# WAL-INTEGRITY-AND-CHECKPOINT.md — WAL 完整性与检查点（05.44 权威）

工单：USIT-STAGE-B-DESIGN-05.44
工作包：B44-05（修复 B43-R08，作为 POST-COMMIT-FAILURE-CLOSURE 的支撑文档）
证据等级：VERIFIED_STATIC（单进程）+ NOT_PROVEN（跨进程/跨重启）

---

## 1. WAL 记录完整性

### 1.1 完整 WAL 记录字段

每个 WAL 记录必须包含：

| 字段 | 说明 |
|---|---|
| `txn_id` | 事务逻辑标识（锁内分配） |
| `commit_seq` | 全局单调提交顺序（锁内分配，COMMIT 时唯一） |
| `op` | 操作类型（ADMIT/RELEASE/REVOKE/FREEZE/UNFREEZE/SET_DENIED 等） |
| `snapshot` | 完整候选状态快照（COMMIT 记录） |
| `checksum` | 记录校验 |
| `status` | 记录状态（PREPARE/COMMIT/APPLY/RESERVATION/CONFIRM/ISOLATED） |
| `req_id` | 请求幂等键 |

### 1.2 记录完整性校验

- 每条记录写入前计算 checksum。
- 每条记录写入后 fsync。
- 恢复扫描时校验 checksum，损坏 → ISOLATED。

---

## 2. WAL 截断与压缩

- **禁止删除 WAL 记录**：WAL 记录不可删除，只记录检查点以加快恢复。
- **检查点机制**：记录最近一次成功恢复的 commit_seq，便于快速定位恢复起点。
- **历史保留**：检查点只记录位置，不删除记录。

---

## 3. 检查点伪代码（唯一权威）

```python
def checkpoint_wal(last_commit_seq):
    # 记录最近一次成功恢复的 commit_seq
    cp_rec = build_record(last_commit_seq=last_commit_seq, timestamp=now())
    if not wal_append_fsync(cp_rec, status='CHECKPOINT'):
        return ('ERROR', None, None)   # 截断失败
    return ('OK', None, None)
```

**关键**：
- 检查点记录与 WAL 记录统一格式。
- 检查点写入失败 → ERROR，不截断。

---

## 4. 恢复与检查点边界

### 4.1 恢复起点

- **从检查点开始**：恢复时从检查点记录的 commit_seq 开始扫描 WAL。
- **跳过检查点之前的记录**：检查点之前的记录已恢复过，跳过。
- **恢复终点**：WAL 末尾（或损坏记录处）。

### 4.2 恢复策略

- **重放**：从 WAL 重放，跳过检查点之前的记录。
- **恢复失败**：检查点无效 → ISOLATED。
- **恢复中断**：检查点有效，从未完成处恢复。

---

## 5. WAL 检查点与 req_id 映射保留

- **req_id 映射**：req_id→txn_id 映射随 WAL COMMIT 记录持久化，历史保留。
- **检查点与 req_id**：检查点不删除 req_id 映射，历史保留。
- **清理规则**：仅当资源退役且所有历史请求不可能再到达时，才允许归档清理 req_id 映射。

---

## 6. WAL 记录校验失败

### 6.1 校验失败情况

| 情况 | 处理 |
|---|---|
| 记录 checksum 损坏 | ISOLATED，DENY，不猜测 |
| 记录截断（半条记录） | ISOLATED，DENY |
| 记录格式错误 | ISOLATED，DENY |
| 记录类型未知 | ISOLATED，DENY |

---

## 7. 与 05.43 的差异

| 项 | 05.43 | 05.44 |
|---|---|---|
| WAL 截断/压缩 | 未定义 | CHECKPOINT 机制，不删除记录 |
| 检查点记录 | 无 | WAL 统一格式记录 |
| 恢复起点 | WAL 开头 | 检查点开始 |
| 校验失败 | 单点 checksum | 完整校验矩阵 |

---

## 8. 证据来源与证明边界

- 事务顺序：TXN-ORDER-AND-WAL-REPLAY.md
- 恢复屏障：RECOVERY-BARRIER-AND-ADMISSION.md
- 崩溃矩阵：CRASH-AND-CONCURRENCY-MATRIX.md

**证明等级**：单进程 WAL 完整性与检查点 → VERIFIED_STATIC；跨进程检查点一致性、跨重启 → NOT_PROVEN，维持 DENY。
