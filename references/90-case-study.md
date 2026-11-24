# 90 · 匿名案例数据（0.18 µm / 1P6M SoC）

> **什么时候读我**：需要一个具体数字做对照、想知道「这个方法的量级对不对」，或写报告要举例时。
> **这是全 skill 唯一出现实测数值的文件**；正文（00~85）只写方法，需要举例时指回这里。
> 纪律：每个数字后面都跟来源类型（脚本打印 / 报告字段 / 交付表）；同一量在不同文件冲突时，**以「判决」节为准**。

**数值声明**：本文数字为**量级示例**，用于说明方法能收敛到什么程度；已去掉项目名、公司名、主机、路径、
哈希与日期，并对几何/计数/时序做区间化处理，**不指向任何具体项目或客户**。
工艺相关的层号、规则名、尺寸一律用占位符，**不要当成你所用 PDK 的常量**。

规模：四万余 P&R 实例、百级 pad、5 个 SRAM 宏、6 层金属（1P6M），单时钟单电源域。

---

## 1. 环境与工具版本（仅供对照，不构成要求）

| 项 | 值 |
|---|---|
| PDK | 0.18 µm CMOS，std cell `<STDCELL_LIB>`（core 1.8 V）+ IO `<IO_LIB>`（3.3 V） |
| 工程根 | `<PROJ>` |
| 环境脚本 | `<ENV_IC>`（Innovus / Calibre / calibredrv / v2lvs）、`<ENV_SY>`（DC / PT / StarRC / Formality） |
| 工具版本 | 老版 DC 与 Formality（2015~2016 代）、Innovus 19.x、Calibre 2019.x（**补丁号不给**，它只是站点指纹） |
| 运行方式 | 远程 Linux 服务器跑重活，本地只做 `<ls/grep/cat>` 与文档 |
| 金属栈 | 早期 1P5M，**出片 1P6M** |

> 工具版本是这一批数字的**上下文**：换版本后规则数、优化结果都可能变，别把本文件的数字直接搬到新跑里。

---

## 2. 几何与 pad

| 量 | 值 | 来源类型 |
|---|---|---|
| die | **约 1958 × 1958 µm**（≈1.96 mm，3.83 mm²） | die 框实测 + 交付表 |
| BORDER（`<BORDER_LAYER>`） | 2000 × 2000 µm（die 每侧多出约 21 µm，取整到 2000） | `config.tcl` |
| core | 左下 (200,200)；边长 ≈ die − 2×200 ≈ **1558 µm** | `config.tcl` |
| pad 环外沿距 die 边 | 约 **22**（floorplan 标称） | `config.tcl` |
| Calibre 抽取后器件原点 | 约 22，与标称差 0.002（属库单元内偏移，别当两个设计参数） | LVS 报告 |
| pad 尺寸 / pitch | `<PAD_W>` × `<PAD_H>`，pitch = `<PAD_PITCH>`（从 IO LEF 读）；corner 为方形；pad 库十来个宏 | LEF / pad 库 |
| pad 环总数 | 百级 = 二十余信号 + 十余电源（含 2 颗 3.3 V）+ 4 corner + 百余 IO filler | floorplan 脚本 |
| site / row | `<CORE_SITE>`（名字与尺寸从 std cell LEF 读）；初始约 300 行，`cutRow` 后四百余段 | floorplan 自检 |
| SRAM | 5 个宏 fixed + 20 µm soft halo；沿 core 两条边成对摆放（R90 / R270 两种朝向各一处） | `config.tcl` |
| 标准单元 | 四万余个 / 约 1.1 mm²；alloc 约 1.34 mm² | place 日志 |
| density | place 前约 0.8 → place 后约 77%；overflow 0.00%H / 0.04%V | place 日志 |
| P&R 实例 | 四万余实例 + 二万六千 filler | place 日志 |
| 信号 pad | 二十六 → **二十三**（去掉 3 颗 DFT pad，交付网表对应 `_nodft` 版本） | 网表 diff |

---

## 3. 密封环（交付件实测）

> 这一段是**量过的结论**，不是推断。旧链条把环画在 die 外侧，新链条改到 die 内侧——两者不可混用，
> 引用时必须说明是哪一条链的 GDS。

