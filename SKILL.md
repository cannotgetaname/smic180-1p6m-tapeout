---
name: smic180-1p6m-tapeout
description: "SMIC 0.18 µm 数字后端到 MPW tapeout 的方法：DC 综合/SDC/LEC → Innovus floorplan/PG/place/CTS/route → 1P5M↔1P6M 迁移（6MT pad、官方密封环、DMF 填充）→ Calibre DRC/LVS/天线清零 → TLUPlus→nxtgrd RC 重抽 → PrimeTime 五角 → 交付打包与 MPW 表格。Use when the task mentions SMIC 180 / 1P6M / 1P5M / tapeout / 流片 / metal option / DRC / LVS / antenna / 天线 / latch-up / LU / seal ring / 密封环 / DMF / dummy fill / StarRC / nxtgrd / SPEF / PrimeTime / LEC / Formality, or asks to close DRC/LVS/antenna to zero before tapeout."
license: CC-BY-4.0
allowed-tools: Bash, Read, Write, Edit, Grep, Glob
metadata:
  version: 1.0.0
---

# SMIC 0.18 µm 数字后端 → MPW tapeout 全流程

一句话：**从 RTL 综合一路做到可交给 foundry 的 MPW 交付包**，其中 1P5M→1P6M 迁移这一段把 DRC / 天线 / LVS 全部打到零。

本 skill 是**方法 + 判据 + 坑**，不是某一块芯片的回忆录：正文只写通用做法，一个已脱敏的完整案例数据集中在
[`references/90-case-study.md`](references/90-case-study.md)。需要举例时正文写「示例见 90」。

> 示例项目成绩（只用来说明体系能收敛到什么程度，**不是**通用验收线）：DRC 0（全部规则）/ 天线 0（全部规则）/
> LVS CORRECT（Nets N/N，零豁免）/ PT 五角 0 违例；出片频率降了约 6% 换签核。
>
> 文中 `<...>` 是占位符，**粘进 shell 前必须替换并加引号**（否则 `<TOP>` 会被当成输入重定向）。
> 工艺相关的**层号、规则名、尺寸、方阻、DRM 上限主要用占位符**；示例里保留的少数数值只用于说明量级，
> **必须按你自己的 DRM / deck / ITF 重查**。

## 0. 开工前先填这张变量表

正文与所有 reference 用这些占位符；换项目改**这两张表** + 每个 ref 末尾的「换项目改哪几行」。

| 变量 | 含义 | 填法（示例，换项目照填） |
|---|---|---|
| `<PROJ>` | 工程根（绝对路径；本地只读挂载另记） | `/path/to/proj` |
| `<WORKDIR>` | 当前步骤 cwd（后端常在 `<PROJ>/pnr`） | `<PROJ>/pnr` |
| `<TOP>` | 顶层 cell / LVS `LAYOUT PRIMARY` 名 | `mychip` |
| `<DIE_UM>` | die 边长（方形；非方形时拆 X/Y） | 千 µm 量级，取 die 框实测 |
| `<CORE_W>` | core 边长；core 左下角 = `<IO2CORE>` + `<PAD_H>`（pad 偏移 + gap + pad 高） | die − 2×core 左下角 |
| `<BORDER_UM>` | 掩模/划片标记框边长（**不是 die**，不能算面积） | 按 MPW 要求或 die + 两侧余量取整；**不由 die 直接推导** |
| `<N_PAD>` | pad 环实例总数（信号 + 电源 + corner + filler） | 百级 |
| `<N_RULES_DRC>` | DRC deck 执行规则数（`RuleChecks Executed`） | **从 summary 读**（百级） |
| `<N_RULES_ANT>` | 天线 deck 规则数（取 `RuleChecks Executed`，别数 results） | 十级 |
| `<METAL_COUNTS>` | deck 金属层选项 | `5` / `6` 等 |
| `<TOP_METAL>` | deck 顶层金属厚度档 | 查 deck 头 `set top_metal` |
| `<CORNERS>` | PT 角集合 | `ss ff tt` + 温度/电压极端角 |
| `<FREQ>` / `<PERIOD>` | 出片频率 / 周期 | 由时序收敛结果定，别先拍 |
| `<GDSMAP>` | Innovus 四列层映射文件 | `<PROJ>/pnr/flow/gds.map.innovus` |
| `<ENV_IC>` | Innovus / Calibre / calibredrv / v2lvs 环境脚本 | `/path/to/env_ic.csh` |
| `<ENV_SY>` | DC / PT / StarRC / Formality 环境脚本 | `/path/to/env_sy.csh` |
| `<PDK>` | PDK 根 | `/path/to/pdk` |

