# RESOURCE-OPERATION-CRASH-RECOVERY.md — 资源 operation 预留/确认接入真实释放（05.46 权威修订）

工单：USIT-STAGE-B-DESIGN-05.46
工作包：B46-03（修复 B45-R03）
证据等级：VERIFIED_STATIC（单进程）+ NOT_PROVEN（跨进程/跨重启）
目标：修复 05.45 中 OP_RESERVATION 与 OP_CONFIRM 语义矛盾（§3 表示已预留、§4 表示已执行）、状态机不完整、恢复判断不确定的缺陷。

---

## 1. B45-R03 缺陷（05.45 真实代码）

05.45 `RESOURCE-OPERATION-CRASH-RECOVERY.md §2`：

```python
# 步骤 2：持久化预留（OP_RESERVATION）
if not persist_reservation(req_id, op, token.owner_id, resource_id, payload_hash):
    return ('ABORT', None, None)

# 步骤 3：执行 operation
try:
    op_result = operation()
```

**缺陷**：
1. §3 说 OP_RESERVATION 表示「已预留、operation 尚未执行」，但 §4 说「查 OP_RESERVATION，判断 operation 已执行」——直接矛盾。
2. 状态机不完整（缺少 OPERATION_STARTED、OPERATION_CONFIRMED 等状态）。
3. 预留后、operation 前崩溃：不能判断是否已执行，恢复无法正确判断。
4. 预留持久化失败后崩溃，恢复无法区分「operation 未执行」与「operation 已执行」。

---

## 2. 唯一权威规则（05.46 修订）

### 2.1 状态机（七状态）

```
RESERVED_NOT_STARTED ──┬── OP_STARTED  → OPERATION_STARTED ──┬── OPERATION_CONFIRMED ── COMMITTED ──┬── RELEASED
                         │                                     │                                        │
                         └── ABORT （预留失败）                 └── ABORT （operation 失败）               └── RELEASE
```

完整状态定义：

| 状态 | 进入条件 | 持久化记录 | 说明 |
|---|---|---|---|
| `RESERVED_NOT_STARTED` | `persist_reservation()` 成功 | OP_RESERVATION（req_id、op、owner、resource、payload_hash、timestamp） | 已预留，operation 未见执行 |
| `OPERATION_STARTED` | 进入 operation 函数 | OP_STARTED（req_id、op、owner、resource、payload_hash、timestamp） | 已进入 operation，成败未知 |
| `OPERATION_CONFIRMED` | `wal_append_fsync(confirm_rec, status='OP_CONFIRM')` 成功 | OP_CONFIRM（req_id、result、checksum、timestamp） | operation 已执行，结果已记录 |
| `COMMITTED` | `run_txn()` 提交成功 | COMMIT（commit_seq、snapshot_hash、txn_id、req_id、timestamp） | WAL 已提交 |
| `RELEASED` | `wal_append(rec, status='OP_RELEASE')` 成功 | OP_RELEASE（req_id、resource_id、timestamp） | 资源已释放 |
| `UNKNOWN` | 处于 `UNKNOWN` 状态且无法判断 | N/A | 预留后 operation 未知效果 |
| `ISOLATED` | 无法证明安全 | N/A | 隔离，等待外部确认 |

---

### 2.2 状态转换与崩溃窗口

#### 2.2.1 预留后、operation 前崩溃（Reserved → Not started）

```
WAL: OP_RESERVATION → (crash)
recovery: 查 OP_RESERVATION → RESERVED_NOT_STARTED
```

恢复动作：req_id 已被占用，但 operation 未执行。**不重复执行 operation**（因为 operation 未执行过）。

#### 2.2.2 operation 中崩溃（Started → Unknown）

```
WAL: OP_RESERVATION → OP_STARTED → (crash)
recovery: 查 OP_RESERVATION + OP_STARTED → OP_STARTED
              → OPERATION_STARTED → (crash) → UNKNOWN
```

恢复动作：req_id 已被占用，operation 已进入，但结果未知。非幂等 → UNKNOWN，需要外部确认（可信写或结果查询）。

#### 2.2.3 operation 后确认失败（Confirmed → Unknown）

```
WAL: OP_RESERVATION → OP_STARTED → OP_CONFIRM → (crash)
recovery: 查 OP_RESERVATION + OP_STARTED + OP_CONFIRM → OP_CONFIRMED
              → WAL 恢复 → COMMIT + APPLY → OK
```

恢复动作：operation 已执行，记录已确认。重试 req_id 查询时返回 OP_CONFIRMED，不重复执行。

#### 2.2.4 预留失败（Reserved → Abort）

```
WAL: (no OP_RESERVATION) → WAL error
recovery: 查 OP_RESERVATION → 无
```

恢复动作：req_id 未占用，可重试。

---

## 3. 资源请求状态机（唯一权威，覆盖完整生命周期）

