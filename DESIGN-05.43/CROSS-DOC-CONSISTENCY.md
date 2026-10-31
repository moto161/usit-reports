# CROSS-DOC-CONSISTENCY.md — 跨文档一致性（05.43）

工单：USIT-STAGE-B-DESIGN-05.43
工作包：B43-08（修复 B42-R09）
证据等级：STATIC_ANALYSIS

---

## 1. 问题编号统一（修复 B42-R09）

05.42 主文档与执行报告的问题编号映射不一致。05.43 统一为：

| 编号 | 名称 | 主文档 | 执行报告 | ISSUES | 反例引用 |
|---|---|---|---|---|---|
| B43-01 | WAL 恢复严格幂等 | ✓ | ✓ | ✓ | WAL_RECOVERY_IDEMPOTENCY |
| B43-02 | 提交前不变式验证 | ✓ | ✓ | ✓ | PRECOMMIT_VALIDATION |
| B43-03 | COMMIT 后异常与重试 | ✓ | ✓ | ✓ | POST_COMMIT_ERROR |
| B43-04 | txn_lock 唯一所有权 | ✓ | ✓ | ✓ | LOCK_OWNERSHIP |
| B43-05 | 资源端释放异常与恢复 | ✓ | ✓ | ✓ | RESOURCE_FAILURE |
| B43-06 | 资源端 Owner 授权 | ✓ | ✓ | ✓ | OWNER_AUTHORIZATION |
| B43-07 | Token 身份字段统一 | ✓ | ✓ | ✓ | TOKEN_IDENTITY |
| B43-08 | 跨文档编号一致性 | ✓ | ✓ | ✓ | — |
| B43-09 | 反例重新设计 | ✓ | ✓ | ✓ | 各类别 |

主文档与执行报告使用**完全相同**的编号与名称，无映射偏差。

---

## 2. 核心定义一致性核对

| 定义 | 权威来源 | 其他文档是否一致 |
|---|---|---|
| COMMIT = 提交点，APPLY = 进度标记（非变更触发点） | WAL-RECOVERY-IDEMPOTENCY §2 | TRANSACTION-PRECOMMIT-VALIDATION §2 一致 |
| 提交时序 = 构造候选→验证→PREPARE→COMMIT→APPLY | TRANSACTION-PRECOMMIT-VALIDATION §2 | 全文档唯一，无 05.42 的两种矛盾顺序 |
| 锁唯一持有层级 = run_txn | LOCK-OWNERSHIP-AND-CALL-GRAPH §2 | 其他文档引用一致 |
| Token 身份 = (slot, generation, owner_id, resource_id) | TOKEN-IDENTITY-CANONICAL-MODEL §2 | 其他文档引用一致 |
| state_version = 审计字段 | TOKEN-IDENTITY-CANONICAL-MODEL §2 | 与身份定义不冲突 |
| owner_id 来源 = 可信上下文注入 | RESOURCE-OWNER-AUTHORIZATION §2 | TOKEN-IDENTITY §4 一致 |
| COUNT == popcount(TOKEN_ACTIVE) 在 COMMIT 前验证 | TRANSACTION-PRECOMMIT-VALIDATION §5 | CRASH-AND-CONCURRENCY-MATRIX 一致 |

---

## 3. 已删除的 05.42 矛盾

| 矛盾 | 05.42 位置 | 05.43 处理 |
|---|---|---|
| 先 CAS 状态字 vs 先写 TRT | TRANSACTION-COMMIT-PROTOCOL（05.42 无此矛盾，05.41 有） | 已消除 |
| COMMIT 后回滚 vs 提交前验证 | STATE-INVARIANTS（05.42） | 统一为提交前验证 |
| COMMIT+APPLY 双记录重复 apply | TRANSACTION-CRASH-RECOVERY（05.42） | 只取 COMMIT apply 一次 |
| admit 持锁 + run_txn 持锁 | TOKEN-ALLOCATION-RELEASE-REUSE（05.42） | 唯一持有层级 run_txn |
| state_version 身份+审计矛盾 | TRANSACTION-DATA-MODEL §5（05.42） | 明确审计字段 |

---

## 4. 字段/状态/术语一致性

| 项 | 主文档 | 协议文档 | 反例 | 一致 |
|---|---|---|---|---|
| VERSION_EXHAUSTED_DENY | ✓ | ✓ | B43-VER-001 | ✓ |
| GENERATION_EXHAUSTED_DENY | ✓ | ✓ | B43-VER-002 | ✓ |
| ALREADY_RELEASED | ✓ | ✓ | B43-CONC-002 | ✓ |
| FENCED_OUT | ✓ | ✓ | B43-CONC-001 | ✓ |
| OWNER_MISMATCH | ✓ | ✓ | B43-OWN-001 | ✓ |
| STALE_TOKEN | ✓ | ✓ | B43-OWN-002 | ✓ |
| COMMITTED_PENDING_APPLY | ✓ | ✓ | B43-VAL-003 | ✓ |
| ISOLATED | ✓ | ✓ | B43-RES-001 | ✓ |

---

## 5. 统计与 HASH 一致性

- 反例 JSON：33 条，ID 唯一，统计与 COUNTEREXAMPLE-RESULTS.md 一致。
- 历史统计：HISTORICAL-COUNTEREXAMPLE-AUDIT.md 与 COUNTEREXAMPLE-RESULTS.md §6 一致。
- HASH：全部交付文件 SHA256 + 字节数，以 Git blob（LF 口径）计算，见 HASH-MANIFEST.txt 与 EXECUTION-REPORT.md。

---

## 6. 结论

跨文档编号、核心定义、状态术语、反例引用、统计数字全部一致，05.42 的编号映射偏差已修复。
