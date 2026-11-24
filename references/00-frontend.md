# 00 · 前端：库准备 / SDC / DC 综合 / LEC / DFT

> **什么时候读我**：从 RTL 起步，要准备库与约束、跑 Design Compiler 综合、做 Formality LEC、或判断这个项目要不要做 DFT 时。

> 约定：正文用 `<PROJ>` / `<PDK>` / `<ENV_*>` 占位（见 SKILL.md §0），本项目实测值一律在 `90-case-study.md`。
> 纪律：数字只来自脚本 / 报告 / 交付文档；源文件里没有的写「源文件未见，需向设计者确认」，不拿业界惯例冒充本项目事实；同一个量在不同文件对不上时，注明以哪份为准。
> 本 PDK 的库名：std cell `<STDCELL_LIB>`、IO `<IO_LIB>` —— 名字、电压档、corner 名随 PDK 变，换工艺全部重查。

---

## 一、库清单与来源

后端只认 4 类库 + 1 套 RC tech。**开工第一件事是把每一类的「该有什么」和「怎么验」列出来**，缺一样后面全崩。

| 目录 | 是什么 | 该有什么（检查点） |
|---|---|---|
| `lib/stdcell` | 标准单元（**可能只有单阈值档**，本项目只有 RVT，无 HVT/LVT） | `liberty/<电压>/`、`lef/{macro,tf}`、`gds`、`verilog`（`.v`/`_pg.v`/`_neg.v`）、`fastscan/*.atpg`、`astro/tluplus/<金属栈>`、`cell_list` |
| `lib/io` | IO pad 库（电压和 core 不同，本项目 core 1.8 V / IO 3.3 V） | `syn/<电压>/*.{db,lib}`、`lef/*_{5MT,6MT}.lef`、`gds`、`lvs`、`ibis`、`atpg` |
| `lib/sram` | memory compiler 宏（每个宏一套 LEF/GDS/V/CDL/MBIST/PDF + 多 corner `.lib`） | 宏数量要和网表里 `ref_name` 对得上；`.lib` 的 corner 覆盖要够 DC/PT 用 |
| `rc/` | RC tech | `<栈>.tf` + 每个角一份 `*.tluplus`（注意 `grd_ver`，见 50）+ StarRC cmd/SPEF 输出目录 |

**四条会反复踩的库坑：**

1. **corner 数对不上**：库里可能给了 5 套 `.db/.lib`，但综合脚本的 `CORNER` 只暴露 3 档（ss/tt/ff）且**标签写死**。多出来的角（如 `tt_85c`、`ff_0c`）要自己改脚本才能用，后仿 / PT 侧另算。**别以为「库里有 = 流程能用」。**
2. **符号链接**：README 写 `lib/stdcell -> <真实目录>`，但 sftp / 同步视图里链接常被**展开成实体目录**（`readlink -f` 指回自身）。换机器时按 `scripts/env.csh` 的 `$STDCELL/$IOLIB` 手工重建——重建命令仓库里常常没有。
3. **SRAM 必须吃预转 `.db`**：本机 DC 进程内 `read_lib` 会失败（`LCSH-3: Library Compiler`），所以综合脚本默认指向一个**预先用 `lc_shell` 转好的 `.db` 目录**（环境变量可覆盖），找不到才退回 `.lib`。
4. **IO 库只能用厂商 `.db`**：历史 bug 是 IO 库从来没被加载过、pad 在网表里是空壳——这种错综合不报错，到 P&R 才炸。

**金属栈选择**：`lib/stdcell/astro/tluplus/` 里通常有 1P3M~1P6M 全栈，但 **IO LEF 只有 5MT/6MT** ⇒ 实际候选只有 5 层或 6 层。早期阶段用什么栈、出片用什么栈要显式记录（本项目早期 1P5M、出片 1P6M）。`rc/README.md` 的「默认 5M」只对早期成立。

---

## 二、SDC 约束体系：一条基线 + 一批「只改 `create_clock`」的派生版

**做法**：维护**一条基线**，每个新频率都「复制上一版、只改 `create_clock`」，文件名带频率/周期。这样频率演进可追溯，签核 SDC 与 P&R SDC 可以分开管理。

派生版的清单要能回答三个问题：谁复制谁、`period`/`waveform` 实际值、用在哪一步（DC / Innovus / PT）。

**四条硬规矩：**