> 环境脚本名与内容随站点不同（谁管 Innovus/Calibre、谁管 DC/PT/StarRC/Formality）。**不要在正文里写死**，只写 `<ENV_IC>` / `<ENV_SY>`。

**工艺常量表（第二张表，换 PDK 必填）**：20/30/50/60/90 里出现的 `<...>` 都指这一张，值一律自己查，本文不给。

| 占位符 | 含义 | 从哪取 |
|---|---|---|
| `<M5_LAYER>` `<M6_LAYER>` | 1P5M / 1P6M 的顶层金属层号 | DRC deck 的 `LAYER MAP` 段，或 `calibredrv` 看层表 |
| `<V4_LAYER>` `<V5_LAYER>` | 顶层通孔层号（1P5M / 1P6M） | 同上 |
| `<V4_SIZE_THICK>` / `<V4_SIZE_NORM>` | 顶层厚通孔 / 普通通孔的边长（mask 单位） | DRM 的通孔尺寸表；两档要能区分开 |
| `<T_THICK>` / `<T_RPSQ>` | 厚顶层金属的厚度 / 方阻 | ITF 或 nxtgrd 头部 `CONDUCTOR metalN { THICKNESS=… RPSQ=… }` |
| `<T_THICK_THIN>` / `<T_RPSQ_THIN>` | 被让出的那层金属在 1P6M 下的厚度 / 方阻 | 同上（同一次输出里两档都有） |
| `<M6TXT_LAYER>` | 顶层金属的 TEXT 层号 | deck 的 `LAYER MAP`（打错层 = 白做） |
| `<BORDER_LAYER>` | 掩模框层号 | deck 的 `LAYER MAP` |
| `<MARKER_1>` / `<MARKER_2>` | 两个豁免 marker 层的「层号:datatype」 | deck 里 `NOT INSIDE <MARKER_*>` 引用的那两个层 |
| `<DUMMY_BLOCK_LAYER>` | DMF 的 dummy 禁区层（若用 DUMBM 方案） | DMF deck 头部 |
| `<CORE_SITE>` | 标准单元 site 名与尺寸 | std cell LEF 的 `SITE` 定义 |
| `<PAD_W>` / `<PAD_H>` / `<PAD_PITCH>` | pad 宽 / 深 / 间距 | IO LEF 的 `MACRO` 尺寸 |
| `<FREQ_TAG>` | 频率标签（目录/文件名用） | 自己定，与 SDC 的 period 对应 |

**命名类占位符（就地替换，不需要查 PDK）**：

