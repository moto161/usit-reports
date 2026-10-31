# RECOVERY-BARRIER-AND-ADMISSION.md — 恢复屏障与事务准入（05.46 权威修订）

工单：USIT-STAGE-B-DESIGN-05.46
工作包：B46-01（修复 B45-R01）
证据等级：VERIFIED_STATIC（单进程）+ NOT_PROVEN（跨进程/跨重启）
目标：修复 05.45 中 admission_cond 与 recovery_cond 分离导致的事务退出通知发错 Condition、恢复线程无法唤醒的确定性死锁，以及 barrier_flag 跨锁读写竞态。

---

## 1. B45-R01 缺陷（05.45 真实代码）

05.45 `RECOVERY-BARRIER-AND-ADMISSION.md §3`：

```python
admission_cond = threading.Condition()
recovery_cond = threading.Condition()

def txn_exit():
    with admission_cond:
        in_flight_count -= 1
        if in_flight_count == 0:
            admission_cond.notify_all()     # <-- 通知 admission_cond

def recovery_barrier_guard():
    with recovery_cond:
        barrier_flag = True
        while in_flight_count > 0:
            recovery_cond.wait()            # <-- 等待 recovery_cond
```

**缺陷**：
1. 事务退出通知发给 `admission_cond`，恢复线程却在 `recovery_cond.wait()` 上等待；**两个 Condition 不是同一个**，计数归零也无法唤醒恢复线程。
2. `barrier_flag` 在恢复锁（`recovery_cond`）下写入，在准入锁（`admission_cond`）下读取，没有共同的同步保护，存在 TOCTOU 竞态。

---

## 2. 唯一权威规则（05.46 修订）

### 2.1 单一同步权威：`barrier_cond`

采用**单一 `barrier_cond`**（一个 Condition 同时保护屏障标志、在途计数、进入/退出条件），消除双 Condition 不同步：

```python
barrier_cond = threading.Condition()        # 唯一同步权威
barrier_flag = False                        # 屏障标志（barrier_cond 保护）
in_flight_count = 0                         # 在途计数（barrier_cond 保护）

# 事务进入
def txn_enter():
    global in_flight_count
    with barrier_cond:
        if barrier_flag:
            raise RecoveryInProgress()
        in_flight_count += 1

# 事务退出
def txn_exit():
    global in_flight_count
    with barrier_cond:
        in_flight_count -= 1
        if in_flight_count == 0:
            barrier_cond.notify_all()        # 同一 Condition，正确唤醒恢复线程

# 恢复屏障
@contextmanager
def recovery_barrier_guard():
    global barrier_flag
    with barrier_cond:
        barrier_flag = True
        while in_flight_count > 0:
            barrier_cond.wait()              # 同一 Condition，正确等待
        try:
            yield
        except Exception:
            barrier_flag = True               # 异常保持 ISOLATED
            raise
        else:
            barrier_flag = False              # 成功解除屏障
            barrier_cond.notify_all()
```

**关键**：
1. **单一 `barrier_cond`**：屏障标志、在途计数、进入/退出、通知/等待都由同一个 Condition 保护，消除双 Condition 不同步。
2. **正确唤醒**：`txn_exit()` 的 `barrier_cond.notify_all()` 与 `recovery_barrier_guard()` 的 `barrier_cond.wait()` 是**同一个 Condition**，计数归零时正确唤醒恢复线程。
3. **屏障标志同步**：`barrier_flag` 的读写都在 `barrier_cond` 保护下，消除 TOCTOU。

---

## 3. 资源操作完整生命周期计入 in_flight

工单要求：「所有资源操作必须从执行 operation 前开始计入 in_flight，直到 OP_RELEASE、COMMIT/APPLY 或 ISOLATED 处理结束。不得仅在 run_txn 内部计数。」

```python
def atomic_release(token, req_id, trusted_caller_owner, operation, resource_id,
                   op, owner_id, resource_id_param, payload_hash):
    txn_enter()                              # 进入屏障（覆盖完整 operation 生命周期）
    try:
        with global_request_lock:
            existing = lookup_req_in_global(req_id)
            if existing:
                return ('ALREADY_PROCESSED', existing.txn_id, 'ALREADY_RELEASED')
            start_req(req_id, op, owner_id, resource_id_param, payload_hash)

        with resource_lock(resource_id):
            persist_reservation(req_id, op, owner_id, resource_id_param, payload_hash)
            try:
                op_result = operation()
            except Exception:
                mark_failed_req(req_id, 'OP_FAILED_AFTER_RESERVATION')
                return isolate_release(token, req_id, trusted_caller_owner, 'OP_FAILED_AFTER_RESERVATION')
            confirm_rec = build_record(req_id=req_id, result=op_result)
            if not wal_append_fsync(confirm_rec, status='OP_CONFIRM'):
                return ('UNKNOWN', None, None)
            txn_result = run_txn(req_id, 'RESOURCE_RELEASE', token, trusted_caller_owner,
                                 op_record=op_result, resource_state=...)
            if txn_result.status != 'OK':
                mark_failed_req(req_id, txn_result.status)
                return isolate_release(token, req_id, trusted_caller_owner, 'TXN_ABORT_AFTER_OP')
            wal_append(rec, status='OP_RELEASE')
            return ('OK', txn_result.txn_id, None)
    finally:
        txn_exit()                           # 退出屏障（任何路径都执行）
```

