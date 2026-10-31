# USIT-STAGE-B-DESIGN-05.24 哈希回执（HASH-RECEIPT）

> 记录 MANIFEST 自身哈希 + ZIP 内文件独立解压核验结果。
> **ZIP 自身 SHA-256 与字节数见外部回执 `USIT-STAGE-B-DESIGN-05.24-ZIP-RECEIPT.txt`（不纳入 ZIP，避免自引用循环）。**

## 1. MANIFEST 自身

| 文件 | 字节数 | SHA-256 |
|------|--------|---------|
| USIT-STAGE-B-DESIGN-05.24-HASH-MANIFEST.md | 1859 | fd6e3ec6b16791b48776a8bbcdf42dde2c907a77f7a9f6bc9555b275033af98d |

## 2. ZIP 独立解压核验（真实执行）

解压工具：`unzip`（Info-ZIP，Windows 环境）
解压目录：`_verify/`（独立临时目录）

### 2.1 文件清单（10 个文件，无缺失/无额外/无重复）

> 下表列出 ZIP 内除本回执外的 9 份文件；本回执自身是第 10 份，其哈希见外部回执。

| 文件 | 长度 | CRC-32 | 源文件字节数 | 逐字节比对 |
|------|------|--------|-------------|-----------|
| USIT-STAGE-B-DESIGN-05.24.md | 54133 | 9d529c1b | 54133 | MATCH |
| USIT-STAGE-B-DESIGN-05.24-ISSUES.md | 2122 | 3ceba47b | 2122 | MATCH |
| USIT-STAGE-B-DESIGN-05.24-DIFF.md | 2982 | dc8e3afd | 2982 | MATCH |
| USIT-STAGE-B-DESIGN-05.24-CROSS-REVIEW.md | 1996 | 395951fc | 1996 | MATCH |
| USIT-STAGE-B-DESIGN-05.24-EXECUTION-REPORT.md | 3900 | daab46cf | 3900 | MATCH |
| USIT-STAGE-B-DESIGN-05.24-HASH-MANIFEST.md | 1859 | b100cc86 | 1859 | MATCH |
| usit_source_baseline_05.24_pre.txt | 2465 | 6b90bae2 | 2465 | MATCH |
| usit_source_baseline_05.24_post.txt | 2466 | 56c8616e | 2466 | MATCH |
| usit_source_baseline_05.24_diff.txt | 684 | b1608b36 | 684 | MATCH |

ZIP 内共 **10 个文件**（上表 9 份 + 本回执），总解压长度 74953 字节。

### 2.2 安全核验

- 路径穿越条目：无（所有条目为平铺文件名，无 `../` 或绝对路径）；
- 大小写冲突：无；
- 目录条目：无（仅文件）；
- 敏感信息：无（未包含 token、密钥、凭据）；
- 临时文件/旧版本：无。

### 2.3 结论

**ZIP 独立解压核验：PASS。**

核验时间：2026-10-31（GMT+8）
核验工具：Info-ZIP unzip、GNU sha256sum、GNU cmp
