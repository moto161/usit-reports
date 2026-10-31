# USIT-STAGE-B-DESIGN-05.28 — HASH-RECEIPT

**工单编号**：USIT-STAGE-B-DESIGN-05.28
**采集时间**：2026-10-31
**哈希算法**：SHA-256

本 RECEIPT 记录 MANIFEST 自身的真实字节数与 SHA-256，以及 ZIP 独立解压、逐文件 CRC32/SHA-256、逐字节比较结果。ZIP 自身哈希仅由外部 ZIP-RECEIPT 记录（避免 ZIP 内自引用循环）。ZIP 内文件逐文件哈希由 ZIP-RECEIPT 详列。

## MANIFEST 自身

| 文件 | 字节数 | SHA-256 |
|---|---|---|
| USIT-STAGE-B-DESIGN-05.28-HASH-MANIFEST.md | 2626 | b63cd8d47cde2deadc2c6678b7038bbac88d3cc4db9e4bea19de61e57333ec74 |

## DELIVERY.zip 自身（由外部 ZIP-RECEIPT 记录，本表不重复）

ZIP 自身哈希见 `USIT-STAGE-B-DESIGN-05.28-ZIP-RECEIPT.md`（外部回执，不在 ZIP 内）。

## ZIP 独立解压核验

- 解压工具：unzip（GNU Info-ZIP）
- 解压命令：`unzip -o USIT-STAGE-B-DESIGN-05.28-DELIVERY.zip -d _verify/`
- 逐文件 SHA-256 / CRC32 / 逐字节比对：见 ZIP-RECEIPT.md
- 结果：14 文件全部 MATCH（逐字节一致）

## 说明

- RECEIPT 自身 SHA-256 不在本文件内自引用；ZIP 自身哈希由外部 ZIP-RECEIPT 记录，不在本文件内（避免自引用循环）。
- ZIP 内不含 ZIP-RECEIPT 与 DELIVERY.zip 自身。
- 所有哈希均为本轮实际采集，未使用历史哈希或占位符。
