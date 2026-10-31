# CROSS-DOC-CONSISTENCY.md — 跨文档一致性（05.45）

工单：USIT-STAGE-B-DESIGN-05.45
工作包：B45-09（反例一致性）
证据等级：STATIC_ANALYSIS

---

## 1. 问题编号统一（沿用 05.44 规则）

05.45 统一为：

| 编号 | 名称 | 主文档 | 执行报告 | ISSUES | 反例引用 |
|---|---|---|---|---|---|
| B45-01 | 恢复屏障 | ✓ | ✓ | ✓ | RECOVERY_BARRIER |
| B45-02 | 全局请求幂等 | ✓ | ✓ | ✓ | REQUEST_ID_ATOMIC_IDEMPOTENCY |
| B45-03 | operation 预留/确认 | ✓ | ✓ | ✓ | RESOURCE_OPERATION_CRASH_RECOVERY |
| B45-04 | 事务顺序与 WAL 恢复边界 | ✓ | ✓ | ✓ | TXN_ORDER_AND_WAL_REPLAY |
| B45-05 | 原子发布、查询与隔离持久化 | ✓ | ✓ | ✓ | POST_COMMIT_FAILURE_CLOSURE |
| B45-09 | 反例重新设计 | ✓ | ✓ | ✓ | 各类别 |

主文档与执行报告使用**完全相同**的编号与名称，无映射偏差。

---

## 2. 核心定义一致性核对

| 定义 | 权威来源 | 其他文档是否一致 |
|---|---|---|
| commit_seq 为唯一提交顺序权威（锁内分配） | TXN-ORDER-AND-WAL-REPLAY.md §2 | POST-COMMIT-FAILURE-CLOSURE.md §2 一致 |
| PREPARE 无 commit_seq，不参与排序 | TXN-ORDER-AND-WAL-REPLAY.md §2 | 其他文档引用一致 |
| commit_seq 耗尽保护 | TXN-ORDER-AND-WAL-REPLAY.md §2 | 其他文档引用一致 |
| 检查点含状态快照 + 请求幂等索引 + 摘要 | TXN-ORDER-AND-WAL-REPLAY.md §6 | 其他文档引用一致 |
| req_id 全局唯一性在全局请求锁内完成 | REQUEST-ID-ATOMIC-IDEMPOTENCY.md §2 | 其他文档引用一致 |
| 预留/确认协议接入真实释放 | RESOURCE-OPERATION-CRASH-RECOVERY.md §2 | 其他文档引用一致 |
| 原子发布 = 不可变聚合快照 + 单指针切换 | POST-COMMIT-FAILURE-CLOSURE.md §2 | 其他文档引用一致 |
| 恢复屏障 = Condition + 锁分离 + 上下文管理器 | RECOVERY-BARRIER-AND-ADMISSION.md §2 | 其他文档引用一致 |
| baseline_status/revision_status 分离 | COUNTEREXAMPLES.json | COUNTEREXAMPLE-RESULTS.md 一致 |

---

## 3. 已删除的 05.44 矛盾

| 矛盾 | 05.44 位置 | 05.45 处理 |
|---|---|---|
| threading.Lock().wait()（死锁） | RECOVERY-BARRIER-AND-ADMISSION.md §3 | Condition + 锁分离 + 上下文管理器 |
| req_id 仅资源锁（并发重复提交） | REQUEST-ID-ATOMIC-IDEMPOTENCY.md §5 | 全局请求锁 + 参数摘要校验 |
| operation 先执行后查 req_id | RESOURCE-OPERATION-CRASH-RECOVERY.md §2 | 先占用 req_id，后执行 operation |
| 预留/确认未接入真实释放 | RESOURCE-OPERATION-CRASH-RECOVERY.md §2 | 真实释放流程调用 persist_reservation() |
| PREPARE 参与 commit_seq 排序 | TXN-ORDER-AND-WAL-REPLAY.md §3 | PREPARE 无序号，不排序 |
| 检查点无快照/索引 | WAL-INTEGRITY-AND-CHECKPOINT.md（05.44） | 检查点含完整快照 + 索引 + 摘要 |
| 恢复起点未定义 | TXN-ORDER-AND-WAL-REPLAY.md（05.44） | 检查点存在时从快照恢复 |
| publish 锁内依次赋值 | POST-COMMIT-FAILURE-CLOSURE.md §2 | 不可变聚合快照 + 单指针切换 |

---

## 4. 状态术语一致性

| 项 | 主文档 | 协议文档 | 反例 | 一致 |
|---|---|---|---|---|
| VERSION_EXHAUSTED_DENY | ✓ | ✓ | — | ✓ |
| GENERATION_EXHAUSTED_DENY | ✓ | ✓ | — | ✓ |
| COMMIT_SEQ_EXHAUSTED | ✓ | ✓ | B45-ORD-002 | ✓ |
| ALREADY_PROCESSED | ✓ | ✓ | B45-ID-008 | ✓ |
| FENCED_OUT | ✓ | ✓ | — | ✓ |
| OWNER_MISMATCH | ✓ | ✓ | B45-ID-004 | ✓ |
| STALE_TOKEN | ✓ | ✓ | — | ✓ |
| COMMITTED_PENDING_APPLY | ✓ | ✓ | B45-PC-002 | ✓ |
| ISOLATED | ✓ | ✓ | B45-ORD-003 | ✓ |
| CONFLICT_xxx | ✓ | ✓ | B45-ID-002/003/004/005 | ✓ |
| RECOVERY_BARRIER | ✓ | ✓ | B45-BAR-003 | ✓ |
| OP_RESERVATION/OP_CONFIRM/OP_RELEASE | ✓ | ✓ | B45-RES-001/002 | ✓ |

---

## 5. 统计与 HASH 一致性

- 反例 JSON：41 条，ID 唯一，统计与 COUNTEREXAMPLE-RESULTS.md 一致。
- 历史统计：HISTORICAL-COUNTEREXAMPLE-AUDIT.md 与 COUNTEREXAMPLE-RESULTS.md §6 一致。
- HASH：全部交付文件 SHA256 + 字节数，以 Git blob（LF 口径）计算，见 HASH-MANIFEST.txt 与 EXECUTION-REPORT.md。

---

## 6. 结论

跨文档编号、核心定义、状态术语、反例引用、统计数字全部一致。05.44 的矛盾已全部修复为 05.45 的统一权威规则。
