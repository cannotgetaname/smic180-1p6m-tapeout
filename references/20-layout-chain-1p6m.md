# 20 · 1P6M 版图链（8 步）— 输入/输出契约、开关锁定、marker 与豁免机制

> **什么时候读我**：要把 1P5M 的 P&R GDS 迁到 1P6M（或反向），按 8 步重建带官方密封环 + DMF 填充 + marker 的交付版图时。
> 本篇是这条链的**唯一权威**：其它 reference 提到 pad swap / 烘焙 via / DMF / marker 时都指回这里。
> 所有 8 步的实测计数在 `90-case-study.md`（本篇只写自检项名与判据）。

---

## 一、开工前锁死六处开关（错一处整轮白跑）

PDK 随包 deck **经常带着错的上限注释**：注释写 1P6M、代码里还是 `metal_counts 5`。这是本项目最大的返工根因。

| # | 位置（用 `grep -n` 定位，行号随 deck 版本变） | 必须是 | 踩过的坑 |
|---|---|---|---|
| 1 | DRC / 天线 deck 的 `set metal_counts` | `<METAL_COUNTS>` | 注释 1P6M、实际 5 ⇒ 整轮按 1P5M 签核 |
| 2 | 同文件 `set top_metal` | `<TOP_METAL>`（厚顶层档） | 选错厚度档会按别的几档厚度判，全错 |
| 3 | pad 库变体 | **`_6MT`**（1P6M 要 M6 + V5） | 5MT pad 没有 M6/V5 ⇒ deck 的 `PA_1` 悬空 ⇒ LU 规则**假阴性** |
| 4 | 密封环顶层金属 | 顶层 M + 顶层 V | 官方角块单元自带的 V1~V5 是顶层通孔尺寸，只靠 marker 层豁免 |
| 5 | DMF deck 的 `total_metal_counts` + `M6_insert` | `<METAL_COUNTS>`；`M6_insert YES` | 少一层 ⇒ dummy 少填一层 |
| 6 | DMF 的 `TM1_TM2_used_as_top_metal` / `TM2_insert` | **NO** | 设 YES 会把 M6 dummy 写到 **TM2 的 GDS 号**，而本工艺 M6 是另一个 GDS 号，白做 |

**自检命令（一行锁六处）：**

```bash
grep -nE '^ *set (metal_counts|top_metal)|^ *(M6_insert|TM2_insert|TM1_TM2_used_as_top_metal|total_metal_counts)' <WORKDIR>/rules/*.drc <WORKDIR>/rules/*.ant <WORKDIR>/rules/*.dmf
```

**判据：同一轮里 DRC / 天线 / DMF 的 metal option 必须完全一致**，且与「买到的层数」一致。写进报告的每个结果都要带 metal option。

---

## 二、metal option 的工艺常量（换工艺 / 换 metal option 全部重查）

| 项 | 1P5M | 1P6M | 影响 |
|---|---|---|---|
| 顶层金属 GDS 层 | M5 = `<M5_LAYER>` | **M6 = `<M6_LAYER>`** | 层表、天线上限、DMF 目标层都跟着变 |
| 顶层通孔 GDS 层 | V4 = `<V4_LAYER>` | **V5 = `<V5_LAYER>`** | 同上 |
| 顶层通孔尺寸（mask 单位） | 厚顶层通孔 **`<V4_SIZE_THICK>`** | 普通通孔 **`<V4_SIZE_NORM>`** | 老 GDS 直接用会报 `V4_1` 上千条 |
| 被让出的那层金属：厚度 / 方阻 | `<T_THICK>` / `<T_RPSQ>`（1P5M 下它是厚顶层） | **`<T_THICK_THIN>` / `<T_RPSQ_THIN>`**：厚度约 **0.56×**、方阻约 **3×**（两个倍数别混成一个） | 长线电阻 ≈3×，时序一定变差 |
| 新顶层金属：厚度 / 方阻 | — | `<T_THICK>` / `<T_RPSQ>`（沿用厚顶层档） | 厚金属从 M5 让给 M6 |

