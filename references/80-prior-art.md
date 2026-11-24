# 80 · 同类 skill 调研与差距分析（外部资料）

> **什么时候读我**：想知道本 skill 与业界同类 agent skill 的差别、要给别人解释"为什么这么设计"、
> 或者想补上本 skill 目前没覆盖的签核维度（IR/EM、DFT、waiver 治理、证据规范）时。
> 本文只记录外部调研结论，不含本项目实测数字（那些在 90）。

调研范围：用 web_search 检索 "agent skill / Claude skill + EDA / tapeout / physical design"，
逐个打开读原文。下面只列**真实存在且读到了正文**的。

---

## 一、找到的同类 skill

### 1. `vibeic/vibe-ic` → `tapeout-checklist`（最接近本 skill 的定位）

- 仓库：<https://github.com/vibeic/vibe-ic/tree/main/vibe-ic-marketplace/plugins/vibe-ic/skills/tapeout-checklist>
- 目录页：<https://skills.rest/skill/tapeout-checklist-vibeic>
- 依据其公开的 `SKILL.md` 摘要（转述，非转载）。

**它的做法**：不做流程，只做**放行闸门**（pre-tapeout gate）。八个类别：
functional（回归/形式/LEC）、timing（MCMM STA）、power integrity（IR-drop/EM）、
physical（DRC/LVS/antenna/density/ERC）、DFT coverage、power intent（UPF）、
documentation、GDS release（含 checksum）。

**它比我们强的 5 个概念（已吸收进 SKILL.md §六）**：

1. **`NOT_MEASURED` 是一等公民**：报告缺失/没看的版图 = `NOT_MEASURED`，
   不能因为"文件存在""有签名""哈希对得上"就升成 PASS。**这条比我们"0 条才算过"更严格**。
2. **waiver 治理 schema**：每个 waiver 必须有 ID / 理由 / ticket / **具名人类审批人**，
   **AI 自批或自评一律拒收**；`PASS_WITH_WAIVERS` 不算 tapeout ready。
3. **GDS 不能用 md5 直接比对**：GDSII 的 `BGNLIB`/`BGNSTR` 会写入时间戳，
   同一版图两次 streamOut 的 md5 必然不同 → 要先把日期字段清零再做 digest
   （它给了 `gds_canonical_digest.py` 这个思路）。**我们之前只写"记下 md5"，漏了这个警告。**
4. **release manifest 结构**：`release_manifest.json` + `SHA256SUMS` + 相对 `archive/` 树，
   并把"最终 GDS / 内部 STA-SDC-SPEF / 外部 PDK 依赖 / 供应商上传子集"分开列。
5. **die 的定义**：die 是 **PDK 密封环框**，不是"所有层的 box"。它专门写了一个真实案例：
   SKY130 的 `advSeal_6um_gen` 会把角标 octagon 画到 die 外面，
   于是"版图 bbox > 申报 die"其实是正常现象——判据应该是**"密封环那一步实测出的框"**。

### 2. `hdl-tools/digital-chip-design-agents`（覆盖面最广的 marketplace）

- 仓库：<https://github.com/hdl-tools/digital-chip-design-agents>
- npm 包：<https://app.unpkg.com/digital-chip-design-agents@1.9.1/files/README.md>

16 个 plugin / 17 个 skill 文件，15 个领域：architecture、rtl、verification、formal、
synthesis、dft、sta、hls、**pd**、soc、memory-ip、compiler、firmware、fpga…
其中 `chip-design-pd` 就是 floorplan→placement→CTS→routing→sign-off。

**对我们的意义**：它是"按领域切分"的组织方式，我们的 skill 是"按一条真实项目流水线切分"。
两者不冲突：如果要扩展成团队用的 skill 集，可以借它的领域切法。

### 3. `icagenthub.com / chip-design-pd`（中文平台，中科麒芯）

- <https://www.icagenthub.com/skills/chip-design-pd>

定位：PD flow 的**规划与评审**（PDK 分析 → floorplan → placement → CTS → routing →
timing/power/area 优化 → GDS signoff），同时支持开源 OpenROAD/KLayout **和** 商用 Innovus/ICC2/Calibre。

**对我们的意义**：它显式声明"两套工具链都覆盖"。我们的 skill 目前只覆盖商用（Innovus/Calibre），
如果要更通用，可以加一节"开源替代路径"（OpenROAD/OpenLane、KLayout、Netgen）。

### 4. `OpenADA`（证据规范做得最严的 EDA skill）

- <https://raw.githubusercontent.com/simra-tech/OpenADA/main/skills/openada/SKILL.md>

