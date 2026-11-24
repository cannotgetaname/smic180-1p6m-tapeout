# 10 · Innovus P&R 线性流程（floorplan → PG → place → CTS → hold → route → export）

> **什么时候读我**：要在 Innovus 里从 LEF + 网表跑到 DEF/GDS，或九步里某一步 `check.rpt` 报 FAIL 时。
> 天线逐版 ECO 不在这篇，见 `50-ant-rc-pt.md`。

> 本文件用到的参数（**具体取值见 `90-case-study.md` 的「P&R 参数表」**，这里只写关系式）：
> `<PAD_H>` pad 深度、`<PAD_OFF>` pad 环外沿距 die 边、`<GAP>` pad 内沿到 core、`<CORE_W>`、`<IO2CORE>`、`<RING_W>/<RING_SP>/<RING_OFFSET>`、`<STRIPE_W>/<STRIPE_SP>/<STRIPE_PITCH>/<STRIPE_OFF>`、`<MACRO_STRAP_W>/<MACRO_STRAP_N>`、`<HALO>`。

---

## 一、怎么跑：一条命令 + 一个 PASS 判据

```csh
# 单步（cwd 必须是 innovus/；脚本内部 source config.tcl 定位全路径）
csh -c "source <ENV_IC> >& /dev/null; innovus -no_gui -files flow/01_floorplan.tcl -log logs/01_floorplan.log >& logs/01_floorplan.stdout"

# 批量（flow/run.sh 封装上面这条：逐 step 跑、失败即停）
bash flow/run.sh 00_init 01_floorplan 02_pg_ring 03_pg_route 04_place
```

- ⚠️ **Innovus 即使 Tcl 里 `error`（`fl::fail`）退出码仍是 0**。判据只有 `reports/<step>/check.rpt` 末行 `RESULT : PASS`；`run.sh` 就是靠 `grep -q "RESULT : PASS"` 卡关的。
- **参数唯一源 `flow/config.tcl`**：几何 / 层 / 宽度 / 库路径 / 宏表 / pad 表都在这里，步骤脚本里不出现魔数。改工艺先改这一份。
- `flow/lib.tcl` 提供 `fl::begin/chk/need/rlog/restore/save/wns/show`：`fl::need` 是**真断言**（不满足整步 FAIL），`fl::rlog` 把单条命令的 log 单独落到 `reports/<step>/<name>.log`。
- ⚠️ **config.tcl 的注释会过期**：本项目有注释还写着「密封环在 die 外侧」，而交付 GDS 实测环在 die 内侧（判决见 90）。**注释只当线索，几何以 `fl::need` 断言 + 实测 bbox 为准。**

---

## 二、九个步骤脚本（`innovus/flow/`）

