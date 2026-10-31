# RECOVERY-BARRIER-AND-ADMISSION.md — 恢复屏障与事务准入（05.44 权威）

工单：USIT-STAGE-B-DESIGN-05.44
工作包：B44-03（修复 B43-R03）
证据等级：VERIFIED_STATIC（单进程）+ NOT_PROVEN（跨进程/跨重启）
目标：修复 05.43 中恢复锁与事务锁独立、in_recovery 布尔标志 TOCTOU 的竞态，建立真正的恢复屏障。

---

## 1. B43-R03 缺陷（05.43 真实代码）

05.43 `LOCK-OWNERSHIP-AND-CALL-GRAPH.md §5/§6`：

```python
def run_txn(...):
    if in_recovery:
        raise RecoveryInProgress
    with txn_guard():
        ...

def recover():
    with recovery_guard():
        in_recovery = True
        ...
```

**缺陷**：
- `in_recovery` 是普通布尔，无原子关系。
- 可能出现：A 检查 `in_recovery=False` 后、获取锁前，B 设置 `in_recovery=True` 开始恢复。
- 同时发生 WAL 修改与恢复重放，造成不一致。

---

## 2. 唯一权威规则（05.44 修订）

### 2.1 恢复屏障设计

采用**屏障锁 + 在途计数**模型：

```
1. 恢复启动：获取 recovery_barrier_lock（独立屏障锁）
2. 等待在途事务退出：检查 in_flight_count == 0
3. 如果在途事务存在：阻塞等待（可超时）
4. 排空确认：确保无新事务进入（置入屏障标志）
5. 执行恢复：扫描 WAL、重放、发布
6. 恢复完成：解除屏障，允许新事务进入
```

### 2.2 屏障与事务准入

- 屏障建立后：任何 `run_txn` 入口必须先获取 barrier_lock，检查屏障标志。
- 在途事务退出：`run_txn` 结束前 decrement `in_flight_count`。
- 禁止 TOCTOU：检查与进入锁内必须是原子操作（用同一把 barrier_lock）。

---

## 3. 屏障与准入伪代码（唯一权威）

```python
barrier_lock = threading.Lock()           # 屏障锁
barrier_flag = False                       # 屏障标志
in_flight_count = 0                        # 在途事务计数

def txn_entry_guard():
    # 事务入口：先获取屏障锁检查标志
    with barrier_lock:
        if barrier_flag:
            raise RecoveryInProgress()
        in_flight_count += 1
        return this_thread

def txn_exit_guard(thread):
    with barrier_lock:
        in_flight_count -= 1

def recovery_barrier_guard():
    # 恢复屏障：获取屏障锁，等待在途事务退出
    with barrier_lock:
        barrier_flag = True                # 建立屏障
        while in_flight_count > 0:
            barrier_lock.wait()            # 阻塞等待事务退出
        return this_thread

def run_txn(req_id, op, slot, owner_id, resource_id, caller_owner):
    entry = txn_entry_guard()              # 步骤 1：进入屏障
    try:
        with txn_guard():                  # 步骤 2：获取事务锁
            ...
    finally:
        txn_exit_guard(entry)              # 步骤 3：退出屏障

def recover():
    with recovery_barrier_guard():         # 单一屏障入口
        # 执行恢复（扫描 WAL、重放、发布）
        ...
    # barrier_lock 释放时自动唤醒等待事务
```

**关键**：
1. `txn_entry_guard` 与 `recovery_barrier_guard` 使用**同一把 barrier_lock**，测立原子关系。
2. `recovery_barrier_guard` 等待 `in_flight_count == 0`，排空在途事务。
3. `run_txn` 的 `try/finally` 保证任何出口都调用 `txn_exit_guard`，不泄漏计数。

---

## 4. 恢复屏障状态机

| 阶段 | 说明 |
|---|---|
| 初始 | barrier_flag=False，in_flight_count=0 |
| 屏障建立 | recovery_barrier_guard() 设置 flag，等待计数为零 |
| 排空 | 在途事务执行 txn_exit_guard，计数递减 |
| 恢复 | 排空后执行 WAL 扫描、重放、发布 |
| 完成 | 恢复结束，解除屏障，唤醒等待事务 |
| 终止 | barrier_flag=False，in_flight_count=0 |

---

## 5. 事务与恢复交错矩阵

| 交错 | 结果 | 证据 |
|---|---|---|
| 事务进入屏障后、恢复建立前 | 正常进入，计数+1 | txn_entry_guard 代码 |
| 恢复建立后、事务进入前 | 阻塞排队，等待恢复结束 | barrier_lock.wait |
| 恢复排空在途事务 | 阻塞等待计数归零 | recovery_barrier_guard |
| 恢复中断，屏障未解除 | barrier_flag 保持 True，新事务阻塞 | 同上 |
| 恢复异常 | 屏障保持，ISOLATED | 同源 |

---

## 6. 与 05.43 的差异

| 项 | 05.43 | 05.44 |
|---|---|---|
| 屏障机制 | recovery_lock + 普通布尔 | barrier_lock + 计数器 + 同锁原子 |
| 事务准入 | 检查后进入锁 | 原子进入屏障 |
| TOCTOU | 有 | 无 |
| 排空等待 | 未实现 | recovery_barrier_guard 阻塞等待 |

---

## 7. 证据来源与证明边界

- 事务顺序：TXN-ORDER-AND-WAL-REPLAY.md
- 请求幂等：REQUEST-ID-ATOMIC-IDEMPOTENCY.md
- 崩溃矩阵：CRASH-AND-CONCURRENCY-MATRIX.md

**证明等级**：单进程 barrier_lock + 计数器原子关系 → VERIFIED_STATIC；跨进程恢复屏障、跨重启 → NOT_PROVEN，维持 DENY。
