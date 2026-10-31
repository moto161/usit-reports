# RESOURCE-OPERATION-CRASH-RECOVERY.md — 资源 operation 预留/确认接入真实释放（05.45 权威修订）

工单：USIT-STAGE-B-DESIGN-05.45
工作包：B45-03（修复 B44-R05）
证据等级：VERIFIED_STATIC（单进程）+ NOT_PROVEN（跨进程/跨重启）
目标：修复 05.44 中非幂等 operation 先持久化预留、再执行副作用、预留后未确认崩溃无法判断缺陷，确保预留协议接入真实释放流程。

---

## 1. B44-R05 缺陷（05.44 真实代码）

05.44 `RESOURCE-OPERATION-CRASH-RECOVERY.md §2`：

```python
def operation_reservation(req_id, operation_id, description):
    with resource_lock(resource_id):
        if not wal_append_fsync(rec, status='OP_RESERVATION'):
            return ('ABORT', None, None)
        result = operation()
        ...
```

05.44 `RESOURCE-RELEASE-SERIALIZATION.md §3`：

```python
def atomic_release(token, req_id, trusted_caller_owner, operation):
    with resource_lock(resource_id):
        ...
        start_req(req_id, resource_id)     # <-- 未定义持久化语义，未调用预留协议
        op_result = operation()
```

**缺陷**：
1. `start_req()` 未调用预留/确认协议。
2. 预留存在但未确认崩溃后，无法判断 operation 是否已发生。
3. 预留记录持久化失败时，恢复无法判断，可能重复执行非幂等操作。

---

## 2. 唯一权威规则（05.45 修订）

### 2.1 预留/确认机制

采用 **预留（Reservation）+ 确认（Confirm）+ 释放（Release）** 三阶段协议，所有 WAL 记录：

1. **预留**：`OP_RESERVATION`，含 `req_id`、`op`、`payload_hash`、`owner_id`、`resource_id`、`timestamp`。
2. **确认**：`OP_CONFIRM`，含 `req_id`、`operation result`、`checksum`。
3. **释放**：`OP_RELEASE`，含 `req_id`、`resource state`。

### 2.2 释放流程接入预留/确认

释放流程必须调用预留/确认协议，不能直接用 `start_req()` 替代：

```python
def atomic_release(token, req_id, trusted_caller_owner, operation, resource_id):
    with global_request_lock:
        # 步骤 1：全局 req_id 原子占用
        ...
        start_req(req_id, op, token.owner_id, resource_id, payload_hash)

    with resource_lock(resource_id):
        # 步骤 2：持久化预留（OP_RESERVATION）
        if not persist_reservation(req_id, op, token.owner_id, resource_id, payload_hash):
            return ('ABORT', None, None)

        # 步骤 3：执行 operation
        try:
            op_result = operation()
        except Exception:
            mark_failed_req(req_id, 'OP_FAILED_AFTER_RESERVATION')
            return isolate_release(token, req_id, trusted_caller_owner, 'OP_FAILED_AFTER_RESERVATION')

        # 步骤 4：持久化确认（OP_CONFIRM）
        confirm_rec = build_record(req_id=req_id, result=op_result)
        if not wal_append_fsync(confirm_rec, status='OP_CONFIRM'):
            return ('UNKNOWN', None, None)

        # 步骤 5：WAL 事务提交（RESOURCE_RELEASE）
        txn_result = run_txn(req_id, 'RESOURCE_RELEASE', token, trusted_caller_owner,
                             op_record=op_result, resource_state=...)
        if txn_result.status != 'OK':
            mark_failed_req(req_id, txn_result.status)
            return isolate_release(token, req_id, trusted_caller_owner, 'TXN_ABORT_AFTER_OP')

        # 步骤 6：释放记录（OP_RELEASE）
        wal_append(rec, status='OP_RELEASE')

        return ('OK', txn_result.txn_id, None)
```

**关键**：
1. **预留/确认接入真实释放**：释放流程调用 `persist_reservation()` 和 `wal_append_fsync(confirm_rec, status='OP_CONFIRM')`。
2. **预留持久化失败 → ABORT**：无法安全执行 operation。
3. **确认持久化失败 → UNKNOWN**：operation 可能已发生但记录失败。
4. **确认失败后的恢复**：查询 WAL 中的 OP_RESERVATION，判断是否已发生，ISOLATED 或重放。

---

## 3. 预留/确认状态机

| 状态 | 说明 |
|---|---|
| OP_RESERVATION | 已预留，operation 未执行 |
| OP_CONFIRM | operation 已执行，记录 result |
| OP_RELEASE | 已释放，request 完成 |
| OP_FAILED | operation 抛异常 |
| ABORT | 预留失败 |
| UNKNOWN | 确认失败 |

---

## 4. 崩溃恢复矩阵

| 崩溃点 | 恢复动作 |
|---|---|
| 预留未持久化 | 未执行，可重试 |
| 预留已持久化、确认未持久化 | 查 OP_RESERVATION，若存在表示已执行 → ISOLATED，外部确认 |
| 确认已持久化 | 已执行，返回 OK |

---

## 5. 与 05.44 的差异

| 项 | 05.44 | 05.45 |
|---|---|---|
| 预留/确认接入 | 仅文档声称 | 真实释放流程调用 |
| 预留持久化失败 | 未定义 | ABORT |
| 确认持久化失败 | 未定义 | UNKNOWN，查 OP_RESERVATION |
| 恢复判断 | 无法判断 operation 是否发生 | 查 OP_RESERVATION |

---

## 6. 证据来源与证明边界

- 事务顺序：TXN-ORDER-AND-WAL-REPLAY.md（05.44 保留）
- 请求幂等：REQUEST-ID-ATOMIC-IDEMPOTENCY.md（05.45 修订）
- 资源释放串行化：RESOURCE-RELEASE-SERIALIZATION.md（05.45 修订）
- 崩溃矩阵：CRASH-AND-CONCURRENCY-MATRIX.md

**证明等级**：单进程预留/确认协议 → VERIFIED_STATIC；跨进程预留/确认、跨重启 → NOT_PROVEN，维持 DENY。