- **改 period 必须同时改 waveform**。只改 period 会让高电平时间不动，占空比漂移（本项目 25 ns 那版就是高电平 14.25 ns，占空比 57%）。
- **注释会骗人**：头注释写「XX MHz 版」而 `create_clock` 实际是另一个周期的情况真实存在（本项目案例见 90）。**以代码行为准**，注释只当线索。
- **注释与代码两条都「对」却互相打脸时，先判断哪个是过期的那条**：改频率时只改 period 不改 waveform、复制上一版却忘了改头注释，都是常见来源。
- **P&R 专用 SDC ≠ DC 基线**：P&R 版通常要去掉 `set_dont_touch_network [get_clocks ...]` 与 `set_ideal_network [get_ports clock]`，否则 CCOpt 拒绝插缓冲（`IMPCCOPT-2314`）。本项目就有一份 `_pr.sdc` 专门干这个。
- **签核 SDC 单独一份**，并在报告里写清用的是哪份。

**核心约束项（每一项都要有明确取舍理由）：**

| 项 | 典型写法 | 判断要点 |
|---|---|---|
| 时钟 | `create_clock -period <P> -waveform {0 P/2} [get_ports clock]` | 单时钟 / 多时钟？`clk_o` 只是引出给外部器件 |
| uncertainty | `set_clock_uncertainty -setup / -hold` | 值太松会签过假时序 |
| transition | `set_clock_transition -max / -min` | 与库最小 transition 量级对齐（见 CTS 的 `target_max_trans`） |
| 复位 | `set_false_path -from [get_ports reset]` + `set_ideal_network` | **只在线性同步器之后安全**；不要在同步器输出上加 false_path/dont_touch |
| 准静态脚 | `set_false_path -from/to ...`（配置脚、输出时钟） | 逐个说明为什么是准静态 |
| IO | `set_input_delay/-output_delay -max/-min`、`set_load` | 注释写 5 pF、代码写 10.0 这种不一致，**以代码为准** |
| 硬核 | `set_dont_touch [get_cells -hierarchical -filter {ref_name =~ <SRAM>}]` | 按 `ref_name` 过滤，别按实例名 |

**两条命门（本项目真实事故）：**

1. **SDC 必须 `source`，不能 `read_sdc`**。`read_sdc` 只认纯 SDC 命令，`set_dont_touch` / `get_cells -filter` / `remove_from_collection` 会被**静默跳过**（PT 里表现为 CMD-005/CMD-010 后中止解析，后半个文件的 IO delay / false_path / set_load 全丢）。
2. **拼写错的约束永远是「生效 0 条」**：本项目 `set_inputs_no_clk` 少个空格 → 输入延迟**历史上一直没生效**，修掉后 core 面积涨约 1%。修完要能看到面积/时序变化，没变化说明还是没生效。

**用降频买 slack 的前提**：先确认关键路径**恒定**且落在某个不可优化的模块里（案例是一段寄存器到 SRAM 宏的路径，与逻辑综合无关）。这样 slack 随周期线性增长，降频有效。否则先修逻辑。

---

## 三、DC 综合

**跑法**：一个 `run_dc.sh` 包装 + 一个 `syn_dc.tcl` 主脚本；环境靠 `<ENV_SY>` + `scripts/env.csh`。

```csh
source <ENV_SY>
source <PROJ>/scripts/env.csh
cd <PROJ>
bash dc/scripts/run_dc.sh                     # 正式综合（ss 角）
DRY_RUN=1 bash dc/scripts/run_dc.sh           # 只打印将执行的命令/库路径，不启动 DC
DC_INCR_OPT=1 DC_CRITICAL_RANGE=1.5 DC_MACRO_MAX_TR=0.4 bash dc/scripts/run_dc.sh   # 收边缘违例（实测必开）
DC_DFT=1 DC_SYN_SUFFIX=_dft bash dc/scripts/run_dc.sh                               # 带扫描链（见第五节）
```

**主干步骤（按脚本出现顺序，坑都在这里）：**