> 读数来源是 ITF/nxtgrd 头部的 `CONDUCTOR metalN { THICKNESS=… RPSQ=… }`。**这三条决定了「1P6M 时序为什么变差」，不是工具/约束问题**（决策见 50）。

---

## 三、8 步链的输入 → 输出契约

这是一套**命名约定**，不是硬性要求；新项目可以换名字，但每一步的「输入 → 输出 → 自检计数」必须成立。脚本名可换，**功能不能省**。

| step | 做什么 | 输入 → 输出 | 自检项（必须打印出来） |
|---|---|---|---|
| 0 | Innovus DB 导出 | `<DB>.enc.dat` → `f0.gds` + LVS 网表 `.v` | 必须用 `<GDSMAP>`；`verifyConnectivity -type all` 为空；**via 引用数 > 0**、refs 数、cell 数 |
| 1 | pad 5MT→6MT | `f0.gds` + `pads6/*.gds`（N 个宏）→ `g1_pad6.gds` | 先记录 `<N_PAD>` 个 ref 的 (x,y,mirror,angle,mag) → 删旧同名 cell → `import append` → 按原坐标回填；`TOPREF_FINAL == TOPREF_BEFORE`、`REFS_RESTORED = N/N` |
| 2 | V4 缩孔 | `g1_pad6.gds` → `g2_v4.gds` | 只匹配 `w==<$V4_SIZE_THICK> && h==<$V4_SIZE_THICK>` 就地缩成 `<$V4_SIZE_NORM>` 见方（中心不动）；打印 `TOTAL_V4` 与 `SHRUNK` |
| 3 | 电源 pad 金属标签 | `g2_v4.gds` → `g3_lbl.gds` | 在**顶层金属**层给每个电源 pad 打 VDD/VSS 文本；`TEXTS_ADDED = N` |
| 4 | 官方密封环 + BORDER | `g3_lbl.gds` + `sealring_src.gds` → `h2_ring.gds` | 4 个角块实例 + 4 根直条 + BORDER 矩形；打印环带坐标 / 图形数 |
| 5 | 烘焙 via 实例 | `h2_ring.gds` → `h3_novia.gds` | `<TOP>_VIA*` 全部按 (x,y,mi,an,mg) 变换写进父单元后删 ref/cell；`BAKED_INSTANCES = N`、cells 数变化 |
| 6 | DMF 填充 | `h3_novia.gds` → `dummy_h3.gds` | 输出是**平行顶层** `<TOP>_dummy`；覆盖率目标（按 foundry 规范） |
| 7 | 合并填充 | `h3_novia.gds` + `dummy_h3.gds` → `h4_fill.gds` | 逐个 `create ref` 挂到 `<TOP>` 再删 `_dummy`；`FILL_REFS_CREATED = N/N` |
| 8 | 重建 marker 层 | `h4_fill.gds` → **`h5_final.gds`** | 一次性重建 `<MARKER_1>` + `<MARKER_2>`；先清 `gdsout` 留下的 `.0` 残留；`LAYERS_AFTER` 里必须同时出现两个 marker 层的**原始 datatype** |
| 9 | 三份签核 | `h5_final.gds` | DRC 0（`<N_RULES_DRC>` 规则）/ 天线 0（`<N_RULES_ANT>` 规则）/ LVS CORRECT |

**每步脚本的退出码约定**（本项目所有脚本统一）：

| 退出码 | 含义 | 处理 |
|---|---|---|
| 0 | 成功，自检计数已打印 | 继续下一步 |
| 2 | 参数/文件缺失 | 查 argv 与输入路径 |
| 4 | 自检计数不符（断言失败） | **停下**，看该步 log；不要带着坏输入往下跑 |

> **没看到自检计数就等于没跑成功**——不要只看退出码。

---

## 四、完整命令模板

