# TXN-ORDER-AND-WAL-REPLAY.md — 事务 ID 与提交顺序（05.44 权威）

工单：USIT-STAGE-B-DESIGN-05.44
工作包：B44-01（修复 B43-R01）
证据等级：VERIFIED_STATIC（单进程）+ NOT_PROVEN（跨进程/跨重启）
目标：修复 05.43 中 txn_id 在获取事务锁之前分配、恢复按 txn_id 升序重放导致的提交顺序反转候选快照覆盖缺陷。

---

## 1. B43-R01 缺陷（05.43 真实代码）

05.43 `TRANSACTION-PRECOMMIT-VALIDATION.md §3`：

```python
def run_txn(op, slot, owner_id, resource_id, caller_owner):
    txn_id = next_txn_id()          # <-- 在锁外分配
    with txn_guard():
        ...
```

05.43 `WAL-RECOVERY-IDEMPOTENCY.md §3`：

```python
for txn_id in sorted(groups.keys()):    # 按 txn_id 升序重放
    ...
```

**缺陷**：txn_id 在获取 txn_lock **之前**分配。可能出现：

- 线程 A 先取得 txn_id=100，尚未获得锁。
- 线程 B 取得 txn_id=101，先获得锁并提交。
- 线程 A 随后获得锁并提交。
- **实际提交顺序 = 101 → 100，恢复顺序 = 100 → 101**。

若 COMMIT 记录保存完整候选状态快照，后提交的旧候选状态会覆盖新状态。

---

## 2. 唯一权威规则（05.44 修订）

### 2.1 提交顺序的权威字段：`commit_seq`

- `commit_seq`：全局单调递增 u64，**唯一提交顺序权威**，由 `COMMIT` 记录携带。
- `txn_id`：事务逻辑标识，与 commit_seq 解耦，用于幂等重放去重。
- `commit_seq` **只在 COMMIT 提交点分配**，任何提前抢号、回退、跳跃都不允许。

### 2.2 序列化边界

事务的执行严格按以下唯一顺序：

```
1. 获取事务串行化权（txn_lock / recovery_barrier_guard）
2. 读取旧状态
3. 构造候选状态
4. 提交前不变式验证（05.43 已建立）
5. 写 PREPARE 记录 → fsync
6. 分配 commit_seq（单调递增）                  <-- 唯一提交顺序分配点
7. 写 COMMIT 记录（携带 commit_seq + 完整候选状态快照）→ fsync   <-- 提交点
8. APPLY：原子发布已验证候选状态
9. 写 APPLY 记录（可选）
10. 释放事务串行化权
```

**关键**：
1. `commit_seq` 在步骤 6 分配，**只在持锁状态下**，保证其严格单调且无顺序反转。
2. `COMMIT` 记录保存**完整候选状态快照**，不是可验证增量——恢复时不需重建状态，直接重放快照。
3. 恢复顺序**只按 commit_seq 升序**，与 txn_id 无关。

---

## 3. 提交协议伪代码（唯一权威）

```python
def run_txn(op, slot, owner_id, resource_id, caller_owner, req_id):
    with txn_guard():                       # 步骤 1：唯一事务串行化权
        # 步骤 2：读取旧状态
        state_old = read_state()
        trt_old = read_trt()

        # 步骤 3：构造候选
        state_new, trt_new = build_candidate(op, state_old, trt_old, ...)

        # 步骤 4：提交前不变式验证（05.43 已建立）
        if not invariants_hold(state_new, trt_new):
            return ('ABORT', None, None)
        if version_exhausted(state_new):
            return ('ABORT', None, 'VERSION_EXHAUSTED_DENY')

        # 步骤 5：PREPARE
        txn_id = next_txn_id()              # <-- 在锁内分配，顺序与 commit_seq 一致
        rec = build_record(txn_id, op, state_new, trt_new, req_id, ...)
        if not wal_append_fsync(rec, status='PREPARE'):
            return ('ABORT', txn_id, None)

        # 步骤 6：分配 commit_seq（单调，唯一提交顺序）
        commit_seq = next_commit_seq()      # <-- 在锁内严格单调

        # 步骤 7：COMMIT（提交点，携带 commit_seq + 完整候选快照）
        rec.set_commit_seq(commit_seq)
        rec.set_snapshot(state_new, trt_new)
        if not wal_append_fsync(rec, status='COMMIT'):
            return ('UNKNOWN', txn_id, None)

        # 步骤 8：APPLY（原子发布）
        publish_atomic(state_new, trt_new)

        # 步骤 9：APPLY 记录（可选）
        wal_append(rec, status='APPLY')

        return ('OK', txn_id, None)
```

