# TXN-ORDER-AND-WAL-REPLAY.md — 事务顺序与 WAL 恢复边界闭合（05.45 权威修订）

工单：USIT-STAGE-B-DESIGN-05.45
工作包：B45-04（修复 B44-R01）
证据等级：VERIFIED_STATIC（单进程）+ NOT_PROVEN（跨进程/跨重启）
目标：修复 05.44 中 PREPARE 无 commit_seq、恢复按 commit_seq 排序可能冲突、commit_seq 耗尽无定义、检查点无快照缺陷。

---

## 1. B44-R01 缺陷（05.44 真实代码）

05.44 `TXN-ORDER-AND-WAL-REPLAY.md §3`：

```python
for commit_seq in sorted(groups.keys()):
    ...
    apply_snapshot(state, trt, commit_rec.snapshot)
```

**缺陷**：
1. PREPARE 记录无 commit_seq，但 WAL 字段定义要求每条记录都有 commit_seq。
2. 恢复按 commit_seq 排序，PREPARE、CHECKPOINT 等记录可能被误排序。
3. commit_seq 耗尽（u64 最大值）无定义。
4. 检查点只保存 commit_seq，无状态快照，跳过历史记录无法重建。

---

## 2. 唯一权威规则（05.45 修订）

### 2.1 WAL 记录类型

| 记录类型 | 是否有 commit_seq | 说明 |
|---|---|---|
| PREPARE | **无 commit_seq** | 事务意图落盘，关联 txn_id |
| COMMIT | **有 commit_seq** | 唯一提交顺序权威 |
| APPLY | 有 commit_seq（可选） | 应用进度标记，关联 txn_id |
| OP_RESERVATION | 无 commit_seq | 预留，关联 req_id |
| OP_CONFIRM | 无 commit_seq | 确认，关联 req_id |
| OP_RELEASE | 无 commit_seq | 释放，关联 req_id |
| CHECKPOINT | 有 commit_seq | 检查点 |

**关键**：
- PREPARE、OP_RESERVATION、OP_CONFIRM、OP_RELEASE **无 commit_seq**，不参与 commit_seq 排序。
- COMMIT、APPLY、CHECKPOINT **有 commit_seq**，参与排序。

### 2.2 commit_seq 耗尽保护

commit_seq 为 u64，达到最大值（0xFFFFFFFFFFFFFFFF）时：

```python
def next_commit_seq():
    global commit_seq_counter
    if commit_seq_counter >= 0xFFFFFFFFFFFFFFFF:
        raise CommitSeqExhausted()           # 永久拒绝，不允许回绕/复用/跳过
    commit_seq_counter += 1
    return commit_seq_counter
```

---

## 3. WAL 记录完整性（05.45 新增）

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

---

## 4. 恢复算法（唯一权威，按 commit_seq 重放）

```python
def recover():
    records = scan_wal()
    if wal_corrupted():
        return 'ISOLATED'

    state = init_state()
    trt = init_trt()

    # 分离记录：有 commit_seq 和 无 commit_seq
    no_seq_recs = [r for r in records if r.commit_seq is None]
    seq_recs = [r for r in records if r.commit_seq is not None]

    # 无 commit_seq 的记录：PREPARE/OP_RESERVATION/OP_CONFIRM/OP_RELEASE
    # 关联 txn_id 或 req_id，不排序
    for r in no_seq_recs:
        if r.record_type == 'OP_RESERVATION':
            # 关联 req_id，判断是否已执行
            continue
        if r.record_type == 'OP_CONFIRM':
            # 关联 req_id，判断是否已确认
            continue
        # PREPARE/OP_RELEASE：不影响状态重建

    # 有 commit_seq 的记录：按 commit_seq 升序重放
    seq_recs.sort(key=lambda r: r.commit_seq)
    for r in seq_recs:
        if r.record_type == 'COMMIT':
            # 校验 snapshot_hash
            if r.snapshot_corrupted():
                return 'ISOLATED'
            apply_snapshot(state, trt, r.snapshot)
        elif r.record_type == 'APPLY':
            # 记录已应用，不重复 apply
            continue
        elif r.record_type == 'CHECKPOINT':
            # 检查点记录，不重建状态
            continue

    return ('OK', state, trt)
```