```csh
#!/bin/csh -f
source <ENV_IC>
cd <PROJ>

# 79 类 via 单元由 gdsout 按需生成，个数随设计变；用 seq 生成，别手抄
set VIAS=`seq -f "${TOP}_VIA%g" 0 78`      # 注意加引号，否则 < > 会被 csh 当重定向

# step 1  pad 换 6MT（宏名与 pads6/*.gds 一一对应）
calibredrv <WORKDIR>/scripts/p6_swap.tcl -- <WORKDIR>/gds/f0.gds <WORKDIR>/gds/pads6 <WORKDIR>/gds/g1_pad6.gds \
  <PAD_MACRO_1> <PAD_MACRO_2> ... >& <WORKDIR>/reports/g1.log

# step 2  厚顶层通孔 -> 普通通孔尺寸
calibredrv <WORKDIR>/scripts/v4_shrink.tcl -- <WORKDIR>/gds/g1_pad6.gds <WORKDIR>/gds/g2_v4.gds >& <WORKDIR>/reports/g2.log

# step 3  电源标签（顶层金属 TEXT 层号从 deck 的 LAYER MAP 查，不能猜）
calibredrv <WORKDIR>/scripts/add_pwr_labels.tcl -- <WORKDIR>/gds/g2_v4.gds <WORKDIR>/gds/g3_lbl.gds <M6TXT_LAYER> >& <WORKDIR>/reports/g3.log

# step 4  官方环 + BORDER
calibredrv <WORKDIR>/scripts/mk_frame.tcl -- <WORKDIR>/gds/g3_lbl.gds <WORKDIR>/gds/sealring_src.gds <WORKDIR>/gds/h2_ring.gds >& <WORKDIR>/reports/h2.log

# step 5  烘焙 via 单元
calibredrv <WORKDIR>/scripts/bake_vias.tcl -- <WORKDIR>/gds/h2_ring.gds <WORKDIR>/gds/h3_novia.gds $VIAS >& <WORKDIR>/reports/h3_bake.log

# step 6  DMF（也走 calibre -drc，且必须 hier）
calibre -drc -hier <WORKDIR>/rules/dmf.dmf >& <WORKDIR>/reports/dmf_h3.log

# step 7  合并填充
calibredrv <WORKDIR>/scripts/merge_fill.tcl -- <WORKDIR>/gds/h3_novia.gds <WORKDIR>/gds/dummy_h3.gds <WORKDIR>/gds/h4_fill.gds >& <WORKDIR>/reports/h4.log

# step 8  重建 marker（环带宽度参数，见 90）
calibredrv <WORKDIR>/scripts/fix_all_markers.tcl -- <WORKDIR>/gds/h4_fill.gds <WORKDIR>/gds/h5_final.gds <BAND_NM> >& <WORKDIR>/reports/h5.log

# step 9  三份签核（-hier 不能省；天线也用 -drc 跑）
calibre -drc -hier <WORKDIR>/rules/drc_h5.drc >& <WORKDIR>/reports/drc_h5.log
calibre -drc -hier <WORKDIR>/rules/ant_h5.ant >& <WORKDIR>/reports/ant_h5.log
calibre -lvs -hier <WORKDIR>/rules/lvs_h5.lvs >& <WORKDIR>/reports/lvs_h5.log
```

---

## 五、每一步的机理与坑

### step 1 · pad swap：`import append` 会按名覆盖

- 做法：**先记坐标 → 删旧同名 cell → `import append` 新库 → 按原坐标回填**。直接 append 会留下旧 cell 或错位。
- **副作用（重要）**：pad 库 GDS 自带 `<TOP>_VIA<层对>` 一类通孔单元，`import append` 会**按名字覆盖**设计里的同名单元。实测效果可能是「顶层通孔被换成普通通孔尺寸」——正好是 1P6M 要的，但这是**意外修好，不许依赖**：step 2 必须独立做，且要能解释 V4 统计数为什么和 step 1 之前不同。
- 验证：宏 bbox 一致、开口层与 M6 一致、新增 M6/V5 层、顶层 ref 数不变。

### step 2 · V4 缩孔

