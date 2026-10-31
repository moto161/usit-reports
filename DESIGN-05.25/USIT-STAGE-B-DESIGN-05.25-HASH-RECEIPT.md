# USIT-STAGE-B-DESIGN-05.25 哈希回执（HASH-RECEIPT）

> 记录 MANIFEST 自身哈希 + ZIP 内文件独立解压核验结果。
> **ZIP 自身 SHA-256 与字节数见外部回执 `USIT-STAGE-B-DESIGN-05.25-ZIP-RECEIPT.txt`（不纳入 ZIP，避免自引用循环）。**

## 1. MANIFEST 自身

| 文件 | 字节数 | SHA-256 |
|------|--------|---------|
| USIT-STAGE-B-DESIGN-05.25-HASH-MANIFEST.md | 1882 | bed1233cb0c6c064c0c5746bbef7e600640854e483db7b847437a38b7a3232b2 |

## 2. ZIP 独立解压核验（真实执行）

解压工具：`unzip`（Info-ZIP，Windows 环境）
解压目录：`_verify/`（独立临时目录）

### 2.1 文件清单（10 个文件，无缺失/无额外/无重复）

> 下表列出 ZIP 内除本回执外的 9 份文件；本回执是第 10 份，其哈希见外部回执。

| 文件 | 长度 | 源文件字节数 | 逐字节比对 |
|------|------|-------------|-----------|
| USIT-STAGE-B-DESIGN-05.25.md | 37523 | 37523 | MATCH |
| USIT-STAGE-B-DESIGN-05.25-ISSUES.md | 2039 | 2039 | MATCH |
| USIT-STAGE-B-DESIGN-05.25-DIFF.md | 2894 | 2894 | MATCH |
| USIT-STAGE-B-DESIGN-05.25-CROSS-REVIEW.md | 1736 | 1736 | MATCH |
| USIT-STAGE-B-DESIGN-05.25-EXECUTION-REPORT.md | 4204 | 4204 | MATCH |
| USIT-STAGE-B-DESIGN-05.25-HASH-MANIFEST.md | 1882 | 1882 | MATCH |
| usit_source_baseline_05.25_pre.txt | 2465 | 2465 | MATCH |
| usit_source_baseline_05.25_post.txt | 2466 | 2466 | MATCH |
| usit_source_baseline_05.25_diff.txt | 885 | 885 | MATCH |

（CRC-32 逐项见外部回执）

### 2.2 安全核验

- 路径穿越条目：无；
- 大小写冲突：无；
- 符号链接：无；
- 目录条目：无（仅文件）；
- 敏感信息：无；
- 临时文件/旧版本：无。

### 2.3 结论

**ZIP 独立解压核验：PASS。**

核验时间：2026-10-31（GMT+8）
核验工具：Info-ZIP unzip、GNU sha256sum、GNU cmp
