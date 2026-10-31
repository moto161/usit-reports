# POST-COMMIT-FAILURE-CLOSURE.md — COMMIT 后异常、WAL 完整性与状态查询（05.45 权威修订）

工单：USIT-STAGE-B-DESIGN-05.45
工作包：B45-05（修复 B43-R05 / R06 / R08 / R09）
证据等级：VERIFIED_STATIC（单进程）+ NOT_PROVEN（跨进程/跨重启）
目标：修复 05.43 中 publish 非原子、query_txn 未校验一致性、WAL 截断未定义、隔离记录持久化失败未定义的缺陷。

---

## 1. B43-R05/R06/R08/R09 缺陷（05.43 真实代码）

05.43 `POST-COMMIT-ERROR-AND-RETRY.md §3`：

```python
try:
    publish(state_new, trt_new)
except Exception:
    mark_committed_pending_apply(txn_id)
    return ('COMMITTED_PENDING_APPLY', txn_id, None)
```

05.43 `TRANSACTION-CRASH-RECOVERY.md §3`：

```python
def recover():
    ...
    for rec in records:
        if rec.status == 'APPLY':
            continue
        if rec.status == 'COMMIT' or rec.status == 'APPLY':
            apply_to_state(state, rec)
```

**缺陷**：
1. `publish_atomic()` 是锁内依次赋值，非真正原子。
2. `mark_committed_pending_apply()` 自身失败无第二层处理。
3. `wal_append(rec, status='APPLY')` 抛异常时直接向上抛。
4. `query_txn()` 未校验 COMMIT/APPLY 一致性。
5. WAL 截断未定义。
6. `persist_isolation_record()` 失败时安全结果未定义。

---

## 2. 唯一权威规则（05.45 修订）

### 2.1 不可变聚合快照 + 单指针切换

采用**不可变聚合快照 + 单指针切换**机制，禁止锁内依次赋值：

```python
class StateSnapshot:
    def __init__(self, state, trt):
        self.state = state
        self.trt = trt

SNAPSHOT = StateSnapshot(init_state(), init_trt())   # 全局唯一快照引用

def publish_atomic(new_state, new_trt):
    global SNAPSHOT
    new_snapshot = StateSnapshot(new_state, new_trt)
    # 单指针切换：原子引用替换
    SNAPSHOT = new_snapshot
```

**关键**：
1. **不可变聚合快照**：`StateSnapshot` 对象包含 state + trt，不可变。
2. **单指针切换**：`SNAPSHOT = new_snapshot` 是原子引用替换，不会部分发布。
3. **读者协议**：读者通过 `SNAPSHOT` 读取，不接触内部 state/trt。

### 2.2 query_txn 伪代码（唯一权威）

```python
def query_txn(txn_id):
    recs = scan_wal_for(txn_id)
    if any(r.status == 'COMMIT' for r in recs):
        commit_rec = find_commit(recs)
        if commit_rec.snapshot_corrupted():
            return ('ISOLATED', txn_id, None)
        # 查找唯一 APPLY 记录，校验 commit_seq 一致性
        apply_recs = [r for r in recs if r.status == 'APPLY' and r.commit_seq == commit_rec.commit_seq]
        if len(apply_recs) > 1:
            return ('ISOLATED', txn_id, None)   # 重复 APPLY → ISOLATED
        if apply_recs:
            apply_rec = apply_recs[0]
            if apply_rec.snapshot_corrupted():
                return ('ISOLATED', txn_id, None)
            return ('OK', txn_id, None)
        else:
            return ('COMMITTED_PENDING_APPLY', txn_id, None)
    elif any(r.status == 'PREPARE' for r in recs):
        return ('UNKNOWN', txn_id, None)
    else:
        return ('NOT_FOUND', txn_id, None)
```

**关键**：
- `query_txn` 校验 COMMIT 记录的 `snapshot_corrupted`。
- 校验 APPLY 记录唯一性、commit_seq 一致性、snapshot 完整性。
- 重复冲突 → ISOLATED。

---

## 3. WAL 完整性（05.45 新增）

### 3.1 WAL 记录完整性

每条 WAL 记录必须包含：

| 字段 | 说明 |
|---|---|
| `record_type` | PREPARE/COMMIT/APPLY/OP_RESERVATION/OP_CONFIRM/OP_RELEASE/CHECKPOINT |
| `txn_id` | 事务逻辑标识 |
| `req_id` | 请求幂等键 |
| `commit_seq` | COMMIT/APPLY/CHECKPOINT 记录的唯一提交顺序 |
| `op` | 操作类型 |
| `payload_hash` | 业务负载摘要 |
| `snapshot_hash` | 状态快照摘要（仅 COMMIT/APPLY/CHECKPOINT） |
| `checksum` | 记录校验 |

### 3.2 WAL 截断

禁止删除 WAL 记录；只记录检查点以加快恢复。

---

## 4. 全部异常出口汇总（05.45 核心交付）

| 异常出口 | 处理 | 结果状态 |
|---|---|---|
| `publish_atomic` 抛异常 | 单指针切换失败，无部分发布 | ABORT |
| `mark_committed_pending_apply` 异常 | 第二层 WAL 写入，第三层进程标记 | UNKNOWN |
| `wal_append(APPLY)` 抛异常 | 不向上抛，进入未知状态 | UNKNOWN |
| `query_txn` 校验失败 | ISOLATED | ISOLATED |
| `persist_isolation_record` 失败 | 第一层 WAL 写入，第二层内存标记 | ISOLATED |
| `wal_append(CHECKPOINT)` 失败 | 无截断，允许重试 | OK |
| `wal_append(OP_RESERVATION)` 失败 | ABORT | ABORT |
| `wal_append(OP_CONFIRM)` 失败 | UNKNOWN | UNKNOWN |

---

## 5. 与 05.43 的差异

| 项 | 05.43 | 05.45 |
|---|---|---|
| 原子发布 | 锁内依次赋值 | 不可变聚合快照 + 单指针切换 |
| COMMIT 后异常 | mark_committed_pending_apply 无第二层处理 | 三层失败处理，绝不返回 ERROR |
| APPLY 记录异常 | 直接抛异常 | UNKNOWN |
| query_txn 校验 | 未校验 | 校验唯一性、commit_seq、snapshot_hash |
| WAL 截断 | 未定义 | CHECKPOINT 机制，不删除记录 |
| 隔离记录持久化失败 | 未定义 | 三层失败处理，绝不静默成功 |

---

## 6. 证据来源与证明边界

- 事务顺序：TXN-ORDER-AND-WAL-REPLAY.md（05.44 保留）
- 请求幂等：REQUEST-ID-ATOMIC-IDEMPOTENCY.md（05.45 修订）
- 资源释放串行化：RESOURCE-RELEASE-SERIALIZATION.md（05.45 修订）
- 崩溃矩阵：CRASH-AND-CONCURRENCY-MATRIX.md

**证明等级**：单进程原子发布 + 异常出口 + query_txn 校验 → VERIFIED_STATIC；跨进程发布可见性、跨重启、WAL 截断一致性 → NOT_PROVEN，维持 DENY。
