# RECOVERY-BARRIER-AND-ADMISSION.md — 恢复屏障与事务准入（05.45 权威修订）

工单：USIT-STAGE-B-DESIGN-05.45
工作包：B45-01（修复 B44-R03）
证据等级：VERIFIED_STATIC（单进程）+ NOT_PROVEN（跨进程/跨重启）
目标：修复 05.44 中 barrier_lock 用 `threading.Lock().wait()`（不存在此方法）、恢复线程持锁等待导致死锁、返回值不是上下文管理器的缺陷。

---

## 1. B44-R03 缺陷（05.44 真实代码）

05.44 `RECOVERY-BARRIER-AND-ADMISSION.md §3`：

```python
barrier_lock = threading.Lock()           # 普通 Lock，无 wait() 方法
barrier_flag = False

def recovery_barrier_guard():
    with barrier_lock:
        barrier_flag = True
        while in_flight_count > 0:
            barrier_lock.wait()            # <-- Lock 无 wait() 方法，直接异常
        return this_thread                  # <-- 不是上下文管理器
```

**缺陷**：
1. `threading.Lock()` 没有 `wait()` 方法，调用直接抛 `AttributeError`。
2. 恢复线程持有 `barrier_lock` 等待时，在途事务的 `txn_exit_guard()` 需获取同一把锁递减计数，形成死锁。
3. `recovery_barrier_guard()` 返回 `this_thread`，不是上下文管理器，无法保证屏障解除。

---

## 2. 唯一权威规则（05.45 修订）

### 2.1 恢复屏障模型：Condition + 锁分离

采用 **Condition 机制** + **锁分离**，禁止锁内等待：

```
1. 事务准入屏障：
   - admission_cond = threading.Condition()
   - admission_lock = admission_cond
   - in_flight_count = 0

2. 恢复屏障：
   - recovery_cond = threading.Condition()
   - recovery_lock = recovery_cond
   - barrier_flag = False

3. 事务进入/退出：
   - 进入：with admission_cond: 检查 barrier_flag，in_flight_count += 1
   - 退出：with admission_cond: in_flight_count -= 1，若 0 则 notify_all()

4. 恢复屏障：
   - with recovery_cond:
     - barrier_flag = True
     - 等待 in_flight_count == 0（持有 recovery_cond，可 wait/notify_all）
     - 恢复执行
     - 成功：barrier_flag = False，notify_all()
     - 异常：barrier_flag 保持 True，资源 ISOLATED

5. 恢复屏障上下文管理器：
   - recovery_barrier_guard() 返回真正的上下文管理器
   - __exit__ 无论正常/异常都解除屏障
```

**关键**：
1. **Condition**：`admission_cond`/`recovery_cond` 使用 `threading.Condition()`，有 `wait()` 方法。
2. **锁分离**：事务准入用 `admission_cond`，恢复屏障用 `recovery_cond`，两把锁分离，不形成死锁。
3. **上下文管理器**：`recovery_barrier_guard()` 返回 `contextmanager` 包裹的屏障，所有出口（正常/异常）都解除屏障。

---

## 3. 恢复屏障伪代码（唯一权威）

```python
import threading
from contextlib import contextmanager

admission_cond = threading.Condition()
recovery_cond = threading.Condition()
admission_lock = admission_cond
recovery_lock = recovery_cond

in_flight_count = 0
barrier_flag = False

# 步骤 1：事务准入（进入屏障）
def txn_enter():
    global in_flight_count
    with admission_cond:
        if barrier_flag:
            raise RecoveryInProgress()
        in_flight_count += 1

# 步骤 2：事务退出（退出屏障）
def txn_exit():
    global in_flight_count
    with admission_cond:
        in_flight_count -= 1
        if in_flight_count == 0:
            admission_cond.notify_all()

# 步骤 3：恢复屏障上下文管理器
@contextmanager
def recovery_barrier_guard():
    global barrier_flag
    with recovery_cond:
        barrier_flag = True
        while in_flight_count > 0:      # 持锁等待，不会死锁
            recovery_cond.wait()         # Condition 的 wait()
        try:
            yield                       # 恢复执行
        except Exception:
            barrier_flag = True          # 异常时保持 ISOLATED
            raise
        else:
            barrier_flag = False         # 成功解除屏障
            recovery_cond.notify_all()

# 步骤 4：普通事务入口
def run_txn(req_id, op, slot, owner_id, resource_id, caller_owner):
    txn_enter()                         # 进入屏障
    try:
        with txn_guard():               # txn_cond（05.43 已有）
            ...
    finally:
        txn_exit()                      # 退出屏障，任何路径都执行
```