| 项 | 值 | 证据 |
|---|---|---|
| 环位置（**终稿**） | **die 内侧**，三项（装配隔离 / 环宽 / 合计）都取 DRM 下限，三者相加大致等于「die 边到环内沿」的距离 | 交付 GDS 顶层 `iterator poly` 量环带：x 从 die 边 −(隔离+环宽) 到 die 边 −隔离 |
| 环位置（**已废弃**） | 比终稿向 die 内侧挪了 2 µm ⇒ 装配隔离只剩「下限 − 2」，**低于 DRM**；更早一版把环画在 die 外侧，同样不合规 | DRM SR.7/8/9（**DRC deck 标 `[NC]`，不查**，只能人工核） |
| 环与 pad 的缝 | pad 外沿 ~22 − 环外沿 ~20 = **约 2 µm**（按终稿算；用废弃版会算成约 4 µm，**差 2 µm 正好吃掉装配隔离余量**） | `config.tcl` + GDS 实测 |
| 角块 | 官方密封环角块单元，bbox 约 60 µm 见方（几何只落在隔离带内），四角用 4 种 mirror/angle 组合 | 建环脚本（`<O>`、`<ARM>`） |
| 直边剖面 | 各层在环带内的宽度区间从官方剖面图逐层量；**直边无通孔阵列** | 官方直条单元 via 层为空 |
| DMF dummy 越界（**坑**） | dummy 金属画到 BORDER 内侧 2 µm（LVS `DATABASE EXTENT` 比 die 还外扩约 19 µm），即**环外到 BORDER 之间也有 dummy**；且 EXTENT 是 LVS 层并集，**不能用来判环位置** | LVS 日志 |

---

## 4. 8 步链的自检计数（产出交付件那条链）

| step | 计数 |
|---|---|
| 0 | P&R 导出 GDS 约 84 MB；cells 五百余；顶层 refs 三十九万；via 引用十几万（**非 0 才说明 map 对**） |
| 1 | `TOPREF` 前后一致（三十九万级）；`REFS_RESTORED` = pad 总数 / pad 总数 |
| 2 | `TOTAL_V4` 十万余、`SHRUNK` 一千余；对照：换 pad 前另一版 GDS 里有七万余个厚通孔 |
| 3 | `TEXTS_ADDED` 十余（顶层金属 TEXT 层） |
| 4 | 环带 + BORDER；角块四角 |
| 5 | `BAKED_INSTANCES` 一万一千余；cells 五百余 → 四百余 |
| 6 | DMF 覆盖率目标**按 foundry 规范填**（各层上限不同，本文不给数） |
| 7 | `FILL_REFS_CREATED` 一万五千余 / 一万五千余（旧链少约 1.7 千） |
| 8 | 清掉 `<MARKER_1>.0` 残留 4 条、`<MARKER_1>` 4 条、`<MARKER_2>` 的 pad 方框（个数 = pad 数）、环带 4 条（band ≈180 µm） |
| 终版 refs | `TOPREF` 三十九万级（比 step 0 多出的部分是 fill ref） |

---

## 5. DRC

| 项 | 值 |
|---|---|
| 规则数 / 结果 | **421 条规则**（该版 deck 的执行数，换 deck 版本必重算），`TOTAL DRC Results Generated = 0 (0)` |
| 耗时 | 约 8 分钟（CPU 与 REAL 差数秒，属正常） |
| 零条是怎么来的 | **全部由项目 waiver deck 豁免**，四类合计数**恰好等于规则数**（巧合，别读成「421 条规则全有问题」） |
| 　四类之一 | via-stack 族 3 条规则各约 136；十几条在 8 个 pad 单元内、一百二十余条在相邻 pad 顶层缝隙 |
| 　四类之二 | 残余 LU，个位数，全在信号 pad 内（5 个 pad 型号），同一 ESD/guard-ring 子单元，pad 局部 bbox 完全相同 |
| 　四类之三 | 残余 LU 一条，在某个 SRAM 宏内部 |
| 　四类之四 | 环带内细缝：N-well 细缝 3（十几 nm × 数百 nm）+ 两条金属间距族规则各 2 |
| 核心区 | **0 条**（这是接受豁免的底线） |
| deck 与官方差异 | 37 行：头部（metal option、`LAYOUT PATH`、输出格式、`DRC CELL NAME`/`CELL SPACE XFORM`）+ 判定式 5 处，覆盖 7 个规则名 |
| 自动 waiver | 官方 `DRC Automated Waiver Flow` 缺 license feature，只能走自建 waiver deck + marker |

