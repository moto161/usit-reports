# TRANSACTION-PRECOMMIT-VALIDATION.md — 提交前不变式验证（05.43 权威）

工单：USIT-STAGE-B-DESIGN-05.43
工作包：B43-02（修复 B42-R02）
证据等级：VERIFIED_STATIC（单进程）+ NOT_PROVEN（跨进程/跨重启）
目标：消除 05.42 中「先 APPLY 再验证 COUNT、违反则回滚且称未提交」与「PREPARE→COMMIT→APPLY」两套顺序的自相矛盾，建立唯一权威的提交前验证时序。

---

## 1. B42-R02 缺陷（05.42 真实矛盾）

05.42 存在两套互斥的顺序：

- `TRANSACTION-COMMIT-PROTOCOL.md §1`：`PREPARE → COMMIT(fsync) → APPLY`。
- `STATE-INVARIANTS-AND-TRANSITIONS.md §3`：先执行 APPLY，再检查 `COUNT == popcount(TOKEN_ACTIVE)`，违反则**回滚 APPLY**，并称「此时事务尚未提交」。

**矛盾本质**：COMMIT fsync 成功即已提交，不可能再把一个「已提交」事务当作「尚未提交」来回滚。APPLY 之后的回滚既无法撤销已经 fsync 的 COMMIT 记录，也会让内存状态与 WAL 权威记录永久不一致。

---

## 2. 唯一权威时序（05.43 修订）

每个事务严格按以下顺序，全文档唯一：

```
1. 读旧状态（state_old, trt_old），在锁内
2. 构造候选新状态（state_new, trt_new）  ← 只在内存构造，未发布
3. 验证全部不变式（针对候选状态）：
   - COUNT == popcount(TOKEN_ACTIVE)
   - VERSION 单调递增且未耗尽
   - 每槽位 generation 单调递增且未耗尽
   - DENIED / GATE / FROZEN / TOKEN / VERSION 约束
4. 任一不变式不满足 → ABORT（不写 WAL，不发布，不回滚——因为从未发布）
5. 不变式全部满足 → 写 PREPARE 记录 → fsync
6. 写 COMMIT 记录 → fsync                        ← 提交点（唯一权威）
7. APPLY：把已验证的候选状态发布到内存缓存       ← 应用（不再验证、不再回滚）
8. 写 APPLY 记录（可选，进度标记）
9. 返回 OK
```

**关键修正**：

1. **不变式验证在 COMMIT 之前、针对候选状态完成**。候选状态尚未发布，不满足时直接丢弃（ABORT），不存在「回滚已发布状态」的问题。
2. **COMMIT 之后禁止任何回滚**。COMMIT fsync 成功 = 已提交，其后只允许执行「把已验证的候选状态发布」这一确定性动作。
3. **APPLY 只是发布**，不是「验证 + 发布 + 可能回滚」。验证已经在步骤 3 完成，APPLY 不再验证。

---

## 3. 提交协议伪代码（唯一权威）

```python
def run_txn(op, slot, owner_id, resource_id, caller_owner):
    txn_id = next_txn_id()
    with txn_guard():                       # 锁由 run_txn 统一持有（见 LOCK-OWNERSHIP）
        state_old = read_state()
        trt_old = read_trt()

        # 步骤 2：构造候选状态（纯内存）
        state_new, trt_new = build_candidate(op, state_old, trt_old, slot, owner_id, resource_id)

        # 步骤 3：提交前不变式验证
        if not invariants_hold(state_new, trt_new):
            return 'ABORT'                  # 候选状态违反不变式，未发布，未写 WAL
        if version_exhausted(state_new):
            return 'VERSION_EXHAUSTED_DENY'
        if generation_exhausted(trt_new, slot):
            return 'GENERATION_EXHAUSTED_DENY'

        rec = build_record(txn_id, op, state_new, trt_new, ...)

        # 步骤 5：PREPARE
        if not wal_append_fsync(rec, status='PREPARE'):
            return 'ABORT'                  # PREPARE 未落盘，未提交

        # 步骤 6：COMMIT（提交点）
        if not wal_append_fsync(rec, status='COMMIT'):
            return 'UNKNOWN'                # 提交不确定，见 POST-COMMIT-ERROR

        # 步骤 7：APPLY（发布已验证的候选状态）
        publish(state_new, trt_new)         # 原子发布到状态字 + TRT

        # 步骤 8：APPLY 记录
        wal_append(rec, status='APPLY')

        return 'OK'
```

