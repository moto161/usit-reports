# CROSS-DOC-CONSISTENCY.md — 跨文档一致性（05.44）

工单：USIT-STAGE-B-DESIGN-05.44
工作包：B44-09（反例一致性）
证据等级：STATIC_ANALYSIS

---

## 1. 问题编号统一（沿用 05.43 规则）

05.44 统一为：

| 编号 | 名称 | 主文档 | 执行报告 | ISSUES | 反例引用 |
|---|---|---|---|---|---|
| B44-01 | 事务 ID 与提交顺序 | ✓ | ✓ | ✓ | TXN_ORDER_AND_WAL_REPLAY |
| B44-02 | req_id 原子幂等 | ✓ | ✓ | ✓ | REQUEST_ID_ATOMIC_IDEMPOTENCY |
| B44-03 | 恢复屏障与事务准入 | ✓ | ✓ | ✓ | RECOVERY_BARRIER_AND_ADMISSION |
| B44-04 | 资源端释放与接管串行化 | ✓ | ✓ | ✓ | RESOURCE_RELEASE_SERIALIZATION |
| B44-05 | COMMIT 后异常、WAL 完整性、状态查询 | ✓ | ✓ | ✓ | POST_COMMIT_FAILURE_CLOSURE |
| B44-09 | 反例重新设计 | ✓ | ✓ | ✓ | COUNTEREXAMPLE_REDESIGN |

主文档与执行报告使用**完全相同**的编号与名称，无映射偏差。

---

## 2. 核心定义一致性核对

| 定义 | 权威来源 | 其他文档是否一致 |
|---|---|---|
| commit_seq 为唯一提交顺序权威 | TXN-ORDER-AND-WAL-REPLAY.md §2 | POST-COMMIT-FAILURE-CLOSURE.md §2 一致 |
| COMMIT 记录保存完整候选快照 | TXN-ORDER-AND-WAL-REPLAY.md §2 | POST-COMMIT-FAILURE-CLOSURE.md §3 一致 |
| req_id 原子幂等查询在锁内 | REQUEST-ID-ATOMIC-IDEMPOTENCY.md §2 | RESOURCE-RELEASE-SERIALIZATION.md §2 一致 |
| 恢复屏障 = barrier_lock + 计数器 | RECOVERY-BARRIER-AND-ADMISSION.md §2 | 其他文档引用一致 |
| Token 身份 = (slot, generation, owner_id, resource_id) | TOKEN-IDENTITY-CANONICAL-MODEL.md §2 | 其他文档引用一致 |
| 原子发布 = txn_guard 内同步更新 + 读锁/快照 | POST-COMMIT-FAILURE-CLOSURE.md §2 | 其他文档引用一致 |

---

## 3. 已删除的 05.43 矛盾

| 矛盾 | 05.43 位置 | 05.44 处理 |
|---|---|---|
| txn_id 锁外分配 → 恢复按 txn_id | TRANSACTION-PRECOMMIT-VALIDATION.md（05.43） | commit_seq 锁内分配，恢复按 commit_seq |
| req_id 锁外查询 → 并发重复提交 | POST-COMMIT-ERROR-AND-RETRY.md（05.43/05.43 沿用） | 锁内查询 + 原子幂等 |
| in_recovery 布尔 TOCTOU | LOCK-OWNERSHIP-AND-CALL-GRAPH.md（05.43） | barrier_lock + 计数器 |
| 幂等查询并发重复 operation | RESOURCE-FAILURE-AND-RECOVERY.md（05.43） | 资源锁串行化 |
| mark_committed_pending_apply 无第二层处理 | POST-COMMIT-ERROR-AND-RETRY.md（05.43） | 三层失败处理 |

---

## 4. 状态术语一致性

| 项 | 主文档 | 协议文档 | 反例 | 一致 |
|---|---|---|---|---|
| VERSION_EXHAUSTED_DENY | ✓ | ✓ | B44-ORD-003 | ✓ |
| GENERATION_EXHAUSTED_DENY | ✓ | ✓ | — | ✓ |
| ALREADY_RELEASED | ✓ | ✓ | B44-RES-001 | ✓ |
| FENCED_OUT | ✓ | ✓ | — | ✓ |
| OWNER_MISMATCH | ✓ | ✓ | B44-ID-003 | ✓ |
| STALE_TOKEN | ✓ | ✓ | — | ✓ |
| COMMITTED_PENDING_APPLY | ✓ | ✓ | B44-PC-001 | ✓ |
| ISOLATED | ✓ | ✓ | B44-ORD-003 | ✓ |
| CONFLICT_xxx | ✓ | ✓ | B44-ID-002～005 | ✓ |
| RECOVERY_BARRIER | ✓ | ✓ | B44-BAR-001 | ✓ |

---

## 5. 统计与 HASH 一致性

- 反例 JSON：26 条，ID 唯一，统计与 COUNTEREXAMPLE-RESULTS.md 一致。
- 历史统计：HISTORICAL-COUNTEREXAMPLE-AUDIT.md 与 COUNTEREXAMPLE-RESULTS.md §6 一致。
- HASH：全部交付文件 SHA256 + 字节数，以 Git blob（LF 口径）计算，见 HASH-MANIFEST.txt 与 EXECUTION-REPORT.md。

---

## 6. 结论

跨文档编号、核心定义、状态术语、反例引用、统计数字全部一致。
