# TXN-ORDER-AND-WAL-REPLAY.md — 事务顺序与 WAL 恢复边界（05.46 权威修订）

工单：USIT-STAGE-B-DESIGN-05.46
工作包：B46-05 支撑（沿用 05.45 边界，修订 WAL 记录字段与排序规则）
证据等级：VERIFIED_STATIC（单进程）+ NOT_PROVEN（跨进程/跨重启）

---

## 1. WAL 记录完整性（05.46 新增）

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

## 2. 恢复算法（唯一权威，按 commit_seq 重放）

```python
def recover():
    records = scan_wal()
    if wal_corrupted():
        return 'ISOLATED'

    # 查找最近检查点（单一恢复入口，见 WAL-INTEGRITY-AND-CHECKPOINT 05.46 修订）
    checkpoint = find_latest_checkpoint(records)
    if checkpoint is None:
        return recover_from_wal_start(records)
    return recover_from_checkpoint(checkpoint, records)
```

**关键**：
- 统一恢复入口 `recover()`，消除 05.45 双入口冲突。
- recover_from_checkpoint() 从快照恢复 + 跳过检查点前记录（见 WAL-INTEGRITY-AND-CHECKPOINT 05.46 修订）。

---

## 3. 冲突记录处理

| 情况 | 处理 |
|---|---|
| 同一 txn_id 出现多条 COMMIT | ISOLATED |
| 同一 commit_seq 出现多条 COMMIT | ISOLATED |
| 同一 commit_seq 出现多条 APPLY | ISOLATED |
| 同一 txn_id 不同 commit_seq | ISOLATED |
| 缺失 PREPARE 但存在 COMMIT | ISOLATED（不完整记录） |
| 缺失 COMMIT 但存在 APPLY | ISOLATED |
| 缺失 OP_RESERVATION 但存在 OP_CONFIRM | 未知效果，ISOLATED |
| 缺失 OP_CONFIRM 但存在 OP_RESERVATION | 未知效果，ISOLATED |
| checksum 损坏 | ISOLATED |

---

## 4. 与 05.45 的差异

| 项 | 05.45 | 05.46 |
|---|---|---|
| 恢复入口 | recover()/recover_from_checkpoint() 双入口 | 统一 recover() 单一入口 |
| 检查点存在时恢复起点 | 未定义冲突 | 从快照恢复并跳过 checkpoint.commit_seq 前记录 |
| WAL 记录字段 | 未完整定义 | 完整定义（含 wal_position） |

---

## 5. 证据来源与证明边界

- 事务顺序：TXN-ORDER-AND-WAL-REPLAY.md（05.45 保留）
- 请求幂等：REQUEST-ID-ATOMIC-IDEMPOTENCY.md（05.46 修订）
- 资源释放串行化：RESOURCE-RELEASE-SERIALIZATION.md（05.46 修订）
- 崩溃矩阵：CRASH-AND-CONCURRENCY-MATRIX.md

**证明等级**：单进程 WAL 完整 + 统一恢复入口 → VERIFIED_STATIC；跨进程检查点一致性、跨重启 → NOT_PROVEN，维持 DENY。