它是"开源 EDA 的 CLI 契约层"：Xschem/ngspice/KLayout DRC/Netgen LVS/Yosys/OpenSTA。

**它比我们强的两个概念（已吸收）**：

1. **`execution.status` 与 `engineering.status` 必须分开**：
   `completed` 只代表进程跑完，**不代表 DRC/LVS 通过**；必须读到 `engineering.status: pass`。
   而且 `unknown`（证据不足）**不许**升格成 pass。
2. **"一次只证明一个断言"**：`openada doctor --assertion <one>` 一次只跑一个最小前置检查，
   不要一上来推荐整条 flow。这条对我们"每改一版只验必要的三件套"有借鉴意义。

### 5. 其他（只看到目录条目，未读到正文）

- `skills.lc` 上的 `semiconductor-chip-design-engineer`（抓取超时）
- `hardware-agency-agents`（PCB/嵌入式向，不相关）

---

## 二、差距表：本 skill vs 同类

| 维度 | 本 skill 现状 | 同类做法 | 处理 |
|---|---|---|---|
| 定位 | **一条完整流水线**（综合→P&R→迁移→签核→交付） | 多数是**单点**（PD 或 checklist） | 保持；这是我们最大差异点 |
| 证据纪律 | 有"每个数字带来源" | `execution/engineering/unknown` 三级 | **已补进 §六** |
| waiver | 有"只能用于 PDK 内部并逐条说明" | 有 schema + 人类审批人 | **已补进 §六** |
| 缺失项判定 | "0 条才算过" | `NOT_MEASURED` 概念 | **已补进 §六** |
| GDS 比对 | 只说"记 md5" | 时间戳陷阱 + canonical digest | **已补进 §六** |
| die 定义 | 有（本项目的 ring/BORDER/die 三个口径之争） | 有（seal frame 模型） | **已补进 §六** |
| **IR-drop / EM** | **完全没有** | 双方都当作必查项 | **本 skill 的已知空洞**，见下 |
| **DFT / ATPG** | 本项目决定不做 | checklist 要求 ≥95% stuck-at | **本 skill 的已知空洞** |
| MCMM | 只做了 5 corner 单模式 | 强调 MCMM | 记为限制 |
| 开源工具链 | 只覆盖商用 | 有的两者都覆盖 | 可选扩展 |

## 三、本 skill 目前明确的空洞（不要假装没有）

1. **没有 IR-drop / EM 检查**：本项目只做了 DRC/LVS/天线/时序。上电完整性没验。
   如果要严格签核，必须补 Voltus/RedHawk 之类，或至少跑 PDN 静态 IR。
2. **没有 DFT/ATPG**：本项目明确决定不插扫描链（见 00-frontend 的 DFT 段），
   所以没有覆盖率。foundry 要求 ≥95% stuck-at 时这条是硬伤。
3. **没有 UPF / 低功耗意图**：设计是单电源域，没有 power intent 文件。
4. **没做 SI/串扰、没做 EM**。
5. **RC 只抽到 M1–M5**（1P6M 的 M6 没进 SPEF），所以时序结论不覆盖 M6。

## 四、是否要照抄它们的结构

**不要。** 它们的坑在于：
- 单点 skill（只做 checklist）**不告诉你版图怎么改**；
- 依赖自己的 CLI/工具链（OpenADA）或自己的 flow engine（vibe-ic 的 `flow_engine.json`），
  **换一个环境就废**；
- 大量篇幅在"证据协议"而非"工程判断"，对一线的帮助有限。

我们的取舍：**保留"一条真实流水线 + 每步的判据 + 真实踩坑"**，
只把它们的**证据纪律**（三级状态、NOT_MEASURED、waiver schema、canonical digest、die 定义）吸收过来。

---

## 五、外部来源清单

- vibe-ic tapeout-checklist：<https://github.com/vibeic/vibe-ic> / <https://skills.rest/skill/tapeout-checklist-vibeic>
- digital-chip-design-agents：<https://github.com/hdl-tools/digital-chip-design-agents> / <https://app.unpkg.com/digital-chip-design-agents@1.9.1/files/README.md>
- IC Agent Hub chip-design-pd：<https://www.icagenthub.com/skills/chip-design-pd>
- OpenADA：<https://github.com/simra-tech/OpenADA> / <https://raw.githubusercontent.com/simra-tech/OpenADA/main/skills/openada/SKILL.md>
- Anthropic 官方 skills 目录（对照格式）：<https://github.com/anthropics/skills>