---

## 6. 天线

- **规则数取 `TOTAL DRC RuleChecks Executed`**（案例 15 条，随 deck 版本变）：覆盖 GT / CT / 各层金属 / 各层通孔 / MIM / 顶层厚金属。
  ⚠️ 案例里工程内三份文档都少写 2 条（顶层通孔与顶层金属各一）——**以 summary 为准**。
- 交付件：`0 (0)`，全部规则全 0。
- 5M / 6M 对跑的做法：**同一份 GDS** 用 `metal_counts 5` 与 `6` 两份 deck 各跑一次，比「条数 + 规则名」是否逐条一致（案例里那一版两边都是 6 条、逐条相同）⇒ 结论是**违规与 metal option 无关**。
  对交付件（0 条）再跑一次 0 对 0 只能说明「换 deck 不引入新违规」，**不能**当等价证明。

### 天线逐版迭代（二十余版，全部实测）

每轮：`restoreDesign` → `streamOut`（`<GDSMAP>`）→ Calibre 天线 → summary。各轮 deck 正文逐字节相同（只差路径），所以列内可比。

| # | 候选版 | summary | 违规（M2/M3/M4） | 本轮手段 |
|---|---|---|---|---|
| 0 | V0 | `r01` | **0** | 基线，但 GDS 用坏 map 导出、**没有布线** ⇒ 0 条是假的 |
| 1 | V1 | `r02` | 93（3/11/79） | 违例框内候选网就近 `attachDiode` |
| 2 | V2 | `r03` | 92 | 脚本已改名/合并，手段（源文件未体现） |
| 3 | V3 | `r04` | 92 | 脚本已不在 `flow/` |
| 4 | V4 | `r05` | 92 | 违例处插 `<BUF_CELL>` 拆长网 |
| 5 | V5 | `r06` | 93 | 正确姿势：删 filler → `attachDiode` → `ecoRoute` → 补 filler |
| 6 | V6 | `r07` | 100 | 违例网改 `top_preferred_routing_layer` + `editDelete` 后重布 |
| 7 | V7 | `r08` | 110 | 负载单元 `ecoChangeCell` 升 1 档 |
| 8 | V8 | `r09` | **11** | 按 `(inst,pin)` 精确映射插 376 颗（`ant_culprit_pins.txt`） |
| 9 | V9 | `r10` | 56 | 对精确定位到的网插 buffer |
| 10 | V10 | `r11` | 9 | 第二轮残余：换引脚再插二极管 |
| 11 | V11 | `r12` | 9 | 沿元凶网找空闲负载脚补二极管 |
| 12 | V12 | `r13` | 3 | 时钟网末段强制走高一层金属 |
| 13 | V13 | `r14` | 8 | 同上扩到更多网 |
| 14 | V14 | `r15` | 3 | 违例引脚就近插 buffer（`-relativeDistToSink 0.2`） |
| 15 | V15 | `r16` | **0** | 违例脚叠 6 颗二极管——但**没删 filler**，与 filler 重叠埋下 DRC |
| 16 | V16 | `r17` | **77** | 把二极管删光按网重新合法安置（反而崩） |
| 17 | V17 | `r18` | 2 | 一次性做干净：376 精确 + 残余叠 5 颗 + 7 条时钟网升层 |
| 18 | V18 | `r19` | 2 | 残余再叠 8 颗 |
| 19 | V19 | `r20` | **0** ✅ | 最后 2 条整网升层后重布 → **真正收敛** |
| 20 | V20 | `r21` | **6**（0/4/2） | 从 V11 起、**刻意不动上层金属**只叠 8 颗二极管 |