| 阶段 | 目标 | 关键命令（实参形状） | 产物 | 最容易出错的地方 |
|---|---|---|---|---|
| `00_init` | LEF + 网表 + MMMC(ss) | `create_library_set/delay_corner/constraint_mode/analysis_view`、`init_design -setup/-hold` | `db/<TOP>_00_init.enc` | PDK 只有 TLUPlus 没 ICT ⇒ `generateCapTbl` 出的 captable 只有表头；脚本用**文件大小阈值**判断后回退 LEF 级 RC。SRAM 宏数与信号 pad 数必须识别到（本项目 DFT pad 已从网表删） |
| `01_floorplan` | die/core/row + 宏 + pad 环 + IO filler | `deleteRow -all` → `floorPlan -site <SITE> -noSnapToGrid -s <CORE_W> <CORE_W> <IO2CORE>*4`；`placeInstance ... -fixed`；`createPlaceBlockage -type soft`；`cutRow -area ... -keepCell`；`fl::place_cell`/`fl::io_fill` | `01_floorplan.enc` + `db/geom.tcl` | `init_design` 自带默认 floorplan 和几百行 row，不先 `deleteRow -all` 就只是 resize；不加 `-noSnapToGrid` 会被 track 对齐挪位；`cutRow` 不带 `-keepCell` 会把 fixed 宏 unplace |
| `02_pg_ring` | 电源环 + 宏 PG strap + 竖 stripe | `globalNetConnect VDD/VSS -type pgpin -override`；`addRing -type core_rings -follow core`；`addStripe`（宏 strap 先做，再 core stripe） | `02_pg_ring.enc` | 给宏上方加 M4/M5 route blockage 是**错的**（SRAM 只有 M1–M3，封死上层 ⇒ 几十条 DRC）；`addRing -around io` 在某些 pad 组合上必然失败（`IMPPP-4051`） |
| `03_pg_route` | sroute 五趟 + 连通性/DRC 验收 | `sroute -connect {floatingStripe}`、`{corePin} -corePinTarget {stripe ring}`、`{blockPin} -layerChangeRange {METAL3 METAL5}`、`{padPin}` 按 net 分开跑 | `03_pg_route.enc` | `corePin` 用默认 `nearestTarget` 会把 followpin 拉进宏里；`blockPin` 若允许 M2 就退化成贴 VSS 环的 M2 竖线；电源 pad 引脚只到 M2 时必须按 net 分开 |
| `04_place` | `place_opt_design` + preCTS | `setPlaceMode -place_global_place_io_pins false`；`place_opt_design`；`reportCongestion -overflow`；`report_area`；`timeDesign -preCTS`；`verify_drc` | `04_place.enc` | 只验「无 DRC 级重叠」不够，必须同时看 **density 和 overflow** |
| `05_cts` | ccopt + postCTS opt | 换 P&R SDC → `create_ccopt_clock_tree_spec` → `set_ccopt_property buffer_cells/target_max_trans/target_skew` → `ccopt_design` → `set_propagated_clock [all_clocks]` → `optDesign -postCTS -setup -hold` | `05_cts.enc` | SDC 里 `set_dont_touch_network`/`set_ideal_network` 没清 ⇒ `IMPCCOPT-2314` 全树不插 buffer；19.11 这两条**没有 true/false 参数**，写 `... false` 是语法错会被 `catch` 静默吞掉 |
| `06b_hold_fix` | ff 角 hold 修复 | 建 `libs_ff`/`dc_ff`/`view_ff` → `set_analysis_view -setup view_ss -hold view_ff` → `deleteFiller` → `optDesign -postRoute -hold`（≤N 轮）→ `addFiller` → `globalNetConnect` | `06b_hold_fix.enc` | **不先删 filler 就修 hold 一定失败**：`Stopping hold fixing due to density exceeding max design density`，#Buffer=0 |
| `06_route` | 详细布线 + postRoute | `setNanoRouteMode -routeWithTimingDriven true`、`-drouteUseMultiCutViaEffort high`；`routeDesign`；`optDesign -postRoute -setup -hold`；`addFiller`；`ecoRoute -fix_drc`（≤N 轮）；`timeDesign -postRoute`；`verifyConnectivity` | `06_route.enc` | 不加 filler 就出 GDS ⇒ Calibre well/diff 规则上万条；`editDelete` 会把设计状态打回「未布线」（之后 `timeDesign` 报 `must be routed before running extractRC`） |
| `07_export` | 交 DEF/网表/GDS | `defOut -floorplan -netlist -routing`；`saveNetlist`；`streamOut ... -mapFile <GDSMAP> -merge <库 GDS 列表>` | `deliv/<TOP>.{def,pnr.v,gds}` | **层映射必须是 `<GDSMAP>` 四列**，用 Astro 三列 `gds.map` 会静默丢掉全部布线与 via |
| `08_export_lvs` | LVS 网表 + 兜底 SPEF | `saveNetlist ... -includePowerGround -excludeLeafCell`；`extractRC` + `rcOut -spef` | `deliv/<TOP>.lvs.v`、`*_lefrc.spef` | `07` 的普通 `saveNetlist` **不带 VDD/VSS**（LVS 判所有 cell 电源脚悬空 → 上万条 ERC） |

---

## 三、floorplan 的几何关系（全部从 config.tcl 推）

| 量 | 关系式 | 说明 |
|---|---|---|
| die | `<DIE_UM>` | 由用户/面积硬约束给定，链条里任何一步都不许改 |
| pad 环外沿距 die 边 | `<PAD_OFF>` | 与密封环位置、BORDER 余量共同决定 |
| pad 深度 / pitch | `<PAD_H>` / `<PAD_PITCH>` | LEF 尺寸（**库常量**，从 LEF 直接读，换库重查） |
| pad 内沿 → core | `<GAP>` | |
| floorPlan 的四个 margin | `<IO2CORE> = <PAD_OFF> + <GAP>` | **不是到 die 的距离**，是 core 到 pad 环的距离 |
| core 左下角 | `<IO2CORE> + <PAD_H>` | config 的 `cx0` |
| core 边长 | `<CORE_W> = <DIE_UM> − 2 × (<IO2CORE> + <PAD_H>)` | 交叉核对：`get_db designs .boundary` 与配置差 <3 µm 即算对 |
| site / row | `<CORE_SITE>`（如 0.66 × 5.04，**库常量**，从 LEF 读） | `cutRow` 挖掉宏底下后行数会变多（断成更多段）；断言「宏底下 0 段 row」 |

**floorplan 六条硬动作：**

