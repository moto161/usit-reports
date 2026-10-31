# CROSS-DOC-CONSISTENCY.md — 跨文档一致性（05.46）

工单：USIT-STAGE-B-DESIGN-05.46
工作包：B46-09（反例一致性）
证据等级：STATIC_ANALYSIS

---

## 1. 问题编号统一（沿用 05.45 规则）

05.46 统一为：

| 编号 | 名称 | 主文档 | 执行报告 | ISSUES | 反例引用 |
|---|---|---|---|---|---|
| B46-01 | 恢复屏障 | ✓ | ✓ | ✓ | RECOVERY_BARRIER |
| B46-02 | 全局请求幂等 | ✓ | ✓ | ✓ | REQUEST_ID_ATOMIC_IDEMPOTENCY |
| B46-03 | operation 预留/确认 | ✓ | ✓ | ✓ | RESOURCE_OPERATION_CRASH_RECOVERY |
| B46-04 | WAL 与检查点 | ✓ | ✓ | ✓ | WAL_INTEGRITY_AND_CHECKPOINT |
| B46-05 | 原子发布、查询与隔离持久化 | ✓ | ✓ | ✓ | POST_COMMIT_FAILURE_CLOSURE |
| B46-06 | 反例重新设计 | ✓ | ✓ | ✓ | 各类别 |

主文档与执行报告使用**完全相同**的编号与名称，无映射偏差。

---

## 2. 核心定义一致性核对

| 定义 | 权威来源 | 其他文档是否一致 |
|---|---|---|
| commit_seq 为唯一提交顺序权威（锁内分配） | TXN-ORDER-AND-WAL-REPLAY.md §2 | POST-COMMIT-FAILURE-CLOSURE.md §2 一致 |
| PREPARE 无 commit_seq，不参与排序 | TXN-ORDER-AND-WAL-REPLAY.md §2 | 其他文档引用一致 |
| commit_seq 耗尽保护 | TXN-ORDER-AND-WAL-REPLAY.md §2 | 其他文档引用一致 |
| 检查点含完整状态快照 + req_id_index + operation_state_index + wal_position + snapshot_hash | WAL-INTEGRITY-AND-CHECKPOINT.md §2.1 | 其他文档引用一致 |
| 恢复入口 = 统一 recover() | WAL-INTEGRITY-AND-CHECKPOINT.md §2.2 | 其他文档引用一致 |
| 恢复屏障 = Condition + 单一 `barrier_cond` + 上下文管理器 | RECOVERY-BARRIER-AND-ADMISSION.md §2 | 其他文档引用一致 |
| 全局请求锁 = global_request_lock → resource_lock → txn_guard | REQUEST-ID-ATOMIC-IDEMPOTENCY.md §2.3 | 其他文档引用一致 |
| 状态转换完整七状态 = RESERVED_NOT_STARTED/OPERATION_STARTED/OPERATION_CONFIRMED/COMMITTED/RELEASED/UNKNOWN/ISOLATED | RESOURCE-OPERATION-CRASH-RECOVERY.md §2.2 | 其他文档引用一致 |
| 原子发布 = ImmutableSnapshot + 单指针切换 | POST-COMMIT-FAILURE-CLOSURE.md §2.1 | 其他文档引用一致 |
| baseline_status/revision_status 分离 | COUNTEREXAMPLES.json | COUNTEREXAMPLE-RESULTS.md 一致 |

---

## 3. 已删除的 05.45 矛盾

| 矛盾 | 05.45 位置 | 05.46 处理 |
|---|---|---|
| admission_cond 与 recovery_cond 分离（死锁） | RECOVERY-BARRIER-AND-ADMISSION.md §3 | 单一 `barrier_cond`，通知/等待同一 Condition |
| barrier_flag 跨锁读写（TOCTOU） | RECOVERY-BARRIER-AND-ADMISSION.md §1 | 单一 `barrier_cond` 保护 |
| `op`、`payload_hash`、`resource` 未定义变量 | REQUEST-ID-ATOMIC-IDEMPOTENCY.md §3 | 完整参数定义 |
| `start_req()` 未调用 WAL 事务持久化 | REQUEST-ID-ATOMIC-IDEMPOTENCY.md §2.2 | WAL 事务持久化 |
| operation 先执行后查 req_id | REQUEST-ID-ATOMIC-IDEMPOTENCY.md §3 | 先占用 req_id，后执行 operation |
| OP_RESERVATION 与 OP_CONFIRM 语义矛盾 | RESOURCE-OPERATION-CRASH-RECOVERY.md §2 | 七状态状态机 |
| CHECKPOINT 无快照/索引/checkpoint 双入口 | WAL-INTEGRITY-AND-CHECKPOINT.md（05.45 旧） | 完整快照 + 索引 + 摘要，统一 recover() 单一入口 |
| publish 锁内依次赋值/StateSnapshot 未冻结 | POST-COMMIT-FAILURE-CLOSURE.md §2 | ImmutableSnapshot + 受保护访问 |
| query_txn() 未校验唯一 COMMIT/APPLY | POST-COMMIT-FAILURE-CLOSURE.md §2 | 唯一 COMMIT + 唯一 APPLY + commit_seq 一致性 |

---

## 4. 状态术语一致性

| 项 | 主文档 | 协议文档 | 反例 | 一致 |
|---|---|---|---|---|
| VERSION_EXHAUSTED_DENY | ✓ | ✓ | — | ✓ |
| GENERATION_EXHAUSTED_DENY | ✓ | ✓ | — | ✓ |
| COMMIT_SEQ_EXHAUSTED | ✓ | ✓ | — | ✓ |
| ALREADY_PROCESSED | ✓ | ✓ | B46-ID-002 | ✓ |
| FENCED_OUT | ✓ | ✓ | — | ✓ |
| OWNER_MISMATCH | ✓ | ✓ | B46-ID-004 | ✓ |
| STALE_TOKEN | ✓ | ✓ | — | ✓ |
| COMMITTED_PENDING_APPLY | ✓ | ✓ | B46-PC-003 | ✓ |
| ISOLATED | ✓ | ✓ | B46-ORD-003 | ✓ |
| CONFLICT_xxx | ✓ | ✓ | B46-ID-002/003/004/005 | ✓ |
| RECOVERY_BARRIER | ✓ | ✓ | B46-BAR-003 | ✓ |
| OP_RESERVATION/OP_CONFIRM/OP_RELEASE | ✓ | ✓ | B46-RES-001/002 | ✓ |

---

## 5. 统计与 HASH 一致性

- 反例 JSON：38 条，ID 唯一，统计与 COUNTEREXAMPLE-RESULTS.md 一致。
- 历史统计：HISTORICAL-COUNTEREXAMPLE-AUDIT.md 与 COUNTEREXAMPLE-RESULTS.md §6 一致。
- HASH：全部交付文件 SHA256 + 字节数，以 Git blob（LF 口径）计算，见 HASH-MANIFEST.txt 与 EXECUTION-REPORT.md。

---

## 6. 结论

跨文档编号、核心定义、状态术语、反例引用、统计数字全部一致。05.45 的矛盾已全部修复为 05.46 的统一权威规则。
