# USIT-STAGE-B-DESIGN-05.28 — HASH-MANIFEST

**工单编号**：USIT-STAGE-B-DESIGN-05.28
**采集时间**：2026-10-31
**工作区**：C:/Users/Administrator/WorkBuddy/2026-10-15-22-09-47/usit_p0
**交付目录**：C:/Users/Administrator/WorkBuddy/2026-10-15-22-09-47/usit_p0_05.28_reports
**哈希算法**：SHA-256
**采集工具**：sha256sum (coreutils) / stat -c %s (GNU)

本 MANIFEST 记录纳入交付 ZIP 的 12 份内容文件（9 份 Markdown + 3 份源码基线）的真实字节数与 SHA-256。MANIFEST 自身与 DELIVERY.zip 的哈希由 HASH-RECEIPT 与 ZIP-RECEIPT 记录，避免自引用循环。

| 文件 | 字节数 | SHA-256 |
|---|---|---|
| USIT-STAGE-B-DESIGN-05.28.md | 38056 | 9ab48cdda4bce4c1ba97d596d73c027539f75698ce9bb40cf99e89988fca314d |
| USIT-STAGE-B-DESIGN-05.28-ISSUES.md | 2094 | 54e76dc6141622ca9e674007ecbee4b26e5dc494ea80f9634a17ad67c8829e34 |
| USIT-STAGE-B-DESIGN-05.28-DIFF.md | 2171 | c7804086ba256a9490d0c5c8b28d1fa7168bb6178d85502653eb456e25dd6bad |
| USIT-STAGE-B-DESIGN-05.28-CROSS-REVIEW.md | 2300 | ff9c7da283d60e0881bdab05fab14fa22f50db72f629be450614477fba6a96a7 |
| USIT-STAGE-B-DESIGN-05.28-EXECUTION-REPORT.md | 3688 | b8b475d932819e73c1d2b3a499daf1142e4e891275040640447669ac0e4a1b9f |
| USIT-STAGE-B-DESIGN-05.28-TIMELINES.md | 22611 | 8fcef6b4d70ddff7667e0d2c4cf4afb9250b3343b2a88ce7b9bbf4993ac585fe |
| USIT-STAGE-B-DESIGN-05.28-COUNTEREXAMPLES.md | 19944 | 9a501a80bfa89691cf5e1afd207cecd7b2eae905d1a0ee249b48a49738ee850d |
| USIT-STAGE-B-DESIGN-05.28-CROSS-MODULE-CHAINS.md | 7200 | d66653f21150f1ff69bd39a78923df0b6c737336d1a02784fdbf287173ac0106 |
| USIT-STAGE-B-DESIGN-05.28-INVARIANTS.md | 13071 | 8888656747df1bad56259234d6bfa6a34fb08bdf44f9b1dbb7d3b9b22518192d |
| usit_source_baseline_05.28_pre.txt | 2465 | 346955405adbcd7d1ef8b6a074e300a10c8e202fba4fc91c1d3c03ded79106fa |
| usit_source_baseline_05.28_post.txt | 2466 | 6e9a58632876943a40d5a81e4cdb83cd988d92eaeee88998a49646340c01d751 |
| usit_source_baseline_05.28_diff.txt | 648 | 157532e697b6c38377be9e5e762fbfb40ab3ca3a5f012ccb36abc9c85307405d |

## 源码基线一致性

PRE 与 POST 逐文件比对（`diff <(tail -n +4 pre) <(tail -n +4 post)`）结果为**空**，25 个 `.py` 文件 SHA-256 与字节数全部一致。三个冻结文件与历史基线一致：

- scanner.py = 58ba5f25483429ce7482ae1d852deabce6ea0490f2e106c6955c6ff5323b45e3（MATCH）
- cli.py = 879a3735838e183f41dd08054095b514ffc0e81e2b43776b75a77e25d138b8d9（MATCH）
- file_understanding.py = 71995a09da04427c028282cfeba17c6b12ebdbcae97796adc2998af7c455ce5b（MATCH）

**结论**：本轮未修改任何 USIT 源码。
