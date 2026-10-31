# CROSS-DOCUMENT-CONSISTENCY.md

## 目标
跨文档一致性报告。

## 1. 检查范围

05.33 全部交付文档 + 引用的 05.32 文档。

## 2. 冲突项及处理

| # | 冲突 | 处理 | 结果 |
|---|---|---|---|
| 1 | 05.32 coord 内调 _close_admit/_assert_no_commit 违反锁顺序 | 改用原子字 CAS，废除 coord 内业务锁获取 | 已消除 |
| 2 | 05.32 Wait-for Graph 说 coord 孤立但实际等业务归零 | 区分「条件等待」与「锁等待」，coord 在锁图中孤立、冻结者等条件变量 | 已消除 |
| 3 | 05.32 Receipt 锁 Fork 期间可能仍被持有 | 快照前置 receipt_write_count==0，明确隔离/阻止矩阵 | 已消除 |

## 3. 冲突数量

- 发现冲突：3 项（05.32 遗留）。
- 已处理：3 项。
- 未解释冲突：0。

## 4. 单一权威定义

| 主题 | 权威文档 |
|---|---|
| 冻结协议 | FORK-FREEZE-PROTOCOL.md |
| 准入原子性 | GATE-ADMISSION-ATOMICITY.md |
| 并发时序/线性化点 | FORK-CONCURRENCY-TIMELINE.md |
| 快照一致性 | BUSINESS-RECEIPT-SNAPSHOT-CONSISTENCY.md |
| 事务所有权 | FORK-TRANSACTION-OWNERSHIP.md |
| 锁顺序 | GLOBAL-LOCK-ORDER.md |
| 死锁分析 | LOCK-WAIT-FOR-GRAPH.md |

## 5. 05.32 安全约束继承检查

| 约束 | 继承 |
|---|---|
| 身份防回滚 NOT_PROVEN → 禁用 | ✅ |
| Receipt 认证 NOT_PROVEN → 禁用 | ✅ |
| 未认证 Receipt → QUARANTINED | ✅ |
| DENIED 不自动恢复 | ✅ |
| Break-glass 审计信任根 NOT_PROVEN → 禁用 | ✅ |
| 不将 NOT_PROVEN 改写为 DESIGN_PROVEN | ✅ |
| 不把静态推演写为运行时验证 | ✅ |

## 6. 结论

跨文档冲突已消除，无未解释冲突。05.32 安全约束全部继承，未弱化。

## 7. 证据等级
一致性检查 VERIFIED_STATIC；运行时 NOT_TESTED。
