# 40 · LVS 调试手册

> **什么时候读我**：LVS 报 layout-only / source-only 上百上千、只剩个位数网、报 `Power shorted to ground`、或要在源网表侧补 stub / 分岛 / 定点改连接时。

> 案例里的网名、单元名都做了占位（`<NET>` / `<INV_CELL>` 这类），读的时候关注机理，不要照抄名字；机理见 90 §7。
> 数据对照表（各阶段的 Nets 数）在 `90-case-study.md`。

---

## 0. 报告怎么读（先看三处，30 秒定性）

| 位置（报告内） | 看什么 | 判据 |
|---|---|---|
| `OVERALL COMPARISON RESULTS` | `CORRECT` 还是 `INCORRECT`，下面的 `Error:` 行 | 第一眼只看这里 |
| `INFORMATION AND WARNINGS` → `Nets:` 行 | `Matched Layout / Matched Source / Unmatched Layout / Unmatched Source` | **判总数只能看这一行** |
| `INCORRECT OBJECTS` → `INCORRECT NETS` | `DISC#` 列表 + `** no similar instance **` | ASCII 报告**只列前 50 条**，不能拿样本外推 |

**两条容易误判的读数规则：**

- `Warning: Ambiguity points were found and resolved arbitrarily.` **在最终 CORRECT 报告里依然存在**。所以 **它不是判据**，判据永远是 `Nets:` 行的 0/0。反过来，改了网表之后同样的 warning 会真的把网配错（见 §6）。
- **数码网名**（`Net 39/40/41/42`…）意味着两侧都没对上名字。pad 的电源脚名（如 `VSSD`/`VDD33`）常常不在 deck 的 `LVS POWER NAME`/`LVS GROUND NAME` 通配里，所以它们在报告里就是普通网 —— 这是**判读规则，不是改动建议**。

---

## 1. 上千个假网：先把 deck 的大小写开关翻过来

- **现象**：同一份 GDS、同一份 SPICE，LVS 报上千 layout-only / 几百 source-only，`DISC` 条目封顶；多出来的几乎全是「2 脚小网」，而**器件实例 100% 匹配**、Innovus `verifyConnectivity -type all` 也是空的。
- **根因**：PDK LVS deck 里 `#DEFINE CASE_SENSITIVITY FALSE` ⇒ 后面的 `#IFDEF` 段不生效 ⇒ `LAYOUT CASE`/`SOURCE CASE` 全为 NO ⇒ Calibre **大小写不敏感**比对。而设计里存在大量**只差大小写**的同名网（同一个 DB 里是两个网，脚数不同）。关掉大小写敏感后它们在源侧各自并成一个网 ⇒ 版图侧多出的那段就成了 layout-only。
- **对策**：把该行改 `#DEFINE CASE_SENSITIVITY TRUE`，让 deck 的 `LAYOUT CASE YES / SOURCE CASE YES` 生效。**一行改动，不动版图。**
- **验证**：改前/改后的 `Nets:` 四元对照，DISC 从几千条掉到个位数（对照表见 90）。
- **先排除的错误方向**（都实跑过、全都没变化）：解 BOX 单个/全部单元、扁平 GDS + 扁平源、`LVS IGNORE PORTS NO`、`LVS REDUCE SPLIT GATES NO`。器件数、via 数、DRC 通孔规则都干净 ⇒ **不要再往「版图开路 / 缺孔」查**。

---

## 2. `streamOut` 生成的 via 实例：只能烘焙，不能 FILTER

- **现象**：Innovus `gdsout` 会生成几十种 `<TOP>_VIA*` 物理单元（实例上万个）承载电源/信号通孔阵列，**网表里根本不存在这些 cell** ⇒ 大量 `missing instance` + 相关网断裂。
- **根因**：这些单元只有几何、没有器件；而且它们的端口是 VDD/VSS 一起进出。用 `LVS FILTER ... OPEN` 当「忽略器件」处理，等于把同一实例的电源脚和地脚一起摘掉 / 让两侧电源网在 source 侧合并 ⇒ Calibre 直接报
  `Error: Power shorted to ground while merging nets or filtering instances in source.`