1. `file delete -force $SVF` → `set_svf $SVF`：**必须先删再 set**。用 `-append` 会让同一文件叠两份互相冲突的引导，Formality 直接 FAILED。
2. `target_library` = 标准单元 ss 角 `.db`；`link_library = "*" + stdcell + 每个 SRAM + IO`。
3. 宏/包含路径从 `rtl/defines.f`、`rtl/incdirs.f` 读，**不写死**；综合再补 `SYNTHESIS` define。
4. 读 RTL：**逐个 `analyze -format sverilog -define <defs>`，不能用 `read_file`**。工程里有 SV interface 时 `read_file` 不生成 design template，消费方全报 `ELAB-385`。全部读完再 `elaborate <TOP>` → `link` → `check_design`。
5. `set_dont_touch`：**所有 pad 类型** + **SRAM 包装里的时钟门控单元**。漏掉后者的后果是 DC 把门控优化成 `PULL1 clk_en` + `PULL0 CEN`，PTPX 按 SAIF 报出假功耗。
6. `source <sdc>`（可用环境变量覆盖）。
7. 时钟门控：`set_clock_gating_style -sequential_cell latch -control_point before -minimum_bitwidth 3 -max_fanout 32`；给 LEC 用可关（`DC_NO_GATE_CLOCK=1`）。
8. **`set_max_area` 不要写 0**（会制造永远存在的假违例，写一个真实上界）；`set_critical_range`；SRAM 引脚 `max_transition`；`set_host_options -max_cores`。
9. `compile_ultra -no_autoungroup -gate_clock`；增量收一轮 `compile_ultra -incremental ...`（注意 Tcl 展开写法，拼错就变成两个非法选项）。
10. `ungroup -flatten tc_io_*` 之类的 **pad 包装层**：P&R 摆 pad ring 需要顶层就是真 pad cell。
11. 报告 + 写盘。

**输出物**：各角 `dc_{ss,tt,ff}.log`、`dc_{area,qor,timing_max,timing_min,violators,clock,clock_gating,power,check_design}.rpt`、`<TOP>.syn.sdc/.sdf`、`dc/netlist/*.v` + `.ddc`、`dc/svf/*.svf`。实测值（面积 / critical path / ICG 数 / 残余 min_cap）见 90。

**SVF 是什么、给谁用**：`set_svf` 让 DC 边综合边记录它做的等价变换（寄存器重命名/复制、常量传播、时钟门控替换、边界优化）。Formality 读它才不会把这些当不等价；**没有 SVF 的 LEC 基本必挂**。它只服务 LEC，P&R 不读。

---

## 四、LEC 形式验证：分两段，脚本自己生成 `fm_shell` 脚本

| 段 | 比什么 | 需要 SVF | 目录约定（r=参考, i=实现） |
|---|---|---|---|
| **LEC-A** | RTL ↔ 综合网表 | **要** | r=RTL，i=`dc/netlist/*.v` |
| **LEC-B** | 综合网表 ↔ 布线后网表 | 不要（两边都是门级） | r=综合网表，i=交付网表 |

```csh
# LEC-A
csh -c "source <ENV_SY> >& /dev/null; cd <PROJ> && bash dc/scripts/run_lec.sh --netlist <syn.v> --svf <syn.svf>"
# LEC-B
csh -c "source <ENV_SY> >& /dev/null; cd <PROJ> && bash dc/scripts/run_lec_n2n.sh"
```

**脚本内部固定动作（照做，别省）：**

- 在 `/tmp` 下生成 `lec.tcl`：`set_svf`、`read_db`（std cell + 所有 SRAM + IO）、`read_sverilog -container r` 读 RTL、`read_verilog -netlist` 读实现。
  **一律 `read_sverilog`**：本工程 `.v` 里也有 `always_comb`，`read_verilog` 会 `FMR_VLOG-481` 直接挂。
- 两侧 pad 与 SRAM 宏 `set_black_box`；`reset` 常 0。
- **`match` 之后才 `set_dont_verify_points`**（`scan_out_o`、`*clk_gate*`、`*Memory_reg*`），顺序反了会漏点。
- `verify` 后用 grep `Verification SUCCEEDED/FAILED` 定 exit code；脚本自检 license（`lmstat`），挂了用 `lmgrd` 拉起。

**结论必须分开写，别混：**

- **LEC-B 通过** = 「Innovus 的 place/CTS/opt 没改逻辑」，这是**可交付的**结论。
- **LEC-A 未关闭** = 「综合网表逻辑未证」。常见根因是**工具版本引导不兼容**（老版 Formality 拒绝接受新版 DC 的 `reg_merging` / `guide_replace` 引导）+ 一批无复位初值寄存器被判 `Constrained 0X`。正确写法是「未证，需 waiver 或升级工具后用现有网表/SVF 重跑」，**不能照抄工程 README 里更早一轮的 `Verification SUCCEEDED (N/N)`**——那通常没有日志留存。