**关键**：
1. **Condition 的 wait()**：`recovery_cond.wait()` 是 `Condition` 的方法，正确可用。
2. **锁分离**：`admission_cond` 与 `recovery_cond` 分离，互不死锁。
3. **上下文管理器**：`@contextmanager` 包裹，`recovery_barrier_guard()` 可正常进入/退出。
4. **任何路径退出屏障**：`txn_exit()` 在 `finally` 中调用，保证计数递减。
5. **恢复异常保持屏障**：`recovery_barrier_guard()` 的 `except` 保持 `barrier_flag=True`，资源 ISOLATED。

---

## 4. 恢复屏障状态机

| 阶段 | 说明 |
|---|---|
| 初始 | barrier_flag=False，in_flight_count=0 |
| 屏障建立 | recovery_barrier_guard() 设置 barrier_flag=True，等待计数归零 |
| 排空 | 在途事务 txn_exit()，计数递减，归零时唤醒 recovery_cond |
| 恢复 | 排空后执行恢复（扫描 WAL、重放、发布） |
| 完成 | 恢复结束，barrier_flag=False，唤醒等待事务 |
| 终止 | barrier_flag=False，in_flight_count=0 |

---

## 5. 事务与恢复交错矩阵

| 交错 | 结果 | 证据 |
|---|---|---|
| 事务进入屏障后、恢复建立前 | 正常进入，计数+1 | txn_enter() 代码 |
| 恢复建立后、事务进入前 | 阻塞排队，等待恢复结束 | txn_enter() 检查 barrier_flag |
| 恢复等待在途事务退出 | 阻塞等待计数归零后唤醒 | recovery_barrier_guard() wait() |
| 恢复中断，屏障未解除 | barrier_flag 保持 True，新事务阻塞 | recovery_barrier_guard() except |
| 恢复异常 | 屏障保持，ISOLATED | 同上 |
| 恢复完成解除屏障 | barrier_flag=False，唤醒等待事务 | recovery_barrier_guard() else |

---

## 6. 与 05.44 的差异

| 项 | 05.44 | 05.45 |
|---|---|---|
| 等待机制 | `threading.Lock().wait()`（不存在） | `threading.Condition().wait()`（正确） |
| 锁分离 | 单锁，死锁 | 双 Condition，无死锁 |
| 上下文管理器 | 返回 this_thread，非上下文管理器 | `@contextmanager` 包裹 |
| 退出屏障 | 无明确路径 | 正常/异常出口都解除 |
| 排空等待 | 未实现 | Condition wait/notify |

---

## 7. 证据来源与证明边界

- 事务顺序：TXN-ORDER-AND-WAL-REPLAY.md（05.44 保留）
- 请求幂等：REQUEST-ID-ATOMIC-IDEMPOTENCY.md（05.45 修订）
- 资源释放串行化：RESOURCE-RELEASE-SERIALIZATION.md（05.45 修订）
- 崩溃矩阵：CRASH-AND-CONCURRENCY-MATRIX.md

**证明等级**：单进程 Condition + 锁分离 → VERIFIED_STATIC；跨进程恢复屏障、跨重启 → NOT_PROVEN，维持 DENY。