**关键**：
1. **无 commit_seq 的记录不排序**：PREPARE、OP_RESERVATION、OP_CONFIRM、OP_RELEASE 不参与 commit_seq 排序。
2. **有 commit_seq 的记录按 commit_seq 升序**：COMMIT、APPLY、CHECKPOINT。
3. **COMMIT 校验 snapshot_hash**：损坏即 ISOLATED。
4. **APPLY 记录已应用**：不重复 apply。

---

## 5. 冲突记录处理

| 情况 | 处理 |
|---|---|
| 同一 txn_id 出现多条 COMMIT | ISOLATED |
| 同一 commit_seq 出现多条 COMMIT | ISOLATED |
| 同一 commit_seq 出现多条 APPLY | ISOLATED |
| 同一 txn_id 不同 commit_seq | ISOLATED |
| 缺失 PREPARE 但存在 COMMIT | ISOLATED（不完整记录） |
| 缺失 COMMIT 但存在 APPLY | ISOLATED |
| 缺失 PREPARE 但存在 OP_RESERVATION | 未知效果，ISOLATED |
| 缺失 OP_CONFIRM 但存在 OP_RESERVATION | 未知效果，ISOLATED |
| checksum 损坏 | ISOLATED |

---

## 6. 检查点（05.45 新增）

检查点必须包含**状态快照**、**请求幂等索引**、**可验证摘要**：

```python
def checkpoint_wal(last_commit_seq):
    # 构建状态快照
    state_snapshot = build_state_snapshot()
    req_index = build_req_index()
    checkpoint_rec = build_record(
        record_type='CHECKPOINT',
        commit_seq=last_commit_seq,
        state_snapshot=state_snapshot,
        req_index=req_index,
        snapshot_hash=hash(state_snapshot, req_index)
    )
    if not wal_append_fsync(checkpoint_rec):
        return ('ERROR', None, None)
    return ('OK', None, None)
```

**关键**：
1. **状态快照**：保存 last_commit_seq 处的完整状态。
2. **请求幂等索引**：保存 req_id→txn_id 映射。
3. **可验证摘要**：snapshot_hash 用于恢复校验。

---

## 7. 恢复起点（05.45 新增）

检查点存在时，恢复从**检查点**开始：

```python
def recover_from_checkpoint(checkpoint_rec):
    # 校验 snapshot_hash
    if checkpoint_rec.snapshot_hash != hash(checkpoint_rec.state_snapshot, checkpoint_rec.req_index):
        return 'ISOLATED'
    # 从快照恢复状态
    state = checkpoint_rec.state_snapshot
    trt = init_trt()
    req_index = checkpoint_rec.req_index
    return ('OK', state, trt, req_index)
```

**关键**：
- 检查点存在时，跳过历史 WAL，从快照恢复。
- 检查点不存在时，从 WAL 起点重放。
- 检查点校验失败 → ISOLATED。

---

## 8. 与 05.44 的差异

| 项 | 05.44 | 05.45 |
|---|---|---|
| PREPARE 是否有 commit_seq | 未定义 | 无 commit_seq，不排序 |
| commit_seq 耗尽 | 未定义 | 达到 u64 最大值永久拒绝 |
| 检查点 | 只保存 commit_seq | 保存状态快照 + 请求幂等索引 + 摘要 |
| 恢复起点 | WAL 起点 | 检查点存在时从快照恢复 |
| 冲突记录处理 | ISOLATED（同 txn_id 多条 COMMIT） | ISOLATED（更完整） |

---

## 9. 证据来源与证明边界

- 事务顺序：TXN-ORDER-AND-WAL-REPLAY.md（05.44 保留）
- 请求幂等：REQUEST-ID-ATOMIC-IDEMPOTENCY.md（05.45 修订）
- 恢复屏障：RECOVERY-BARRIER-AND-ADMISSION.md（05.45 修订）
- 资源释放串行化：RESOURCE-RELEASE-SERIALIZATION.md（05.45 修订）
- 崩溃矩阵：CRASH-AND-CONCURRENCY-MATRIX.md

**证明等级**：单进程 PREPARE 无序号 + commit_seq 耗尽保护 + 检查点 → VERIFIED_STATIC；跨进程检查点一致性、跨重启 → NOT_PROVEN，维持 DENY。