| 占位符 | 含义 |
|---|---|
| `<DB>` / `<SRAM1..3>` | 你的 Innovus DB 名 / 各 SRAM 宏名 |
| `<STDCELL>` `<STDCELL_LIB>` `<IO>` `<IO_LIB>` | 标准单元 cdl 文件 / 库名、IO 的 cdl / 库名 |
| `<INV_CELL>` `<PAD_FILLER>` `<PAD_CORNER>` `<PAD_MACRO_*>` | 反相器单元名、pad filler 前缀、corner pad 前缀、各 pad 宏名 |
| `<NET>` | 示例里的网名（`<NET>[6]` 这类，含总线位与转义标识符两种写法） |
| `<SITE>` / `<CORE_SITE>` | 同一个东西：标准单元 site 名（两处写法已统一，见 10 §三） |
| `<GAP>` / `<IO2CORE>` / `<PAD_OFF>` / `<HALO>` | pad 内沿→core、core←→pad 环、pad 外沿→die 边、宏 halo（布局参数，90 §12 有示例表） |
| `<RING_W>` `<RING_SP>` `<RING_OFFSET>` `<STRIPE_*>` `<MACRO_STRAP_*>` | 电源环 / stripe / 宏 strap 的宽、间距、偏移（布局参数，90 §12） |
| `<ARM>` `<O>` | 密封环角块臂长 / 环带起点偏移（建环脚本参数） |
| `<BAND>` / `<BAND_NM>` | marker 环带宽度（µm / nm 两种单位写法） |
| `<MARKER>` | 泛指「带 datatype 的 marker 层」，具体指 `<MARKER_1>` / `<MARKER_2>` |

## 1. 什么时候用 / 什么时候不要用

**用**（具体触发句，比「跑一遍后端」更准）：

- 「要做 SMIC 180 的 tapeout / 出片包 / MPW 表格」「1P5M 迁到 1P6M（或反向）」「pad 换 6MT」「重建密封环」「加大 dummy 填充」
- 「DRC 报 LU_1_G / LU_3_3a_G / LU_3_3b_G 上千条」「V4_1 爆 1000 条」「DRC 是 0 但不知道跑的是哪份 GDS」
- 「LVS 报 layout-only 上千个网」「LVS 只剩 3 个网」「Power shorted to ground while merging nets」
- 「天线怎么改都是 6 条降不下去」「最新改完的那版天线反而更脏」
- 「StarRC 报 EX-443 / SX-1936」「PT 读 SDC 丢约束」「ICC/Innovus 导出的 GDS 里没有 via」
- 「CTS 后不插 buffer / IMPCCOPT-2314」「hold 修不动，density 95% 卡住」

**不要用**：

- 纯模拟 / 混合信号版图设计；纯前端（只到 RTL 级）
- 别的 foundry 的 deck（规则名、层号、豁免机制全不同）
- 别的金属层选项 / 别的 PDK 版本：本 skill 里提到的**工艺常量**（顶层金属与通孔的 GDS 层号、通孔尺寸、各层厚度与方阻）一律按 `查该 metal option 的 DRM / ITF / deck` 处理，不写死

## 2. 硬规则（违反必出事，且大多无法事后自检）

1. **环境**：EDA 一律 `csh -c "source <ENV_x> >& /dev/null; <cmd> >& <log>"`。csh 重定向是 `>&`，写 `2>&1` 直接报 `Ambiguous output redirect`。
2. **calibredrv**：固定 `calibredrv <script.tcl> -- <args...>`，脚本内 `[lindex $argv 0]`；`-s`/`-a` 会报 `Tcl requires the first script argument to begin with '-'`。
3. **Innovus `streamOut` 必须用 `<GDSMAP>`（四列 `layerObjName layerObjType layerNumber dataType`）**。Astro/Virtuoso 三列 `gds.map` 会让工具**静默丢弃全部布线/电源/via**（GDS 反而更小、via 引用数 0）。**拿到 GDS 第一件事是数 via 引用，0 就是坏 map。**
4. **带 datatype 的 marker 层（deck 判豁免用的那两个 `层号:datatype`）在 `gdsout` 后会退化成 `.0`**，整个豁免失效。所有 marker 统一留到**最后一次 gdsout 之前**一次性重建（本文记作 `<MARKER_1>` / `<MARKER_2>`）。
5. **每个签核 deck 跑之前先 `grep -n 'LAYOUT PATH' <deck>`** 确认指向当期 GDS。真实翻车：deck 路径没换，DRC 的 0 条其实跑在另一个 GDS 上。
6. **不能按时间/编号猜哪版最好**。天线修复里"最后一次改动"反而有违规，早一版是 0 —— 候选版必须**逐版实测**。
7. **一次只跑一个重活**（DRC / DMF / 烘焙 / nxtgrd 编译都是 10~40 分钟级），用后台 job + 日志轮询。
8. **不要写 Python 动 GDS**。版图/网表用 calibredrv(Tcl) / Calibre / Innovus / sed / awk；只有 Excel 表格用 openpyxl。
9. **LVS 一律带 `-hier`**（flat 跑源侧会被展平成 0 个器件，结果是垃圾）；**SDC 一律 `source`，不用 `read_sdc`**（`read_sdc` 只认纯 SDC 命令，会静默丢掉 `set_dont_touch` / `get_cells -filter` / `remove_from_collection`）。
10. **数字必须绑定「文件 + md5 + deck + metal option」**；不同文件的数字混写=作废。同一条结论只认 `文件 + md5` 同一组。