---

## 4. 读者如何避免观察半成品状态

05.42 未回答「状态字已更新但 TRT 尚未更新」的半成品读。05.43 明确：

- **发布是原子的**：`publish(state_new, trt_new)` 在 txn_lock 内一次性更新状态字和 TRT 两个内存结构，读者通过**读锁（共享锁）**或**快照**读取。
- 读者要么看到旧的一致状态（APPLY 前），要么看到新的一致状态（APPLY 后），**不会看到「状态字已变、TRT 未变」的中间态**，因为发布在同一个临界区内完成。
- 若跨进程读，读者必须读 WAL 的 COMMIT 记录作为权威，不直接读内存缓存；跨进程内存一致性**NOT_PROVEN**，维持 DENY。

---

## 5. 不变式清单（每次提交前针对候选状态验证）

| 编号 | 不变式 | 验证时机 |
|---|---|---|
| I-1 | `COUNT == popcount(TOKEN_ACTIVE)` | COMMIT 前 |
| I-2 | `VERSION` 单调递增，无回退/跳跃/回绕 | COMMIT 前 |
| I-3 | 每槽位 `generation` 单调递增，无回绕 | COMMIT 前 |
| I-4 | `DENIED` 位置后不可清除 | COMMIT 前 |
| I-5 | `GATE`/`FROZEN` 状态合法转移 | COMMIT 前 |
| I-6 | 候选状态无非法位组合 | COMMIT 前 |
| I-7 | Token 身份 (slot, generation, owner_id, resource_id) 完整绑定 | COMMIT 前 |

任一不变式不满足 → ABORT，候选状态不发布。

---

## 6. APPLY 失败的隔离

- APPLY（`publish`）在 COMMIT 之后，理论上是确定性内存赋值。若因进程崩溃未完成 APPLY，恢复流程会依据 WAL 中的 COMMIT 记录重新 APPLY（见 WAL-RECOVERY-IDEMPOTENCY.md）。
- 若 APPLY 抛异常（例如内存错误），**不得返回 ERROR 诱导调用方重试**，而是进入 `COMMITTED_PENDING_APPLY`（见 POST-COMMIT-ERROR-AND-RETRY.md），由恢复流程续 APPLY。

---

## 7. 与 05.42 的差异

| 项 | 05.42 | 05.43 |
|---|---|---|
| 验证时机 | COMMIT 后、APPLY 后回滚（矛盾） | COMMIT 前、针对候选状态 |
| 回滚语义 | 「未提交」回滚已 fsync 事务 | 候选状态未发布，ABORT 丢弃，无回滚 |
| APPLY 职责 | 验证 + 发布 + 可能回滚 | 仅发布已验证候选状态 |
| 半成品读 | 未说明 | 临界区内原子发布 + 读锁/快照 |

---

## 8. 证据来源与证明边界

- 恢复幂等：WAL-RECOVERY-IDEMPOTENCY.md
- 提交后异常：POST-COMMIT-ERROR-AND-RETRY.md
- 锁所有权：LOCK-OWNERSHIP-AND-CALL-GRAPH.md
- 崩溃矩阵：CRASH-AND-CONCURRENCY-MATRIX.md

**证明等级**：单进程、txn_lock 内构造+验证+发布 → VERIFIED_STATIC；跨进程发布可见性、fsync 原子性 → NOT_PROVEN，维持 DENY。