- **三个 V4 数字必须解释清楚，否则读者会以为哪个写错**：① 换 pad 前 DB 里 `<V4_SIZE_THICK>` 方孔总数；② 换 pad 后 GDS 里剩余的同类方孔总数（含 pad 库自带的 V4）；③ 脚本实际缩掉的个数。三者可以差几倍，因为 `TOTAL_V4` 统计的是**整个 GDS** 里 `<V4_LAYER>` 层的多边形，`SHRUNK` 只动尺寸匹配的那批。写报告时三个数各自绑定来源。
- 只缩尺寸、**中心不动** ⇒ 连接关系不变，包覆只会更宽裕。

### step 3 · 电源 pad 的标签：为什么必须是顶层金属

- deck 靠**金属层 TEXT 上的 VDD/VSS 字符串**认电源 pad（`DEFINE_PAD_BY_TEXT` / `PADTEXT_TOP` / `POWER_NAME` / `GROUND_NAME`）。P&R 导出**没有任何文本** ⇒ deck 判不出电源 pad ⇒ 整片电源网上的 N 扩散被当成 IO 器件 ⇒ LU 规则爆几千条（案例见 30）。
- **标签要落在真正连通的金属上**：本项目打在 M5 上无效（pad 局部那块 M5 是悬空 plate，与它相交的 V5 数为 0），打在 M6 上才有效——M6 覆盖整个 pad 且有几千个 V5 接回 M5。
- 判据：**标签层必须同时满足「覆盖整个 pad」+「与 pad 电源网有 via 连通」**。打之前先在 pad 局部数 via，别凭层号猜。

### step 4 · 密封环：按官方结构重建，绝不「平移」现成环

- 官方只给**角块单元 + 剖面**：直条的层组合与宽度**从官方剖面图逐层量**（层名与区间自己记，本文不给数），**直边通常没有通孔阵列**（以官方直条单元自带的 via 层是否为空为准）。
- 做法：`4 个角块实例（4 种 mirror/angle 组合）+ 4 根直条`，删掉不属于本 metal option 的层：角块单元里属于别的 metal 选项的层要删，它自带的标记框层也删掉，BORDER 改用自画的矩形。
- **位置是设计决定，不是库给的**：band 起点、BORDER 外框、dummy 禁区层都得自己定。交付版按 DRM 的**最小装配隔离**放在 die 内侧（环外沿与 die 边之间留出 DRM 规定的隔离带，同时环本身占满环宽）；也见过把环放在 die 外侧的做法——**两种都可能，必须按交付 GDS 实测写**（示例见 90）。
- 🔴 **整环平移会毁拐角**：只平移不缩短直条 ⇒ 四角多出 `pad_off` 量级的重叠 ⇒ 拐角一堆 spacing 违例。必须按结构重建。
- **BORDER（`<BORDER_LAYER>`）只是标记框**：画到让掩模 bbox 等于 `<BORDER_UM>` 即可，**不是 die，不能算面积**。
- ⚠️ **不要用 LVS 报告的 `DATABASE EXTENT` 推断环的位置**：它是所有 LVS 层的并集，最外的常常是 **DMF dummy 金属**而不是环。判环位置只能用 calibredrv 量环带坐标（`iterator poly` 看顶层 marker 层或密封环直条层的 x/y 范围）。

### step 5 · 烘焙 via 实例（不能 delete，也不能 FILTER）

- Innovus `streamOut` 会生成 `<TOP>_VIA0…VIAN` 这类**只有几何、网表里不存在**的单元，LVS 会报 missing instance。
- 两条错误路线：① 直接 `delete cell` 会连 SREF 一起删、把版图挖洞（网数爆炸）；② `LVS FILTER ... OPEN` 对纯几何单元无效，还会把同一实例的电源脚和地脚一起摘掉 ⇒ `Power shorted to ground`。
- 正确做法：**烘焙（bake）进父单元** —— 按 (x,y,mirror,angle,mag) 变换顶点写进父单元，再删 ref 和 cell。之后两侧都不存在该 cell。

### step 6–7 · DMF 填充与合并