## 3. 全流程地图

```
RTL ─► 综合(DC) ─► LEC-A(RTL↔syn, 需 SVF) ─► [DFT?] ─► Innovus
                                                         ├─ 00_init      LEF+网表+MMMC
                                                         ├─ 01_floorplan die/core/row/pad/macro
                                                         ├─ 02_pg_ring   环 + 宏 strap
                                                         ├─ 03_pg_route  sroute 五趟
                                                         ├─ 04_place
                                                         ├─ 05_cts       ccopt + propagated clock
                                                         ├─ 06b_hold_fix ff 角 hold，先删 filler
                                                         ├─ 06_route     NanoRoute + postRoute
                                                         └─ 07/08_export DEF/GDS/LVS 网表
                                                                │
              ┌─────────────────────────────────────────────────┘
              ▼
     1P6M 版图链（8 步，唯一权威见 references/20）
     pad 6MT → V4 缩孔 → M6 电源标签 → 官方密封环 → 烘焙 via → DMF → 合并填充 → 重建 marker
              │
              ▼
     DRC / 天线 / LVS ─► RC 重抽(tluplus→nxtgrd→StarRC) ─► PT 五角 ─► MPW 交付包
```

阶段之间只允许一个方向：**版图链产出 GDS → 三份 deck 签核 → 交付**。任何"回到 Innovus 改版图"都必须整条链重跑，不允许在 GDS 上打补丁后跳过前面的自检。

## 4. 每次改完版图必跑的验收清单

1. `grep -n 'LAYOUT PATH' <drc_deck> <ant_deck> <lvs_deck>` —— 三个都指向新 GDS
2. 层表检查：marker 层是否丢 datatype、有没有不该出现的层、**via 引用数 > 0**
3. **DRC `TOTAL DRC Results Generated = 0`**，且 `RuleChecks Executed = <N_RULES_DRC>`
4. **天线 `TOTAL ... = 0`**，`RuleChecks Executed = <N_RULES_ANT>`
5. **LVS `CORRECT`**，且 `Nets: N / N`，`0 layout-only / 0 source-only / 0 不匹配实例 / 0 缺失引脚`
6. **回读验收**：`calibredrv` 读一遍（层数/结构数/bbox/顶层名），并请人在 Virtuoso 里实导一次 —— **不许自己宣布「没问题」**
7. 面积口径：按 die 切割线算，**不用 BORDER**
8. `md5sum` + `cksum` 记下来（交付表要填），本地与服务器一致
9. **交付件与被测件必须是同一个文件**（`cmp`/md5 对账）
10. **时序要不要重跑**：只改 pad / 环 / 填充 / marker 这类不碰核心布线的东西 → RC 不变，不重跑；**动了 M5/M6 长线或电源 → 必须重抽 RC + 重跑 PT**

## 5. 坑清单速查（完整版见对应 reference）

