# USIT-STAGE-B-DESIGN-05.17-HASH-MANIFEST

> 交付文件 SHA-256 与字节数清单
> 生成：2026-10-31（GMT+8）
> 验证方法：`sha256sum`（本地自检，非第三方独立验证）

## 前五份文档 SHA-256 与字节数

| 文件 | 字节数 | SHA-256 |
|------|--------|---------|
| USIT-STAGE-B-DESIGN-05.17.md | 40131 | d9f9aec3d33a81682a7188cf4fa16a03063ec541251827c6d80e0ff36d929ae9 |
| USIT-STAGE-B-DESIGN-05.17-ISSUES.md | 3008 | 07b766aefd02041cf1f195ca797ae54329321d3d8a73fb65d5d86ed27276c1c3 |
| USIT-STAGE-B-DESIGN-05.17-DIFF.md | 2081 | 07ec9bd401de7e57b22f2c98a1474e27de5d29c1797198930f9618ca4e40a8be |
| USIT-STAGE-B-DESIGN-05.17-CROSS-REVIEW.md | 2231 | 575fe7efc3c03df55ef6a1f757d264184205c590dd79363c3de86a5ccc6fcdd5 |
| USIT-STAGE-B-DESIGN-05.17-EXECUTION-REPORT.md | 3782 | 5c45f70679548fcafa53d146a18a7061a3be1d339e08244e71f684413ebf764a |

## 分层清单说明（解决 ZIP 自引用）

采用**分层清单 + 独立外部封存回执**，避免循环引用：

- **第 1 层**：本 MANIFEST 记录前五份文档 SHA-256。
- **第 2 层**：HASH-RECEIPT.txt 记录本 MANIFEST 的 SHA-256（外部封存，非自引用）。
- **第 3 层**：ZIP 的 SHA-256 在打包后单独计算，记录在 RECEIPT 与最终答复。

不声称任何"循环自校验"。

## 说明

- 验证方法为本地自检（`sha256sum`），**非第三方独立验证**，verification_status = NOT_VERIFIED。
- Git Blob SHA 不作为文件 SHA-256 使用。