1. `deleteRow -all` —— 不吃掉 `init_design` 的默认 floorplan，后面全是 resize。
2. `floorPlan -site ... -noSnapToGrid -s ...` —— 不加 `-noSnapToGrid` 会被 track 对齐挪位（几十 nm 级，足以让 pad 与环缝对不齐）。
3. 宏 `placeInstance ... -fixed` + `createPlaceBlockage -type soft`（halo `<HALO>`）—— **footprint 要按旋转后的宽高填**（R90/R270 交换 w/h），别拿 LEF 原尺寸。
4. `cutRow -area <宏 bbox> -keepCell` —— 不带 `-keepCell` 会把 fixed 宏 unplace（真实踩过）。
5. pad 朝向按「键合焊盘朝 die 外、信号/电源脚朝 core」：下 R0 / 上 R180 / 右 R90 / 左 R270。
6. IO filler 宽度分档、按剩余空隙选；放完必须断言 pad 环无 overlap、无 gap。

---

## 四、PG：环 + stripe + 宏 strap

**顺序**：先宏 strap（宏的电源入口），再 core 内 stripe，最后 sroute 补连接。

- **电源环**：`addRing -nets {VDD VSS} -type core_rings -follow core -layer {top METAL5 bottom METAL5 left METAL4 right METAL4} -width <RING_W> -spacing <RING_SP> -offset <RING_OFFSET>`。左右 M4 / 上下 M5，落在 pad 内沿与 core 之间的边带里；corner via 由工具自动补。
- **竖向 stripe**：`addStripe -nets {VDD VSS} -layer METAL4 -direction vertical -width <STRIPE_W> -spacing <STRIPE_SP> -set_to_set_distance <STRIPE_PITCH> -start_from left -start_offset <STRIPE_OFF> -stop_offset 0 -area <core box>`。严格限制在 core 内，伸到环的那段由 `sroute floatingStripe` 补。
- **宏 PG strap 的宽度不是随便定的**：同一侧 VDD/VSS strap 的中心距由 SRAM pin ring 自身间距决定（库常量）；deck 规则（如 `M4_11_V234`/`M3_11_V234`）要求「带 ≥3 层堆叠 via array 的金属间距 ≥ 3.0 µm」⇒ **宽度上限 ≈ 中心距 − 3.0**。按 core stripe 的宽度去做宏 strap 会爆几百条间距违例。本项目最终值见 90。
- 配 `setAddStripeMode -stacked_via_bottom_layer ... -stacked_via_top_layer ...`（只打需要的 via 层）；普通 stripe/strap 一律 `setAddStripeMode -skip_via_on_pin {block standardcell}`（自动 ViaGen 往宏里打 via 会撞 obs）。
- **sroute 分五趟**：`floatingStripe` → `followpin` → `blockpin` → `padPin`(VDD) → `padPin`(VSS)。每趟都记 open 数，**任何一趟 open 不为 0 都要先查清**（个别 followpin open 可能是已知无害基线，要写进报告）。
- 若 sroute 在某个坐标向下扎到不该去的地方，用一个小的 `createRouteBlk` 禁区挡掉，**sroute 跑完立刻 `deleteRouteBlk` 还资源**；禁区坐标要**跟随 core 原点**（用 `cx0` 偏移），die 尺寸一变硬编码就摆错。
- PG 验收三件套：`verifyConnectivity -type special`（open/dangling）、纯 PG 的 DRC、`verifyPowerVia`。

---

## 五、CTS：ccopt 设置与「传播时钟」

1. 先把约束模式换成 **P&R 专用 SDC**（去掉 `set_dont_touch_network [get_clocks ...]` / `set_ideal_network [get_ports clock]`），脚本里再兜底 `reset_ideal_network` / `set_dont_touch ... false`。
2. `create_ccopt_clock_tree_spec -file reports/05_cts/ctspec.tcl` 后 `source` 进来。
3. **显式指定缓冲单元**：`set_ccopt_property buffer_cells <CLKBUF 列表>`。本库只有 CLKBUF 没有 CLKINV（10 种驱动档），不指定 CCOpt 会拿普通 BUF 建树。
4. `target_max_trans` 要按**库的最小 transition 量级**给（180 nm 库约 0.671 ns，CCOpt 默认 0.15 不现实）；`target_skew` 按设计给。
5. **结果判据不是 buffer 数**：脚本闸门可能写「buffer 数 > N」，但真正的判据是 ccopt 的 **slew/cap 违例**（本项目修前 226 slew → 修后 1）。而且**脚本头注释与代码里的阈值常常不一致，以代码为准**。
6. **`set_propagated_clock [all_clocks]` 必须放在 `optDesign -postCTS` 之前**，此后所有 `timeDesign` 与后续步骤都是 propagated clock 的真实延迟。
   ⚠️ 后续步骤不会重复敲这条，靠它存在 DB 里；**如果从 `04_place` 直接起跑 route，签核仍是 ideal clock**，必须自己补一句。

