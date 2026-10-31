# RESOURCE-RELEASE-SERIALIZATION.md — 资源端释放与接管串行化（05.44 权威）

工单：USIT-STAGE-B-DESIGN-05.44
工作包：B44-04（修复 B43-R04）
证据等级：VERIFIED_STATIC（单进程）+ NOT_PROVEN（跨进程/跨重启）
目标：修复 05.43 中幂等查询、授权校验、req_id 占用、operation 执行、WAL 事务提交之间缺乏统一串行化边界，以及授权后接管无法防止旧操作继续执行的缺陷。

---

## 1. B43-R04 缺陷（05.43 真实代码）

05.43 `RESOURCE-FAILURE-AND-RECOVERY.md §3`：

```python
if req_id in consumed_records:
    return 'ALREADY_RELEASED'

op_result = operation()          # <-- 幂等查询之后、WAL 事务之前执行
txn_result = run_txn(...)
```

**缺陷**：两个相同 req_id 并发调用同时通过 `req_id in consumed_records` 查询，然后分别执行 `operation()`，外部业务副作用被执行两次。此外，授权校验与资源接管之间无统一线性化点，旧 Owner 可能在校验通过后、operation 执行前被接管。

---

## 2. 唯一权威规则（05.44 修订）

### 2.1 资源端串行化模型

资源端释放采用**单资源锁 + 幂等键 + 串行化点**：

```
1. 获取资源锁（resource_lock，唯一资源串行化权）
2. 锁内查询幂等键（req_id 与资源绑定状态）
3. 三重授权校验（token + owner + resource）
4. Fencing 检查（generation 单调推进）
5. req_id 占用（标记为已启动，防止重复）
6. 执行 operation（业务副作用）
7. 提交 WAL 事务（含 req_id→txn_id、operation result、resource state）
8. 写 consumed_records（已提交幂等记录）
9. 释放资源锁
```

### 2.2 授权与接管的串行化

- 三重授权校验（token + owner + resource）在资源锁内完成。
- 资源接管在同一资源锁内，与释放操作串行化。
- 旧 Owner 通过授权后、operation 执行前被接管：分配 generation 已推进，旧 Owner 的 token 因 generation 不匹配而拒绝。

---

## 3. 资源端释放伪代码（唯一权威）

```python
def atomic_release(token, req_id, trusted_caller_owner, operation):
    with resource_lock(resource_id):        # 步骤 1：唯一资源串行化权
        # 步骤 2：锁内查询幂等键
        existing = lookup_consumed_in_lock(req_id, resource_id)
        if existing:
            return ('ALREADY_RELEASED', existing.txn_id, None)

        # 步骤 3：三重授权校验（在锁内）
        if not validate_token_in_lock(token, trusted_caller_owner, resource):
            return ('FENCED_OUT', None, None)

        # 步骤 4：Fencing 检查
        if not fencing_check_in_lock(token, resource):
            return ('STALE_TOKEN', None, None)

        # 步骤 5：req_id 占用（标记已启动）
        start_req(req_id, resource_id)

        # 步骤 6：执行 operation（仅在通过授权+Fencing 后）
        try:
            op_result = operation()
        except Exception:
            # 异常处理：标记 req_id 为 ABORT，不得重放
            mark_failed_req(req_id, resource_id, 'OP_FAILED')
            return isolate_release(token, req_id, trusted_caller_owner, 'OP_FAILED')

        # 步骤 7：WAL 事务提交（在锁内，包含 req_id→txn_id、operation result、resource state）
        txn_result = run_txn(req_id, 'RESOURCE_RELEASE', token, trusted_caller_owner,
                             op_record=op_result, resource_state=...)
        if txn_result.status != 'OK':
            # 从事务返回错误，标记 req_id 为 ABORT
            mark_failed_req(req_id, resource_id, txn_result.status)
            return isolate_release(token, req_id, trusted_caller_owner, 'TXN_ABORT_AFTER_OP')

        # 步骤 8：已提交幂等记录（consumed_records 由 WAL 恢复重放处理）
        # 无需显式写 consumed_records，恢复时从 WAL 重建

        return ('OK', txn_result.txn_id, None)
```

**关键**：
1. **资源锁串行化**：所有步骤（幂等查询、授权、Fencing、req_id 占用、operation、WAL 提交）在单一资源锁内完成。
2. **幂等查询在锁内**：两个并发请求只有一个能查询到 req_id 不存在，第二个查询时已存在 ALREADY_PROCESSED。
3. **req_id 占用**：`start_req` 标记 req_id 为已启动，防止后续操作重复执行 operation。
4. **接管串行化**：资源接管在同一资源锁内，与释放操作互斥。

---

## 4. 资源端崩溃恢复

| 崩溃点 | 恢复动作 |
|---|---|
| operation 前崩溃 | req_id 未占用，无 operation，可重试 |
| operation 中 | 副作用可能部分发生，恢复时检查幂等性 |
| WAL COMMIT 前崩溃 | req_id 已占用但未提交，恢复时查询 WAL，若无 COMMIT 则 ISOLATED |
| consumed_records 未写（实际由 WAL 恢复处理） | 恢复重放 WAL，从 COMMIT 重建 consumed_records |

---

## 5. 跨进程与跨重启边界

- 单进程内资源锁 → VERIFIED_STATIC。
- 跨进程资源锁、共享内存竞态 → NOT_PROVEN，维持 ISOLATED/DENY。
- 跨进程接管串行化、跨重启 operation 幂等 → NOT_PROVEN，ISOLATED。

---

## 6. 与 05.43 的差异

| 项 | 05.43 | 05.44 |
|---|---|---|
| 幂等查询位置 | 无 | 资源锁内 |
| operation 与 WAL 边界 | operation 在外，WAL 在事务内 | operation 在锁内，WAL 在事务内 |
| 接管授权 | 仅在授权时 | 串行化在同一资源锁 |
| 重复 operation | 可能 | 锁内幂等查询消除 |

---

## 7. 证据来源与证明边界

- 事务顺序：TXN-ORDER-AND-WAL-REPLAY.md
- 资源端异常：RESOURCE-FAILURE-AND-RECOVERY.md（05.42，保留）
- 崩溃矩阵：CRASH-AND-CONCURRENCY-MATRIX.md

**证明等级**：单进程资源锁串行化 + 锁内 usms → VERIFIED_STATIC；跨进程资源锁、跨重启 → NOT_PROVEN，维持 ISOLATED/DENY。
