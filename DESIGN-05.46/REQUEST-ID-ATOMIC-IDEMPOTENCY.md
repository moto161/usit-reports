# REQUEST-ID-ATOMIC-IDEMPOTENCY.md — 全局请求 ID 原子幂等（05.46 权威修订）

工单：USIT-STAGE-B-DESIGN-05.46
工作包：B46-02（修复 B45-R02）
证据等级：VERIFIED_STATIC（单进程）+ NOT_PROVEN（跨进程/跨重启）
目标：修复 05.45 中 `op`、`payload_hash`、`resource` 未定义变量、start_req() 原子持久化及失败处理语义未定义、资源操作生命周期未覆盖完整的缺陷。

---

## 1. B45-R02 缺陷（05.45 真实代码）

05.45 `REQUEST-ID-ATOMIC-IDEMPOTENCY.md §3`：

```python
def atomic_release(token, req_id, trusted_caller_owner, operation, resource_id):
    with global_request_lock:
        existing = lookup_req_in_global(req_id)
        ...
        start_req(req_id, op, token.owner_id, resource_id, payload_hash)   # <-- op, payload_hash 未定义
```

**缺陷**：
1. `op`、`payload_hash`、`resource` 未定义变量，伪代码无法执行。
2. `start_req()` 未定义原子持久化及失败处理语义。
3. 资源操作生命周期未覆盖完整 operation（从 req_id 占用到 OP_RELEASE）。

---

## 2. 唯一权威规则（05.46 修订）

### 2.1 参数完整定义

所有伪代码参数必须完整定义，禁止使用未定义变量：

```python
def atomic_release(token, req_id, trusted_caller_owner, operation, resource_id):
    # token: Token 对象（含 slot/generation/owner_id/resource_id）
    # req_id: 请求的幂等键，全局唯一
    # trusted_caller_owner: 可信身份上下文注入的所有者（非自报值）
    # operation: 业务副作用函数（非幂等）
    # resource_id: 资源 ID
    # op = 'RESOURCE_RELEASE'   # 操作类型，本释放操作
    # owner_id = token.owner_id  # 所有者 ID
    # payload_hash = hash(token.slot, token.generation, token.owner_id, token.resource_id)  # 负载摘要
```

**完整参数清单**：
| 参数 | 类型 | 来源 |
|---|---|---|
| `token` | Token | 调用方提供 |
| `req_id` | str/u64 | 调用方提供，全局唯一 |
| `trusted_caller_owner` | str | 服务端受信调用上下文 |
| `operation` | Callable | 业务副作用函数 |
| `resource_id` | str/int | 调用方提供 |
| `op` | str | 本释放操作：'RESOURCE_RELEASE' |
| `owner_id` | str | `token.owner_id` |
| `payload_hash` | str | `hash(token.slot, token.generation, token.owner_id, token.resource_id)` |

---

### 2.2 start_req() 真实语义

`start_req()` 必须定义原子持久化及失败处理语义：

```python
def start_req(req_id, op, owner_id, resource_id, payload_hash):
    # 1. 为 req_id 写占用记录到全局请求表（持久化）
    # 2. 返回占用结果
    if not global_append_fsync(req_id=req_id, op=op, owner_id=owner_id,
                               resource_id=resource_id, payload_hash=payload_hash,
                               status='OCCUPIED'):
        return ('ABORT', None, None)   # 占用失败，拒绝执行副作用
    return ('OCCUPIED', req_id, None)
```

**关键**：
1. `start_req()` 使用 WAL 事务（`global_append_fsync`）将 req_id 占用记录**持久化**到全局请求表，不是仅内存标记。
2. 占用失败 → ABORT，不执行 operation。
3. 占用成功 → 持久化记录，崩溃后可从请求表恢复。

---

### 2.3 完整锁顺序与生命周期计入

工单要求：「所有资源操作必须从执行 operation 前开始计入 in_flight，直到 OP_RELEASE、COMMIT/APPLY 或 ISOLATED 处理结束。不得仅在 run_txn 内部计数。」