| # | 坑 | 对策 | 详见 |
|---|---|---|---|
| 1 | 三列 `gds.map` 静默丢光布线/via | 用 `<GDSMAP>` 四列；先数 via 引用 | 10 |
| 2 | csh 写 `2>&1` 报错 | 用 `>&` | 全部 |
| 3 | marker 层 datatype 变 `.0` | 最后一次 gdsout 前一次性重建 | 20/30 |
| 4 | deck 的 `LAYOUT PATH` 没换 | 每次跑前 grep，交付前与包内 summary 对账 | 60 |
| 5 | LU 规则爆 4000+ 条 | 给电源 pad 的 **M6** 补 VDD/VSS TEXT（deck 靠 TEXT 认 pad） | 30 |
| 6 | `V4_1` 爆 1000 条 | 顶层厚通孔在 1P6M 降级为普通通孔，尺寸按新 DRM 缩（中心不动） | 20 |
| 7 | `import ... append` 按名覆盖同名 cell | 导入前查对方 cell 列表（pad 库曾偷改 V4） | 20 |
| 8 | LVS 上千假网 | deck 的 `CASE_SENSITIVITY` 改 `TRUE`（设计里有大量只差大小写的网） | 40 |
| 9 | `LVS FILTER <TOP>_VIA* OPEN` 报电源短路 | 改为烘焙进父单元 | 40 |
| 10 | pad filler 缺 stub | 源网表补空 `SUBCKT`，**引脚顺序与版图一致** | 40 |
| 11 | `\a[6]` 与 `a[6]` 被 v2lvs 合并 | 扁平 SPICE 定点改，**别改 Verilog 层次** | 40 |
| 12 | FP 虚脚版图被 pad 环切成多段 | 网表侧按版图分岛，不改版图、不豁免 | 40 |
| 13 | 「最新版反而更脏」 | 逐版实测，不按编号排序 | 50 |
| 14 | 天线 0 条但版图非法（二极管压 filler / 无布线 GDS） | 判 0 前先看 GDS 大小 + `verify_drc` + `verifyConnectivity` | 50 |
| 15 | StarRC 报 `EX-443` | tluplus 抠出 ITF → `grdgenxo` 编 nxtgrd | 50 |
| 16 | StarRC 报 `SX-1936` | 1P6M 里**降级成普通通孔的那一层**要映射到 tluplus 的普通通孔名（它在 1P5M 是厚顶层通孔，两个名字只差一个字母）；照抄上一代 map 必踩 | 50 |
| 17 | `grdgenxo` 产物不在 ITF 旁边 | 它在**当前目录**生成，手动归位再改命令文件 | 50 |
| 18 | PT 读 SDC 丢约束 | `source` 而不是 `read_sdc` | 50 |
| 19 | 密封环「整体平移」毁拐角 | 按官方结构重建（4 个角块单元 + 4 根直条），不许平移现成环 | 20 |
| 20 | dummy 填到密封环外侧 | DMF 要设禁区（`<DUMMY_BLOCK_LAYER>` 或等价手段），**环外到 BORDER 之间保持无 dummy** | 20 |
| 21 | 用 LVS `DATABASE EXTENT` 推断环位置 | EXTENT 是所有 LVS 层并集，最外的常是 dummy 金属；**用 calibredrv 量环带坐标** | 20/90 |
| 22 | 金属层上任何 TEXT 都变 pin 名 | 交付 GDS 剥掉非必要 TEXT；剥文本件与交付件分开管理 | 60 |
| 23 | flat LVS（不带 `-hier`） | 一律 `-hier`，否则源侧 0 器件 | 40 |
| 24 | 只跑 Calibre 就算过 | calibredrv 回读 + Virtuoso 实导 | 60 |
| 25 | 拿只抽了 M1–M5 的 SPEF 当完整 1P6M | 查 map 里有没有 `METAL6`/`VIA56`，写明抽取边界 | 50 |
| 26 | 不删 filler 就修 hold / 插二极管 | 先 `deleteFiller`，做完补回 | 10/50 |
| 27 | Innovus 里 `error` 了但退出码 0 | 只认 `reports/<step>/check.rpt` 的 `RESULT : PASS` | 10 |
| 28 | `restoreDesign` 指向 `.enc` | 必须指 `.enc.dat` 目录 | 10/50 |
| 29 | SDC/配置注释与代码不一致 | 以**代码行**为准，注释只当线索 | 00 |
| 30 | 上游文档数字连锁传抄 | 回 summary/summary 的 `RuleChecks Executed` 核，别抄文档 | 90 |

