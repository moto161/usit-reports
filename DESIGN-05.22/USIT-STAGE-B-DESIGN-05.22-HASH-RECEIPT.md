# USIT-STAGE-B-DESIGN-05.22-HASH-RECEIPT｜哈希核验回执

> 阶段 B · DESIGN-05.22 · 哈希验证回执
> 生成：2026-10-31（GMT+8）
> 哈希算法：SHA-256

---

## 回执记录

| 项 | 值 |
|----|----|
| verification_status | NOT_VERIFIED |
| manifest_sha256 | df91557a9c338a9a56dcc1ea9820de2154b9407fb02aaa6248ad4f2cd8eaa88f |
| manifest_bytes | 1486 |
| manifest_file | USIT-STAGE-B-DESIGN-05.22-HASH-MANIFEST.md |
| zip_sha256 | （见下，ZIP 打包后填入）|
| zip_bytes | （见下）|
| verify_time | 2026-10-31 12:40 GMT+8 |
| zip_extract | （见下）|
| note_blob_sha | Git Blob SHA 不得冒充 SHA-256，本回执所有哈希均为 SHA-256 |
| note_zip | ZIP 自身 SHA-256 与字节数记录于本回执（ZIP 外部），避免循环自引用 |

## ZIP 独立解压与核验结果

（打包后填入：文件数量、每个文件名称/字节数/SHA-256/CRC32、与原始交付文件逐字节比较结果、核验工具与版本）

核验工具：`unzip`（Info-ZIP）、`sha256sum`（GNU coreutils）、`powershell Compress-Archive`。

---