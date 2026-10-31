# USIT-STAGE-B-DESIGN-05.27 — HASH-RECEIPT

**工单编号**：USIT-STAGE-B-DESIGN-05.27
**采集时间**：2026-10-31
**哈希算法**：SHA-256

本 RECEIPT 记录 MANIFEST 自身与 DELIVERY.zip 自身的真实字节数与 SHA-256，并记录 ZIP 独立解压、逐文件 CRC32/SHA-256、逐字节比较结果。ZIP 内文件的逐文件哈希由 ZIP-RECEIPT 详列。

## MANIFEST 自身

| 文件 | 字节数 | SHA-256 |
|---|---|---|
| USIT-STAGE-B-DESIGN-05.27-HASH-MANIFEST.md | 2652 | 63a8989b85b21fbfe13f8fde5dc4e1bb808f2e19b166ce36246ac0b2563174f5 |

## DELIVERY.zip 自身

| 文件 | 字节数 | SHA-256 |
|---|---|---|
| USIT-STAGE-B-DESIGN-05.27-DELIVERY.zip | 30839 | 1789c3785f5c33d0f2ae3164331559486fcc0e0252df54db2eebff150b047de8 |

## ZIP 独立解压核验

- 解压工具：unzip（GNU Info-ZIP）
- 解压命令：`unzip -o USIT-STAGE-B-DESIGN-05.27-DELIVERY.zip -d _verify/`
- 逐文件 SHA-256 / CRC32 / 逐字节比对结果：见 ZIP-RECEIPT.md
- 结果：14 文件全部 MATCH（逐字节一致）

## 说明

- RECEIPT 自身 SHA-256 不在本文件内自引用；ZIP 内不含 ZIP-RECEIPT 与 DELIVERY.zip 自身。
- 所有哈希均为本轮实际采集，未使用历史哈希或占位符。