**交付用的是 V19（DB28）**，不是最后编号的 V20（6 条）——「最新版反而更脏」的活样本。
V20 的 6 条明细：M3 族 4 / M4 族 2，CPU 42 s / REAL 46 s。

---

## 7. LVS

| 阶段 | 报告 | Nets 匹配 | unmatched（layout / source） |
|---|---|---|---|
| deck 里 `CASE_SENSITIVITY FALSE` | `lvs_case_false.rep` | 约 4.28 万 / 4.28 万 | 一千九百余 / 二百余 |
| 改 `TRUE` 后 | `lvs_case_true.rep` | 约 4.47 万（两侧差 3） | 0 / 0（余 3 网） |
| SPI 定点修 2 个 bit | `lvs_bit_fix.rep` | 约 4.47 万 | 3 / 0（只剩 FP） |
| FP 网表侧分岛（最终） | `lvs_final.rep` | **两侧完全一致** | **0 / 0** |

- 最终：`CORRECT`，`Nets:` 两侧同数、后面两个 0，**零豁免**。
- 大小写：设计里有一千六百余对只差大小写的同名网；改 deck 的 `CASE_SENSITIVITY TRUE` + 启用 `LAYOUT CASE YES / SOURCE CASE YES`。
- VIA 单元：近 80 种 / 一万一千余个实例；烘焙后 GDS 里 `grep -a -c "${TOP}_VIA"` = 0（**加引号**，否则 `<` 变重定向，0 命中会被误读成通过）。
- pad filler stub：9 个空 `SUBCKT`，引脚顺序 `VSSD FP VDD33 VSS VDD`；某个 filler 型号例外保留 5 脚，3 个实例单独挂 FP 岛网。
- FP 岛网：按版图分成 **4 个岛**，脚数 9 / 24 / 2 / 1，合计 36（另一份记录写 9/25/2/1，差 1 脚，以最终 `.spi` 的 `$PINS FP=` 计数为准）。
- 转义标识符案例：`\<NET>[6]` / `<NET>[6]`；反例改 Verilog 后立刻出现四十几条 `no similar instance`。
- 修 2 个 bit 后到终稿的 **+3 网来自 FP 一分四**，不是新增逻辑。

---

## 8. RC + PT

| 项 | 值 |
|---|---|
| PDK tluplus 版本 | 老 `grd_ver`（两位数，2007 格式）⇒ 新版 StarRC 报 `EX-443` |
| ITF 产物 | 三角 ITF（CMAX/CMIN/TYP）各约 4 KB，由 tluplus 内嵌文本剥出 |
| nxtgrd 产物 | 二十几 MB 一份；每角约 20–40 min |
| 1P6M 物理参数 | 被让出的那层金属：厚度约 **0.56×**、方阻约 **3×**（读数在 ITF 的 `CONDUCTOR metalN` 段）；新顶层金属沿用原厚顶层档 |
| StarRC cmd | 每角一份，`LEF_FILE` 6 个（tf + stdcell macro + IO + 3 SRAM）；`TOP_DEF_FILE` 是**上一代 1P5M DEF** |
| 映射文件 | 只有 `METAL1..5` + `VIA12..45`，**无 `METAL6`/`VIA56`** ⇒ **M6 未抽取** |

**PT 五角（1P6M SPEF + propagated clock）：**

| 周期 | ss | 结论 |
|---|---|---|
| 38.0 ns（26.32 MHz） | setup WNS **约 −2.31 ns**，20 条违例，TNS **约 −271 ns** | 不达标 |
| 40.4 ns（24.75 MHz） | 全五角 0 违例 | **接受、出片** |

24.75 MHz 五角（每角一份 `summary.txt`）：

| 角 | setup WNS | hold WNS |
|---|---|---|
| ss | 0.000（`SETUP_REG_WNS` +0.09） | +0.48 |
| ff | +8.98 | +0.07 |
| ff0 | +8.09 | +0.09 |
| tt | +1.74 | +0.20 |
| tt85 | +0.11 | +0.23 |

- 最差路径：流水线某级寄存器 → cache tag 的 SRAM 宏。
- PT 自动 ECO 只给出 3 个 size_cell 改动（一个反相器、一个 buffer、一个 NAND4），远不够补 2.31 ns。
- ⚠️ 另一版 PT 报告里 ss 角是一百八十余条 / TNS 约 −273，与本表口径不同；引用时要写清目录与版本。