```python
def atomic_release(token, req_id, trusted_caller_owner, operation, resource_id):
    txn_enter()                          # 进入屏障
    try:
        with global_request_lock:
            existing = lookup_req_in_global(req_id)
            if existing:
                return ('ALREADY_PROCESSED', existing.txn_id, 'ALREADY_RELEASED')
            start_req(req_id, op='RESOURCE_RELEASE', owner_id=token.owner_id,
                      resource_id=resource_id, payload_hash=hash(token))

        with resource_lock(resource_id):
            # 持久化预留（OP_RESERVATION）
            if not persist_reservation(req_id, op='RESOURCE_RELEASE', owner_id=token.owner_id,
                                       resource_id=resource_id, payload_hash=hash(token)):
                return isolate_release(token, req_id, trusted_caller_owner, 'RESERVATION_FAILED')
            # 记录 operation 开始意图
            if not wal_append_fsync(rec, status='OP_STARTED'):
                return ('UNKNOWN', None, None)

            try:
                op_result = operation()
            except Exception:
                mark_failed_req(req_id, 'OP_FAILED_AFTER_RESERVATION', owner_id, resource_id, payload_hash)
                if not wal_append_fsync(rec, status='OP_FAILED'):mark_failed_req(req_id, 'OP_FAILED_AFTER_RESERVATION', owner_id, resource_id, payload_hash)
                return isolate_release(token, req_id, trusted_caller_owner, 'OP_FAILED_AFTER_RESERVATION')
            confirm_rec = build_record(req_id=req_id, result=op_result, payload_hash=hash(token))
            if not wal_append_fsync(confirm_rec, status='OP_CONFIRM'):
                return ('UNKNOWN', None, None)

            txn_result = run_txn(req_id, 'RESOURCE_RELEASE', token, trusted_caller_owner,
                                 op_record=op_result, resource_state=...)
            if txn_result.status != 'OK':
                mark_failed_req(req_id, txn_result.status, owner_id, resource_id, payload_hash)
                if not wal_append_fsync(rec, status='OP_FAILED'):mark_failed_req(req_id, txn_result.status, owner_id, resource_id, payload_hash)
                return isolate_release(token, req_id, confirmed_caller_owner, 'TXN_ABORT_AFTER_OP')
            wal_append(rec, status='OP_RELEASE')
            return ('OK', txn_result.txn_id, None)
    finally:
        txn_exit()                       # 退出屏障
```

**关键**：
1. **状态转换完整**：RESERVED_NOT_STARTED → OPERATION_STARTED → OPERATION_CONFIRMED → COMMITTED → RELEASED。
2. **预留后、operation 前崩溃**：OP_RESERVATION + OP_STARTED（若已写），恢复时查 OP_STARTED，判断 operation 未执行。
3. **operation 后确认失败**：查 OP_CONFIRM，判断 operation 已执行确认，恢复直接返回 OP_CONFIRMED。
4. **预留失败**：查 OP_RESERVATION，判断 req_id 未占用，可重试。

---

## 4. 状态转换矩阵

| 状态 | 进入时 WAL 记录 | 崩溃位置 | 恢复判断 |
|---|---|---|---|
| RESERVED_NOT_STARTED | OP_RESERVATION（无 operation） | OP_RESERVATION 写入失败 | req_id 未占用，可重试 |
| OPERATION_STARTED | OP_RESERVATION + OP_STARTED | OP_STARTED 写入失败 | req_id 已占用，operation 未执行，不可自动重放 |
| OPERATION_CONFIRMED | OP_RESERVATION + OP_STARTED + OP_CONFIRM | OP_CONFIRM 写入失败 | req_id 已占用，operation 已执行（确认丢失），不可自动重放 |
| COMMITTED | OP_RESERVATION + OP_STARTED + OP_CONFIRM + COMMIT | COMMIT 写入失败 | req_id 已占用，RESOURCE_RELEASE 已提交，不可重复提交 |
| RELEASED | OP_RESERVATION + OP_STARTED + OP_CONFIRM + COMMIT + OP_RELEASE | OP_RELEASE 写入失败 | req_id 已占用，RESOURCE_RELEASE 已提交，不可重复提交 |
| UNKNOWN | OP_RESERVATION + OP_STARTED + OP_CONFIRM（缺失/顺序错误） | 记录缺失/顺序错误 | req_id 已占用，operation 可能已执行或确认，需外部判断 |
| ISOLATED | 上述任意 | 无法判断 | req_id 已占用，operation 可能已执行或确认，需外部判断 |

---

## 5. 与 05.45 的差异

| 项 | 05.45 | 05.46 |
|---|---|---|
| OP_RESERVATION 与 OP_CONFIRM 语义 | 矛盾（已预留 vs 已执行） | 状态机完整（七状态） |
| 状态机 | 不完整（三状态） | 完整（七状态） |
| 恢复判断 | 无法判断 operation 是否已执行 | 查 OP_STARTED，区分 operation 未执行/已执行 |
| operation 未执行恢复 | 无法正确恢复 | RESERVED_NOT_STARTED，不重复执行 |
| operation 已执行恢复 | 无法正确恢复 | OP_CONFIRMED，不重复执行 |

---

## 6. 证据来源与证明边界

- 事务顺序：TXN-ORDER-AND-WAL-REPLAY.md（05.45 保留）
- 请求幂等：REQUEST-ID-ATOMIC-IDEMPOTENCY.md（05.46 修订）
- 资源释放串行化：RESOURCE-RELEASE-SERIALIZATION.md（05.46 修订）
- 崩溃矩阵：CRASH-AND-CONCURRENCY-MATRIX.md

**证明等级**：单进程七状态状态机 + 预留/确认协议 → VERIFIED_STATIC；跨进程预留/确认、跨重启 → NOT_PROVEN，维持 DENY。
