# CRASH-AND-CONCURRENCY-MATRIX.md — 崩溃与并发矩阵（05.43 权威）

工单：USIT-STAGE-B-DESIGN-05.43
工作包：B43-05/B43-06 支撑（B42-R05/R06 的崩溃窗口与并发交错汇总）
证据等级：VERIFIED_STATIC（单进程）+ NOT_PROVEN（跨进程/跨重启）
目标：以矩阵形式汇总全部崩溃窗口、并发交错、恢复动作，作为反例与跨文档一致性的统一依据。

---

## 1. 事务提交崩溃矩阵

| 崩溃点 | WAL 状态 | 恢复动作 | 证据 |
|---|---|---|---|
| PREPARE fsync 前 | 无记录 | 未提交，丢弃 | TRANSACTION-PRECOMMIT-VALIDATION §2 |
| PREPARE fsync 后、COMMIT 前 | PREPARE（无 COMMIT） | 未提交，丢弃 | 同上 |
| COMMIT fsync 前（写中） | 半条记录/checksum 失败 | ISOLATED，DENY | WAL-RECOVERY-IDEMPOTENCY §4 |
| COMMIT fsync 后、APPLY 前 | COMMIT（无 APPLY） | 重放 COMMIT 一次 | WAL-RECOVERY-IDEMPOTENCY §3 |
| COMMIT + APPLY 均持久化 | COMMIT + APPLY | 只取 COMMIT apply 一次 | WAL-RECOVERY-IDEMPOTENCY §5 |
| APPLY 记录写入中 | COMMIT + 半条 APPLY | 忽略 APPLY，取 COMMIT | 同上 §4 |

---

## 2. 恢复幂等矩阵

| 记录组合 | 是否 apply | apply 次数 |
|---|---|---|
| 仅 PREPARE | 否 | 0 |
| PREPARE + COMMIT | 是 | 1 |
| PREPARE + COMMIT + APPLY | 是 | 1（忽略 APPLY） |
| 仅 COMMIT（无 PREPARE，异常） | 是 | 1 |
| 仅 APPLY（无 COMMIT，异常） | 否 | 0（APPLY 非变更触发点） |
| 重复 COMMIT（同 txn_id） | 是 | 1（first_commit + applied 集合） |

---

## 3. 锁与并发矩阵

| 并发场景 | 结果 | 证据 |
|---|---|---|
| 两事务并发 admit | txn_lock 串行化，逐一提交 | LOCK-OWNERSHIP-AND-CALL-GRAPH §3 |
| admit 持锁调 run_txn（自锁） | 05.42 死锁 → 05.43 消除（上层不持锁） | 同上 §1/§2 |
| 恢复与普通事务并发 | in_recovery 标志阻塞普通事务 | 同上 §6 |
| 线程持锁崩溃 | 进程内锁遗留 → NOT_PROVEN | 同上 §7 |
| 进程崩溃（跨重启） | WAL 恢复，锁自动消失 | 同上 §7 |

---

## 4. 资源端释放崩溃矩阵

| 崩溃点 | 恢复动作 | 证据 |
|---|---|---|
| 授权后、operation 前 | 无副作用，可重试 | RESOURCE-FAILURE-AND-RECOVERY §4 |
| operation 中 | 幂等则重放，否则 ISOLATED | 同上 |
| operation 后、WAL COMMIT 前 | 查 WAL，无 COMMIT → 判 operation 幂等 | 同上 |
| WAL COMMIT 后、APPLY 前 | 续 APPLY | 同上 |
| consumed_records 后 | ALREADY_RELEASED | 同上 §3 |

---

## 5. 版本与代际耗尽矩阵

| 边界 | 结果 | 证据 |
|---|---|---|
| VERSION == 0xFFFFFFFF，需递增 | VERSION_EXHAUSTED_DENY | TRANSACTION-PRECOMMIT-VALIDATION §3 |
| generation 达到槽位最大值，需递增 | GENERATION_EXHAUSTED_DENY | 同上 |
| VERSION 回退/跳跃/回绕 | ABORT（不变式 I-2 拒绝） | 同上 §5 |
| generation 回绕复用旧值 | STALE_TOKEN（旧代际被拒） | TOKEN-IDENTITY-CANONICAL-MODEL §5 |

---

## 6. 证明等级总表

| 能力 | 单进程 | 跨进程 | 跨重启 |
|---|---|---|---|
| WAL 恢复幂等 | VERIFIED_STATIC | NOT_PROVEN | NOT_PROVEN |
| 提交前不变式验证 | VERIFIED_STATIC | NOT_PROVEN | NOT_PROVEN |
| 锁所有权/出口审计 | VERIFIED_STATIC | NOT_PROVEN | NOT_PROVEN |
| Token 身份四字段校验 | VERIFIED_STATIC | NOT_PROVEN | NOT_PROVEN |
| 资源端 Owner 授权 | VERIFIED_STATIC | NOT_PROVEN | NOT_PROVEN |
| operation 幂等重放 | 视业务而定 | NOT_PROVEN | NOT_PROVEN |
| consumed_records 持久化 | VERIFIED_STATIC（WAL） | NOT_PROVEN | NOT_PROVEN |

**结论**：所有跨进程、跨重启能力维持 DENY/ISOLATED，与历轮一致。

---

## 7. 证据来源

- WAL-RECOVERY-IDEMPOTENCY.md
- TRANSACTION-PRECOMMIT-VALIDATION.md
- POST-COMMIT-ERROR-AND-RETRY.md
- LOCK-OWNERSHIP-AND-CALL-GRAPH.md
- RESOURCE-FAILURE-AND-RECOVERY.md
- RESOURCE-OWNER-AUTHORIZATION.md
- TOKEN-IDENTITY-CANONICAL-MODEL.md
