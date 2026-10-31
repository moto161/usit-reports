# CROSS-DOCUMENT-CONSISTENCY.md

## 目标
跨文档一致性报告，列出冲突数量及处理结果。

## 1. 检查范围

对以下文档进行交叉一致性检查：
- USIT-STAGE-B-DESIGN-05.31.md（主）
- FORK-PARENT-CHILD-ISOLATION.md
- FORK-LOCK-ORDER-UNIFIED.md
- IDENTITY-ANTI-ROLLBACK.md
- RECEIPT-AUTH-BOUNDARY.md
- DENIED-BREAK-GLASS-STATE-MACHINE.md
- GLOBAL-PROTOCOL-INDEX.md
- 以及引用到的 05.30 文档（PARENT-RESTORE-PROTOCOL、FORK-LOCK-OWNERSHIP、FORK-LOCK-LIFECYCLE、LOCK-ORDER-PROOF、PROCESS-TREE-IDENTITY、RECEIPT-AUTHENTICITY、RECEIPT-IDEMPOTENCY、DENIED-STATE-MACHINE、INVARIANTS）

## 2. 冲突项及处理

| # | 冲突 | 处理 | 结果 |
|---|---|---|---|
| 1 | 05.30「DENIED 永久不可逆」vs「manual_reenable 可恢复」 | 重定义为「对普通路径不可逆；Break-glass 管理协议可解除」 | 已消除 |
| 2 | 05.30 锁顺序「coord 不与业务锁嵌套」vs 伪代码隐式嵌套 | 统一到 FORK-LOCK-ORDER-UNIFIED 权威定义 | 已消除 |
| 3 | 05.30 身份「树内严格唯一」vs 快照回滚可回退序号 | 降级为「单次运行内严格；跨快照 NOT_PROVEN」 | 已消除 |
| 4 | 05.30「Receipt 四元组」vs「认证分离」 | 统一到 RECEIPT-AUTH-BOUNDARY 权威定义 | 已消除 |
| 5 | 锁重建/释放/所有权混为一谈 | 在 FORK-LOCK-ORDER-UNIFIED 明确区分 | 已消除 |
| 6 | 分支路径编码歧义 | 定长编码规则明确 | 已消除 |

## 3. 冲突数量统计

- 发现冲突：6 项（均为 05.30 遗留）。
- 已处理：6 项。
- 未解释冲突：**0**。

## 4. 剩余重复定义 / 遗漏 / 未经证明假设

| 项 | 状态 |
|---|---|
| 锁顺序重复定义 | 已收敛到单一权威 |
| 身份唯一性边界 | 已明确（单次 vs 跨快照） |
| Receipt 认证 | 如实 NOT_PROVEN，不假设 |
| 防回滚锚点 | 如实 NOT_PROVEN，不假设 |
| Break-glass 双人审批 | 明确为设计约束（DESIGN_PROPOSAL，未实现） |

## 5. 结论

跨文档协议冲突已全部消除，无未解释冲突。身份防回滚与 Receipt 认证如实标注 NOT_PROVEN，未做未经证明的假设。

## 6. 证据等级
一致性检查：VERIFIED_STATIC（静态核对）；运行时：NOT_TESTED。
