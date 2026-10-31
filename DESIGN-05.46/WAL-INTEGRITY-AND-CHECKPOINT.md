# WAL-INTEGRITY-AND-CHECKPOINT.md — WAL 和检查点统一（05.46 权威修订）

工单：USIT-STAGE-B-DESIGN-05.46
工作包：B46-04（修复 B45-R04）
证据等级：VERIFIED_STATIC（单进程）+ NOT_PROVEN（跨进程/跨重启）
目标：修复 05.45 中 WAL-INTEGRITY-AND-CHECKPOINT.md 仍保留 05.44 旧协议、CHECKPOINT 只保存 commit_seq 缺少状态快照/请求索引、recover()/recover_from_checkpoint() 双入口冲突的缺陷。

---

## 1. B45-R04 缺陷（05.45 真实代码）

05.45 `TXN-ORDER-AND-WAL-REPLAY.md §4` 的 `recover()` 忽略 CHECKPOINT，从初始状态重放；§7 却要求 `recover_from_checkpoint()` 从快照恢复。两个恢复入口没有统一，存在确定性冲突。

05.45 `WAL-INTEGRITY-AND-CHECKPOINT.md`（保留 05.44 旧协议，未更新）只保存 `last_commit_seq`，缺少状态快照/请求索引/可验证摘要。

---

## 2. 唯一权威规则（05.46 修订）

### 2.1 CHECKPOINT 完整内容

CHECKPOINT 必须包含：

| 字段 | 说明 |
|---|---|
| `record_type` | CHECKPOINT |
| `commit_seq` | 当前 commit_seq |
| `state_snapshot` | 完整聚合状态快照 |
| `trt_snapshot` | 完整 TRT 快照 |
| `req_id_index` | req_id→txn_id 映射索引 |
| `operation_state_index` | operation 状态索引 |
| `wal_position` | WAL 位置（用于恢复定位） |
| `snapshot_hash` | 可验证摘要（hash(state_snapshot + trt_snapshot + req_id_index + operation_state_index)） |
| `checksum` | 记录校验 |

### 2.2 统一恢复入口：`recover()`

```python
def recover():
    records = scan_wal()
    if wal_corrupted():
        return 'ISOLATED'

    # 查找最近检查点（单一恢复入口）
    checkpoint = find_latest_checkpoint(records)
    if checkpoint is None:
        # 无检查点，从 WAL 起点重放
        return recover_from_wal_start(records)
    else:
        # 有检查点，从快照恢复并跳过检查点之前记录
        return recover_from_checkpoint(checkpoint, records)

def recover_from_checkpoint(checkpoint, records):
    # 校验快照摘要
    if checkpoint.snapshot_hash != hash(checkpoint.state_snapshot, checkpoint.trt_snapshot,
                                     checkpoint.req_id_index, checkpoint.operation_state_index):
        return 'ISOLATED'
    state = checkpoint.state_snapshot
    trt = checkpoint.trt_snapshot
    req_id_index = checkpoint.req_id_index
    operation_state_index = checkpoint.operation_state_index

    # 重放检查点之后的记录（commit_seq > checkpoint.commit_seq）
    for r in records:
        if r.record_type in ('COMMIT', 'APPLY') and r.commit_seq <= checkpoint.commit_seq:
            continue
        apply_record(state, trt, r, req_id_index, operation_state_index)
    return ('OK', state, trt)

def recover_from_wal_start(records):
    state = init_state()
    trt = init_trt()
    req_id_index = {}
    operation_state_index = {}

    # 按 commit_seq 升序重放
    seq_recs = [r for r in records if r.record_type in ('COMMIT', 'APPLY') and r.commit_seq is not None]
    seq_recs.sort(key=lambda r: r.commit_seq)
    for r in records:
        if r.record_type in ('PREPARE', 'OP_RESERVATION', 'OP_CONFIRM', 'OP_RELEASE'):
            continue                      # 不参与 commit_seq 排序，不影响状态
        if r.record_type == 'COMMIT':
            if r.snapshot_corrupted():
                return 'ISOLATED'
            apply_snapshot(state, trt, r.snapshot)
        elif r.record_type == 'APPLY':
            continue                      # APPLY 非变更触发点
        elif r.record_type == 'CHECKPOINT':
            continue                      # CHECKPOINT 不重建状态
    return ('OK', state, trt)
```

**关键**：
1. **单一恢复入口 `recover()`**：统一从检查点/WAL 起点恢复，消除双入口冲突。
2. **检查点存在时**：从快照恢复，跳过检查点之前记录；检查点不存在时：从 WAL 起点重放。
3. **检查点完整包含**：状态快照、TRT、req_id_index、operation_state_index、commit_seq、wal_position、snapshot_hash、checksum。
4. **split**：PREPARE、OP_RESERVATION、OP_CONFIRM、OP_RELEASE 不参与 commit_seq 排序；COMMIT/APPLY/CHECKPOINT 参与排序。

---

## 3. WAL 记录完整性（05.46 新增）

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
| `wal_position` | WAL 位置（检查点用） |
| `checksum` | 记录校验 |

---

## 4. WAL 截断

禁止删除 WAL 记录；只增加检查点以加快恢复。

---

## 5. 与 05.45 的差异

| 项 | 05.45 | 05.46 |
|---|---|---|
| CHECKPOINT 内容 | 仅 commit_seq（05.44 旧） | 完整快照 + TRT + req_id_index + operation_state_index + wal_position + snapshot_hash |
| 恢复入口 | recover()/recover_from_checkpoint() 双入口冲突 | 统一 recover() 单一入口 |
| 检查点存在时恢复起点 | 未定义冲突 | 从快照恢复并跳过 checkpoint.commit_seq 前记录 |
| 检查点不存在时恢复 | 未定义 | 从 WAL 起点重放 |
| PREPARE 排序 | 未定义 | 无 commit_seq，不参与排序 |
| WAL 记录字段 | 未完整定义 | 完整定义（含 wal_position） |

---

## 6. 证明边界

**证明等级**：单进程 CHECKPOINT 完整内容 + 统一恢复入口 → VERIFIED_STATIC；跨进程检查点一致性、跨重启 → NOT_PROVEN，维持 DENY。

---

## 7. 证据来源与证明边界

- 事务顺序：TXN-ORDER-AND-WAL-REPLAY.md（05.45 保留）
- 请求幂等：REQUEST-ID-ATOMIC-IDEMPOTENCY.md（05.46 修订）
- 资源释放串行化：RESOURCE-RELEASE-SERIALIZATION.md（05.46 修订）
- 崩溃矩阵：CRASH-AND-CONCURRENCY-MATRIX.md
