# USIT-STAGE-B-DESIGN-05.21-HASH-RECEIPT｜哈希核验回执

> 阶段 B · DESIGN-05.21 · 哈希验证回执
> 生成：2026-10-31（GMT+8）
> 哈希算法：SHA-256

---

## 回执记录

| 项 | 值 |
|----|----|
| verification_status | NOT_VERIFIED |
| manifest_sha256 | baf11597d69f47530439124914d6d08b74a91dae62237fea98d0af0fa8dbf549 |
| manifest_bytes | 1486 |
| manifest_file | USIT-STAGE-B-DESIGN-05.21-HASH-MANIFEST.md |
| zip_sha256 | （见下，ZIP 打包后填入）|
| zip_bytes | （见下）|
| note_blob_sha | Git Blob SHA 不得冒充 SHA-256，本回执所有哈希均为 SHA-256 |
| note_zip | ZIP 自身 SHA-256 与字节数记录于本回执（ZIP 外部），避免循环自引用 |

## ZIP 自引用说明

ZIP 自身哈希不入 MANIFEST（避免循环自引用），记录于本 RECEIPT。ZIP 名：`USIT-STAGE-B-DESIGN-05.21-DELIVERY.zip`。

---