---

## 五、DFT 现状与「要不要做」的判断

**先分清三件事：**

1. **库侧模型齐不齐**：std cell 与 IO 都有 `.atpg` 模型 ⇒ 缺的可能只是**工具**（服务器上没有 ATPG 可执行），不是库。
2. **DC 侧能不能插链**：`set_scan_configuration -chain_count N -style multiplexed_flip_flop` + 端口 `scan_en/scan_in/scan_out`（顶层可能是 `*_i/_o` 命名，脚本用 `_pick_port` 自适应），产物 `*_dft.v` / `*_dft.svf` / scan 报告。
3. **插完链必须补 `compile_ultra -incremental`**：案例里不补直接崩到 −200 ns 量级 / 近万条违例；补完恢复 0 违例。

**本项目为什么最终没做 DFT（可复用的判断）**：链头会落在 **pad 外侧 bond 节点、绕过 3.3V→1.8V 输入缓冲器**，这条链不可用。正确做法是把 RTL 拆成 `chipTop`（pads）/ `coreTop`（纯逻辑）再在 `coreTop` 上做 DFT —— 那是**结构性重构**。所以当时的决定是整体不做：从 `<TOP>.syn.v` 派生 `<TOP>_pnr_nodft.v`，删掉 DFT pad 实例与顶层端口，**全文 diff 只有 4 处 14 行、无功能改动**，P&R 全流程重跑。

⇒ 交付网表要**实测** `SD*` 扫描触发器个数（grep），为 0 就写「未做 ATPG、出片网表没有扫描链、扫描覆盖率无法测量」。遗留的 `*_dft.v` 不进 P&R。

---

## 六、前端交给后端的接口（谁读哪个文件）

| 交付物 | 谁吃 / 怎么吃 |
|---|---|
| 综合网表（送 P&R） | Innovus `config.tcl` 的网表变量；`00_init.tcl` 用 `init_design` 读，`init_design_netlisttype VERILOG` |
| 签核网表（含全部 pad） | 对照 / 派生母本 |
| 约束 | P&R 用 `_pr.sdc`（`create_constraint_mode -sdc_files`）；DC 用基线；PT 用签核版。**别拿 `dc/reports/*.syn.sdc` 当 P&R 约束**（那是 DC 回写的综合后 sdc） |
| SVF | 只给 Formality（LEC-A） |
| 文件列表 | DC `analyze` 与 Formality `read_sverilog` 共用同一份，保证读同一 RTL |
| 面积/时序/QoR | 前端自检 + floorplan 的初始面积依据 |
| SDF | 门级后仿（PT 侧另有签核 SDF） |
| 网表差异证明 | 说明 `syn.v → _pnr_nodft.v` 只删 DFT pad；LEC-B 的 REF 就是它 |

**复跑顺序**：`DRY_RUN=1 run_dc.sh` 对路径 → `run_dc.sh` → `run_lec.sh`（LEC-A）→ 派生 `_pnr_nodft.v` → P&R → `run_lec_n2n.sh`（LEC-B）。

---

## 七、换项目改哪几行

| 要改的东西 | 在哪 |
|---|---|
| PDK 根 / 库目录 / 金属栈 | `scripts/env.csh`（`PDK`、`STDCELL`、`IOLIB`、`METAL_STACK`、`STACK_TAG`） |
| corner 暴露的档数 + 写死的标签 | `syn_dc.tcl` 的 `CORNER` / `STD_TAG` |
| SRAM `.db` 预转目录 | `run_dc.sh` 的 `<SRAM_DB_DIR>`（或等价环境变量） |
| RTL 文件/宏/包含路径 | `rtl/{filelist,defines,incdirs}.f` |
| SDC 基线 + P&R 派生版 + 签核版 | `sdc/` 三处 + `config.tcl` 的 `::SDC` |
| pad 类型清单（`set_dont_touch`） | `syn_dc.tcl` 的 7 类 pad 列表 |
| LEC 的 black box / dont_verify 清单 | `dc/scripts/run_lec*.sh` |
| 是否需要 DFT、拆不拆 chipTop/coreTop | 设计决策，先看链头是否绕过输入缓冲器 |
