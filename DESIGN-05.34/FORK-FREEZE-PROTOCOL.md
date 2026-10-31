# FORK-FREEZE-PROTOCOL.md

## 目标
闭合条件变量丢失唤醒窗口，给出完整等待协议与线性化点。

## 1. 同步原语

| 原语 | 用途 |
|---|---|
| `_drain_lock`（Mutex） | 保护 admit_count、admit_epoch、predicate 检查 |
| `_drain_cond`（Condition，绑定 _drain_lock） | 等待业务归零 |
| `admit_epoch`（generation 计数器） | 记录每次 admit_count 变化，辅助通知先于等待 |

## 2. 完整等待协议

predicate：`admit_count == 0`。由 `_drain_lock` 保护，检查与等待注册**在同一锁内原子完成**。

### 冻结者等待（消除丢失唤醒）

```python
def _wait_admit_drain(self, timeout):
    deadline = now() + timeout
    with self._drain_lock:                      # 持锁进入
        while self._admit_count != 0:           # 持锁检查 predicate（循环重查）
            remaining = deadline - now()
            if remaining <= 0:
                raise DrainTimeout()
            # wait 原子地：释放锁 → 睡眠 → 唤醒后重新持锁 → 循环重查
            self._drain_cond.wait(remaining)
        # predicate 满足，返回（已持锁）
        return
```

### 业务准入退出（通知）

```python
def _admit_exit(self):
    with self._drain_lock:                      # 持锁
        self._admit_count -= 1                  # 递减
        self._admit_epoch += 1                  # generation 递增
        if self._admit_count == 0:
            self._drain_cond.notify_all()       # 通知所有冻结者
```

## 3. 关键交错时间线：归零恰在冻结者准备等待时

```
业务线程 B：持锁 → admit_count: 1→0 → notify_all → 释放锁
冻结者 F：  等待 _drain_lock（B 释放后获得）
           → 持锁检查 predicate：admit_count==0 → 直接返回（不进入 wait）
```

**结论**：由于 predicate 检查在持锁下进行，且业务递减也在持锁下进行，两者串行化。若业务先归零，冻结者持锁后立即看到 predicate 满足，直接返回，**不进入 wait** → 无丢失唤醒。

## 4. 虚假唤醒 / 超时 / 取消 / 中断 / 通知先于等待

| 情况 | 处理 |
|---|---|
| 虚假唤醒 | `while` 循环重查 predicate，不满足继续 wait |
| 超时 | deadline 检查，超时抛 DrainTimeout，回滚冻结 |
| 取消 | 抛异常，finally 释放锁，回滚冻结 |
| 中断 | 同取消，安全终态 |
| 通知先于等待 | 持锁重查 predicate，已归零则不 wait，直接返回 |

## 5. 多冻结者并发

`_fork_coord_lock` 串行化冻结发起者（同一时刻只有一个冻结者进入等待）。即使多个冻结者等待，notify_all 唤醒全部，各自持锁重查 predicate，正确。

## 6. 线性化点

- 业务退出线性化点：`admit_count -= 1` 的持锁执行点。
- 冻结者「观察到归零」线性化点：持锁下 `admit_count == 0` 检查为真的点。
- 两者在 `_drain_lock` 下全序化，无竞争窗口。

## 7. 不变量

- I-FF-1：admit_count 仅在 `_drain_lock` 持锁下增减。
- I-FF-2：冻结者 predicate 检查与 wait 注册在同一锁内原子完成。
- I-FF-3：通知在持锁下发出，先于等待的归零可被重查发现。
- I-FF-4：冻结后 admit_count 单调不增。

## 8. 证据等级
等待协议 DESIGN_PROVEN（基于 Mutex+Condition 标准语义，模型级）；真实运行时 NOT_TESTED。
