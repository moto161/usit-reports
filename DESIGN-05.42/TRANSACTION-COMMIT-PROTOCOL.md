# TRANSACTION-COMMIT-PROTOCOL.md — 事务提交协议（05.42 权威）

证据等级：VERIFIED_STATIC（单进程）+ NOT_PROVEN（跨进程/跨重启）
目标：修复 B41-R01/R07——给出唯一权威的 PREPARE/COMMIT/APPLY 提交时序，消除 05.41 的提交顺序矛盾。

---

## 1. 唯一权威提交时序

每个事务严格按以下顺序，**全文档唯一**：

```
1. 构造 WAL 记录（含全部字段变化 + checksum）
2. 写 PREPARE 记录 → fsync          ← PREPARE 持久化
3. 写 COMMIT 记录 → fsync           ← 提交点（唯一权威）
4. APPLY：更新状态字 + TRT 内存     ← 应用（缓存更新）
5. 写 APPLY 记录（可选，用于恢复加速）
6. 返回 OK
```

**提交点 = 步骤 3 的 COMMIT 记录 fsync 成功**。这是唯一权威提交点，全文档统一，不再有「先写 TRT 再 CAS」或「先 CAS 再写 TRT」的歧义。

---

## 2. 事务接口（唯一，try/finally 释放锁）

```python
def run_txn(op, slot, owner_id, resource_id):
    txn_id = next_txn_id()          # 全局单调
    rec = build_record(txn_id, op, slot, owner_id, resource_id, ...)

    try:
        # 步骤 2：PREPARE
        if not wal_append_fsync(rec, status='PREPARE'):
            return 'ABORT'           # PREPARE 落盘失败 → 未提交，丢弃

        # 步骤 3：COMMIT（提交点）
        if not wal_append_fsync(rec, status='COMMIT'):
            return 'UNKNOWN'         # COMMIT 落盘失败 → 状态不确定，见恢复

        # 步骤 4：APPLY（更新内存缓存）
        apply_to_state(rec)
        apply_to_trt(rec)

        # 步骤 5：APPLY 记录（可选）
        wal_append(rec, status='APPLY')   # 非阻塞，失败不影响已提交

        return 'OK'
    except Exception:
        return 'ERROR'               # 异常，见 §4 锁与恢复
```

---

## 3. 事务 ID 与版本

| 字段 | 规则 |
|---|---|
| txn_id | 全局单调递增 u64，唯一，禁止回绕（耗尽 DENY） |
| txn_version | 事务内版本，重放幂等 |
| checksum | 记录校验，损坏 → 隔离 |

---

## 4. 锁生命周期（B42-03，详细见 LOCK-LIFECYCLE-AND-OWNERSHIP.md）

- `run_txn` 内部通过**事务上下文管理器**持有锁，`try/finally` 保证所有路径释放。
- 所有返回路径（ABORT/UNKNOWN/ERROR/OK）都在 finally 中释放锁。
- 禁止在提前返回分支遗留锁。

---

## 5. 崩溃窗口（提交协议视角）

| 崩溃点 | 结果 |
|---|---|
| PREPARE fsync 前 | 未提交，丢弃 |
| PREPARE fsync 后、COMMIT 前 | 未提交，丢弃（PREPARE 无 COMMIT） |
| COMMIT fsync 前 | 未提交（可能丢失） |
| COMMIT fsync 后 | 已提交，重放 APPLY |

**核心**：提交点只有 COMMIT fsync 一处。fsync 前崩溃 = 未提交；fsync 后 = 已提交。无歧义。

---

## 6. 与 05.41 的差异

| 项 | 05.41 | 05.42 |
|---|---|---|
| 提交顺序 | 两种矛盾（先 TRT / 先 CAS） | 唯一：PREPARE→COMMIT(fsync)→APPLY |
| 提交点 | CAS 成功 或 锁释放（矛盾） | COMMIT fsync（唯一） |
| 恢复依据 | 猜状态字/TRT 领先 | 重放 WAL |

---

## 证据来源

- 数据模型：TRANSACTION-DATA-MODEL.md
- 崩溃恢复：TRANSACTION-CRASH-RECOVERY.md
- 方案选型：WAL-OR-ATOMIC-STORAGE-DESIGN.md
- 证明等级：VERIFIED_STATIC（单进程）+ NOT_PROVEN（跨进程/跨重启）
