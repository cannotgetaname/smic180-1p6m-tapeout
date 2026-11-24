# 85 · 放行闸门（Signoff Gate）：八类签核逐项清单

> **什么时候读我**：GDS 要交给 foundry 前的最后 1~2 周；或要回答"这版能不能交"。
> 本文是**闸门**，不是流程：它不教你怎么改版图，只定义"什么算过、怎么证明、缺项怎么记"。
> 流程细节回 `00~60`；案例数据到哪一步见 `90`；外部依据见 `70`、`80`。

## 0. 用法（三条铁律）

1. **每项只能落四种状态**：`PASS` / `FAIL` / `NOT_MEASURED` / `WAIVED`。
   **没有 `SKIP`，也不许留空。** 文件存在、有签名、哈希对得上，都不能把 `NOT_MEASURED` 变成 `PASS`。
2. **`PASS` 必须能指到"哪份报告、哪个字段、什么值"**。指不出就是 `NOT_MEASURED`。
3. **`WAIVED` 必须有**：`ID / 规则或指标 / 实测值 / 为什么可接受 / 残留风险 / 具名人类审批人`。
   **AI 自批无效**；`PASS_WITH_WAIVERS` ≠ tapeout ready。

放行判据：**八类里没有任何 `FAIL` 与 `NOT_MEASURED`**，`WAIVED` 全部经人工批准并列入风险清单。

---

## 1. Functional（功能）

| 项 | 工具（商用 / 开源） | 判据 | 陷阱 |
|---|---|---|---|
| 回归 | 任意仿真器 / 自有 CI | 目标用例 100% 通过，零新增 fail | 只跑冒烟就当过 |
| 覆盖 | VCS/Xcelium/Questa coverage | 代码/功能/断言覆盖达冻结目标 | 覆盖率达标但 corner 用例没跑 |
| **LEC（RTL↔网表）** | Formality / Conformal（开源：Yosys `eq`、ABC） | `0 failing points`，且 **unmatched/undetermined = 0** | 只报「通过七千多点」却漏掉「近七千点未判定」—— 那不是通过 |
| LEC（网表↔网表） | 同上 | 同上 | 用它冒充 RTL↔网表 |
| 形式 | VC Formal / JasperGold（开源 SymbiYosys） | 属性全 proved；报告要写**假设与证明深度（bounded）** | 把 bounded 证明说成无界证明 |

> 本项目状态见 `90`：LEC-B（网表↔网表）通过；**LEC-A（RTL↔网表）未闭合**，属未证项。

## 2. Timing（MCMM STA）

| 项 | 工具 | 判据 |
|---|---|---|
| setup / hold / recovery / removal / pulse-width / max-trans / max-cap | PrimeTime / Tempus（开源 OpenSTA） | **每个 corner × 每个 mode** 的 WNS ≥ 0 且 NVP = 0 |
| corner 集 | — | 至少 ss/ff/tt（含温度极端），项目有多电压/多模式必须成矩阵 |
| OCV / derate / CPPR | — | 报告里必须能读到实际用的 derate 值与是否开了 CPPR |
| SI（串扰） | PT-SI / Tempus SI | 开 SI 后重跑，delta 要记录；不开 SI 要在报告里声明 |
| 与版图一致性 | — | SPEF 必须来自**最终版图**；版图改过就要重抽 |

**假 0 的三个来源**（都真实发生过）：
- SPEF 是旧版图抽的 → 时序"很好"但没意义；
- SDC 用 `read_sdc` 被截断 → 约束没生效，假通过（要 `source`）；
- 用理想时钟（`set_ideal_network`）当签核 → CTS 后必须 `set_propagated_clock`。

> **RC 抽取边界必须写清**：本次只抽了 M1–M5（映射文件无 METAL6/VIA56），所以结论不覆盖 M6。

## 3. Power Integrity（IR-drop / EM）—— 本项目**没做**

| 项 | 工具（商用 / 开源） | 判据 | 说明 |
|---|---|---|---|
| 静态 IR | Voltus / RedHawk / Ansys（开源 OpenROAD `pdnsim`） | 最差压降 ≤ 预算（常见 **5% VDD**，PDK/业主要求更严时从严） | 必须写清：用哪个 VDD、什么 activity、full-chip 还是 core-only |
| 动态 IR | Voltus / RedHawk | 峰值压降 ≤ 预算，且看 di/dt 场景 | 需要 VCD/FSDB 活动率 |
| EM（信号/电源） | Voltus / RedHawk | `J / Jmax ≤ 1`（DC 与 AC 分开判），含 via 电流 | 电源网格最常超，via array 数量决定成败 |
| PDN 电阻 | 同上或自建 | 从 pad 到最远实例的 R 在预算内 | 与 IR 一起报 |

**这些工具不在本 skill 的流水线里**，本 skill 只做到"把该查的查完"。若要补，最小可做：`pdnsim` 静态 IR（开源可跑）+ 电源网格 via 数量核对。

## 4. Physical（DRC / LVS / Antenna / Density / ERC / Latch-up / ESD）

