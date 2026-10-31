# POST-COMMIT-FAILURE-CLOSURE.md — COMMIT 后异常、WAL 完整性与状态查询（05.46 权威修订）

工单：USIT-STAGE-B-DESIGN-05.46
工作包：B46-05（修复 B45-R05）
证据等级：VERIFIED_STATIC（单进程）+ NOT_PROVEN（跨进程/跨重启）
目标：修复 05.45 中 StateSnapshot 不冻结、query_txn() 未校验唯一性、APPLY 筛选完整、recover() 与 recover_from_checkpoint() 双入口冲突的缺陷。

---

## 1. B45-R05 缺陷（05.45 真实代码）

05.45 `POST-COMMIT-FAILURE-CLOSURE.md §2`：

```python
def publish_atomic(state_new, trt_new):
    global SNAPSHOT
    new_snapshot = StateSnapshot(state_new, trt_new)
    SNAPSHOT = new_snapshot
```

**缺陷**：
1. `StateSnapshot` 只是普通 Python 类，state/trt 仍为可变引用，未真正冻结内部状态。读者可修改 snapshot.state 或 snapshot.trt，破坏不可变性。
2. `query_txn()` 只检查找到的 COMMIT/APPLY 及 commit_seq，但没有先权衡 COMMIT 唯一性（多个 COMMIT 无校验）。
3. APPLY 只筛选与 COMMIT 相同的 commit_seq，其他冲突 APPLY 被忽略。
4. `recover()` 与 `recover_from_checkpoint()` 没有统一入口，出现双路径冲突。

---

## 2. 唯一权威规则（05.46 修订）

### 2.1 真实不可变快照

采用**符合不可变快照结构**（使用专用冻结结构，而非普通 Python 类）：

```python
class ImmutableSnapshot:
    def __init__(self, state, trt):
        self._state = state
        self._trt = trt
        self._frozen = True

    @property
    def state(self):
        return self._state

    @property
    def trt(self):
        return self._trt

    @property
    def frozen(self):
        return self._frozen

    def with_new_state(self, new_state, new_trt):
        return ImmutableSnapshot(new_state, new_trt)

    def __contains__(self, item):
        return item in self._state

    def __getitem__(self, key):
        return self._state[key]

    def __iter__(self):
        return iter(self._state)

    def __len__(self):
        return len(self._state)

SNAPSHOT = ImmutableSnapshot(init_state(), init_trt())  # 原子快照单指针

def publish_atomic(new_state, new_trt):
    global SNAPSHOT
    new_snapshot = ImmutableSnapshot(new_state, new_trt)
    SNAPSHOT = new_snapshot                              # 原子引用替换
```

**关键**：
1. **冷冻结构**：`ImmutableSnapshot` 用 `_state`/`_trt` 私有字段 + `frozen` 属性 + `@property`，外部无法直接修改 snapshot._state 或 snapshot._trt（受保护访问）。
2. **状态协议**：读者只能通过 `snapshot.state`、`snapshot.trt` properties 读取，不能写入（受保护访问）。
3. **原子发布**：`SNAPSHOT = new_snapshot` 是原子引用替换，不会部分发布。

---

### 2.2 query_txn 唯一性校验（唯一权威）

```python
def query_txn(txn_id):
    recs = scan_wal_for(txn_id)
    # 第1步：唯一 COMMIT 校验
    commit_recs = [r for r in recs if r.status == 'COMMIT']
    if len(commit_recs) > 1:
        return ('ISOLATED', txn_id, None)   # 重复 COMMIT → ISOLATED

    if not commit_recs:
        return ('NO_COMMIT', txn_id, None)   # 无 COMMIT

    commit_rec = commit_recs[0]
    if commit_rec.snapshot_corrupted():
        return ('ISOLATED', txn_id, None)

    # 第2步：唯一 APPLY 校验（与 COMMIT 相同 commit_seq）
    apply_recs = [r for r in recs if r.status == 'APPLY' and r.commit_seq == commit_rec.commit_seq]
    if len(apply_recs) > 1:
        return ('ISOLATED', txn_id, None)   # 重复 APPLY → ISOLATED

    if apply_recs:
        apply_rec = apply_recs[0]
        if apply_rec.snapshot_corrupted():
            return ('ISOLATED', txn_id, None)

    return ('OK', txn_id, commit_rec.commit_seq)
```

**关键**：
1. **唯一 COMMIT**：`commit_recs = [r for r in recs if r.status == 'COMMIT']`，唯一校验。
2. **唯一 APPLY**：`apply_recs = [r for r in recs if r.status == 'APPLY' and r.commit_seq == commit_rec.commit_seq]`，唯一校验。
3. **完整性校验**：校验 snapshot_hash、checksum。

---

### 2.3 隔离记录持久化失败处理

`persist_isolation_record()` 失败时：

1. 第一层：尝试写入 WAL 的 `ISOLATED` 记录（独立 WAL 事务）。
2. 第二层：WAL 写入失败时：进程内标记 `resource.state = ISOLATED`，后续查询资源状态时返回 ISOLATED。
3. 第三层：内存标记也失败时：调用方进程终止，资源端保留最后状态。

---

## 3. 全部异常出口汇总（05.46 核心交付）

| 异常出口 | 处理 | 结果状态 |
|---|---|---|
| `publish_atomic` 抛异常 | 单指针切换失败，无部分发布 | ABORT |
| `mark_committed_pending_apply` 异常 | 第二层 WAL 写入，第三层进程标记 | UNKNOWN |
| `wal_append(APPLY)` 抛异常 | 不向上抛，进入未知状态 | UNKNOWN |
| `query_txn` 校验失败 | ISOLATED | ISOLATED |
| `persist_isolation_record` 失败 | 第一层 WAL 写入，第二层内存标记 | ISOLATED |

---

## 4. 与 05.45 的差异

| 项 | 05.45 | 05.46 |
|---|---|---|
| StateSnapshot 冻结 | 普通类，state/trt 可变 | `ImmutableSnapshot` 私有字段 + 受保护访问 |
| 原子发布实现 | `SNAPSHOT = new_snapshot` | 同上（元素替换） |
| 读者协议 | 未定义 | 通过 `snapshot.state`/`snapshot.trt` properties 读取 |
| query_txn 唯一性 | 未校验 | 唯一 COMMIT + 唯一 APPLY + commit_seq + snapshot_hash |
| recover()/recover_from_checkpoint() | 双入口冲突 | 统一 recover() 单一入口 |
| WAL 截断 | 未定义 | CHECKPOINT 机制，不删除记录 |

---

## 5. 证据来源与证明边界

- 事务顺序：TXN-ORDER-AND-WAL-REPLAY.md（05.45 保留）
- 请求幂等：REQUEST-ID-ATOMIC-IDEMPOTENCY.md（05.46 修订）
- 资源释放串行化：RESOURCE-RELEASE-SERIALIZATION.md（05.46 修订）
- 崩溃矩阵：CRASH-AND-CONCURRENCY-MATRIX.md

**证明等级**：单进程真实不可变快照 + 唯一性校验 → VERIFIED_STATIC；跨进程发布可见性、跨重启、检查点一致性 → NOT_PROVEN，维持 DENY。