## 6. 判定标准（签核口径）

- **DRC**：`TOTAL DRC Results Generated = 0`。waiver 只能用在**上游 PDK 宏内部 / pad 接缝**这类「非本设计错误」上，标准是 **核心区 0 条**；每类要列规则名、条数、坐标区间，不掩盖、不夸大。
- **天线**：`TOTAL ... = 0`，规则数取 summary 的 `RuleChecks Executed`（不是 results 行数，ASCII results 每规则默认截 1000 条）。1P6M 不能沿用 1P5M 的 0 条结论。
- **LVS**：报告出现 `CORRECT`，`Nets: N / N`、`0 / 0`，**不用 waiver**。`Ambiguity points were found` 这句警告在正确报告里依然存在，不是判据。
- **PT**：`<CORNERS>` 全部 0 违例；必须写明用的哪版 SDC + SPEF 的抽取边界（本项目 M6 未抽，见 90）。
- **交付**：`md5` + `cksum` 双登记，本地与服务器一致，包内 GDS 与签核 GDS 同一文件。

### 6.1 证据纪律（从同类 skill `vibe-ic/tapeout-checklist` 与 `OpenADA` 吸收，见 `references/80-prior-art.md`）

1. **进程跑完 ≠ 检查通过**。要分开读：
   `execution.status = completed` 只代表工具跑完；`engineering.status = pass` 才是通过。
   `unknown`（证据不足/报告不可解析）**不许**升格成 pass。
2. **`NOT_MEASURED` 是一等结论**。报告缺失、版图没看、日志被截断 → `NOT_MEASURED`。
   **文件存在、有签名、哈希对得上，都不能替代"跑过并读了结果"。** 缺项就是缺项。
3. **waiver 要有 schema**：每条必须给 `ID / 规则名 / 条数 / 坐标区间 / 为什么不是本设计错误 / 具名审批人`。
   **AI 自批或自评无效**；`PASS_WITH_WAIVERS` 不等于 tapeout ready，必须显式列出残留风险。
4. **两份 GDS 不能用 md5 直接比**：GDSII 的 `BGNLIB`/`BGNSTR` 会写时间戳，
   同一版图两次 `streamOut` 的 md5 **通常会不同**。要比内容，先清零日期字段再做 digest；
   而交付表里填的 md5，是**那一个文件**的 md5（用于校验传输完整性，不是用来判两份版图是否相同）。
5. **die 的定义**：die 应以**密封环那一步实测出的框**为准，不是"所有层的 bbox"。
   密封环生成器可能把角标/identification marker 画到 die 外面（SKY130 就有一个半径 152 µm 的角标
   octagon）。判"版图越界"之前先看**是哪一层**越界。常见同款现象：BORDER / die / 密封环三者各是一个口径（示例见 90 §2–§3）。

### 6.2 签核八类（借它的分类框架自查，本 skill 只覆盖到第 4 类）

| # | 类别 | 本 skill 覆盖？ |
|---|---|---|
| 1 | functional（回归 / 形式 / LEC） | 部分（LEC 见 00，回归不在本 skill） |
| 2 | timing（MCMM STA） | 是（50），但本项目只做单模式五角 |
| 3 | power integrity（IR-drop / EM） | **没有** ← 已知空洞（要补就跑 pdnsim/Voltus，见 85 §3） |
| 4 | physical（DRC / LVS / antenna / density / ERC） | 是（20/30/40/50） |
| 4b | **密封环 SR.7/8/9（DRC 标 `[NC]` = 不检查！）** | 必须**读 DRM 人工核**：环宽、环到芯片边的装配隔离、两者之和，三项各查 DRM 的下限；**本文不给数，案例值也不能照抄到别的 die 上** |
| 5 | DFT coverage | **没有**（本项目决定不插链；要补见 85 §5） |
| 6 | power intent（UPF） | **没有**（单电源域） |
| 7 | documentation | 部分（60） |
| 8 | GDS release（checksum / manifest） | 是（60） |