---

## 六、hold fix 到底干了什么

1. 补 ff 视图：`create_library_set libs_ff`（std cell ff + IO 快角 + 所有 SRAM ff）→ `dc_ff` → `view_ff`；`set_analysis_view -setup {view_ss} -hold {view_ff}`（setup 签慢角，hold 签快角）。
2. `setAnalysisMode -analysisType onChipVariation -cppr both`；`setOptMode -holdFixingCells {<延迟单元列表>}`（**本库的命名要自己查**，老库可能没有 `<DEL_CELL>` 这种小档（延迟单元名从库 cell 列表查））。
3. **先删 filler**（`deleteFiller`），否则 density=100%、一个 buffer 都塞不进。
4. 迭代 `optDesign -postRoute -hold` 最多 N 轮，每轮用 `get_timing_paths -delay_type min` 读最差 slack 判断是否收敛。
5. **补回 filler** + `globalNetConnect` 重绑 PG，然后复检：setup ss / hold ff 的 `WNS/TNS/Viol`、`verify_drc 0`、`verifyConnectivity -type regular 0 problem`。

---

## 七、布线

- 全脚本显式的 NanoRoute 设置通常只有两条：`setNanoRouteMode -routeWithTimingDriven true`、`setNanoRouteMode -drouteUseMultiCutViaEffort high`。天线 ECO 阶段临时关时序驱动时用 `-routeWithTimingDriven false -drouteFixAntenna false -drouteEndIteration <N>`。
- **45° 布线：脚本里没有就是没有**。别把工具默认行为写成「我们开了 45°」；要确认只能在 DB 里看实际走线角度。
- 顺序：**routeDesign → `optDesign -postRoute`（需先 `setAnalysisMode -analysisType onChipVariation -cppr both`，否则 `IMPOPT-6080`）→ addFiller → 修 DRC → 再 timeDesign**。`ecoRoute -fix_drc` 要设轮数上限 + 硬门 `ndrc == 0`。
- **filler 计数用 `llength [dbGet top.insts]` − `sizeof_collection [get_cells -hierarchical *]`**：filler 是 physical-only，`get_cells` 抓不到。

---

## 八、导出（`07` / `08`）

- `07_export` 源 DB 自动选「有 `06b_hold_fix` 就用，否则 `06_route`」：`defOut -floorplan -netlist -routing`、`saveNetlist`、`streamOut -mapFile <GDSMAP> -structureName <TOP> -libName ... -merge <标准单元/IO/SRAM 五份库 GDS>`。**不 merge 库 GDS ⇒ Calibre 报 `Cell XXX is referenced but not defined`。**
- 🔴 **`<GDSMAP>` 必须是四列 `layerObjName layerObjType layerNumber dataType`**。三列时 Innovus 报 `IMPOGDS-1556/1557` 后**静默丢弃全部布线/电源/via**：GDS 明显变小、**via 引用 0**。四列格式必须显式列出 `NET`、`SPNET`、`VIA`、`DIEAREA` 四类对象，且 via 的上下层金属要同时出现在 `* VIA` 行，否则会引出一堆 `V*_3_4` 包覆/最小面积违例。
- **名字层通常是故意注释掉的**（`#NAME METAL1/NET 61 1`）：几万条 net name 会被 Calibre LVS 当成 port。顶层 pin label 另用脚本单独加。
- `08_export_lvs` 在**同一个 DB** 上重导带电源地的网表：`saveNetlist ... -includePowerGround -excludeLeafCell`，并断言「VDD/VSS 各出现 >1000 次」「`^module` 只有 1 个（只剩顶层）」。顺带 `extractRC` + `rcOut -spef` 只作兜底；**签核 SPEF 必须用 StarRC + tluplus/nxtgrd**（见 50）。

---

## 九、换项目改哪几行

| 要改的东西 | 在哪 |
|---|---|
| die / core / pad / ring / stripe / 宏表 / pad 表 | `innovus/flow/config.tcl`（唯一源） |
| 库路径、GDS map、merge 列表 | `config.tcl` 的 `::GDSMAP` / `::GDS_MERGE` |
| SRAM 实例名与旋转方向 | `config.tcl` 宏表（footprint 用旋转后尺寸） |
| P&R SDC 路径 | `config.tcl` 的 `::SDC`（用 `_pr` 版） |
| hold fixing cell 列表 | `06b_hold_fix.tcl`（查库有哪些延迟单元） |
| CLKBUF 列表 / target_max_trans | `05_cts.tcl` |
| 各步 pass 断言 | `lib.tcl` 的 `fl::need` 与本步脚本末尾 |
| 步骤清单 | `flow/run.sh` 的参数顺序 |