---

## 9. 前端（综合 / LEC / DFT / 频率）

| 项 | 值 |
|---|---|
| RTL 文件数 | 九十几（`rtl/filelist.f`）、incdirs 18、defines 19 |
| 综合实测（ss） | Error 0、critical path 约 27 ns、setup/hold 违例 0、cell 面积约 2.3 mm²（含 26 pad）、core ≈2.0 mm²、三百余 ICG / 六千六百寄存器被门控（87.5%）、残余十几条 `min_capacitance` |
| SDC 核心约束 | uncertainty 0.20/0.10；transition 0.15/0.10；input/output delay 2.0/0.5；`set_load 10.0`；单时钟 `clk` |
| 频率演进（综合） | 100 MHz −18.26 ns / 5,579 条；50 MHz −8.17 ns / 2,329 条；**35 MHz（28.5 ns）0.00 ns / 0 条** |
| SDC 派生链 | 28.5 → 33.333 → 35.714 → 38.0（`_pr` + 签核 26m32）→ **40.4（24m75，出片）**；某个 25 ns 版只改 period 未改 waveform（占空比 57%） |
| LEC-B | **通过**：七千九百余个比较点、0 failing、约 1 分钟 |
| LEC-A | **未关闭**：最好一轮一千一百余通过 / 20 失败 / **近七千未判定**（10 种配置试完）；根因是老版 Formality 不接受新版 DC 的 `reg_merging` / `guide_replace` 引导 + 三百余个无复位初值寄存器被判 `Constrained 0X` |
| DFT | **未做**。DC 侧插过 1 条链（七千五百余级），补 4 轮 incremental 后 0 违例；但链头绕过输入缓冲器 ⇒ 交付网表 `SD*` = **0** |
| `set_inputs_no_clk` 事故 | 少个空格 ⇒ 输入延迟一直没生效；修掉后 core 面积 +约 1% |

---

## 10. 交付件：**只记方法，不记指纹**

本文件**刻意不留** GDS 的 md5 / cksum / 字节数——它们只能证明「是哪个文件」，对读者没有复现价值，
却能反过来把某个具体交付件对上号。要记的是**怎么算、记在哪、和谁对账**：

| 要记的 | 怎么取 |
|---|---|
| 文件名 / 顶层 cell | 交付表写顶层名，不写文件名 |
| 大小 | `ls -l` 字节数；打包前后一致 |
| `cksum` / `md5` | `cksum`、`md5sum`；本地与服务器各算一次 |
| 签核同源 | 交付包内 GDS 与跑 DRC/LVS 的那份**逐字节相同**（`cmp`） |
| 包内 summary | `Layout Primary Cell` = 顶层名；DRC 与天线各自的执行规则数 = 你这次 summary 里的值 |
| 版本对照 | 1P6M 交付件比上一代 1P5M 大约 1 千万字节（≈11 MB）量级，可做粗筛 |

> 记住坑：**校验值只证明文件身份，不证明检查通过**；`ls -l` 只能粗筛，定案必须哈希。

---

## 11. 判决：仓库里互相冲突的口径，以谁为准

| 冲突 | 判决 | 依据 |
|---|---|---|
| 密封环在 die 内侧还是外侧 | **内侧**（交付链） | 交付 GDS 顶层 `iterator poly` 实测；LVS `DATABASE EXTENT` 最外是 **DMF dummy**，不是环 |
| 天线规则数 13 还是 15 | **15** | 两份 summary 都是 15；13 是三份文档的连锁传抄 |
| 交付 GDS 是哪一版 | **交付包里那份**（内链条），文档里另一版是旧链 | 包内 GDS 实测；旧文档自相矛盾 |
| 合并填充 ref 数 15,759 还是 14,073 | **15,759**（交付链）；14,073 属旧链 | 两步日志逐条对 |
| 顶层引用数 391,256 / 395,980 | 391,256 = pad swap 前后；**395,980** = 合并填充与 marker 之后的终版 | 各步日志 |
| pad 偏移 22.2 与 22.198 | **两个量**：22.2 = floorplan 标称；22.198 = Calibre 抽取后器件原点 | 差 0.002 µm |
| 密封环是哪个建环脚本出的 | **未闭合**：旧链与新链用了不同版本的 `mk_frame*.tcl`，日志里 `O` 值不同 | 引用时要标明链 |
| LEC 是否 SUCCEEDED | **LEC-B 通过，LEC-A 未关闭**；工程 README 里更早一轮的 `8106/8106` 无日志留存 | 最新 LEC 报告 |

