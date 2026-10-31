# USIT-STAGE-B-DESIGN-05.28 — ZIP-RECEIPT（外部回执）

**工单编号**：USIT-STAGE-B-DESIGN-05.28
**采集时间**：2026-10-31
**核验机器**：Windows（Git Bash 环境，sha256sum / unzip / cmp）

本文件为 ZIP 外部回执，记录 DELIVERY.zip 自身字节数与 SHA-256，以及 ZIP 内每一文件的字节数、CRC32、SHA-256，和逐字节比对结果。本文件不在 ZIP 内，避免自引用。

## ZIP 自身

| 文件 | 字节数 | SHA-256 |
|---|---|---|
| USIT-STAGE-B-DESIGN-05.28-DELIVERY.zip | 47826 | 3da7056787d074beae6bf9a78986e507af15231979f4f3528e4fa44ae9667004 |

## ZIP 内文件清单（解压后逐文件核验）

| 文件 | 字节数 | CRC32 | SHA-256 |
|---|---|---|---|
| USIT-STAGE-B-DESIGN-05.28.md | 38056 | a5b00028 | 9ab48cdda4bce4c1ba97d596d73c027539f75698ce9bb40cf99e89988fca314d |
| USIT-STAGE-B-DESIGN-05.28-ISSUES.md | 2094 | 0cc6fd4b | 54e76dc6141622ca9e674007ecbee4b26e5dc494ea80f9634a17ad67c8829e34 |
| USIT-STAGE-B-DESIGN-05.28-DIFF.md | 2171 | 9ae9c367 | c7804086ba256a9490d0c5c8b28d1fa7168bb6178d85502653eb456e25dd6bad |
| USIT-STAGE-B-DESIGN-05.28-CROSS-REVIEW.md | 2300 | e6625d93 | ff9c7da283d60e0881bdab05fab14fa22f50db72f629be450614477fba6a96a7 |
| USIT-STAGE-B-DESIGN-05.28-EXECUTION-REPORT.md | 3688 | ddc40278 | b8b475d932819e73c1d2b3a499daf1142e4e891275040640447669ac0e4a1b9f |
| USIT-STAGE-B-DESIGN-05.28-HASH-MANIFEST.md | 2626 | e8168b09 | b63cd8d47cde2deadc2c6678b7038bbac88d3cc4db9e4bea19de61e57333ec74 |
| USIT-STAGE-B-DESIGN-05.28-HASH-RECEIPT.md | 1362 | f8976f9e | 5074e0ad6f7e2af73d6a6f5efaca5703c98e1b411bce87526fef648b9e5b9f78 |
| USIT-STAGE-B-DESIGN-05.28-TIMELINES.md | 22611 | b4df4bd5 | 8fcef6b4d70ddff7667e0d2c4cf4afb9250b3343b2a88ce7b9bbf4993ac585fe |
| USIT-STAGE-B-DESIGN-05.28-COUNTEREXAMPLES.md | 19944 | 45a7828d | 9a501a80bfa89691cf5e1afd207cecd7b2eae905d1a0ee249b48a49738ee850d |
| USIT-STAGE-B-DESIGN-05.28-CROSS-MODULE-CHAINS.md | 7200 | fd502fc0 | d66653f21150f1ff69bd39a78923df0b6c737336d1a02784fdbf287173ac0106 |
| USIT-STAGE-B-DESIGN-05.28-INVARIANTS.md | 13071 | dc9d5b84 | 8888656747df1bad56259234d6bfa6a34fb08bdf44f9b1dbb7d3b9b22518192d |
| usit_source_baseline_05.28_pre.txt | 2465 | 1e526088 | 346955405adbcd7d1ef8b6a074e300a10c8e202fba4fc91c1d3c03ded79106fa |
| usit_source_baseline_05.28_post.txt | 2466 | 230abb04 | 6e9a58632876943a40d5a81e4cdb83cd988d92eaeee88998a49646340c01d751 |
| usit_source_baseline_05.28_diff.txt | 648 | 1b51fad3 | 157532e697b6c38377be9e5e762fbfb40ab3ca3a5f012ccb36abc9c85307405d |

## 核验结论

- 解压工具：unzip（GNU Info-ZIP）
- 解压目录：全新空目录 _verify/
- 逐字节比对：14/14 MATCH
- 路径穿越 / 绝对路径 / 大小写冲突 / 符号链接 / 重复条目 / 额外文件 / 敏感信息 / 临时文件 / 旧版本：未发现（14 个条目全部为预期交付文件，无多余/缺失/重复/路径穿越/绝对路径条目）

## 说明

所有哈希、字节数、CRC32 均为本轮实际采集，未引用历史数据或占位符。
