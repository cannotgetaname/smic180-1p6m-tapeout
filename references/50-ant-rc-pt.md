# 50 · 天线签核 + 逐版搜索 / RC 抽取 + PT 五角

> **什么时候读我**：要跑天线签核、判断「为什么最新改完那版天线反而更脏」、要从 tluplus 建 RC、或跑 PrimeTime 五角并决定降不降频时。
> 逐版实测表、五角数值、物理参数对照见 `90-case-study.md`。

---

# 第一部分：天线（antenna）

## 1.1 deck 与开关

- 用 PDK 的天线 deck（`*.ant`），签核副本只改**开关 + 3 行路径**，原 deck 不加料（判据：文件大小与官方一致，diff 只有那几行）。
- 开关与 DRC 同源：`set metal_counts <METAL_COUNTS>`、`set top_metal <TOP_METAL>`。**1P6M 必须 6**；5/4/3 分别对应 1P5M/1P4M/1P3M。
- deck 头注释通常给出层位（1P6M：顶层金属 M6 = `<M6_LAYER>`、顶层通孔 V5 = `<V5_LAYER>`；1P5M 换成 `<M5_LAYER>` / `<V4_LAYER>`；**一律以本 PDK 的 LAYER MAP 为准**）。**换 metal option 时这两层 GDS 号会变**——这就是「同一份 GDS 在 5M/6M deck 下规则集合不同」的机制。
- 每次必改 3 行：`LAYOUT PRIMARY "<TOP>"` / `LAYOUT PATH "…/当期.gds"` / `DRC RESULTS DATABASE "…"`。

```csh
csh -c "source <ENV_IC> >& /dev/null; cd <PROJ>; \
  calibre -drc -hier -64 -turbo 6 <WORKDIR>/rules/ant_h5.ant >& <WORKDIR>/reports/ant_h5.log"
```

## 1.2 怎么判读

- 只看 `.summary`：每个 `RULECHECK` 的 `TOTAL Result Count` + `TOTAL DRC Results Generated`。
- **规则数取 summary 的 `TOTAL DRC RuleChecks Executed`，不要数 ASCII results 行、也不要数文档里列的规则名**：ASCII 每规则默认截 1000 条；文档漏列规则名是常见事故（案例里上游三份文档都漏列了 2 条——顶层通孔与顶层金属各一，以 summary 为准）。
- 天线和 DRC 共用 `metal_counts`/`top_metal`，但**是两份 deck、两次运行**，不要互相引用结果。

## 1.3 「违规与 metal option 无关」的判法：5M/6M 对跑

同一份 GDS，只换 deck 的 `metal_counts`（5 vs 6）各跑一次：

| 观察 | 结论 |
|---|---|
| 违例集合（规则名 + 条数）**逐条一致** | 违例来自 M1–Mtop 本层，**与顶层金属选项无关**，改 metal option 消不掉 |
| 只在 6M 下出现（如 `ANT_MT_3`、`ANT_VT_1_2`） | 才是顶层金属配置问题 |

- 但 `metal_counts`/`top_metal` 仍必须与买到的层数一致（顶层相关的 GT/VT 规则会重指 M6/V5）⇒ **1P6M 不能沿用 1P5M 的「0 条」结论，必须换 deck 重跑。**

## 1.4 天线 ECO 的搜索方法（本项目最痛的一段）

**每轮固定套路**（不这样做无法归因）：

```
restoreDesign <DB> → 从 DB streamOut 一份 GDS（必须 <GDSMAP>）→ Calibre 天线 → 记 summary
```

- 每轮只允许改 1~2 个手段。否则「变好/变差」无法归因。
- deck 正文逐字节相同（只差 `LAYOUT PATH`/输出路径）⇒ 各轮 summary **可比**；这个前提要自己抽查一次再开始比较。

**七条教训（都能独立复用）：**

1. **不要按时间/编号猜哪版最好，必须逐版实测。** 最后一版（时间最新、改动「更保守」）反而有违规，更早一版是 0 条 —— 最终交付要回退到那版干净的。
2. **每个「0 条」都要验版图有效性。** 本项目的两个反例：① 0 条来自一份**没有布线**、体积明显偏小的 GDS；② 0 条来自「二极管压在 filler 上」的**非法版图**，一重建就回到几十条。判 0 之前先看 GDS 大小 / `verifyConnectivity` / `verify_drc`。
3. **插/叠二极管前必须先删 filler，插完再补。** 不删 filler ⇒ `attachDiode` 只能压在 filler 上 ⇒ AA/NW/CT/M1 上百条 DRC。
4. **`verifyProcessAntenna` 只能粗筛**：LEF 里往往没有 `ANTENNAGATEAREA`/`ANTENNADIFFAREA`（工具报 `IMPLF-200/201` 就是这两条），**真判据只有 Calibre 天线 deck**。
5. **改布线层是立竿见影的手段**：把违例网（尤其时钟网末段）改走顶层金属，天线比值成数量级下降。代价是动顶层金属会牵动时钟时序 —— 所以「不用顶层金属、只合法叠二极管」的版本反而残留几条。**这是取舍，不是 bug，报告里要写清。**
6. **每轮只动 1~2 个手段**：本项目有一串「换手段反而变差」的真实记录（几十条 → 一百多条），正是它们把方向逼到「精确定位到 `(inst, pin)` 再插」——那一步一次性把违规从百级降到十级。
7. **有效手段的排序（经验）**：精确定位 `(inst,pin)` 后插二极管 ＞ 违例网改顶层金属 ＞ 就近插 buffer 拆长网 ＞ 换驱动档位 ＞ 盲目框内插二极管。「盲目」那类只会引入 DRC。
8. **`restoreDesign` 必须指向 `.enc.dat` 目录**：`.enc` 只是几百字节的 stub，拿它 restore 直接失败。

