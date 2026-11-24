# 70 · 通用最佳实践（外部资料版，不含案例实测数字）

> 定位：本文件只收**行业通行做法 / 官方规范要点**，供 skill 复用到别的项目；尺寸、层号、0 条结论一律见 90，本文不重复、不混写。来源分级：**[厂商文档级]** = SMIC/TSMC/SkyWater/Synopsys/Cadence/Siemens 官方 PDK、DRM、UG、白皮书；**[专利/论文]**；**[社区文档]** = 开源 PDK、开源 shuttle、GitHub 教材；**[博客/论坛]** = 个人博客、eetop/mweda 问答；**[经验]** = 无可靠来源，仅工程共识。每条给「结论 / 为什么 / 对本 skill 的意义」。

---

## 一、Seal ring（密封环）设计规范

### 1.1 尺寸是「三段式参数」，不存在跨工艺通用数字
- 结论：seal ring 宽度、scribe lane 宽度、环到 die 边/到 pad 的隔离，是 foundry assembly/DR 给出的**三个独立参数**（design rule 里分别对应 assembly isolation / seal ring width / scribe line width），必须逐项从工艺文档取 ([mweda 如何画 seal ring](http://ee.mweda.com/ask/331698.html))（论坛）；社区经验值：scribe lane 常见 60–100 µm、主流工艺多用 80 µm ([eetop 问答](https://bbs.eetop.cn/forum.php?mod=viewthread&tid=604513)、[划片槽科普](http://mp.weixin.qq.com/s?__biz=MzY4NjM0NDY2NA==&mid=2247484128&idx=1&sn=56f7447e327a6867ba3f8a775984dec1))（博客/论坛）。
- 为什么：scribe lane 里要塞 PCM/TEG、对准标记、环本体，不同 shuttle/fab 塞的东西不同，所以只能由工艺侧定；环宽则由划片应力与金属叠层决定。
- 对本 skill：写成「从 DR 取参数并记录 deck/DR 版本」，而不是写死 µm 数；换工艺只换参数表。

### 1.2 环宽与 corner 是固定剖面，官方 pcell 优先
- 结论：开源 PDK 里 seal ring 以固定截面 pcell 交付，如 Sky130 的 `advSeal_6µm`/`DieSealR`（环宽 6 µm，并带 `areaid.dea` 死区用于金属应力规则检查）([SkyWater 层定义表](https://foss-eda-tools.googlesource.com/skywater-pdk/+/8b5a8cede1f8f1146040dd69abdad41de4b841dc/docs/rules/layers/table-c4b-layer-description.csv))（厂商 PDK 文档）；同一文档还规定 seal ring 左下角应落在原点 (0,0) ([Periphery Rules x.19](https://skywater-pdk.readthedocs.io/en/main/rules/periphery.html))（厂商文档级）。
- 为什么：corner 处金属/孔/有源区叠法与直边不同，自己拿直边拼 corner 是 DRC 与可靠性的高发点。
- 对本 skill：把「直边自绘 + 官方 corner 拼接」写成允许做法，但要求 corner 取官方件、直边剖面与官方一致。

### 1.3 环必须闭合：它是裂纹与水汽的最后一道屏障
- 结论：seal ring 的作用是阻挡划片产生的裂纹扩展和外部水汽/杂质侵入，断环即形成失效通路 ([US10892235](https://patentimages.storage.googleapis.com/be/04/fb/f4eb9714f140e1/US10892235.pdf)、[US20140239455](https://patentimages.storage.googleapis.com/e4/f0/86/f833232dfa1a43/US20140239455A1.pdf))（专利）；工程上它还兼作衬底/噪声隔离与接地通路 ([eetop 讨论](https://bbs.eetop.cn/thread-860793-1-97.html))（论坛）。
- 为什么：刀轮/激光划片沿 scribe lane 走，环一旦不连续，裂纹可绕过断口进入电路区。
- 对本 skill：签核加一条「环连通性/闭合性」目检项，而不是只依赖 DRC。

### 1.4 MARKS/marker 层的作用是「让 DRC 认得这是环」
- 结论：工厂常要求用专用标记层（如公开 PDK 里的 `CAD_SEAL_RING_MARK` 一类，层号随 PDK 而变）把 seal ring 包住，DRC 才能识别环区并对环内施加专用/豁免规则 ([mweda 问答](http://ee.mweda.com/ask/325609.html))（论坛）；开源 PDK 同样用辅助层（`areaid.sl` 等）界定环/划片区，并在 `DieSealR` pcell 内用 `areaid.dea` 标注金属应力死区 ([SkyWater 层定义表](https://foss-eda-tools.googlesource.com/skywater-pdk/+/8b5a8cede1f8f1146040dd69abdad41de4b841dc/docs/rules/layers/table-c4b-layer-description.csv))（厂商文档级）。
- 为什么：标记层不是物理膜层，是给验证工具看的「区域声明」；它丢了，规则要么误报要么整段不查。
- 对本 skill：把「marker 层必须在最后一次 GDS 导出前重建」写成通用纪律（本项目正是踩过），而不是项目偏方。

### 1.5 「自己画」还是「代工厂/shuttle 加」
- 结论：两种都通行——DR 给固定结构，设计方可按规则自绘（或用脚本生成），MPW 时也可请 fab/组织方统一加 ([mweda 如何画 seal ring](http://ee.mweda.com/ask/331698.html))（论坛）；开源 shuttle 即由流程脚本在 floorplan 阶段加环（`addmpwseal.tcl`，"Add caravel floorplan with a preliminary seal ring"）([sky130 mpw-001 slot-029](https://foss-eda-tools.googlesource.com/third_party/shuttle/sky130/mpw-001/slot-029/+/1dd50654a824fc8be748cb945e27c6bdb9940be8%5E%21/scripts/addmpwseal.tcl))（社区文档）。
- 为什么：谁加环决定了谁负责环的 DRC/LVS、谁负责与 pad/IO ring 的间隙、以及最终 GDS 的层归属。
- 对本 skill：交付清单里显式写「环由谁加、在哪个 GDS 版本后加、环的 DRC 由谁出报告」。

---

## 二、1P5M → 1P6M 迁移

### 2.1 金属层数是「metal option」，一层之差就是一套 tech file
- 结论：0.18 µm 平台普遍以 metal option 形式提供 1P5M/1P6M 等组合，且顶层可选厚金属（如 TSMC 0.18 的 `1p6M_4X1U40KA` 选项）([CMC TSMC 0.18 工艺页](https://www.cmc.ca/tsmc-180-nm-cmos/))（厂商/代工服务文档级）。
- 为什么：层数/厚度变化会同时改 LEF、TLU+/nxtgrd、DRC/LVS deck、fill deck 与密度规则，任何一件不同步就是「用 5 层的模型签 6 层的片」。
- 对本 skill：所有签核产物必须带「metal option + deck + tech file 版本」三元组，缺一不可。

### 2.2 厚顶层金属天生该给电源和长线用
- 结论：工艺常把顶部一两层做得更厚以降低时钟、电源、全局信号走线的电阻 ([US8336018](https://patentimages.storage.googleapis.com/3d/30/72/a58053de8d1c88/US8336018.pdf))（专利）；多层金属中上层电阻性阻抗更低、下层电感/电阻比例不同 ([Springer 章节](https://rd.springer.com/chapter/10.1007/978-3-319-29395-0_39))（论文/教材级）。
- 为什么：IR drop ∝ 电流 × 电阻，同样的网格把电源搬到厚金属上，压降改善最直接；长全局线延时也主要由 RC 决定。
- 对本 skill：迁移时的默认动作写成「PG 网格、时钟主干、跨模块长线优先搬厚金属」，而不是全面重布线。

### 2.3 顶层换了，提取与 STA 必须重来
- 结论：每层厚度/介电环境变了，单位长度 RC 与耦合电容随之变，寄生必须以新 stack 的 TLU+/nxtgrd 重新提取 ([StarRC ITF→TLU+ 编译流程](https://geek-doc.cn/eda/starrc-techfile-itf-tluplus))（博客，内容与厂商流程一致）；同时顶层的最小线宽/间距/密度/填充规则与低层不同 ([SkyWater Periphery Rules](https://skywater-pdk.readthedocs.io/en/main/rules/periphery.html))（厂商文档级）。
- 为什么：沿用旧 SPEF 会得到「看起来收敛」的假时序；沿用旧 fill/density 规则会得到假 DRC 0。
- 对本 skill：迁移的验收门槛写成「同一 metal option 下：deck + TLU+ + fill deck + SPEF + STA 同基线」。

---

## 三、Calibre DRC/LVS 签核 checklist

### 3.1 签核是一个集合，不是「DRC 0 + LVS CORRECT」两项
- 结论：tapeout 前通行集合为 DRC、LVS、天线、ERC、密度/填充、latch-up、ESD、电源地连通性、short/open、seal ring/scribe line、bump/pad 检查 ([silicon-atlas 签核清单](https://github.com/microarch-flow/silicon-atlas/blob/main/03_backend_physical_design/07_drc_lvs_signoff.md)、[EcrioniX signoff checklist](https://ecrionix.org/physical-design/signoff/))（社区文档/博客）。
- 为什么：DRC/LVS 只覆盖几何与连接，浮空栅、电源地直连、ESD 通路阻值这些「电学良构」问题两者都看不出来。
- 对本 skill：把 checklist 做成「项目 → 证据文件 → 结论」三列表，逐项留痕。

### 3.2 签核 DRC 必须在最终 GDSII 上重跑
- 结论：以最终导出的 GDSII 跑出零违例才算 clean，P&R 工具内部的 clean 不算 ([EcrioniX](https://ecrionix.org/physical-design/signoff/))（博客，与各家 flow 一致）；填充后版图变了，寄生与 STA 也要用填充后的数据重跑。
- 为什么：GDS 导出会丢层、改层号、烘焙实例，工具内数据库与交付文件不是同一份东西。
- 对本 skill：签核对象一律写「文件名 + md5 + deck + 选项」，禁止用「上次跑过」代替。

### 3.3 ERC/PERC 查的是电学良构
- 结论：浮空栅、未驱动栅、电源地直连、ESD clamp 之间的点对点电阻过大等，由 ERC/PERC 类检查覆盖 ([Siemens PERC 白皮书](https://static.sw.cdn.siemens.com/siemens-disw-assets/public/34xQH4zOfVrddyoNmX2oEc/en-US/Siemens-SW-Advanced-electrical-rule-checking-in-IC-WP-81822-C2.pdf))（厂商白皮书）；latch-up/ESD 的间距类要求则写在工艺假设文档的 Latch-up/ESD Criteria 章节 ([SkyWater 假设文档](https://skywater-pdk.readthedocs.io/en/main/rules/assumptions.html))（厂商文档级）。
- 为什么：这些错误在版图几何上是「合法」的，只在电路语义上错，必须靠提取网表+规则判。
- 对本 skill：把 ERC 单列为一步，不和 LVS 合并。

### 3.4 三种典型「假 0」陷阱
- 结论：(a) 层号/layer map 不一致，工具照跑但图形没被识别 ([eetop 层号讨论](https://bbs.eetop.cn/thread-979507-2-32.html))（论坛）；(b) 干净报告对应的是旧版图，checksum 只能证明文件身份、不能证明 DRC 通过 ([MST MPW 交接清单](https://store.mst-sg.com/community/t/33-mpw-tapeout-handoff-match-gds-verification-reports-and-the))（厂商社区文档）；(c) waiver 里的「边缘错误」若不人工复核就当作已豁免 ([Calibre Auto-Waivers](http://www.mentor.com/products/ic_nanometer_design/verification-signoff/physical-verification/calibre-nmdrc/upload/calibre-auto-waiver-ds.pdf))（厂商文档级）。
- 为什么：三者都表现为「0 errors」，但分别是没查、查错对象、把错误合法化。
- 对本 skill：验收话术固定为「这份 0 是在哪个文件、哪个 deck、哪些选项下得到的」。

---

## 四、天线效应（Antenna）

### 4.1 计算原理：每层「暴露导体面积 / 栅面积」，逐层累计
- 结论：天线规则限制的是等离子刻蚀时与栅相连的导体暴露面积相对栅面积的比值，暴露面积含接触孔/via 面积与导体侧壁面积（周长×厚度）；无二极管时 `EGAR = EA / A_gate`，逐层判 ([SkyWater Antenna Rules](https://skywater-pdk.readthedocs.io/en/main/rules/antenna.html))（厂商文档级）。LEF 级定义给出等价形式：`ratio = [(metal_area + 10×cut_area) − (100×diff_area)] / gate_area`，并把各层 PAR 累加 ([LEF/DEF 参考](https://coriolis.lip6.fr/doc/lefdef/lefdefref/PAE.html))（格式规范级）。
- 为什么：电荷在刻蚀该层时积累，下层已连通的导体仍与栅相连，所以必须从下往上逐层累计。
- 对本 skill：解释「为什么加 via、跳层、切长线都能改比值」，避免把天线当成玄学。

### 4.2 二极管不是万能：它按面积折算进分母
- 结论：用二极管时 `EGAR = EA/A_gate − K×diode_area − diode_bonus`，K 逐层给定、`diode_bonus` 只要用了二极管就给 ([SkyWater Antenna Rules](https://skywater-pdk.readthedocs.io/en/main/rules/antenna.html))（厂商文档级）；商业工具同样按二极管保护能力对比值做缩放 ([IC Compiler User Guide, antenna ratio 计算](https://projects.ece.utexas.edu/courses/spring_20/ee382m-8/project_shared/docs/iccug.pdf))（厂商文档级）。
- 为什么：二极管靠反向漏电泄放电荷，面积越大保护越强，但收益有上限且逐层不同。
- 对本 skill：要求「二极管面积、接的电源/地、所在层」都记录，否则无法复核。

### 4.3 三种主流修法与各自代价
- 结论：常见修法为（1）就近加天线二极管泄放电荷；（2）跳层/换层把长导体分散到不同层；（3）插 buffer 把长线切成两段 ([天线违例批量修复实践](https://blog.csdn.net/weixin_28728279/article/details/166723474))（博客）。
- 为什么：三种分别改「分母/单层面积/连通长度」，适用场景与副作用完全不同。
- 对本 skill：修法选择写成决策表（违例层、违例倍率、是否有空闲二极管位置、时序余量），而不是一律加二极管。

### 4.4 修完必须重跑三项
- 结论：未找到权威来源，属经验——二极管引入额外电容/漏电/面积，跳层改变 RC 与耦合，插 buffer 改变逻辑结构与延时；因此修天线后必须重跑 ANT + LVS + STA（必要时串扰）。
- 为什么：天线修复是在签核末期动版图，最容易把已经 clean 的时序/连接打回去。
- 对本 skill：把「修天线 → 重跑三件套」写成强制回路，禁止只重跑 ANT。

---

## 五、MPW/Shuttle 提交流程

### 5.1 交付 = 版图 + 证据 + 责任人
- 结论：通行交接清单要求：版图文件名/格式/版本/checksum/top cell/单位、工艺与 PDK/deck 版本、layer mapping、DRC/LVS 报告及运行配置、剩余错误与 waiver（含理由与批准人）、pad map 与封装图、测试交接 ([MST MPW tapeout handoff](https://store.mst-sg.com/community/t/33-mpw-tapeout-handoff-match-gds-verification-reports-and-the))（厂商社区文档）。
- 为什么：MPW 是多项目共用 reticle，任何一项对不上都会在拼版阶段返工，且责任无法追溯。
- 对本 skill：把该清单直接做成提交包的目录结构模板。

### 5.2 层次化有硬上限，提交前要 flatten 体检
- 结论：GDSII 层次深度有上限（常见实现限制 64 层），过深层次会被写版/校验工具拒绝 ([GDS 几何处理文档](https://raw.githubusercontent.com/tachsin/photonoxide/refs/heads/main/docs/methods/geometry.md))（工具文档级）；掩模写入设备对层次层数的限制更严，超限需重定层次或扁平化 ([US9292643](https://patentimages.storage.googleapis.com/ba/0d/0b/4ccb1c043f5e71/US9292643.pdf))（专利）。
- 为什么：层次过深/单元过多是提交被退回的常见机械性原因，与设计正确性无关。
- 对本 skill：提交前跑一次「层次深度 + cell 数 + 单层多边形数」体检，超限先处理再提交。

### 5.3 文件大小与压缩是平台硬约束
- 结论：shuttle 平台对上传文件设明确大小上限（示例文档写单文件 1 MB）([UNIC-CASS precheck 材料](https://ieee-cas.org/files/ieeecass/2024-09/UNIC-CASS-2nd-Ed-Precheck-mocktest.pdf))（社区/教学文档）；交付常要求压缩包 + 校验和以便对账 ([MST](https://store.mst-sg.com/community/t/33-mpw-tapeout-handoff-match-gds-verification-reports-and-the))（厂商社区文档）。
- 为什么：限额决定你能否保留层次、是否需要拆包/分卷，早确认能省一轮返工。
- 对本 skill：提交前先读平台限额，再决定「保留层次 / 扁平化 / 压缩分卷」。

### 5.4 `.summary` 汇总日志与 waiver 的通行要求
- 结论：未找到权威来源，属经验——多数 MPW/代工提交包要求一份机器可读汇总（各检查错误数、waiver 列表、工具与 deck 版本、GDS md5），便于 fab 与设计方对账；waiver 需逐条对应到 highlight 与批准人（后者见 [Calibre Auto-Waivers](https://www.saros.co.uk/wp-content/uploads/2024/02/Calibre-Auto-Waivers.pdf)）。
- 为什么：fab 不会逐条读你的 DRC 报告，只会看汇总数与 waiver 是否自洽。
- 对本 skill：要求 `.summary` 由脚本从报告自动生成，禁止手写。

---

## 六、只有 TLUPlus、没有 QRC techfile 时怎么办（StarRC / PrimeTime）

### 6.1 源头是 ITF，TLU+/nxtgrd 是它的编译产物
- 结论：foundry 交付的源头是文本 ITF；TLU+ 是离线场求解预计算出的多维查找表（按 corner 一文件），nxtgrd 是更细粒度的模型格式，二者都由 ITF 编译得到，命令为 StarRC 体系的 `grdgenxo`（`itf2TLUPlus` 语法为 `-input itf -output tlu`）([StarRC 技术文件：ITF→TLU+ 编译流程](https://geek-doc.cn/eda/starrc-techfile-itf-tluplus)、[TLU+ 转换实践](http://mp.weixin.qq.com/s?__biz=MzkxOTYwNzYxNQ==&mid=2247483907&idx=1&sn=5230b56929498a09b67681421d8a16c2))（博客，流程与厂商一致）；QRC techfile 属于 Cadence Quantus 路线，与 StarRC/PT 不是同一套技术文件，社区亦总结「ITF 不能直接给工具用，需转 nxtgrd 给 StarRC、tluplus 给 P&R」([mweda 问答](http://ee.mweda.com/ask/334376.html))（论坛）。
- 为什么：TLU+ 只是「已编译好的一档精度」，不是另一种工艺描述；两条提取路线各有 tech file，缺一方不影响另一方。
- 对本 skill：只有 TLU+ 时走 StarRC/PT 即可、不需要 QRC techfile；查清「谁编译、对应哪个 corner」比纠结格式更重要。

### 6.2 真正的翻车点是 layer mapping，不是格式
- 结论：ITF/TLU+ 用工艺层名、GDS 用 layer/datatype 数字对，中间靠 mapping 文件桥接；漏一层会让该层寄生**静默为 0**、提取照常跑完不报错 ([StarRC 技术文件](https://geek-doc.cn/eda/starrc-techfile-itf-tluplus))（博客）；同类层号不一致问题在 DRC 侧同样存在 ([eetop](https://bbs.eetop.cn/thread-979507-2-32.html))（论坛）。
- 为什么：提取器只对「它认得的层」建模，认不得就跳过，误差只会体现在时序上。
- 对本 skill：每次提取后抽一条已知长度金属线，用 SPEF 电阻对 ITF 方块电阻手算对账。

### 6.3 corner 名、文件名、扰动表必须三者对齐
- 结论：TLU+/nxtgrd 按 corner 分文件，运行时按 corner 名加载；corner 名与文件名/ITF 扰动表对不上，就会「张冠李戴」地用错模型 ([StarRC 技术文件](https://geek-doc.cn/eda/starrc-techfile-itf-tluplus))（博客）。
- 为什么：这类错误不报错、只让某个 corner 的裕量异常，靠人眼很难发现。
- 对本 skill：要求 corner 到文件的映射表随签核包归档，并在 STA 报告里打印实际加载的文件名。

## 使用建议

- 与本项目实测冲突时，**以实测 + 官方 deck 为准**，并把冲突点回写到对应篇的「坑」；新增条目必须带来源等级 + URL + 一句「对本 skill 的意义」。
- 每次换工艺/换 PDK 版本，重读第一节（seal ring 参数）、第二节（metal option）、第六节（提取技术文件）——这三节最依赖具体工艺。