- **对策**：**烘焙进父单元**（按变换把顶点写进父单元，再删 ref 和 cell，见 20 step 5）。
- 两个必须避开的雷：① 直接 `delete cell` 会连 SREF 一起删、把版图挖洞（网数会翻几倍）；② `LVS FILTER ... OPEN` 只对「两侧都有 box 子电路」的单元有效，纯几何单元无解。
- **验证**：`grep -a -c '<TOP>_VIA' <final.gds>` 应该为 **0**，且 `no similar instance` 里不再出现 VIA 单元。

---

## 3. pad filler：标准单元 filler 可以直接 OPEN，pad 的 filler 必须给 stub

- **现象**：标准单元 filler（`F_FILL*`）用 `LVS FILTER ... OPEN` 干净通过；但 pad 环上的 filler（`<PAD_FILLER>*` + `<PAD_CORNER>*`）同样处理就报 `Power shorted to ground` 或缺实例。
- **根因**：`F_FILL*` **没有电源脚**（两端悬空，OPEN 只是「忽略」）；而 pad filler 的版图单元有 5 个脚（`VSSD / FP / VDD33 / VSS / VDD`）并真的接在 pad 环母线上，源端没有对应子电路 ⇒ 5 个脚无处落。
- **对策**：在源网表侧补**空 stub 子电路**，且**引脚顺序必须和版图一致**：

```
.SUBCKT <PAD_FILLER> VSSD FP VDD33 VSS VDD
.ENDS
```

  注意这个顺序**不等于** PDK IO 库的写法（pad 宏本身是 `VDD33 VSSD VSS VDD PAD FP OEN I C` 之类）。pad 宏直接用库自带子电路 + 逐实例 `$PINS` 命名连接；**只有 filler 用自建 stub**。
  deck 侧再用 `LVS BOX <PAD_FILLER>*` / `LVS BOX <PAD_CORNER>*` + 对 corner 用 `LVS FILTER ... SHORT`。
- **特例**：某个 filler 型号的 `FP` 在版图上落在**不同段**，要保留 5 脚并对个别实例单独挂岛网；其余 filler 用**空 body 但保留同样的 5 脚声明**（写成 0 脚会报引脚数不匹配）。**逐实例核对，不要一刀切。**

---

## 4. 案例：转义标识符 `\name[bit]` 与总线位 `name[bit]` 是两个网

- **现象**：只剩 3 个网时，其中 2 个是同一总线的两个 bit；版图侧各是 2 个网、各若干脚，源侧是 1 个网（自环）。
- **根因**（本项目最有价值的发现）：Verilog 里 `\o_j_<NET>[6]` 是**转义标识符**（名字里带方括号，是独立 net），`o_j_<NET>[6]` 是**总线位**。二者在 Verilog / Innovus DB / GDS 里都是两个网：

```
CELL .../<INV_CELL>_id_o_j_<NET>_6
     I → .../id_o_<NET>[6]      Z → .../id_o_j_<NET>[6]
```

  但 **`v2lvs` 会去掉反斜杠** ⇒ 两者在 SPICE 里同名 ⇒ 被并成一条网 ⇒ 把 ECO 插入的 buffer 输入输出短在一起。即「**版图和 DB 都对，只有 v2lvs 生成的 SPICE 错**」。
- **对策**：**不改 Verilog**，直接在**扁平 SPICE** 上做定点修改（每 bit 3 处）：
  1. 删掉 `*.CONNECT <bus_bit> <escaped>`（`v2lvs` 从 `assign` 生成的那行）；
  2. 驱动端归位：把驱动 cell 的 `.ZN` 从 `\o_j_<NET>[n]` 改回 `o_<NET>[n]`；
  3. 负载端归位：把负载（如加法器）的 `a[n]` 从 `\o_j_<NET>[n]` 改回 `o_<NET>[n]`。
- **走过的弯路（别重复）**：只删 `*.CONNECT` ⇒ 更差（原网失去驱动）；把 `.ZN` 挪到端口并删 assign ⇒ 父层把两条网接到同一父网，双驱动 + `Power shorted to ground`。多级反相器链要先逐级确认「哪一级输出接哪条网」再拆。

---

## 5. 案例：FP 浮空虚脚在版图上被 pad 环切成多段