> 二十余版的逐版实测表（候选版 / summary / 违规数 / 手段）在 90 §6「天线逐版迭代」。

## 1.5 天线收尾的判定

- 交付版：`summary` 的 `TOTAL DRC Results Generated = 0`，规则数 = `<N_RULES_ANT>`。
- 同时必须满足：`verify_drc` 0、`verifyConnectivity` 无开路、GDS 有布线（via 引用数正常）。
- 干净版确定后，**后续所有签核都用这一版 GDS**，不要再顺手改版图。

---

# 第二部分：RC 抽取（tluplus → nxtgrd → StarRC）

## 2.1 PDK 只给 tluplus 时的路：EX-443 → 抠出 ITF → grdgenxo

- PDK 的 TLUPlus 常是**老 `grd_ver`**（老 PDK 常见两位数 `grd_ver`，格式是十几年前的），新版 StarRC 直接读报 **`EX-443`**。
- 解法：从 `.tluplus` 里**抠出内嵌 ITF 文本**，再让 `grdgenxo` 编成 nxtgrd。两种等价抠法：
  - 小脚本逐行剥掉注释前缀（`scripts/tluplus2itf.py`，纯文本处理，不碰版图）；
  - `awk` 剥掉每行 `#     ` 前缀（一行命令）。
- 产物按角命名（CMAX/CMIN/TYP 各一份 ITF + 一份 nxtgrd）。每角 **20–40 分钟**（这个量级可作为进度预期）。
- 🔴 **`grdgenxo` 的输出落在当前目录，不在 ITF 旁边**。要么 `cd` 到目标目录再跑，要么跑完手动归位；归位后还要改 StarRC 命令文件里的 `TCAD_GRD_FILE`。

```csh
cd <PROJ>/rc/starrc
csh -c "source <ENV_SY> >& /dev/null; \
  grdgenxo <WORKDIR>/nxtgrd/<...>_CMAX.itf >& grdgenxo_CMAX.log"
# 编完把 .nxtgrd 挪回 rc/starrc/，再改命令文件
```

## 2.2 StarRC 映射文件：格式 + metal option 陷阱

- 映射文件必须带 **section 头**（`conducting_layers` / `via_layers` / `marker_layers` / `remove_layers`）。写成裸的 `METAL1 metal1` 会报 `SX-1907 Section has not been described`。
- **metal option 陷阱**：1P6M 里顶层通孔要映射到 tluplus 里的**普通通孔名**；1P5M 才是**厚顶层通孔名**。两个名字只差一个字母，抄错就报 **`SX-1936`**，照抄上一代 map 是最常见的起手错误。右侧名字取自 tluplus 头部（`CONDUCTOR metal1..metalN / VIA via1..`）。
- StarRC 命令文件（每个角一份，只差 nxtgrd 与输出 SPEF）关键行：`MAPPING_FILE`、`LEF_FILE`（**示例 6 个**：tf + stdcell macro + IO + 3 个 SRAM —— 数一遍再写文档，别凭印象写 5 个）、`TOP_DEF_FILE`、`TCAD_GRD_FILE`、`NETLIST_FILE`。

```csh
csh -c "source <ENV_SY> >& /dev/null; cd <PROJ>; \
  StarXtract -clean rc/starrc/<TOP>_starrc_CMAX.cmd >& rc/starrc/starrc_CMAX.log"
```

## 2.3 🔴 RC 的抽取边界（必须写进报告，否则结论过强）

签核前先查两件事：

1. 映射文件里**有没有顶层金属/顶层通孔**（如 `METAL6`、`VIA56`）；
2. `TOP_DEF_FILE` 是哪一代的 DEF（是不是上一次 P&R 的 1P5M DEF）、`LEF_FILE` 用的是 5MT 还是 6MT pad。

示例项目两条都没满足：映射只有 `METAL1..5` + `VIA12..45`，DEF 是上一代 1P5M 的 ⇒ 那份 SPEF 实际是
**「1P5M 结构 + 1P6M 介电/方阻的 M1–M5 抽取」**，**M6 没抽**。

⇒ 写结论时必须限定：**PT 结果只对 M1–M5 的 RC 变化有效**；顶层金属（M6）上的长线与电源不在模型里。要拿到完整 1P6M 寄生，必须补 `METAL6`/`VIA56` 映射 + 用 1P6M 的 DEF 重抽。**不要把 `StarRC 跑通了` 当成 `拿到了完整 1P6M 寄生`。**