| 项 | 判据 | 本 skill 对应 |
|---|---|---|
| DRC | `TOTAL RESULTS = 0`；waiver 只限上游宏内部并逐条列 | `30-drc-debug` |
| LVS | `CORRECT`，`Nets: N/N`、`0/0`，零 waiver | `40-lvs-debug` |
| Antenna | `0`，规则数取 `RuleChecks Executed` | `50-ant-rc-pt` |
| **密封环 SR.7/8/9** | **DRC 不查**（手册里标 `[NC]`）：环宽 / 环到芯片边的装配隔离 / 两者之和，**三项各查 DRM 下限**（本文不给数） | `20` §五 step 4、（本表下一行） |
| Density（各层 min/max） | 各层密度上下限规则全 0（规则族名查你那份 deck） | `30` |
| ERC（浮栅/浮阱/衬底） | 0 条，或逐条解释 | `30` |
| Latch-up | LU 规则 0 条（**注意靠 TEXT 识别电源 pad 的坑**） | `30` §三 DRC-1/DRC-5 |
| ESD | 按 PDK 规则（pad 到内部器件间距、双二极管） | `20` |

**假 0 的四个来源**：无布线的 GDS（via 全丢）、旧版图跑新 deck、deck 的 `LAYOUT PATH` 没换、非法版图（二极管压 filler）。

## 5. DFT / ATPG —— 案例里**决定不做**

| 项 | 工具 | 判据 |
|---|---|---|
| 扫描链插入 | Tessent / Modus / DFTMAX | 链数/长度与计划一致；`SD*` 触发器数 > 0 |
| stuck-at ATPG | Tessent / Modus（开源 `fault`） | **覆盖率 ≥ foundry floor（常见 95%，目标 98%）** |
| transition / at-speed | 同上 + OCC | 覆盖率达标，或书面具名说明工具限制 |
| BIST（memory） | 同上 | SRAM BIST 通过；否则写"未做" |
| boundary scan | — | 有 BSDL 文件，或声明不需要 |
| test mode 时序 | PT | 扫描模式单独一组 corner 时序，**常被忘** |

**必须诚实写**：`未做 ATPG ⇒ stuck-at 覆盖率不可测`。不许写"DFT 已考虑""可测性良好"这种模糊话。

> 案例教训：插过链（七千五百余级），但链头绕过 3.3V→1.8V pad 缓冲，最后决定整体不做并删掉 3 颗 DFT pad。交付网表 `SD*` 实测 = 0。

## 6. Power Intent（UPF / CPF）

单电源域、无 power gating 的设计可以整类标 `N/A`（**但要显式写 N/A 及理由**，不能留空）。
有多电压/可关断域时：UPF 与网表一致性、isolation cell、level shifter、retention flop、
always-on 缓冲——每项都要有 PA 工具的检查报告。

## 7. Documentation

- 设计说明（功能/时钟/复位/电源域/pad 分配）
- SDC（**哪一版**、周期、derate）
- SPEF（哪个 corner、抽取边界）
- pad list / pin list
- waiver 清单（ID/理由/审批人）
- 已知问题清单（含"本 skill §8 的空洞"这类）

判据：换一个人拿到这包文档能复现签核数字。

## 8. GDS Release

| 项 | 判据 |
|---|---|
| 版图与签核同源 | 交付包里的 GDS 与跑 DRC/LVS 的 GDS 是**同一文件**（同 md5，不是"同名"） |
| 文件命名 | 按 MPW 要求（`<Project>_trial.gds` / `_final.gds`），顶层单元名与 GDS 结构名一致 |
| 层次化 | 必须 hierarchical，不能拍平 |
| 压缩 | `.gds.gz` 或按平台要求的 tar.gz；上传后**回读**校验大小/哈希 |
| 校验值 | `md5` + `cksum` 双登记；本地与服务器两侧比对 |
| **canonical digest** | 若要证明"两份版图内容相同"（跨两次 streamOut），必须先把 GDSII `BGNLIB/BGNSTR` 的时间戳字段清零再算 digest；直接比 md5 一定不同 |
| manifest | 交付清单：文件名 / 大小 / md5 / cksum / 生成链版本 / deck 版本 / 跑签核的时间 |

---

## 9. 放行记录模板（复制到交付文档里）

```
[项目]              [版本 v?]         [日期]
┌────┬──────────────────────────┬──────────┬────────────────────────┬────────┐
│ #  │ 检查项                   │ 状态     │ 证据（文件:字段=值）   │ 审批人 │
├────┼──────────────────────────┼──────────┼────────────────────────┼────────┤
│ 1  │ 功能回归 / 覆盖           │          │                        │        │
│ 2  │ LEC RTL↔网表             │          │                        │        │
│ 3  │ STA 全 corner×mode       │          │                        │        │
│ 4  │ 静态 / 动态 IR           │          │                        │        │
│ 5  │ EM                       │          │                        │        │
│ 6  │ DRC                      │          │                        │        │
│ 7  │ LVS                      │          │                        │        │
│ 8  │ Antenna                  │          │                        │        │
│ 9  │ Density / ERC / LU / ESD │          │                        │        │
│ 10 │ DFT / ATPG 覆盖率         │          │                        │        │
│ 11 │ UPF 一致性               │          │                        │        │
│ 12 │ 文档齐套                 │          │                        │        │
│ 13 │ GDS 与签核同源           │          │                        │        │
│ 14 │ 压缩包回读校验           │          │                        │        │
└────┴──────────────────────────┴──────────┴────────────────────────┴────────┘
结论：PASS / PASS_WITH_WAIVERS(列 n 条) / FAIL
```

## 10. 一句话总结

**"没跑过"和"跑过但没读"都是 `NOT_MEASURED`；"跑了但结果没读全"最危险，因为它看起来像 `PASS`。**
