# RESOURCE-OPERATION-CRASH-RECOVERY.md — 资源 operation 崩溃恢复（05.44 权威）

工单：USIT-STAGE-B-DESIGN-05.44
工作包：B44-05 支撑（B44-04 operation 幂等性进一步展开）
证据等级：VERIFIED_STATIC（单进程）+ NOT_PROVEN（跨进程/跨重启）

---

## 1. 业务 operation 的崩溃窗口

### 1.1 幂等 operation

业务 operation 具有确定效果且可重复，例如数据库 UPDATE / INSERT（幂等键相同）。此类 operation：

- 崩溃后从 WAL COMMIT 恢复，consumed_records 已写，request 直接返回 OK。
- 重试不会重复执行副作用。

### 1.2 非幂等 operation

业务 operation 无法证明幂等，例如外部 API 调用、状态不可回滚的操作。此类 operation：

- 操作前必须**预留**（reservation），否则崩溃后无法判断。
- 预留机制：开启一个独立的 WAL 事务记录 `operation_reservation`，含 request_id 与操作标识。
- 崩溃恢复：检查 `operation_reservation` 是否已持久化。若存在则表示已执行，request 返回 ISOLATED（由外部确认）。

---

## 2. 预留机制（预留/确认/释放）

`operation_reservation` 机制适用于无法证明幂等的业务：

```
1. 预留：写预留记录（WAL）
2. 执行操作（外部副作用发生）
3. 确认：写确认记录（WAL 记录实际效果）
4. 释放：写释放记录（WAL）
```

崩溃窗口：

| 崩溃点 | 恢复动作 |
|---|---|
| 预留未持久化 | 未执行，可重试 |
| 预留已持久化、确认未持久化 | 未知效果 → ISOLATED，外部确认 |
| 确认已持久化 | 已执行，返回 OK |

---

## 3. RESERVATION 机制伪代码

```python
def operation_reservation(req_id, operation_id, description):
    with resource_lock(resource_id):
        # 步骤 1：预留记录（WAL）
        if not wal_append_fsync(rec, status='OP_RESERVATION'):
            return ('ABORT', None, None)

        # 步骤 2：执行操作
        try:
            result = operation()
        except Exception:
            return ('ISOLATED', None, 'OPERATION_FAILED_AFTER_RESERVATION')

        # 步骤 3：确认记录（WAL）
        confirm_rec = build_record(result=result)
        if not wal_append_fsync(confirm_rec, status='OP_CONFIRM'):
            return ('UNKNOWN', None, None)

        # 步骤 4：释放记录（WAL）
        wal_append(rec, status='OP_RELEASE')

        return ('OK', None, None)
```

**关键**：
- 预留记录与确认记录都持久化，崩溃后从 WAL 重放判断 operation 是否已执行。
- 预留未持久化：重试安全。
- 确认未持久化：ISOLATED，外部确认。

---

## 4. RESERVATION 机制与幂等性的权衡

| 模式 | 适用 | 崩溃后处理 |
|---|---|---|
| 幂等 operation | 可重复、确定性 | 恢复直接重放，OK |
| 预留+确认机制 | 不可重复、外部副作用 | 重放判断预留状态，OK 或 ISOLATED |
| 纯幂等 | 不确定 | ISOLATED，外部确认 |

---

## 5. 结论

- 幂等 operation：直接重放 OK。
- 非幂等 operation：使用预留+确认机制，崩溃后从 WAL 判断状态。
- 预留持久化失败 → ISOLATED。
- 跨进程 operation、跨重启预留 → NOT_PROVEN。
