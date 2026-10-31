# POST-COMMIT-FAILURE-CLOSURE.md — COMMIT 后异常、WAL 完整性与状态查询（05.44 权威）

工单：USIT-STAGE-B-DESIGN-05.44
工作包：B44-05（修复 B43-R05 / R06 / R08 / R09）
证据等级：VERIFIED_STATIC（单进程）+ NOT_PROVEN（跨进程/跨重启）
目标：修复 05.43 中 COMMIT 后异常处理不完整、publish 非真正原子、query_txn 未校验 COMMIT/APPLY 一致性、WAL 截断与恢复边界不明确的缺陷。

---

## 1. B43-R05/R06/R08/R09 缺陷（05.43 真实问题）

| 编号 | 缺陷（05.43 代码/文字） |
|---|---|
| R05 | `publish(state_new, trt_new)` 被称为原子发布，但 Python 普通赋值非原子；`mark_committed_pending_apply()` 自身失败无第二层处理 |
| R06 | `query_txn()` 把 COMMIT+APPLY 直接认定 OK，未验证记录一致性 |
| R08 | `recover()` 用 init_state 重建，未定义 WAL 截断/压缩/检查点与历史保留关系 |
| R09 | `persist_isolation_record()` 自身持久化失败时安全结果未定义 |

---

## 2. 真正的状态发布机制（R05 修复）

### 2.1 原子发布协议

- **单临界区发布**：`publish_atomic(state_new, trt_new)` 在 `txn_guard` 内一次性更新状态字 + TRT，读者通过**读锁（共享锁）**或**快照**读取。
- **双写一致性**：状态字与 TRT 在同一 `txn_guard` 内同步更新，不存在「状态字已更新但 TRT 尚未更新」的半成品。
- **异常处理**：`publish_atomic` 抛异常时：
  - 第一层：`mark_committed_pending_apply(txn_id)`，标记该事务为已提交待应用。
  - 第二层：`mark_committed_pending_apply` 本身抛异常时：写入 WAL 的 `COMMITTED_PENDING_APPLY` 记录（独立 WAL 事务，保证持久化）。
  - 第三层：WAL 写入也失败时：进程内状态标记 `txn_id` 为不可恢复，允许查询接口返回 UNKNOWN，但绝不返回普通 ERROR。

### 2.2 APPLY 记录写入异常

- `wal_append(rec, status='APPLY')` 抛异常时：
  - 不直接向上抛，而是进入 `COMMITTED_PENDING_APPLY` 状态。
  - 返回 `(COMMITTED_PENDING_APPLY, txn_id, None)`，允许调用方查询。
  - 查询接口 `query_txn` 会检查 WAL 中的 APPLY 记录是否存在，若缺失则返回 `COMMITTED_PENDING_APPLY`。

---

## 3. 查询接口（R06 修复：一致性校验）

### 3.1 query_txn 伪代码（唯一权威）

```python
def query_txn(txn_id):
    recs = scan_wal_for(txn_id)
    if any(r.status == 'COMMIT' for r in recs):
        commit_rec = find_commit(recs)
        # 校验 COMMIT 记录一致性
        if commit_rec.snapshot_corrupted():
            return ('ISOLATED', txn_id, None)
        # 查找 APPLY 记录
        apply_rec = find_apply(recs)
        if apply_rec and apply_rec.commit_seq == commit_rec.commit_seq:
            # 校验 APPLY 记录一致性
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
- 校验 APPLY 记录与 COMMIT 记录的 `commit_seq` 一致性。
- 任一校验失败 → ISOLATED，禁止直接返回 OK。

---

## 4. WAL 完整性与检查点（R08 修复）

### 4.1 WAL 记录完整性

- **完整 WAL 记录**：包含 `txn_id`、`commit_seq`、`op`、`snapshot`、`checksum`、`status`。
- **截断与压缩**：禁止删除 WAL 记录；只增加检查点以加快恢复。
- **检查点**：记录最近一次成功恢复的 commit_seq，便于快速定位恢复起点。
- **历史保留**：检查点只记录位置，不删除记录。

### 4.2 恢复边界

- **恢复起点**：从检查点开始扫描 WAL。
- **重放终点**：WAL 末尾（或损坏记录处）。
- **恢复策略**：从 WAL 重放，跳过检查点之前的记录。
- **恢复失败**：检查点无效 → ISOLATED。

### 4.3 WAL 截断伪代码

```python
def truncate_wal(commit_seq):
    # 只记录检查点，不删除记录
    checkpoint = {'commit_seq': commit_seq, 'timestamp': now()}
    wal_append(checkpoint, status='CHECKPOINT')
    return ('OK', None, None)
```

---

## 5. 隔离记录持久化失败（R09 修复）

### 5.1 persist_isolation_record 异常处理

`persist_isolation_record()` 自身失败时：

- **第一层**：尝试写入 WAL 的 `ISOLATED` 记录（独立 WAL 事务）。
- **第二层**：WAL 写入失败时：进程内标记 `resource.state = ISOLATED`（内存标记），后续查询资源状态返回 `ISOLATED`。
- **第三层**：内存标记也失败时：调用方进程终止，资源端保留最后状态。

**结论**：隔离记录持久化失败绝不静默成功，必须返回 `ISOLATED` 并隔离资源。

---

## 6. 全部异常出口汇总（B44-05 核心交付）

| 异常出口 | 处理 | 结果状态 |
|---|---|---|
| `publish_atomic` 抛异常 | `mark_committed_pending_apply` 失败 → WAL 写入 `COMMITTED_PENDING_APPLY` | COMMITTED_PENDING_APPLY |
| `wal_append(APPLY)` 抛异常 | 不向上抛，返回 `COMMITTED_PENDING_APPLY` | COMMITTED_PENDING_APPLY |
| `mark_committed_pending_apply` 异常 | 第二层 WAL 写入，第三层进程标记 | UNKNOWN（不 ERROR） |
| `query_txn` 校验失败 | ISOLATED | ISOLATED |
| `persist_isolation_record` 失败 | 第一层 WAL 写入，第二层内存标记 | ISOLATED |
| `wal_append(CHECKPOINT)` 失败 | 无截断，允许重试 | OK |

---

## 7. 与 05.43 的差异

| 项 | 05.43 | 05.44 |
|---|---|---|
| 原子发布 | Python 赋值 | txn_guard 内同步更新 + 读锁/快照 |
| COMMIT 后异常 | mark_committed_pending_apply 无第二层处理 | 三层失败处理，绝不返回 ERROR |
| APPLY 记录异常 | 直接抛异常 | COMMITTED_PENDING_APPLY |
| query_txn 校验 | 无 | snapshot_corrupted + commit_seq 一致性校验 |
| WAL 截断 | 未定义 | CHECKPOINT 机制，不删除记录 |
| 隔离记录持久化失败 | 未定义 | 三层失败处理，绝不静默成功 |

---

## 8. 证据来源与证明边界

- 事务顺序：TXN-ORDER-AND-WAL-REPLAY.md
- 请求幂等：REQUEST-ID-ATOMIC-IDEMPOTENCY.md
- 恢复屏障：RECOVERY-BARRIER-AND-ADMISSION.md
- 资源释放串行化：RESOURCE-RELEASE-SERIALIZATION.md
- 崩溃矩阵：CRASH-AND-CONCURRENCY-MATRIX.md

**证明等级**：单进程原子发布 + 异常出口 + query_txn 校验 → VERIFIED_STATIC；跨进程发布可见性、跨重启、WAL 截断一致性 → NOT_PROVEN，维持 DENY。
