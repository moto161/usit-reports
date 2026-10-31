# USIT-STAGE-B-DESIGN-05.27 — HASH-MANIFEST

**工单编号**：USIT-STAGE-B-DESIGN-05.27
**采集时间**：2026-10-31
**工作区**：C:/Users/Administrator/WorkBuddy/2026-10-15-22-09-47/usit_p0
**交付目录**：C:/Users/Administrator/WorkBuddy/2026-10-15-22-09-47/usit_p0_05.27_reports
**哈希算法**：SHA-256
**采集工具**：sha256sum (coreutils) / stat -c %s (GNU)

本 MANIFEST 记录纳入交付 ZIP 的 12 份内容文件（9 份 Markdown 设计文档 + 3 份源码基线文件）的真实字节数与 SHA-256。MANIFEST 自身与 DELIVERY.zip 的哈希由 HASH-RECEIPT 与 ZIP-RECEIPT 记录，避免自引用循环。

| 文件 | 字节数 | SHA-256 |
|---|---|---|
| USIT-STAGE-B-DESIGN-05.27.md | 21474 | c2c93bb51e110b435dbc36a729a532d0d511243515b0f2f29fd1277597e496d6 |
| USIT-STAGE-B-DESIGN-05.27-ISSUES.md | 1929 | 13bf21201f9f68659579825e9b895a5d36270343fde815fe8f4f6b8a1cd62be0 |
| USIT-STAGE-B-DESIGN-05.27-DIFF.md | 2233 | 5a64148a27b6a90017312e7a005f1e568d338571e84cbdb3205b3e37443f3ea2 |
| USIT-STAGE-B-DESIGN-05.27-CROSS-REVIEW.md | 1650 | 11c001b024ae7b3de61c3b1cf738cb3f6a5501a9f75bbff4b155152352f28712 |
| USIT-STAGE-B-DESIGN-05.27-EXECUTION-REPORT.md | 3107 | 748407876ea2be412e72cf7c6a45ec625f59b727429df39ee5c3b385517022a7 |
| USIT-STAGE-B-DESIGN-05.27-TIMELINES.md | 12993 | 269fe48269f0dd064f62ff155f10430b10a9d6d4ff6f6d073483b0b252f72f97 |
| USIT-STAGE-B-DESIGN-05.27-COUNTEREXAMPLES.md | 4824 | e109f1944321db6ae786c2dc8255eac50808590917e309a22b99e246131b8e49 |
| USIT-STAGE-B-DESIGN-05.27-CROSS-MODULE-CHAINS.md | 2501 | 4d6dc34d4759b03ec40d9bf35f12e86f50a05ea4d906ea1c2e86bfb0e29da76e |
| USIT-STAGE-B-DESIGN-05.27-INVARIANTS.md | 5218 | 2bafcdfc2aa18e23288f9b8001148880e79a7c75b0d7066651327c3a5154471b |
| usit_source_baseline_05.27_pre.txt | 2465 | 4fcd389a24c8fb78f717a73e8dda6460e903bd06dad9dc77a43eab104a2f2c0b |
| usit_source_baseline_05.27_post.txt | 2466 | f09b69dd1ed93cc4c7756f685b88e37f67c4e82a90df93a6ff94c27730d0be81 |
| usit_source_baseline_05.27_diff.txt | 705 | 88fd703d21fd61643f4e543b37cf55151333de0dbf223a55b9f3d90637eb14cb |

## 源码基线一致性

PRE 与 POST 清单逐文件比对（`diff <(tail -n +4 pre) <(tail -n +4 post)`）结果为**空**，25 个 `.py` 文件的 SHA-256 与字节数全部一致。三个冻结文件与历史基线一致：

- scanner.py = 58ba5f25483429ce7482ae1d852deabce6ea0490f2e106c6955c6ff5323b45e3（MATCH）
- cli.py = 879a3735838e183f41dd08054095b514ffc0e81e2b43776b75a77e25d138b8d9（MATCH）
- file_understanding.py = 71995a09da04427c028282cfeba17c6b12ebdbcae97796adc2998af7c455ce5b（MATCH）

**结论**：本轮未修改任何 USIT 源码。
