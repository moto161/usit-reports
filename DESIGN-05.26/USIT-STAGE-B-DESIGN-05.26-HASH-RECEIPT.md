# USIT-STAGE-B-DESIGN-05.26 哈希回执（HASH-RECEIPT）

> 记录 MANIFEST 自身哈希 + ZIP 内文件独立解压核验结果。
> **ZIP 自身 SHA-256 与字节数见外部回执 `USIT-STAGE-B-DESIGN-05.26-ZIP-RECEIPT.md`（不纳入 ZIP，避免自引用循环）。**

## 1. MANIFEST 自身

| 文件 | 字节数 | SHA-256 |
|------|--------|---------|
| USIT-STAGE-B-DESIGN-05.26-HASH-MANIFEST.md | 1882 | ed4409b12c138a0bdbd8d981cd07f65d42f665f198da636f90de75b4b86b8787 |

## 2. ZIP 独立解压核验（真实执行）

解压工具：`unzip`（Info-ZIP，Windows 环境）
解压目录：`_verify/`（独立临时目录）

### 2.1 文件清单（10 个文件，无缺失/无额外/无重复）

> 下表列出 ZIP 内除本回执外的 9 份文件；本回执是第 10 份，其哈希见外部回执。

| 文件 | 长度 | 源文件字节数 | 逐字节比对 |
|------|------|-------------|-----------|
| USIT-STAGE-B-DESIGN-05.26.md | 37101 | 37101 | MATCH |
| USIT-STAGE-B-DESIGN-05.26-ISSUES.md | 4611 | 4611 | MATCH |
| USIT-STAGE-B-DESIGN-05.26-DIFF.md | 2375 | 2375 | MATCH |
| USIT-STAGE-B-DESIGN-05.26-CROSS-REVIEW.md | 1783 | 1783 | MATCH |
| USIT-STAGE-B-DESIGN-05.26-EXECUTION-REPORT.md | 2959 | 2959 | MATCH |
| USIT-STAGE-B-DESIGN-05.26-HASH-MANIFEST.md | 1882 | 1882 | MATCH |
| usit_source_baseline_05.26_pre.txt | 2465 | 2465 | MATCH |
| usit_source_baseline_05.26_post.txt | 2466 | 2466 | MATCH |
| usit_source_baseline_05.26_diff.txt | 885 | 885 | MATCH |

（CRC-32 逐项见外部回执）

### 2.2 安全核验

- 路径穿越/绝对路径：无；
- 大小写冲突：无；
- 符号链接：无；
- 重复/额外条目：无；
- 目录条目：无（仅文件）；
- 敏感信息：无；
- 临时文件/旧版本：无。

### 2.3 结论

**ZIP 独立解压核验：PASS。**

核验时间：2026-10-31（GMT+8）
核验工具：Info-ZIP unzip、GNU sha256sum、GNU cmp