- **现象**：Nets 已基本对齐，只剩 1 类失配：网表里 pad 的 `FP` 是一条几十脚的母线，版图里被 pad 环切成多段（`Net 43 / 810 / 2573 / …`）。
- **根因**：`FP` 是 PDK pad 宏的「floating pad」虚脚，**没有电气功能**；它在版图上断裂是 vendor pad 单元沿环旋转摆放的自然结果 —— 水平 pad 与垂直 pad 的 FP 金属条在四个角不重合，corner 单元不把它们连起来。**不是本设计版图错误，也不是迁移引入的**：完全未改动的原始版图用同一 deck 跑，也是同样几个网。
- **对策**：**按版图分段结构在网表侧分配「岛网」**，让两侧分段完全一致（岛脚数加起来必须等于原 FP 总脚数）。**没动版图、没用 waiver。**
- **反面教训**：先试过「跨角补桥」——FP 确实连成一条，但同时把 FP 与 `VSSD` 短在一起（桥画在了 VSSD 条上）。**没有精确坐标之前不许补桥。**

---

## 6. 反例：把转义标识符「改名」会炸掉 pad 电源网

- **现象**：为「从根上」消除转义标识符，直接改 Verilog（`\o_j_<NET>[6] → o_r_<NET>[6]`，删 `wire` 与 `assign`）。两个 ECO 网结构**确实变对了**，但报告立刻 `INCORRECT`：几十个 `** no similar instance **`，`DISC` 网全是 pad 环的 `VSSD/VDD33/VSS/VDD/FP`。
- **根因**：改动撞上了 Calibre 的**歧义消解**。pad 环上有大量**完全相同**的 filler（同型号几十上百个）无法按名字区分，只能靠拓扑配对；网名/拓扑一变，配对被打乱，pad 电源网整串错位。更深的层：pad 实例在 Verilog 里只连信号脚，电源脚悬空，而版图里它们接在 pad 环母线上 —— 这个「源悬空 / 版图连接」的差在原始网表下刚好被消解掉，**一旦网表变化就暴露**。
- **对策**：**不要动 Verilog 的层次结构，不要改网表里的网名**。要修就按 §4 在扁平 SPICE 上做「删 CONNECT + 驱动/负载归位」的**定点改连接**。真要重命名，正确路径是回到 Innovus 重跑那次 in-place buffer ECO，让工具重新生成主语网表。
- **无效补救（都试过）**：换「干净」deck（原始 box 列表而非重复插入的那份）；给所有 filler + corner 加 `LVS FILTER ... OPEN`；把 filler 烘焙进父单元（单个 filler 两万多个多边形 × 上百实例 = 上百万多边形，直接放弃）。

---

## 7. LVS 从上千网打到 0 的标准动作顺序（10 步）

1. **先自证版图没开路**：Innovus `verifyConnectivity -type all` 必须空 + `verifyPowerVia` 干净。不空就先修版图，**不要碰 LVS 参数**。
2. **换对导出**：`<GDSMAP>` 四列，否则布线/via 被静默丢弃；拿到 GDS 先数 via 引用，0 一定是坏 map。
3. **确认报告口径**：`LVS REPORT` 的汇总段 / svdb，**不信 ASCII 报告的前 50 条**。
4. **查 deck 的 `CASE_SENSITIVITY`** → 改 `TRUE`。通常这一步就消掉 90% 的假网。
5. **清「网表里不存在的几何单元」**：`<TOP>_VIA*` 一律烘焙进父单元（不许 delete cell、不许 FILTER OPEN）。
6. **补 pad 侧 stub**：filler 空 `SUBCKT`（引脚顺序与版图一致）；标准单元 filler 用 `LVS FILTER F_FILL* OPEN`；pad 宏用库子电路 + `$PINS`。
7. **保留 `LVS IGNORE PORTS YES`**。端口名对不上时**不要去打 TEXT 标签**（顶层 TEXT 接不进被 box 的 pad）。
8. **只剩个位数网时逐条看 `INCORRECT NETS`**：区分「网表自环」（§4）/「虚脚母线断裂」（§5）/「电源网配对歧义」（§6），三类修法完全不同。
9. **改网表只改连接、不改名字**：删 `*.CONNECT`、驱动端归位、负载端归位；改完用**同一份 GDS** 复跑，确认只少了目标网、没新增失配。
10. **收尾判据**：`Nets:` 行 0/0 + 所有实例类型 x/x，**不用 waiver**。

