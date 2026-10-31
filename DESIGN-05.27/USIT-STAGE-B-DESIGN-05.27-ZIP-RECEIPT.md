# USIT-STAGE-B-DESIGN-05.27 — ZIP-RECEIPT（外部回执）

**工单编号**：USIT-STAGE-B-DESIGN-05.27
**采集时间**：2026-10-31
**核验机器**：Windows（Git Bash 环境，sha256sum / unzip / cmp）

本文件为 ZIP 外部回执，记录 DELIVERY.zip 自身字节数与 SHA-256，以及 ZIP 内每一文件的字节数、CRC32、SHA-256，和逐字节比对结果。本文件不在 ZIP 内，避免自引用。

## ZIP 自身

| 文件 | 字节数 | SHA-256 |
|---|---|---|
| USIT-STAGE-B-DESIGN-05.27-DELIVERY.zip | 30839 | 1789c3785f5c33d0f2ae3164331559486fcc0e0252df54db2eebff150b047de8 |

## ZIP 内文件清单（解压后逐文件核验）

| 文件 | 字节数 | CRC32 | SHA-256 |
|---|---|---|---|
| USIT-STAGE-B-DESIGN-05.27.md | 21474 | d4af4d66 | c2c93bb51e110b435dbc36a729a532d0d511243515b0f2f29fd1277597e496d6 |
| USIT-STAGE-B-DESIGN-05.27-ISSUES.md | 1929 | 661a57a1 | 13bf21201f9f68659579825e9b895a5d36270343fde815fe8f4f6b8a1cd62be0 |
| USIT-STAGE-B-DESIGN-05.27-DIFF.md | 2233 | 8478306e | 5a64148a27b6a90017312e7a005f1e568d338571e84cbdb3205b3e37443f3ea2 |
| USIT-STAGE-B-DESIGN-05.27-CROSS-REVIEW.md | 1650 | b73605bf | 11c001b024ae7b3de61c3b1cf738cb3f6a5501a9f75bbff4b155152352f28712 |
| USIT-STAGE-B-DESIGN-05.27-EXECUTION-REPORT.md | 3107 | db636338 | 748407876ea2be412e72cf7c6a45ec625f59b727429df39ee5c3b385517022a7 |
| USIT-STAGE-B-DESIGN-05.27-HASH-MANIFEST.md | 2652 | ae9c879d | 63a8989b85b21fbfe13f8fde5dc4e1bb808f2e19b166ce36246ac0b2563174f5 |
| USIT-STAGE-B-DESIGN-05.27-HASH-RECEIPT.md | 1226 | ab2fd865 | 190ad6f0b337d0455a570d1ff13b5e6c35deb14ba6bd89db31d0f4c4bec2c412 |
| USIT-STAGE-B-DESIGN-05.27-TIMELINES.md | 12993 | 0d43bda8 | 269fe48269f0dd064f62ff155f10430b10a9d6d4ff6f6d073483b0b252f72f97 |
| USIT-STAGE-B-DESIGN-05.27-COUNTEREXAMPLES.md | 4824 | 07a8d3d3 | e109f1944321db6ae786c2dc8255eac50808590917e309a22b99e246131b8e49 |
| USIT-STAGE-B-DESIGN-05.27-CROSS-MODULE-CHAINS.md | 2501 | fb5df8b7 | 4d6dc34d4759b03ec40d9bf35f12e86f50a05ea4d906ea1c2e86bfb0e29da76e |
| USIT-STAGE-B-DESIGN-05.27-INVARIANTS.md | 5218 | 41761b0a | 2bafcdfc2aa18e23288f9b8001148880e79a7c75b0d7066651327c3a5154471b |
| usit_source_baseline_05.27_pre.txt | 2465 | 9b588f58 | 4fcd389a24c8fb78f717a73e8dda6460e903bd06dad9dc77a43eab104a2f2c0b |
| usit_source_baseline_05.27_post.txt | 2466 | a60054d4 | f09b69dd1ed93cc4c7756f685b88e37f67c4e82a90df93a6ff94c27730d0be81 |
| usit_source_baseline_05.27_diff.txt | 705 | 087e6dbc | 88fd703d21fd61643f4e543b37cf55151333de0dbf223a55b9f3d90637eb14cb |

## 核验结论

- 解压工具：unzip（GNU Info-ZIP）
- 解压目录：全新空目录 _verify/
- 逐字节比对：14/14 MATCH
- 路径穿越 / 绝对路径 / 大小写冲突 / 符号链接 / 重复条目 / 额外文件 / 敏感信息 / 临时文件 / 旧版本：未发现（14 个条目全部为预期交付文件，无多余/缺失/重复/路径穿越/绝对路径条目）

## 说明

所有哈希、字节数、CRC32 均为本轮实际采集，未引用历史数据或占位符。
