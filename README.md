# SMIC 0.18 µm 数字后端 → MPW tapeout skill

一个给 AI agent 用的 **skill**（也可以当人读的 checklist）：从 RTL 综合一路做到可交给 foundry 的 MPW 交付包，
其中 **1P5M → 1P6M 金属层选项迁移**这一段，把 **DRC / 天线 / LVS 全部打到零**。

不是教程合集，而是**判据 + 坑 + 验收闸门**：每条都写清「怎么判过、拿什么证明、错了长什么样」。

## 它包含什么

| 阶段 | 文件 | 内容 |
|---|---|---|
| 入口 | `SKILL.md` | 变量表 → 硬规则 → 流程图 → 每次改版图必跑的验收清单 → 30 条坑速查 → 签核口径 |
| 前端 | `references/00-frontend.md` | 库/约束准备、DC 综合、LEC 两段、要不要做 DFT |
| P&R | `references/10-pnr.md` | Innovus 九步线性流程（floorplan → PG → place → CTS → hold → route → export） |
| 版图链 | `references/20-layout-chain-1p6m.md` | 1P6M 八步：pad swap / 通孔缩孔 / 电源标签 / 密封环 / 烘焙 via / DMF / 合并填充 / marker |
| DRC | `references/30-drc-debug.md` | 现象 → 根因 → 对策六个案例 + waiver 口径 |
| LVS | `references/40-lvs-debug.md` | 大小写、烘焙、stub、转义标识符、FP 分岛的十步顺序 |
| 天线 / RC / PT | `references/50-ant-rc-pt.md` | 逐版搜索法 + tluplus→nxtgrd→StarRC→PT 五角 + 降频决策 |
| 交付 | `references/60-deliverable.md` | 打包、MPW 表格字段、回读验收、数字绑定纪律 |
| 放行闸门 | `references/85-signoff-gate.md` | 八类签核逐项清单 + 四种状态 + waiver 治理 + 记录模板 |
| 外部依据 | `references/70-external-practice.md` | 行业通行做法 / 公开资料要点（23 条结论、31 个来源，标注来源等级） |
| 同类对照 | `references/80-prior-art.md` | 与业界同类 agent skill 的差距表 + 本 skill 的空洞 |
| 案例数据 | `references/90-case-study.md` | 一份量级化的真实流片数据（只作对照，不指向具体项目） |

## 怎么用

**给 agent 用**：把整个目录放进它的 skill 目录即可，例如

```bash
# DeepSeek Harness
cp -r smic180-1p6m-tapeout ~/.dsh/skills/
# Claude Code / 兼容 agents
cp -r smic180-1p6m-tapeout ~/.claude/skills/
```

或者最简单：直接让 agent「读 `SKILL.md`，按它做」。

**给人用**：`SKILL.md` 的 §4 验收清单和 §5 坑速查可以先当 checklist 抄进报告。

## 用之前必须做的三件事

1. **填 `SKILL.md` §0 的两张表**（工程变量 + 工艺常量）——不填，命令全是占位符。
2. **把工艺常量换成你自己的**：本文**主要**用占位符写层号、通孔尺寸、厚度/方阻、DRM 上限；
   示例里保留的少数数值只用于说明量级。照抄别人的工艺数字是事故的常见起点，
   **一律查你手上 PDK 的 LAYER MAP / ITF / DRM**。
3. **改 deck 的 `LAYOUT PATH` 再跑**——本 skill 反复强调这条，因为「DRC 0 条」跑在旧 GDS 上是最常见的事故。

## 适用边界

- 适用：0.18 µm 级 CMOS 数字后端 + MPW/Shuttle 流片，尤其是**金属层选项迁移**与**签核清零**。
- 不适用：纯模拟/混合信号版图、只到 RTL 的前端工作、别的 foundry 的 deck（规则名与豁免机制全不同）。
- 已知空洞（文档里如实标注，不掩饰）：IR-drop / EM / SI、DFT / ATPG、MCMM mode 交叉、UPF、LEC-A 闭环。
  这些没做就是没做，`references/85-signoff-gate.md` 把它们列为 `NOT_MEASURED`。

## 关于数据

`references/90-case-study.md` 的数值做了量级化与区间化处理：去掉项目名、公司名、主机、路径、
文件哈希与日期，几何/计数/时序只保留能说明方法的量级。文中**不包含**任何 PDK 文件、Calibre deck 原文或厂商手册内容。

## 许可（按内容类型分许可，不是对同一作品二选一）

| 内容 | 许可 | 文本 |
|---|---|---|
| 文档正文（`SKILL.md`、`README.md`、`references/*.md`） | CC BY 4.0 | [`LICENSE`](LICENSE) |
| 文档里的命令、脚本与代码块片段 | MIT | [`LICENSE-MIT`](LICENSE-MIT) |

- 转载/改编文档正文：请保留署名「© 2026 cannotgetaname」+ 来源链接，并注明是否修改（CC BY 4.0 §3(a)）。
- 复用代码片段：请保留 `LICENSE-MIT` 中的版权与许可声明。
- 两份许可文本本身不适用上述许可（CC BY 4.0 正文本身为 CC0）。

推荐引用：`cannotgetaname. (2026). smic180-1p6m-tapeout: SMIC 0.18 µm 数字后端到 MPW tapeout. https://github.com/cannotgetaname/smic180-1p6m-tapeout . CC BY 4.0.`

## 版权与署名

© 2026 **cannotgetaname**（<https://github.com/cannotgetaname>）。

`LICENSE-MIT` 第 3 行与 `CITATION.cff` 已填同一个主体；若换仓库/换账号，这三处一起改。

## 商标与隶属

SMIC / 中芯国际、Calibre（Siemens EDA）、Innovus / Virtuoso / Tempus（Cadence）、
PrimeTime / StarRC / Formality / Design Compiler（Synopsys）、sky130 / SkyWater 等名称与标识
是其各自权利人的商标或注册商标。本文仅在**指称意义上**使用（nominative use），
不表示任何隶属、赞助、审阅或背书；本仓库不是上述公司的官方文档，也未由其审阅。
文中提到的 PDK、DRM、deck 与 EDA 工具权利均归各自权利人。

## 第三方内容

`references/70-external-practice.md` 与 `references/80-prior-art.md` 是对公开网页、厂商文档与开源项目的
**摘要与转述**（附原文链接），未转载原文；观点与数据归原作者/机构。详见 [`NOTICE.md`](NOTICE.md)。

## 免责

流程与判据仅供参考。**签核结论必须由你本人在自己的 PDK 版本、自己的报告上得出**——
本 skill 提供的是「该查什么、怎么算过、错了长什么样」，不是「照抄这个数字就能流片」。
