# USIT-STAGE-B-DESIGN-05.23 哈希回执（HASH-RECEIPT）

> 记录 MANIFEST 自身哈希 + ZIP 内 7 文件独立解压核验结果。
> **ZIP 自身 SHA-256 与字节数见外部回执 `USIT-STAGE-B-DESIGN-05.23-ZIP-RECEIPT.txt`（不纳入 ZIP，避免自引用循环）。**

## 1. MANIFEST 自身

| 文件 | 字节数 | SHA-256 |
|------|--------|---------|
| USIT-STAGE-B-DESIGN-05.23-HASH-MANIFEST.md | 1293 | 164a34e350a4a08c931c06029a8e89b74eecd3cf0797f139772d58f78b734a5d |

## 2. ZIP 独立解压核验结果（真实执行）

解压工具：`unzip`（Info-ZIP，Windows 环境）
解压目录：`_verify/`（独立临时目录）

### 2.1 文件清单（7 个文件，无缺失/无额外/无重复）

> 下表列出 ZIP 内除本回执外的 6 份文档；本回执自身哈希见外部回执，避免自引用。

| 文件 | 长度 | 压缩后 | CRC-32 | 源文件字节数 | 逐字节比对 |
|------|------|--------|--------|-------------|-----------|
| USIT-STAGE-B-DESIGN-05.23.md | 38231 | 12410 | 1919e71e | 38231 | MATCH |
| USIT-STAGE-B-DESIGN-05.23-ISSUES.md | 1897 | 1151 | c65fe205 | 1897 | MATCH |
| USIT-STAGE-B-DESIGN-05.23-DIFF.md | 2700 | 1610 | 994c276f | 2700 | MATCH |
| USIT-STAGE-B-DESIGN-05.23-CROSS-REVIEW.md | 1899 | 1163 | 1fb32178 | 1899 | MATCH |
| USIT-STAGE-B-DESIGN-05.23-EXECUTION-REPORT.md | 4148 | 2004 | 7917489b | 4148 | MATCH |
| USIT-STAGE-B-DESIGN-05.23-HASH-MANIFEST.md | 1293 | 738 | 0b82c96f | 1293 | MATCH |

合计：7 个文件（含本回执），总解压长度见 `unzip -l` 输出（52110 字节）。

### 2.2 结论

- ZIP 内文件数量 = 7，与源交付文件数量一致；
- 无缺失文件、无额外文件、无重复文件；
- 每文件解压后与源交付文件逐字节比对 **全部 MATCH**；
- CRC-32 由 unzip 校验通过（解压过程 CRC 校验无报错）；
- SHA-256 与 MANIFEST 一致。

**ZIP 独立解压核验：PASS。**

核验时间：2026-10-31（GMT+8）
核验工具：Info-ZIP unzip、GNU sha256sum、GNU cmp
