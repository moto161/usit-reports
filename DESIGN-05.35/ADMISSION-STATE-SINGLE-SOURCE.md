# ADMISSION-STATE-SINGLE-SOURCE.md

## 目标
统一准入状态源，删除独立安全判断副本。

## 1. 唯一权威状态源

**唯一准入状态源 = 单一原子状态字 `_admit_state`**。

05.34 的独立 `_admit_count` 被**删除**。count 字段只存在于 `_admit_state` 内部（bit1–15）。禁止任何独立计数器参与安全判断。

## 2. 原子操作清单

| 操作 | 原子实现 | 线性化点 |
|---|---|---|
| 准入增加 | `CAS(admit_state, [gate==OPEN, count+1])` | CAS 成功 |
| 准入退出 | `CAS(admit_state, count-1)` | CAS 成功 |
| Gate 关闭（冻结） | `CAS(admit_state, gate=OPEN→FREEZING)` | CAS 成功 |
| 冻结完成（FROZEN） | `CAS(admit_state, [gate==FREEZING, count==0]→FROZEN)` | CAS 成功 |
| DENIED 置位 | `CAS(admit_state, →DENIED)` | CAS 成功 |

所有操作读写**同一原子字**，无第二套状态源。

## 3. 条件等待协议观察唯一源

predicate = `admit_state.count == 0`，由 `_drain_lock` 保护。冻结者等待时**只读取 `_admit_state` 的 count 字段**，禁止读取独立滞后计数器。

```python
def _wait_admit_drain(self, timeout):
    with self._drain_lock:
        while (self._admit_state.load() & COUNT_MASK) != 0:
            if timeout_expired(): raise DrainTimeout()
            self._drain_cond.wait(remaining)
```

## 4. 辅助计数/缓存规则

本设计**不保留辅助计数/缓存**。若未来需要缓存，必须满足：
1. 缓存不参与任何安全决策（不用于判断排空）。
2. 缓存更新失败 → 丢弃缓存，从原子字重读。
3. 缓存滞后 → 以原子字为准。

## 5. 线性化点关系

- 准入/冻结/退出/CAS 全序化于同一原子字。
- 冻结者观察到 `count==0` 的线性化点 = 持锁下读取原子字为 0 的点。
- 通知在持锁下发出，先于等待的归零可被重查发现。

## 6. 并发时间线（10 条）

| # | 场景 | 结果 |
|---|---|---|
| 1 | 准入与冻结同时 | CAS 全序，先到者赢 |
| 2 | 最后一个准入退出 | 持锁 count→0 + notify |
| 3 | 多冻结请求 | coord 串行化 |
| 4 | 虚假唤醒 | while 重查 |
| 5 | 通知先于等待 | 持锁重查直接返回 |
| 6 | 超时 | DrainTimeout 回滚 |
| 7 | 取消 | finally 释放回滚 |
| 8 | 冻结完成时 count 非零 | 拒绝 FROZEN |
| 9 | 退出与冻结 CAS 竞争 | 原子字全序 |
| 10 | count 读滞后（若引入缓存） | 以原子字为准 |

## 7. 证明

**定理**：冻结成功时唯一权威 count==0，且之后无新准入。

**证明**：
1. count 只存在于 `_admit_state`，冻结完成 CAS 前置 `count==0`。
2. 冻结（Gate 关闭）CAS 成功后，准入 CAS 前置 `gate==OPEN` 失败 → 无新准入。
3. 退出 CAS 递减 count，冻结完成 CAS 观察 count==0 即无在途业务。
4. 单一源 → 无「两套源独立判断排空」的不一致。

结论：DESIGN_PROVEN（单一原子源，模型级）。

## 8. 证据等级
DESIGN_PROVEN（逻辑）；真实运行时 NOT_TESTED。
