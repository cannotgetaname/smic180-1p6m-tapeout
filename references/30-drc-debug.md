# 30 · DRC 调试手册（现象 → 根因 → 对策）

> **什么时候读我**：DRC 报了成百上千条，要判断「是我们版图的错还是 PDK/pad 库的锅」，或者要立 waiver、要解释为什么规则跑得特别慢时。
> 版图链本身在 `20-layout-chain-1p6m.md`；本篇只讲**判读与归因**。

---

## 一、先看 summary，不看 results

- 结论只看 `.summary`：`TOTAL DRC RuleChecks Executed`（规则总数）、`TOTAL DRC Results Generated`（结果数）、`TOTAL CPU Time / REAL Time`。
- **ASCII `.results` 每个规则默认最多 1000 条**（summary 里写着 `Maximum Results/RuleCheck`）。看到「每条都正好 1000」不是巧合，是被截断了 —— **总数只能用 summary**。
- 一次 DRC 的耗时差异可以非常大（全片几分钟，但 LU 类规则单开可能 40 分钟）。**不要用耗时估进度**；用 `grep -c` 数 summary 里已完成的规则。

**判读顺序**：规则名 → 坐标分布（`grep` CN 坐标）→ 是否集中在 pad 环 / SRAM 宏 / 环带 → 再决定是改版图还是查 deck。

---

## 二、deck 改动只有两类，先 `diff` 官方 deck

签核 deck 是官方 PDK deck 的**副本**。任何一次「为什么官方跑 4,000 条、我们跑 0 条」的疑问，都用这一条回答：

```bash
diff <官方 deck> <项目签核 deck>
```

正常只应该有两类差异：

1. **头部**：`metal_counts`、`LAYOUT PATH`、结果/summary 输出路径、`DRC CELL NAME YES` + `CELL SPACE XFORM`（按 cell 归类，输出里能看到违规在哪个宏里）。
2. **判定式若干处**：给指定规则加 `NOT INSIDE <MARKER>`（豁免用）。

**如果 diff 出第三类改动，就是有人偷偷改了规则** —— 这在签核里是不可接受的，必须回退。

---

## 三、六个真实案例（可直接当排查模板）

### DRC-1 LU 类规则爆几千条（最贵的一个坑）

- **现象**：LU 族的四条规则同时爆，规则**每条封顶 1000**，CN 坐标铺满全片。总耗时飙到半小时以上。
- **根因**：deck 靠**金属层 TEXT 上的 VDD/VSS 字符串**认电源 pad（`DEFINE_PAD_BY_TEXT` + `PADTEXT_TOP` + `POWER_NAME`/`GROUND_NAME`）。P&R 导出的 GDS **没有任何这类文本** ⇒ 判不出电源 pad ⇒ 整片 VDD/VSS 网上的 N 扩散全被当成 IO 器件。
- **为什么以前没爆**：旧 pad 库没有顶层金属/通孔时，deck 里那条 pad 天线判定（`CONNECT Mn…` 接 pad 顶层）目标层是悬空的 ⇒ 面积比规则恒空 ⇒ LU **全 0 是假阴性**。换 pad 只是让隐藏缺陷第一次真正触发。
- **对策**：给每个电源 pad 的**顶层金属**补 VDD/VSS 文本（脚本见 20 的 step 3）。补完 LU 归零，DRC 总耗时降一个数量级。
- **关键细节**：标签打在**次顶层**金属上可能无效（那块金属是悬空 plate）。判据是「覆盖整个 pad」+「有 via 连通到电源网」，见 20。

### DRC-2 `V4_1` 爆上千条

- **现象**：`V4_1` 单独成组，与 LU 无关。
- **根因**：设计按 1P5M 出图，V4 是**顶层厚通孔**；到了 1P6M，V4 降级为普通通孔，deck 要求**普通通孔尺寸**。deck 里「厚顶层通孔」与「普通通孔」是两条判定式、各管一层，V4 只命中后者，所以只报 `V4_1` 一组。
- **对策**：把所有 `<V4_SIZE_THICK>`² 方孔就地缩成 `<V4_SIZE_NORM>`²（中心不动）。只改尺寸不改连接 ⇒ 0 条。
- **判据**：缩完再跑，若 `V4_1` 仍非 0，说明还有别的层号/图形没匹配上（比如 pad 库自带的 V4 或某个旋转过的孔），回去数 `TOTAL_V4` 与 `SHRUNK` 的差。

### DRC-3 marker 层 datatype 丢了 ⇒ 整片豁免失效

- **现象**：合并 fill 后再 `gdsout`，marker 图形数从 N 变成 0（deck 侧看到的是 `.0`），LU 与 via-stack 规则**同时回到几百上千条**。
- **根因**：`calibredrv gdsout` 不保留这些 marker 层的 datatype，而 deck 里写的是带 datatype 的层（两个带 datatype 的 marker 层，本文记作 `<MARKER_1>` / `<MARKER_2>`）。
- **对策**：见 20 step 8 —— **不在中途画 marker**，最后一次 gdsout 前一次性重建，先删 `.0` 残留。
- **判据**：重建后 `$L layers` 里同时出现原始 datatype；DRC 里「被豁免的规则」重新归零，而「真违规」不会消失（这点很重要：**豁免只该消掉已知的几百条，不该消掉同规则里其它位置的违规**）。

