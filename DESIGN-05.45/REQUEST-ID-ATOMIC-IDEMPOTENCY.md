# REQUEST-ID-ATOMIC-IDEMPOTENCY.md — 全局请求 ID 原子幂等（05.45 权威修订）

工单：USIT-STAGE-B-DESIGN-05.45
工作包：B45-02（修复 B44-R02）
证据等级：VERIFIED_STATIC（单进程）+ NOT_PROVEN（跨进程/跨重启）
目标：修复 05.44 中 req_id 全局唯一性与资源锁冲突、operation 先执行后查询导致的并发重复提交缺陷。

---

## 1. B44-R02 缺陷（05.44 真实代码）

05.44 `REQUEST-ID-ATOMIC-IDEMPOTENCY.md §5`：

```python
def atomic_release(token, req_id, trusted_caller_owner, operation):
    with resource_lock(resource_id):
        if req_id in consumed_records:
            return 'ALREADY_RELEASED'
        start_req(req_id, resource_id)
        op_result = operation()
        ...
        txn_result = run_txn(req_id, ...)
        if txn_result.status == 'OK':
            return 'OK'
```

**缺陷**：
1. req_id 全局唯一，但 `atomic_release()` 用按 `resource_id` 划分的资源锁。
2. 两个请求用相同 req_id、分别操作资源 A 和 B 时，可同时通过各自的幂等查询。
3. 更严重：释放流程**先执行 `operation()`，之后才调 `run_txn()`**；后者即使发现 req_id 已存在，也只是返回 `ALREADY_PROCESSED`，资源释放流程看到 `status == OK`，仍可能把重复副作用当作成功。

---

## 2. 唯一权威规则（05.45 修订）

### 2.1 全局 req_id 原子占用/查询机制

1. req_id 全局唯一，**必须在执行副作用前完成原子占用**。
2. req_id 占用采用**全局请求锁（`global_request_lock`）**，先于资源锁。
3. req_id 占用、参数摘要校验、操作类型校验、Owner 校验、Resource 校验、payload_hash 校验都在执行副作用前完成。

### 2.2 唯一锁顺序

1. `global_request_lock`（全局请求锁）
2. `resource_lock(resource_id)`（资源锁）
3. `txn_guard()`（事务串行化锁）

锁顺序固定：`global_request_lock → resource_lock → txn_guard`，禁止反序，防止死锁。

---

## 3. 资源释放伪代码（唯一权威）

```python
def atomic_release(token, req_id, trusted_caller_owner, operation, resource_id):
    with global_request_lock:                        # 步骤 1：全局请求锁
        # 步骤 2：全局 req_id 原子占用（先于资源锁）
        existing = lookup_req_in_global(req_id)
        if existing:
            return ('ALREADY_PROCESSED', existing.txn_id, 'ALREADY_RELEASED')

        # 步骤 3：全局 req_id 参数摘要校验
        conflict = check_req_conflict(req_id, op, token.owner_id, resource_id, payload_hash)
        if conflict:
            return (conflict + '_ISOLATED', None, None)

        # 步骤 4：标记 req_id 为已占用
        start_req(req_id, op, token.owner_id, resource_id, payload_hash)

    # 步骤 5：获取资源锁
    with resource_lock(resource_id):
        # 步骤 6：幂等查询（在锁内，确认 req_id 占用）
        if req_id in consumed_records:
            return ('ALREADY_PROCESSED', lookup_txn_id(req_id), 'ALREADY_RELEASED')

        # 步骤 7：三重授权校验
        if not validate_token_in_lock(token, trusted_caller_owner, resource):
            return ('FENCED_OUT', None, None)

        # 步骤 8：Fencing 检查
        if not fencing_check_in_lock(token, resource):
            return ('STALE_TOKEN', None, None)

        # 步骤 9：执行 operation（仅在通过授权 + req_id 占用后）
        try:
            op_result = operation()
        except Exception:
            # 异常处理：标记 req_id 为 ABORT，不得重放
            mark_failed_req(req_id, 'OP_FAILED')
            return isolate_release(token, req_id, trusted_caller_owner, 'OP_FAILED')

        # 步骤 10：WAL 事务提交（在锁内，包含 req_id→txn_id、operation result、resource state）
        txn_result = run_txn(req_id, 'RESOURCE_RELEASE', token, trusted_caller_owner,
                             op_record=op_result, resource_state=...)
        if txn_result.status != 'OK':
            mark_failed_req(req_id, txn_result.status)
            return isolate_release(token, req_id, trusted_caller_owner, 'TXN_ABORT_AFTER_OP')

        return ('OK', txn_result.txn_id, None)
```

**关键**：
1. **全局请求锁**：req_id 原子占用先于资源锁。
2. **参数摘要校验**：req_id、op、owner_id、resource_id、payload_hash 在执行副作用前完成冲突校验。
3. **operation 在 req_id 占用后**：先占用 req_id，再执行 operation，防止重复副作用。
4. **锁顺序**：`global_request_lock → resource_lock → txn_guard`，无死锁。

---

## 4. req_id 冲突规则

| 情况 | 结果 |
|---|---|
| req_id 已存在且 op 相同 | ALREADY_PROCESSED |
| req_id 已存在但 op 不同 | CONFLICT_OP_TYPE → ISOLATED |
| req_id 已存在但 owner 不同 | CONFLICT_OWNER → ISOLATED |
| req_id 已存在但 resource 不同 | CONFLICT_RESOURCE → ISOLATED |
| req_id 已存在但 payload_hash 不同 | CONFLICT_PAYLOAD → ISOLATED |

---

## 5. 并发重复请求的结果

| 场景 | 线程 A | 线程 B | 结果 |
|---|---|---|---|
| 同时查询 req_id（锁外） | 不存在 | 不存在 | 分别进入全局请求锁 |
| A 先进入全局请求锁 | 不存在，占用 | 等待 | A 完成 |
| B 后进入全局请求锁 | — | 存在，返回 ALREADY_PROCESSED | B 幂等返回 |

---

## 6. 与 05.44 的差异

| 项 | 05.44 | 05.45 |
|---|---|---|
| req_id 唯一性 | 仅资源锁 | 全局请求锁 + 资源锁 |
| 锁顺序 | 未定义 | global_request_lock → resource_lock → txn_guard |
| operation 执行顺序 | 先执行，后查 req_id | 先占用 req_id，后执行 operation |
| 并发重复请求 | 可能重复提交 | 全局请求锁 + 参数摘要校验消除 |

---

## 7. 证据来源与证明边界

- 事务顺序：TXN-ORDER-AND-WAL-REPLAY.md（05.44 保留）
- 恢复屏障：RECOVERY-BARRIER-AND-ADMISSION.md（05.45 修订）
- 资源释放串行化：RESOURCE-RELEASE-SERIALIZATION.md（05.45 修订）
- 崩溃矩阵：CRASH-AND-CONCURRENCY-MATRIX.md

**证明等级**：单进程全局请求锁 + 锁顺序 → VERIFIED_STATIC；跨进程 req_id 唯一性 → NOT_PROVEN，维持 DENY。