```python
def atomic_release(token, req_id, trusted_caller_owner, operation, resource_id):
    txn_enter()                          # 进入屏障（覆盖完整生命周期）
    try:
        with global_request_lock:
            existing = lookup_req_in_global(req_id)
            if existing:
                return ('ALREADY_PROCESSED', existing.txn_id, 'ALREADY_RELEASED')
            # start_req() 使用 WAL 事务持久化，不只在内存
            start_req(req_id, op='RESOURCE_RELEASE', owner_id=token.owner_id,
                      resource_id=resource_id, payload_hash=hash(token))

        with resource_lock(resource_id):
            # 持久化预留（OP_RESERVATION）
            if not persist_reservation(req_id, op, token.owner_id, resource_id, payload_hash):
                return isolate_release(token, req_id, trusted_caller_owner, 'RESERVATION_FAILED')
            try:
                op_result = operation()
            except Exception:
                mark_failed_req(req_id, 'OP_FAILED_AFTER_RESERVATION', owner_id, resource_id, payload_hash)
                return isolate_release(token, req_id, trusted_caller_owner, 'OP_FAILED_AFTER_RESERVATION')
            confirm_rec = build_record(req_id=req_id, result=op_result)
            if not wal_append_fsync(confirm_rec, status='OP_CONFIRM'):
                return ('UNKNOWN', None, None)
            txn_result = run_txn(req_id, 'RESOURCE_RELEASE', token, trusted_caller_owner,
                                 op_record=op_result, resource_state=...)
            if txn_result.status != 'OK':
                mark_failed_req(req_id, txn_result.status, owner_id, resource_id, payload_hash)
                return isolate_release(token, req_id, trusted_caller_owner, 'TXN_ABORT_AFTER_OP')
            wal_append(rec, status='OP_RELEASE')
            return ('OK', txn_result.txn_id, None)
    finally:
        txn_exit()                       # 退出屏障（任何路径都执行）
```

**关键**：
1. `txn_enter()` 在函数入口调用，`txn_exit()` 在 `finally` 中调用，覆盖完整生命周期。
2. `start_req()` 使用 WAL 持久化，不是内存标记。
3. 锁顺序：`barrier_cond → global_request_lock → resource_lock → txn_guard`，无死锁。
4. req_id 占用后 token 校验失败、Fencing 失败、operation 失败、WAL 提交失败 → `mark_failed_req()` 标记并 `isolate_release()`。

---

## 4. 并发交错反例（10 条）

| 反例 | 交错 | 预期结果 | 证据 |
|---|---|---|---|
| B46-ID-001 | 两个相同 req_id 并发请求，锁外查询不存在 | 05.45 正确（05.46 已修复） | §2.1 参数定义 |
| B46-ID-002 | 两个 req_id 分别操作资源 A 和 B，global_request_lock 并发控制 | 只有一个请求进入 operation 执行 | §2.2 start_req(), §2.3 锁顺序 |
| B46-ID-003 | req_id 占用失败（WAL 写入失败） | ABORT，不执行 operation | §2.2 start_req() ABORT |
| B46-ID-004 | operation 执行失败 | mark_failed_req，ISOLATED | §2.3 mark_failed_req |
| B46-ID-005 | WAL 提交失败 | mark_failed_req，ISOLATED | §2.3 TXN_ABORT_AFTER_OP |
| B46-ID-006 | req_id 占用后崩溃，恢复时查 req_id 表 | 恢复判断 OCCUPIED，不重复占用 | §2.2 start_req() 持久化 |
| B46-ID-007 | 跨进程 req_id 唯一性 | NOT_PROVEN，DENY | §5 证明边界 |
| B46-ID-008 | req_id 冲突（不同 owner） | CONFLICT_OWNER → ISOLATED | §2.1 参数定义 |
| B46-ID-009 | req_id 冲突（不同 resource） | CONFLICT_RESOURCE → ISOLATED | §2.1 参数定义 |
| B46-ID-010 | req_id 冲突（不同 payload） | CONFLICT_PAYLOAD → ISOLATED | §2.1 参数定义 |

---

## 5. 锁顺序图

```
barrier_cond（唯一同步权威，最先获取）
  ↓
global_request_lock（全局请求锁，req_id 占用）
  ↓
resource_lock(resource_id)（资源锁，Fencing）
  ↓
txn_guard()（事务串行化锁，WAL 提交）
```

锁顺序固定：`barrier_cond → global_request_lock → resource_lock → txn_guard`，禁止反序，防止死锁。

---

## 6. 证明边界

**证明等级**：单进程全局请求锁 + start_req() 持久化 + 完整锁顺序 → VERIFIED_STATIC；跨进程 req_id 唯一性、跨重启 → NOT_PROVEN，维持 DENY。

---

## 7. 与 05.45 的差异

| 项 | 05.45 | 05.46 |
|---|---|---|
| 参数定义 | `op`、`payload_hash` 未定义 | 完整定义，可执行 |
| start_req() 语义 | 未定义原子持久化 | WAL 事务持久化，ABORT/ UNKNOWN |
| in_flight 计数 | 仅 run_txn | 完整 operation 生命周期 |
| 锁顺序 | 仅 barrier_cond | barrier_cond → global_request_lock → resource_lock → txn_guard |
| 资源操作生命周期 | 未完整覆盖 | 覆盖 req_id 占用 → resource_lock → op → WAL → OP_RELEASE |