### DRC-4 pad 接缝的 via-stack 违例（版图解决不了）

- **现象**：via-stack 族的三条规则（叠孔与金属间距类）各一百多条，合计几百，是 waiver 后剩饭里最大的一坨。
- **根因**：一部分落在**pad 单元内部**（pad 局部坐标固定），一部分落在**相邻 pad 之间的顶层缝隙**（每个 pad 边界两侧几 µm 的细条）。决定性证据：这些多边形在「旧交付件 + 6MT pad」与「新链条」两次运行里**坐标逐字节相同**，与环、与 fill 都无关 ⇒ 是 6MT PDK pad 单元自身及 abutment 的固有几何。旧 5MT 阶段为 0 只因那时根本没有顶层金属/通孔。
- **对策**：**不要试图改版图**（改 pad 就动 floorplan，改环也没用）。走项目 waiver deck：给这几条循环生成的规则加 `NOT INSIDE <MARKER_2>`，marker 覆盖 pad 实例方框。**在报告里写清是 vendor 单元固有几何，并给坐标证据。**

### DRC-5 残余 LU（几条，全在 pad / SRAM 内部）

- **现象**：标签修好后 LU 只剩个位数，全在信号 pad 内部，且**各 pad 的局部 bbox 完全相同** ⇒ 同一个 vendor ESD/guard-ring 子单元。
- **根因**：foundry pad / SRAM 库自身的边界效应。
- **对策**：所有 LU 规则输出本身就带 `NOT (<MARKER_1> OR <MARKER_2>)`，所以**不改判定式**，只把 pad / SRAM 宏实例的方框盖进 `<MARKER_2>`。也可以不豁免、直接向 foundry 说明——两条路都要留证据。

### DRC-6 环带里的残余细缝（不是靠改环消掉的）

- **现象**：N-well 细缝类规则（十几 nm 的缝）与两三条金属间距类违例，全部落在**环带内**、pad 行下方十 µm 量级的电源环带结构上。
- **根因**：环带结构与填充边界的固有细缝。
- **对策**：靠 step 8 的 `<MARKER_2>` **环带矩形**覆盖 + deck 豁免归零。**不要**为了消这两三条去动环的几何（动环会毁拐角，代价远大于收益）。

---

## 四、waiver 口径（可直接抄进报告）

1. **前提**：`核心区必须 0 条`。这是接受任何豁免的底线；环带外侧也应为 0。
2. **逐类列明**：规则名、条数、坐标区间、为什么不是本设计错误（vendor 固有几何 / 库内部 / 标记覆盖范围）。
3. **不掩盖、不夸大**：不要写「DRC 0 条」而不说用了 waiver。正确写法是「`TOTAL DRC Results Generated = 0`，其中 N 条经项目 waiver deck 豁免，构成如下…」。
4. **自动 waiver 流程可能不可用**：官方 `DRC Automated Waiver Flow` 需要额外 license feature。拿不到就走**自建 waiver deck + marker 图形**这条路，同样要在报告里说明「不是官方 waiver 记录」——不要伪造 waiver 记录。
5. **豁免数与规则数对不上时先怀疑自己**：`TOTAL` 里的规则总数是 deck 里**实际执行**的规则数（本项目取值见 90），和你肉眼数的规则名个数可能不同（循环生成的规则算多条）。

---

## 五、快速排查清单

| 症状 | 先查 |
|---|---|
| 某规则正好 1000 条 | 被截断了，看 summary 总数 |
| 全片铺满、规则名带 LU | 电源 pad 的 TEXT 标签（DRC-1） |
| 单一规则、数量上千、规则名带 V4 | 顶层通孔尺寸 / datatype（DRC-2） |
| 多条规则同时从 0 变几百 | marker 层丢了 datatype（DRC-3） |
| 违规坐标集中在 pad 边界 ±3 µm | pad abutment 固有几何（DRC-4） |
| 违规在 SRAM 宏 bbox 内 | 库内部，豁免或找 foundry（DRC-5） |
| 违规只在环带内 | 环带 marker 覆盖（DRC-6） |
| DRC 特别慢（半小时+） | LU 规则被触发（DRC-1），先修标签 |
| 0 条但觉得不对 | deck 的 `LAYOUT PATH` 指错了 GDS（见 60） |

---

## 六、换项目改哪几行

| 要改的东西 | 在哪 |
|---|---|
| 规则总数、CPU/REAL 判据 | 每次跑完读 summary（不要抄旧数字） |
| TEXT 层号、`POWER_NAME`/`GROUND_NAME` 通配 | deck 头部（本项目 `PADTEXT_TOP`/`DEFINE_PAD_BY_TEXT` 段） |
| 顶层通孔尺寸判定的两个分支 | deck 的 via 规则段（搜 DRM 的两档尺寸） |
| marker 层号与 datatype | deck 的 `LAYER MAP`（两个带 datatype 的 marker 层，本文记作 `<MARKER_1>` / `<MARKER_2>`） |
| 豁免的规则名清单 | deck 判定式（grep `NOT INSIDE`） |
| pad/SRAM 单元清单（盖 marker 用） | pad 库 `cell_list` + SRAM 宏名 |
| 环带宽度 | marker 脚本参数 |
