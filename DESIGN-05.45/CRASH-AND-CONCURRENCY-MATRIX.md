# CRASH-AND-CONCURRENCY-MATRIX.md — 崩溃与并发矩阵（05.44 权威）

工单：USIT-STAGE-B-DESIGN-05.44
工作包：B44-01/02/03/04/05 支撑
证据等级：VERIFIED_STATIC（单进程）+ NOT_PROVEN（跨进程/跨重启）

---

## 1. 事务提交崩溃矩阵

| 崩溃点 | 恢复动作 | 证据 |
|---|---|---|
| 提交前不变式验证失败 | ABORT，不发布 | TRANSACTION-PRECOMMIT-VALIDATION.md（05.43 保留） |
| PREPARE fsync 前崩溃 | 未提交，丢弃 | 同上 |
| PREPARE fsync 后、COMMIT 前崩溃 | 未提交，丢弃 | 同上 |
| COMMIT fsync 前（写中） | 半条记录/checksum 失败 → ISOLATED | POST-COMMIT-FAILURE-CLOSURE.md |
| COMMIT fsync 后、APPLY 前 | 重放 COMMIT 快照一次 | TXN-ORDER-AND-WAL-REPLAY.md |
| COMMIT + APPLY 均持久化 | 只取 COMMIT apply 一次 | 同上 |
| APPLY 记录写入中 | COMMIT + 半条 APPLY → 忽略 APPLY，取 COMMIT | 同上 |

---

## 2. 恢复幂等矩阵（按 commit_seq）

| 记录组合 | 是否 apply | apply 次数 |
|---|---|---|
| 仅 PREPARE | 否 | 0 |
| PREPARE + COMMIT | 是 | 1 |
| PREPARE + COMMIT + APPLY | 是 | 1（忽略 APPLY） |
| 仅 COMMIT（无 PREPARE，异常） | 是 | 1 |
| 仅 APPLY（无 COMMIT，异常） | 否 | 0（APPLY 非变更触发点） |
| 同 commit_seq 多条 COMMIT | — | ISOLATED |
| 同 txn_id 不同 payload | — | ISOLATED |

---

## 3. 锁与并发矩阵（恢复屏障模型）

| 并发场景 | 结果 | 证据 |
|---|---|---|
| 两事务并发 admit | barrier_lock + txn_lock 串行化，逐一提交 | RECOVERY-BARRIER-AND-ADMISSION.md |
| admit 持锁调 run_txn（自锁） | 05.42 死锁 → 05.44 消除（上层不持锁） | 同上 §1 |
| 恢复与普通事务并发 | in_flight_count 等待事务退出 | 同上 §2 |
| 恢复标志检查后、事务获取锁前恢复启动 | 05.43 缺陷 → 05.44 屏障模型消除 | 同上 §2 |
| 恢复中断与标志清除 | 屏障保持，ISOLATED | 同上 §4 |
| 线程持锁崩溃 | 进程内锁遗留 → NOT_PROVEN | 同上 §7 |
| 进程崩溃（跨重启） | WAL 恢复，锁自动消失 | 同上 §7 |

---

## 4. 资源端释放崩溃矩阵

| 崩溃点 | 恢复动作 | 证据 |
|---|---|---|
| 授权后、operation 前 | 无副作用，可重试 | RESOURCE-RELEASE-SERIALIZATION.md §4 |
| operation 中 | 幂等则重放，否则 ISOLATED | 同上 |
| operation 后、WAL COMMIT 前 | 查 WAL，无 COMMIT → 判 operation 幂等 | 同上 |
| WAL COMMIT 后、APPLY 前 | 续 APPLY | 同上 |
| consumed_records 后 | ALREADY_RELEASED | 同上 |
| operation 部分成功 | 预留+确认机制 | RESOURCE-OPERATION-CRASH-RECOVERY.md |
| Owner 校验后接管 | 串行化在同一资源锁 | RESOURCE-RELEASE-SERIALIZATION.md |

---

## 5. 版本与代际耗尽矩阵

| 边界 | 结果 | 证据 |
|---|---|---|
| VERSION == 0xFFFFFFFF，需递增 | VERSION_EXHAUSTED_DENY | TRANSACTION-PRECOMMIT-VALIDATION.md §3 |
| generation 达槽位最大值，需递增 | GENERATION_EXHAUSTED_DENY | 同上 |
| VERSION 回退/跳跃/回绕 | ABORT（不变式 I-2 拒绝） | 同上 §5 |
| generation 回绕复用旧值 | STALE_TOKEN（旧代际被拒） | TOKEN-IDENTITY-CANONICAL-MODEL.md §5 |

---

## 6. COMMIT 后异常出口矩阵

| 异常出口 | 处理 | 结果状态 |
|---|---|---|
| publish_atomic 抛异常 | mark_committed_pending_apply 失败 → WAL 写入 | COMMITTED_PENDING_APPLY |
| wal_append(APPLY) 抛异常 | 不向上抛，返回 COMMITTED_PENDING_APPLY | COMMITTED_PENDING_APPLY |
| mark_committed_pending_apply 异常 | 第二层 WAL 写入，第三层进程标记 | UNKNOWN（不 ERROR） |
| query_txn 校验失败 | ISOLATED | ISOLATED |
| persist_isolation_record 失败 | 第一层 WAL 写入，第二层内存标记 | ISOLATED |
| wal_append(CHECKPOINT) 失败 | 无截断，允许重试 | OK |

---

## 7. 证明等级总表

| 能力 | 单进程 | 跨进程 | 跨重启 |
|---|---|---|---|
| WAL 恢复幂等 | VERIFIED_STATIC | NOT_PROVEN | NOT_PROVEN |
| 提交前不变式验证 | VERIFIED_STATIC | NOT_PROVEN | NOT_PROVEN |
| 恢复屏障与事务准入 | VERIFIED_STATIC | NOT_PROVEN | NOT_PROVEN |
| 锁所有权/出口审计 | VERIFIED_STATIC | NOT_PROVEN | NOT_PROVEN |
| Token 身份四字段校验 | VERIFIED_STATIC | NOT_PROVEN | NOT_PROVEN |
| 资源端 Owner 授权 | VERIFIED_STATIC | NOT_PROVEN | NOT_PROVEN |
| operation 幂等重放 | 视业务而定 | NOT_PROVEN | NOT_PROVEN |
| consumed_records 持久化 | VERIFIED_STATIC（WAL） | NOT_PROVEN | NOT_PROVEN |
| 原子发布可见性 | VERIFIED_STATIC | NOT_PROVEN | NOT_PROVEN |

**结论**：所有跨进程、跨重启能力维持 DENY/ISOLATED，与历轮一致。

---

## 8. 证据来源

- TXN-ORDER-AND-WAL-REPLAY.md
- REQUEST-ID-ATOMIC-IDEMPOTENCY.md
- RECOVERY-BARRIER-AND-ADMISSION.md
- RESOURCE-RELEASE-SERIALIZATION.md
- RESOURCE-OPERATION-CRASH-RECOVERY.md
- POST-COMMIT-FAILURE-CLOSURE.md
- WAL-INTEGRITY-AND-CHECKPOINT.md
