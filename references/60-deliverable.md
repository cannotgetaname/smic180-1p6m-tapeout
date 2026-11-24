# 60 · 流片交付：打包、MPW 表格、交付前最后一道验收

> **什么时候读我**：签核（DRC / 天线 / LVS / PT）已过，要打 tar 包、填 MPW 表格、或做交付前最终核对时。

---

## 一、交付包里应该有什么

最小三件套（本项目实测结构）：

```
<TOP>_<trial|final>.gz
  <TOP>_<trial|final>.gds                # 交付 GDS，顶层 = <TOP>
  <TOP>_<trial|final>.main.drc.summary   # 主 DRC summary（<N_RULES_DRC> 条规则全 0）
  <TOP>_<trial|final>.ant.drc.summary    # 天线 summary（<N_RULES_ANT> 条规则全 0）
```

- 包内 GDS 必须与签核用的那份 GDS **逐字节相同**（`cmp` / md5）。
- 包内两份 summary 的 `Layout Primary Cell` 必须等于 `<TOP>`，`Layout Path(s)` 必须能对上包内 GDS 的**内容**（见坑 1）。
- trial 与 final 常见是同一个 GDS 打两次（md5 相同，`.gz` 差几字节属 gzip 头差异）—— 要确认，不要假设。

## 二、打包与自检三件套

```bash
tar -czf <TOP>_final.gz <TOP>_final.gds <TOP>_final.main.drc.summary <TOP>_final.ant.drc.summary
tar -tzf <TOP>_final.gz                    # ① 成员名对不对
mkdir -p /tmp/unpack && tar -xzf <TOP>_final.gz -C /tmp/unpack
cksum /tmp/unpack/*.gds; md5sum /tmp/unpack/*.gds   # ② 与源 GDS 对账
grep "Layout Primary\|Layout Path" /tmp/unpack/*.summary   # ③ summary 里的 GDS 名对不对
```

**顺序不能反**：先看成员，再对哈希，最后对 summary —— 任何一步不对就重打，不要「反正内容应该一样」。

## 三、MPW 表格：字段与取值来源

表格字段各 foundry 不同，但**每一栏都必须能追到一次命令或一份 summary**。通用取法：

| 字段 | 取值方式 | 易错点 |
|---|---|---|
| GDS 结构名（Top Cell） | summary 的 `Layout Primary Cell` / `calibredrv` 回读 | 别填文件名 |
| 文件大小 | `ls -l`（字节） | 打包前后要一致 |
| `cksum` | `cksum <gds>` | 与 `md5` 两个都要 |
| `md5` | `md5sum <gds>` | 换版必须重算 |
| die 尺寸 / 面积 | **按 die 切割线**（`<DIE_UM>`），不是 BORDER | 见第四节 |
| 掩模 bbox 右上角 | BORDER 的右上坐标 | 只是标记框 |
| DRC deck 名 + 路径 | 主 DRC summary 的 `Rule File Pathname` | 写全路径 |
| 天线 deck 名 + 路径 | 天线 summary 的 `Rule File Pathname` | |
| LVS deck 名 + 路径 | LVS 报告/命令 | |
| 日志文件名 | 各次运行的 stdout 重定向目标 | 交付时要能一起给 |
| 签核结论 | `DRC 0（N 规则）/ 天线 0（N 规则）/ LVS CORRECT（Nets N/N）` | 规则数取 summary，不抄文档 |

**历史值必须隔离**：另一版 GDS 的 size/md5/cksum 要标成「历史值，禁止混用」，换版一律重算。

## 四、尺寸口径：die 与 BORDER 不是一个东西

- **面积按 die 切割线算**（`<DIE_UM>`²）。表里填 die 尺寸/面积。
- `<BORDER_UM>` 是 GDS 里的**标记框**（`<BORDER_LAYER>`），画得比 die 大一圈，让掩模 bbox 等于整数尺寸。**它不是 die，不能拿来算面积**，也不能写进「芯片尺寸」栏。
- 与上一代交付件的一致性要单独核对：die / BORDER / 环带有没有被顺手改过。**floorplan 一行未改**这种话要有 diff 或坐标证据。

## 五、交付前六道闸（少一道就可能被打回）

1. **deck 路径闸**：`grep -n 'LAYOUT PATH' <三份 deck>` 全部指向当期 GDS；再 `grep "Layout Path\|FILE READ FROM" *.summary` 与包内 GDS 对账。
2. **摘要闸**：`TOTAL DRC Results Generated = 0`、`RuleChecks Executed = <N_RULES_DRC>`、天线同理；LVS `CORRECT` + `Nets: N / N` + `0 / 0`。
3. **回读闸**：`calibredrv` 读交付 GDS（层数 / 结构数 / bbox / 顶层名），并请人在 **Virtuoso 实导一次**。**「自写脚本检查通过」不等于能导入，不许自己宣布没问题。**
4. **哈希闸**：`md5` + `cksum` 双登记，本地挂载与服务器两侧各算一次，一致才算。
5. **同一文件闸**：交付件与被测件必须是同一个文件（`cmp` 或 md5）。`deliv/` 里既有中间版又有交付版，**用中间版数字顶替交付件**是真实踩过的亏。
6. **文本闸**：交付 GDS 里**不许有**非必要的 M1–Mtop TEXT（会被 LVS 当 pin 名，见 40 §9）。剥文本的版本要单独命名管理，别把它当交付件。

## 六、四个坑

1. **summary 里的 `Layout Path` 与包内文件名不一致**：本项目包内 summary 写的是中间文件路径、包里是另一个名字，只因为两者当时逐字节相同才没出事。规矩：**对不上就按新路径重跑再打包**，不要靠「应该一样」。
2. **用 GDS 大小当粗筛可以，但不能定案**：不同 metal option 的交付件体积差一截（示例：1P6M 约 1.1 亿 B、上一代 1P5M 约 0.95 亿 B，只记量级），`ls -l` 一眼分新旧够用；但**体积相同的不同版 GDS 完全可能存在**，定案必须哈希。
3. **tar 的 mtime/时间戳警告可忽略**：sftp 挂载的 mtime 与服务器有偏差，`tar` 会报「file changed as we read it」。只要 `tar -tzf` 能列出成员、解包后哈希与源一致就不用重打（此条为经验，工程内无报告记录）。
4. **仓库里可能有两份都自称「最终」的 GDS**：按 md5 + 生成链判决，以**交付包里那份**为准，另一份在文档里标为上一代并写明为什么。别在两份之间反复横跳。

## 七、换项目改哪几行

| 要改的东西 | 在哪 |
|---|---|
| 包内成员名与 `<TOP>` | 打包脚本 / 表格 |
| 规则数（DRC / 天线） | 每次从 summary 取，**不写死** |
| die / BORDER 坐标 | config.tcl + 实测 bbox |
| deck 路径 | 三份 deck 的 `LAYOUT PATH` + 表格 |
| MPW 表格模板 | foundry 提供，字段名可能不同（照上表逐项映射） |
| 交付目录命名（trial/final/日期） | 打包脚本 |