---

## 8. 命令模板

```csh
# 0) 环境：bash 里 source 不了 csh，必须包一层
csh -c "source <ENV_IC> >& /dev/null; calibre -lvs -hier <WORKDIR>/rules/lvs_h5.lvs >& lvs_h5.log"

# 1) 只抓结论（别读整个 log）
grep -n "OVERALL COMPARISON RESULTS" -A 12 <WORKDIR>/reports/lvs_h5.rep
grep -nE "^ +Nets:" <WORKDIR>/reports/lvs_h5.rep | tail -1

# 2) v2lvs 出源网表（每个 -s 是一个库的 SPICE/CDL；工作目录为项目根）
csh -c "source <ENV_IC> >& /dev/null; v2lvs -v <WORKDIR>/lvs/<TOP>.v -o <WORKDIR>/lvs/<TOP>.spi \
  -s lib/io/lvs/<IO>.sp -s lib/stdcell/cdl/<STDCELL>.cdl \
  -s lib/sram/<SRAM1>.cdl -s lib/sram/<SRAM2>.cdl -s lib/sram/<SRAM3>.cdl >& v2lvs.log"
```

- **`-hier` 一律带上**：flat 跑时 `LVS BOX`/`FILTER` 的作用范围与 hier 完全不同（案例里 flat 跑出现源侧 0 器件、结果全废）。真要在 flat 下调试，先确认两侧器件数都非 0 再读结论。
- `v2lvs` 的 `-s` 个数 = 实际用到的库数（IO + stdcell + 每个 SRAM），少一个就 `no similar instance`。

---

## 9. 另外两个高频坑（与 1P6M 无关，但每个项目都会遇到）

- **M1–M6 上任何 TEXT 都会变成 pin 名**：库里 `& Metric 100.1` 这类**元数据文本**在 deck 的 `TEXT LAYER`/`PORT LAYER TEXT` 下变成假 pin，产生几十个幻影网络、box 单元上多出以网号命名的 pin（某个 pad 上曾多出十几个）。对策：交付 GDS 里**绝不保留**非必要的 M1–M6 TEXT；本项目验证过可剥掉上千条元数据文本。**剥文本只影响 LVS 命名，不影响几何，但要与交付件分开管理，别把 LVS 专用文件当交付件。**
- **`LVS BOX` 可能被重复插入**：deck 里 `LVS BOX` 的行数会随脚本反复追加而暴涨（本项目曾出现上千条 vs 原始的几百条）。自检：`grep -c 'LVS BOX' <deck>`，与官方 deck 对照，差太多就重建 deck。

---

## 10. 报告写法（每句话都要能追到文件）

- 每个 LVS 结论都要带：**GDS 路径 + md5 + deck 路径 + 报告路径**。混用不同文件的数字 = 作废。
- 判 CORRECT 时同时写 `Nets: N / N`、`0 / 0`、以及有没有用 waiver（**本项目是零豁免**）。
- 分阶段数字要在同一张表里按「阶段 / 报告 / 四元数」列，说明每次修改只影响了哪些网（网数增加/减少的原因要能对上，例如「FP 一分四 ⇒ +3 个网」）。

---

## 11. 换项目改哪几行

| 要改的东西 | 在哪 |
|---|---|
| `CASE_SENSITIVITY` 行 | deck 头部（grep `CASE_SENSITIVITY`） |
| `LVS REPORT` / `LVS POWER NAME` / `GROUND NAME` 通配 | deck 头部 |
| `LVS BOX` / `LVS FILTER` 清单 | deck（单元名换成本项目的 pad/SRAM/filler） |
| via 单元前缀与个数 | 由 `<TOP>` 与 `streamOut` 实际生成的名字决定（用 `seq`） |
| filler stub 引脚顺序 | **必须查版图单元的真实脚序**，不能照抄别的项目 |
| v2lvs 的 `-s` 库清单 | 网表头部 `.INCLUDE` + 实际实例化到的库 |
| 岛网命名/脚数 | 按版图分段实测（不要按网表的想当然） |