---

## 12. P&R 参数表（10-pnr 里用的占位符）

| 参数 | 值 | 说明 |
|---|---|---|
| `<PAD_W>` / `<PAD_H>` | 从 LEF 读 | LEF `MACRO` 尺寸（库常量，换库重查） |
| `<PAD_PITCH>` | 从 LEF 读 | 相邻 pad 中心距 |
| `<PAD_OFF>` | 22.2 | pad 环外沿距 die 边 |
| `<GAP>` | 37.8 | pad 内沿 → core |
| `<IO2CORE>` | 60.0 | = pad_off + gap；floorPlan 的四个 margin |
| `<CORE_W>` | 约 1558 | core 左下 (200,200)，边长 ≈ die − 2×200 |
| `<RING_W>` / `<RING_SP>` / `<RING_OFFSET>` | 8.0 / 4.0 / 8.0 | 左/右一层、上/下另一层 |
| ring 中心线（config 旧常量，供对照） | VDD 156.0 / VSS 168.0 | 与最终 `addRing` 实参可能不是同一套，**以脚本实参为准** |
| `<STRIPE_W>` / `<STRIPE_SP>` / `<STRIPE_PITCH>` / `<STRIPE_OFF>` | 3.0 / 10.0 / 150.0 / 15.0 | 竖向 stripe，限制在 core 内 |
| `<MACRO_STRAP_W>` / `<MACRO_STRAP_N>` | **1.4** / 20 根（5 宏 × 4 列） | 加宽到 3 会爆四百余条 via-stack 规则（DRM 对堆叠 via 有金属间距下限，而 strap 中心距只有 4.6） |
| `<HALO>` | 20 µm | 宏 soft halo |
| CTS | CLKBUF **224**（修前 151）；`target_max_trans 0.8`、`target_skew 0.50`；slew 违例 226 → 1 | 库只有 10 种时钟 buffer，无时钟反相器 |
| hold fix | ff 角 WNS −1.071 → −0.075（1 轮）→ **+0.025（2 轮）**；filler 二万六千余 → <100 → 补回二万六千余 | `06b_hold_fix` |
| route | 布完 1 条底层金属间距违例，`ecoRoute -fix_drc` 1 轮归 0 | `06_route` |
| sroute open | floatingStripe 2、followpin 1（已知无害基线）、blockpin 0、pad VDD 0、pad VSS 0；PG open 0 / dangling 3 | `03_pg_route` |
| GDS 层号 | 1P6M：M6=`<M6_LAYER>`、V5=`<V5_LAYER>`、M6TXT=`<M6TXT_LAYER>`；1P5M：M5=`<M5_LAYER>`、V4=`<V4_LAYER>` | deck LAYER MAP |

---

## 13. 脚本与文档索引

**只列功能，不列内部路径**（脚本名可换，功能不能省）：

- 版图链 8 个脚本：`reexport_db*.tcl`、`p6_swap.tcl`、`v4_shrink.tcl`、`add_pwr_labels.tcl`、`mk_frame*.tcl`、`bake_vias.tcl`、`merge_fill.tcl`、`fix_all_markers.tcl`（详见 20 §六）
- P&R：`config.tcl`（唯一参数源）、`00_init…08_export_lvs.tcl`、`run.sh`、`gds.map.innovus`
- 天线 ECO：`ant_diode_pin.tcl`、`ant_m5.tcl`、`reexport_gds_v*.tcl`（逐版导出）
- deck：DRC / 天线 / LVS / DMF 各一份 + 官方原版对照（**diff 37 行**要能逐行解释）
- 报告：三份 `.summary` + 每步 `.log` + 每角 PT `summary.txt`
