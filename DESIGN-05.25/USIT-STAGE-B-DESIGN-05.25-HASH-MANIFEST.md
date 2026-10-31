# USIT-STAGE-B-DESIGN-05.25 哈希清单（HASH-MANIFEST）

> 记录纳入交付的文档与基线文件 SHA-256 与字节数。所有哈希为最终写入后用 `sha256sum` 重算，非 Git Blob SHA-1。
> MANIFEST 自身哈希与 ZIP 哈希见 HASH-RECEIPT / ZIP-RECEIPT。

## 1. 设计文档（前 5 份）

| # | 文件名 | 字节数 | SHA-256 |
|---|--------|--------|---------|
| 1 | USIT-STAGE-B-DESIGN-05.25.md | 37523 | 9d11263fb871ca72dd5f9b217f515e810dba86dbe44a380dd9ca8aaafcbac7b0 |
| 2 | USIT-STAGE-B-DESIGN-05.25-ISSUES.md | 2039 | 8ee5be9e2cde3e9daabbdc3322b5b92f5b39c1dacc3ec6975253a04392b3cb2e |
| 3 | USIT-STAGE-B-DESIGN-05.25-DIFF.md | 2894 | e6a139d70aae57023e45cee84337ed6ce45151b23b2c027d077008c6edb0ce7b |
| 4 | USIT-STAGE-B-DESIGN-05.25-CROSS-REVIEW.md | 1736 | 2e37c05d2ce28b3c94edbb51a78430fcb928a9127c70dbd703b044021de8a922 |
| 5 | USIT-STAGE-B-DESIGN-05.25-EXECUTION-REPORT.md | 4204 | 6751c15d8cc0ca1a1f473b497ef0bfaa9e015f5eb5a6d8e17a89091946770050 |

## 2. 源码基线文件（3 份）

| # | 文件名 | 字节数 | SHA-256 |
|---|--------|--------|---------|
| 6 | usit_source_baseline_05.25_pre.txt | 2465 | c2fb7206d60a1fb1cf05ba6d479e23beb2d8d7d3a851196c61efacfd20ba0c3f |
| 7 | usit_source_baseline_05.25_post.txt | 2466 | 6e928d5d5119b5a31c1d9a2c82c77df6677a3f25245c571d0c23280f73093161 |
| 8 | usit_source_baseline_05.25_diff.txt | 885 | d3609b8c817d2e00cafaa1a0e2abf4e6280af3b5a353490391a80fcbd8f78db6 |

## 核验命令

```
sha256sum USIT-STAGE-B-DESIGN-05.25.md USIT-STAGE-B-DESIGN-05.25-ISSUES.md \
  USIT-STAGE-B-DESIGN-05.25-DIFF.md USIT-STAGE-B-DESIGN-05.25-CROSS-REVIEW.md \
  USIT-STAGE-B-DESIGN-05.25-EXECUTION-REPORT.md \
  usit_source_baseline_05.25_pre.txt usit_source_baseline_05.25_post.txt \
  usit_source_baseline_05.25_diff.txt
```

核验时间：2026-10-31（GMT+8）
核验工具：GNU sha256sum（Windows 环境）
