# USIT-STAGE-B-DESIGN-05.24 哈希清单（HASH-MANIFEST）

> 记录纳入交付的文档文件 SHA-256 与字节数。所有哈希为最终写入后用 `sha256sum` 重算，非 Git Blob SHA-1。
> MANIFEST 自身哈希与 ZIP 哈希见 HASH-RECEIPT。

## 1. 设计文档（前 5 份）

| # | 文件名 | 字节数 | SHA-256 |
|---|--------|--------|---------|
| 1 | USIT-STAGE-B-DESIGN-05.24.md | 54133 | 2fca7811a487425bdfc030fc7138d130aeb77d10302492ca34098cebea1baf74 |
| 2 | USIT-STAGE-B-DESIGN-05.24-ISSUES.md | 2122 | 9824fa59e681577a467a87c1b3090053e18559134c096353e6594583dd301e10 |
| 3 | USIT-STAGE-B-DESIGN-05.24-DIFF.md | 2982 | 6907a1f6d6346a81bd7b6b2c4c5e4a482c0d57a42e3bd25638e703eec93814d7 |
| 4 | USIT-STAGE-B-DESIGN-05.24-CROSS-REVIEW.md | 1996 | ed318949584b63b903c2cfce112dcbb62e790bc7e364ee0a99062011e1b1da5a |
| 5 | USIT-STAGE-B-DESIGN-05.24-EXECUTION-REPORT.md | 3900 | be1392c5aa9d091c8b3059208cccb23c2f020815dc2fe342604f12c10d4da135 |

## 2. 源码基线文件（3 份）

| # | 文件名 | 字节数 | SHA-256 |
|---|--------|--------|---------|
| 6 | usit_source_baseline_05.24_pre.txt | 2465 | 0a498a1bafd53770656910d0f6d1ba7737481719b600c961cacf6f14f5592a77 |
| 7 | usit_source_baseline_05.24_post.txt | 2466 | 9b1eb26ec335b8a816d5683ca17c84837cde8dfaac144b1dd2e8010b0f1ae30b |
| 8 | usit_source_baseline_05.24_diff.txt | 684 | c0c4dd7014757c492693ab654610cd8c3ff21f0e4b9d3326e80006e751c8de6d |

## 核验命令

```
sha256sum USIT-STAGE-B-DESIGN-05.24.md USIT-STAGE-B-DESIGN-05.24-ISSUES.md \
  USIT-STAGE-B-DESIGN-05.24-DIFF.md USIT-STAGE-B-DESIGN-05.24-CROSS-REVIEW.md \
  USIT-STAGE-B-DESIGN-05.24-EXECUTION-REPORT.md \
  usit_source_baseline_05.24_pre.txt usit_source_baseline_05.24_post.txt \
  usit_source_baseline_05.24_diff.txt
```

核验时间：2026-10-31（GMT+8）
核验工具：GNU sha256sum（Windows 环境）