## 7. 时间紧时的优先级

**LVS CORRECT > DRC 0 > 天线 0 > 时序**。时序可以降频出片，DRC/LVS/天线有违规 foundry 直接拒收。

**降频的决策规则**（不是「先降了再说」）：

1. 先算**物理根因**：换金属层选项后，厚度/方阻变了多少？关键路径是不是落在被变薄的层上？（示例见 90 §8「1P6M 物理参数」）
2. 再用 ECO 试收：PT `fix_eco_timing` 出的 cell 改动够不够补差额？只差几个 size_cell 就没戏。
3. ECO 不可能在无硅验证前补上 1 ns 以上 → **降频是唯一零改动出路**；降频后必须在**新周期**下重跑全角，并写明用的哪版 SDC。
4. 若根因是「厚金属被挪走」而不是工具/约束问题，**不要**去改 SDC 的 uncertainty/transition 糊过去。

## 8. 已知未闭合项 / 适用边界（诚实条款）

- **LEC-A（RTL↔综合网表）可能未关闭**：本案终态是「未证，需 waiver 或升级工具后重跑」。别照抄工程 README 里的旧 `Verification SUCCEEDED`，要看最新 LEC 报告。（细节见 00 与 90）
- **DFT 可能整体没做**：插了链但链头落在 pad 外侧、绕过输入缓冲器 ⇒ 交付网表应实测 `SD*` 触发器 = 0。写报告要说「未做 ATPG、扫描覆盖率无法测量」，不许含糊。
- **SPEF 的抽取边界**：映射文件可能只有 `METAL1..5` + `VIA12..45`（无 `METAL6`/`VIA56`），且 `TOP_DEF_FILE` 是上一代 1P5M DEF ⇒ 实际是「1P5M 结构 + 新 metal option 介电/方阻的 M1–M5 抽取」，**M6 没抽**。PT 结论只对 M1–M5 有效。
- **IR-drop / EM / SI 没验**：本项目只做了 DRC/LVS/天线/时序，上电完整性（静态 IR）、电迁移、串扰都没跑。foundry 若要求，这是硬缺口。
- **MCMM 只做了单模式**：五角是按 corner 分别跑的，没有 mode（功能/扫描）交叉。
- **仓库里可能有两份都自称「最终」的 GDS**：按 md5 与生成链（g 链 / h 链）判决，以**交付包里那份**为准，另一份标为上一代。

## 9. 深水区参考文档（按阶段读，不要一次全读）

| 文件 | 什么时候读 |
|---|---|
| `references/00-frontend.md` | 从 RTL 起步，要准备库/约束、跑 DC 综合、做 LEC、判断要不要 DFT 时 |
| `references/10-pnr.md` | 要在 Innovus 里从 LEF/网表跑到 DEF/GDS，或某步 `check.rpt` FAIL 时 |
| `references/20-layout-chain-1p6m.md` | 要把版图迁到 1P6M（或反向），按 8 步重建带密封环 / DMF 的交付版图时 |
| `references/30-drc-debug.md` | DRC 报几百上千条，要判断「是我们的版图还是 PDK/pad 库」，或要立 waiver 时 |
| `references/40-lvs-debug.md` | LVS 报 layout-only/source-only 上百上千、只剩个位数网、或要补 stub/改网表时 |
| `references/50-ant-rc-pt.md` | 要跑天线签核、判断「最新版反而更脏」，或要从 tluplus 建 RC、跑 PT 五角时 |
| `references/60-deliverable.md` | 签核已过，要打流片包、填 MPW 表、做交付前最后一道验收时 |
| `references/90-case-study.md` | 需要一个具体数字做量级对照、或写报告要举实例时（**唯一出现实测数值的文件，已脱敏**） |
| `references/85-signoff-gate.md` | **GDS 要交付、要回答"这版能不能交"、要逐项填放行表时（八类签核 + 四种状态 + waiver 治理 + IR/EM/DFT 的空洞清单）** |
| `references/80-prior-art.md` | 想对标业界同类 agent skill（vibe-ic tapeout-checklist / OpenADA / digital-chip-design-agents）、或要补 IR-drop·EM·DFT·waiver 治理等本 skill 的空洞时 |
| `references/70-external-practice.md` | 想对照行业通行做法 / 官方规范要点（seal ring、天线比值、MPW 提交、ITF→nxtgrd），或要给报告加外部引用时（**只含外部资料，无本项目数字**） |