**关键**：`txn_enter()` 在函数入口调用，`txn_exit()` 在 `finally` 中调用，覆盖从 req_id 占用、resource_lock、operation、WAL 提交到 OP_RELEASE 的**完整生命周期**，不遗漏计数。

---

## 4. 锁顺序图

```
barrier_cond（唯一同步权威，最先获取）
  ↓
global_request_lock（全局请求锁）
  ↓
resource_lock(resource_id)（资源锁）
  ↓
txn_guard()（事务串行化锁）
```

锁顺序固定：`barrier_cond → global_request_lock → resource_lock → txn_guard`，禁止反序，防止死锁。

---

## 5. Condition 通知图

```
事务进入：barrier_cond.acquire() → 检查 barrier_flag → in_flight_count += 1 → release
事务退出：barrier_cond.acquire() → in_flight_count -= 1 → 若 0 则 notify_all() → release
恢复屏障：barrier_cond.acquire() → barrier_flag = True → wait() 直到 in_flight_count == 0
          → 恢复执行 → barrier_flag = False → notify_all() → release
```

**通知链**：`txn_exit()` 的 `notify_all()` 与 `recovery_barrier_guard()` 的 `wait()` 使用**同一个 `barrier_cond`**，通知正确送达。

---

## 6. 并发交错反例（10 条）

| 反例 | 交错 | 预期结果 | 证据 |
|---|---|---|---|
| B46-BAR-001 | 事务 A txn_enter，计数 1 | A 正常进入 | §3 伪代码 |
| B46-BAR-002 | 恢复建立屏障，计数 1，等待 | 阻塞等待 | §2 recovery_barrier_guard |
| B46-BAR-003 | 事务 A txn_exit，计数 0，notify_all() | 唤醒恢复线程 | §2 txn_exit |
| B46-BAR-004 | 事务 B txn_enter，barrier_flag=True | 抛 RecoveryInProgress | §2 txn_enter |
| B46-BAR-005 | 恢复完成，barrier_flag=False，notify_all() | 唤醒等待事务 B | §2 recovery_barrier_guard else |
| B46-BAR-006 | 恢复异常，barrier_flag 保持 True | 新事务阻塞，ISOLATED | §2 recovery_barrier_guard except |
| B46-BAR-007 | 事务检查 barrier_flag=False 后、获取锁前恢复启动 | 单一 Condition，无 TOCTOU | §2 barrier_flag 同步 |
| B46-BAR-008 | 资源操作未计入 in_flight（05.45 缺陷） | 05.46 覆盖完整生命周期 | §3 txn_enter/finally txn_exit |
| B46-BAR-009 | 双 Condition 不同步通知（05.45 缺陷） | 05.46 单一 Condition，正确唤醒 | §2 单一 barrier_cond |
| B46-BAR-010 | 跨进程恢复屏障 | NOT_PROVEN，DENY | §7 证明边界 |

---

## 7. 证明边界

**证明等级**：单进程单一 `barrier_cond` + 完整生命周期计数 + 锁顺序 → VERIFIED_STATIC；跨进程恢复屏障、跨重启 → NOT_PROVEN，维持 DENY。

---

## 8. 与 05.45 的差异

| 项 | 05.45 | 05.46 |
|---|---|---|
| Condition 数量 | 双 Condition（admission/recovery） | 单一 `barrier_cond` |
| 通知送达 | txn_exit 通知 admission_cond，恢复等待 recovery_cond（死锁） | 同一 Condition，正确唤醒 |
| barrier_flag 同步 | 跨锁读写（TOCTOU） | 单一 Condition 保护，无竞态 |
| in_flight 计数范围 | 仅 run_txn 内部 | 完整 operation 生命周期 |
| 锁顺序 | 未完整定义 | barrier_cond → global_request_lock → resource_lock → txn_guard |