---

# 第三部分：PrimeTime 五角

## 3.1 模板 `pt_corner.tcl`

- 角用环境变量 `CORNER` 选，合法值即 `<CORNERS>`；每个角对应「std cell `.lib` + IO `.lib` + 每个 SRAM `.lib` + 一份 SPEF（CMAX/CMIN/TYP）」。

| 角标签 | std cell corner | IO 库 | SPEF |
|---|---|---|---|
| ss | 慢压慢温 | 慢角 | CMAX |
| ff / ff0 | 快压低温 / 快压 0℃ | 快角 | CMIN |
| tt / tt85 | 标称 25℃ / 85℃ | typ | TYP |

- ⚠️ **IO 库的 `_max` / `_min` 可能是反的**：本项目 `*_max.lib` 是 1.62V/125℃ **慢角**，`*_min.lib` 是 1.98V/0℃ **快角**。**每换一个 IO 库都要重新确认 `_max/_min` 的语义**，不要按 std cell 的习惯套。
- 路径用环境变量覆盖：`SPEF`、`VERILOG`、`SDCFILE`、`RPT_DIR`。
- **`read_sdc` 会截断**：SDC 里有 `set_dont_touch` / `get_cells -filter` 等非 SDC 语句时，`read_sdc` 报 CMD-005/CMD-010 并中止解析（后半个文件的 IO delay / false_path / set_load 全丢）。**必须 `source`。**
- **CTS 后必须 `set_propagated_clock [all_clocks]`**（并 `remove_ideal_network`），否则时钟理想零 skew。模板里留 `PROP=0` 退回理想时钟做对照；`DERATE=0.95` 之类开关用于开 OCV 降额。

```csh
csh -c "source <ENV_SY> >& /dev/null; cd <PROJ>; \
  setenv CORNER ss; setenv SDCFILE <PROJ>/deliv/<TOP>_<FREQ_TAG>.sdc; \
  setenv RPT_DIR <WORKDIR>/pt_<FREQ_TAG>/ss; \
  pt_shell -f pt/scripts/pt_corner.tcl -output_log_file <WORKDIR>/pt_<FREQ_TAG>/ss.log >& /tmp/pt_ss.log"
# 批量：pt/scripts/run_pt.sh ss ff ff0 tt tt85
```

## 3.2 读 summary：别被顶层 0.0000 骗了

`summary.txt` 里同时有：`SETUP_WNS` / `HOLD_WNS` / 违例路径数 / `SETUP_WNS / TNS` / 按触发器拆的 `*_REG_WNS`。

- **ICG enable 之类的检查会恒 0.0000，把真 slack 掩盖掉**。必须拆开看**寄存器路径**的 WNS（示例 ss 角顶层 `SETUP_WNS 0.0000`，而 `SETUP_REG_WNS` 是另一个非零数）。
- **报告目录要写到角一级**：`pt/reports/ss/summary.txt` 可能是**另一次运行**的副本，与文档口径不同。引用时写清 `<角> + 目录 + 日期`，只写角名定位不到结果。

## 3.3 降频决策（一条真实决策链，可直接复用）

1. **先算物理根因**：换到 1P6M 后，厚金属从 M5 让给 M6，**M5 变薄** ⇒ 方阻约 ×3（倍数与读数见 90 §8）。原来全在 M5 上的长线与顶层电源电阻直接 ×3。这不是工具/约束问题。
2. **看 ECO 能不能收**：`fix_eco_timing` 出来的改动往往只有几个 size_cell，远不够补 ns 级差额 ⇒ 放弃。
3. **降频是唯一零改动出路**：把周期放宽到能覆盖最差角，再在**新周期**下重跑 `<CORNERS>` 全角。
4. **报告要写**：新频率、新周期、用的哪版 SDC、五角各自的 WNS、以及「若要回到原频率，需要 P&R 级 ECO 把长线/电源搬到顶层金属」这条后续项。

> 案例的频率演进、五角数值、`fix_eco_timing` 的改动清单都在 90 §9。

---

# 换项目改哪几行

| 要改的东西 | 在哪 |
|---|---|
| 天线 deck 的 metal option / 3 行路径 | `<WORKDIR>/rules/*.ant` |
| 天线 ECO 的 DB 版本与总结名 | Innovus `restoreDesign` 参数 + summary 输出路径 |
| nxtgrd 的角名与目录 | `grdgenxo` 的 cwd 与 StarRC 命令文件 |
| 映射文件的 metal/via 行 | `*.map`（**必须与 tluplus 头部逐行对照**） |
| StarRC 的 `LEF_FILE` 数量/顺序、`TOP_DEF_FILE` | `*.cmd`（数一遍，别写错） |
| 角集合与 `.lib` 组合 | `pt_corner.tcl` 的 `CORNER` 分支 |
| IO 库 `_max/_min` 语义 | 查 `.lib` 头部的电压/温度 |
| SDC / 网表 / SPEF 路径 | 环境变量（`SDCFILE` / `VERILOG` / `SPEF`） |