## 10. 文件树

```
smic180-1p6m-tapeout/
├── SKILL.md                              唯一入口：变量表 + 硬规则 + 流程图 + 验收清单 + 坑速查 + 读文档导航
└── references/
    ├── 00-frontend.md                    库准备 / SDC 体系 / DC 综合 / LEC 两段 / DFT 现状 / 前后端接口
    ├── 10-pnr.md                         Innovus 九步线性流程（floorplan→PG→place→CTS→hold→route→export）
    ├── 20-layout-chain-1p6m.md           1P6M 版图链八步：输入输出契约 + 五处开关 + marker/豁免机制（唯一权威）
    ├── 30-drc-debug.md                   DRC 现象→根因→对策六个案例 + waiver 口径 + deck diff 复核法
    ├── 40-lvs-debug.md                   LVS 调试：报告读法 + 大小写/烘焙/stub/转义标识符/FP 分岛 + 10 步顺序
    ├── 50-ant-rc-pt.md                   天线签核与逐版搜索方法 + tluplus→nxtgrd→StarRC→PT 五角 + 降频决策
    ├── 60-deliverable.md                 交付打包 / MPW 表字段 / 回读验收 / 数字绑定纪律
    ├── 70-external-practice.md           行业通行做法 / 官方规范要点（31 条外部来源，标注来源等级）
    ├── 80-prior-art.md                   同类 skill 调研 + 差距表 + 本 skill 已知空洞
    ├── 85-signoff-gate.md               放行闸门：八类签核逐项清单 + 四态判定 + waiver 治理 + 记录模板
    └── 90-case-study.md            匿名案例数据（几何、计数、逐版天线表、PT 五角…；不含指纹与路径）
```

一句话说明：**SKILL.md 负责「什么时候读哪篇 + 不许犯的错」，references 负责「具体怎么跑」与「凭什么这么判」，90 负责「量级对照」。**

---

## 11. 来源、许可与免责

- 本 skill 综合多个真实项目的经验整理而成；案例数字做了区间化/量级化处理，**不指向任何具体项目或客户**，已去掉公司名、设计名、主机、路径、哈希与日期。
- **按内容类型分许可**（不是对同一作品二选一）：文档正文（`SKILL.md`、`README.md`、`references/*.md`）为 **CC BY 4.0**，见 `LICENSE`；文档里的命令/脚本片段为 **MIT**，见 `LICENSE-MIT`。引用请署名「© 2026 cannotgetaname」+ 来源链接，并注明是否修改。
- 文中提到的 PDK / DRM / Calibre deck / EDA 工具与厂商产品名均为其各自权利人所有，仅在**指称意义上**使用（nominative use），不表示任何隶属、赞助、审阅或背书；本仓库不是其官方文档。本 skill **不包含**任何 PDK 文件、deck 原文或厂商手册内容；工艺常量主要用占位符。
- 引用外部资料时只给结论与链接（见 70 / 80），不转载原文。
- 免责：流程与判据仅供参考，**签核结论必须由你本人在自己的 PDK 版本、自己的报告上得出**；照抄别人的「0 条」是事故的常见起点。