---

## 4. 恢复算法（唯一权威，按 commit_seq 重放）

```python
def recover():
    records = scan_wal()
    if wal_corrupted():
        return 'ISOLATED'

    state = init_state()
    trt = init_trt()

    # 按 commit_seq 分组（而非 txn_id）
    groups = group_by_commit_seq(records)   # {commit_seq: [records...]}
    applied_commit_seqs = set()

    for commit_seq in sorted(groups.keys()):
        recs = groups[commit_seq]
        if commit_seq in applied_commit_seqs:
            continue
        # 查找该 commit_seq 的 COMMIT 记录
        commit_rec = find_commit_by_seq(recs, commit_seq)
        if commit_rec is None:
            continue                        # 无 COMMIT，未到达提交点

        # 防冲突：同 commit_seq 多条 COMMIT → ISOLATED
        commit_records = [r for r in recs if r.status == 'COMMIT' and r.commit_seq == commit_seq]
        if len(commit_records) > 1:
            return 'ISOLATED'               # 冲突 COMMIT，隔离

        # 校验快照一致性
        if commit_rec.snapshot_corrupted():
            return 'ISOLATED'

        apply_snapshot(state, trt, commit_rec.snapshot)
        applied_commit_seqs.add(commit_seq)

    return ('OK', state, trt)
```

**关键**：
1. **重放按 commit_seq 升序**，与 txn_id 无关。
2. **COMMIT 记录保存完整候选快照**，恢复直接 apply 快照，无需重建字段变化。
3. **同 commit_seq 多条 COMMIT → ISOLATED**，禁止 first_commit 静默选取。
4. **snapshot_corrupted 校验快照一致性**，损坏即隔离。

---

## 5. 相同 txn_id 重复 COMMIT、冲突 payload、不同 checksum 的处理

| 情况 | 处理 |
|---|---|
| 同一 txn_id 出现多条 COMMIT（重复追加） | `find_commit_by_seq` 只取一条；若多条则 `ISOLATED` |
| 同一 commit_seq 多条 COMMIT | `ISOLATED`，禁止静默选取 |
| COMMIT 记录 snapshot_corrupted | `ISOLATED` |
| 同 txn_id 不同 payload/checksum | `ISOLATED`（在扫描阶段 `wal_corrupted()` 捕获） |
| 缺失 PREPARE 但存在 COMMIT | `ISOLATED`（不完整记录） |

---

## 6. 候选快照覆盖防御

- **COMMIT 记录保存完整候选状态快照**：恢复时直接 apply 快照，不涉及字段增量。
- **按 commit_seq 升序应用**：即使发生 101→100（严格说不可能，因为 commit_seq 持锁分配且单调），恢复严格按 commit_seq 顺序，不会反转。
- **原子发布**：`publish_atomic` 在 txn_lock 内一次性更新 state + trt，读者通过读锁/快照读取（见 POST-COMMIT-FAILURE-CLOSURE.md），不会观察到不一致。

---

## 7. 与 05.43 的差异

| 项 | 05.43 | 05.44 |
|---|---|---|
| 提交顺序权威 | txn_id（锁外分配，可反转） | commit_seq（锁内分配，单调） |
| 恢复重放顺序 | 按 txn_id 升序 | 按 commit_seq 升序 |
| COMMIT 记录 | 候选字段变化 | 完整候选快照 + commit_seq |
| 同 txn_id 重复 COMMIT | first_commit 静默选取 | ISOLATED 隔离 |
| 快照覆盖风险 | 存在 | 消除（快照 + 按 seq 顺序） |

---

## 8. 证据来源与证明边界

- 提交前验证：TRANSACTION-PRECOMMIT-VALIDATION.md（05.43，保留）
- 恢复屏障：RECOVERY-BARRIER-AND-ADMISSION.md
- 请求幂等：REQUEST-ID-ATOMIC-IDEMPOTENCY.md
- 崩溃矩阵：CRASH-AND-CONCURRENCY-MATRIX.md

**证明等级**：单进程持锁内 commit_seq 分配 + 快照重放 → VERIFIED_STATIC；跨进程 commit_seq 一致性、跨重启 → NOT_PROVEN，维持 DENY。