- DMF 输出是**平行顶层**（`<TOP>_dummy`），不是挂在原顶层上。必须逐个 `create ref` 挂回 `<TOP>` 再 `delete cell <TOP>_dummy`（填充 cell 本身无人引用，删顶层安全）。
- 覆盖率目标按 foundry 规范填（本项目取值见 90），不要自己拍。
- 🔴 **dummy 只能填到密封环内侧**。本项目踩过：dummy 一直填到 BORDER 内侧，**填到了环外面**（LVS `DATABASE EXTENT` 的最外坐标就是被 dummy 撑出去的）。环外到 BORDER 之间应保持空。对策：给 DMF 设禁区（`<DUMMY_BLOCK_LAYER>` 自造层或等价手段），并在合并后用环带坐标复核。
- 自检：`FILL_REFS_CREATED` 必须等于 dummy 顶层里的 ref 总数。

### step 8 · marker 层：最后一次 gdsout 之前一次性重建

- 现象：`gdsout` 不保留 marker 层的 datatype（`<MARKER_1>` 被写成 `<层号>:0`），deck 里写的 `<层号>:<datatype>` 完全对不上 ⇒ 豁免失效 ⇒ LU 与 via-stack 规则回到几百上千条。
- 对策：**中途不画 marker**；最后一步统一重建，同一脚本先删 `.0` 残留，再画：
  - `<MARKER_1>`：覆盖**整个环**（含四角）的少量矩形 —— 让官方角块单元里的顶层尺寸通孔合法；
  - `<MARKER_2>`：**每个 pad 实例 + 每个 SRAM 宏的方框** + 一圈**环带矩形**（带宽 `<BAND>`，量级：几十~几百 µm，按环带与 pad 行实际宽度定）—— 让 pad 接缝 / 库内部 / 环带细缝的残余违规有覆盖依据。
- 判据：重建后 `$L layers` 里**同时**出现两个 marker 的原始 datatype，且图形数符合预期。整个豁免机制就是「deck 的判定式里写了 `NOT (<MARKER_1> OR <MARKER_2>)`，我们把不该由设计承担的图形盖住」——**不改判定式本身**（除非确实要加一条新的覆盖规则，并在报告里逐条说明）。

---

## 六、最小可复现集（别抄探针脚本）

8 步流水只依赖这 8 个脚本 + 1 份密封环源 GDS。`scripts/` 下通常还有上百个一次性探针（`*_probe.tcl`、`laydbg.tcl`、`*_dbg.tcl`），**不要当流程用**：

| 功能 | 脚本（名字可换） |
|---|---|
| DB → GDS + 网表 | `reexport_db*.tcl` |
| pad swap | `p6_swap.tcl` |
| V4 缩孔 | `v4_shrink.tcl` |
| 电源标签 | `add_pwr_labels.tcl` |
| 密封环 + BORDER | `mk_frame*.tcl`（**以最终交付那条链实际用的版本为准**，新旧两条链可能用不同版本） |
| 烘焙 via | `bake_vias.tcl` |
| 合并填充 | `merge_fill.tcl` |
| 重建 marker | `fix_all_markers.tcl`（旧版可能只有 `fix_marks.tcl`，只重建一个 marker，不够用） |

---

## 七、换项目改哪几行

| 要改的东西 | 在哪 |
|---|---|
| metal option / top metal / pad 变体 / DMF 开关 | 三份 deck 头部（见本文件第一节的 grep） |
| 顶层金属与顶层通孔的 GDS 层号 | deck 的 LAYER MAP + 脚本里的层号常量 |
| 顶层通孔两档尺寸（查 DRM） | `v4_shrink.tcl` 的匹配条件与目标值 |
| TEXT 层号 | deck LAYER MAP（打错层 = 白做） |
| 环的位置 / 宽度 / BORDER 坐标 | `mk_frame*.tcl` 的 `O`、`ARM`、BORDER 两个矩形 |
| 环带宽度 `<BAND>`、pad/SRAM 方框清单 | `fix_all_markers.tcl` 的参数 + 单元名列表 |
| 填充覆盖率目标 | DMF deck（按 foundry 规范） |
| via 单元名前缀与个数 | 用 `seq` 生成，前缀跟 `<TOP>` 走 |
