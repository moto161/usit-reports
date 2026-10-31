# ATOMIC-STATE-WORD.md

## 目标
原子状态字与计数器边界完整定义，field-aware CAS。

## 1. `_admit_state` 字段定义

| 字段 | 位 | 掩码 | 取值范围 | 合法组合 |
|---|---|---|---|---|
| gate | bit0 | 0x1 | 0=OPEN, 1=FREEZING | — |
| admit_count | bit1–15 | 0x7FFE | 0..32767 | gate 任意 |
| frozen | bit16 | 0x10000 | 0/1 | 仅在 gate==FREEZING 时可为 1 |

合法组合：`frozen==1 → gate==1`（不变量）；`admit_count` 与 gate 无关（冻结时归零，但字段可短暂非零）。

## 2. 溢出/下溢/非法位拒绝策略

| 情况 | 拒绝 |
|---|---|
| admit_count 达 32767 再准入 | 拒绝准入（返回 AdmitLimit） |
| admit_count 下溢（退出多于准入） | 检测，进入隔离/异常 |
| frozen 位在 gate==OPEN 时置 1 | 非法组合，拒绝，进入隔离 |
| 保留位被写 | field-aware CAS 只改目标字段，保留位不变；检测非法写则隔离 |

## 3. field-aware CAS（禁止裸 `state+1`）

```python
def cas_admit_enter(self):
    while True:
        cur = self._admit_state.load()
        if cur & GATE_MASK:                 # gate==FREEZING
            return False
        if (cur >> 1) & COUNT_MASK >= 32767:
            raise AdmitLimit()
        new = cur + ADMIT_ONE                # 只增 admit_count 字段（bit1 起，步进 2）
        if self._admit_state.cas(cur, new):
            return True                      # 活锁：失败则循环重试
```

`ADMIT_ONE = 2`（bit1 步进），保证不触碰 gate/frozen 位。freeze 用 `new = cur | GATE_MASK`，unfreeze 用 `new = cur & ~GATE_MASK`。

## 4. CAS 失败重试与活锁/饥饿

- 失败重试：循环，读取最新值再 CAS。
- 活锁：field-aware 操作只在字段值变化时重试；因其他线程成功推进，最终本线程会成功（无忙等待死循环保证由公平性/退避提供）。
- 饥饿：准入 CAS 在 gate==FREEZING 期间必然失败（正确拒绝，非饥饿）；gate==OPEN 时无饥饿（各线程独立推进计数）。

## 5. frozen 与 FREEZING 冗余性

`frozen` 位与 `gate==FREEZING` 存在冗余。裁决：**保留 frozen 位用于区分「FREEZING（冻结中，尚未归零）」与「FROZEN（已归零，可快照）」**。一致性不变量：`frozen==1 ⟺ gate==1 且 admit_count==0`。

## 6. 状态转换线性化点

| 转换 | 线性化点 | 失败后状态 |
|---|---|---|
| OPEN→FREEZING | cas_freeze 成功 | 保持 OPEN |
| FREEZING→OPEN（回滚） | cas_unfreeze 成功 | FREEZING |
| FREEZING→FROZEN | drain 完成 + frozen 置位 | FREEZING |
| FROZEN→分叉 | fork 返回 | 回滚 OPEN |

## 7. `commit_count` / `receipt_write_count`

| 项 | 定义 |
|---|---|
| 线性化点 | 提交/回执写入开始时原子增，结束时原子减（各自 field-aware CAS） |
| 事务归属 | 每个计数归属于发起它的线程/事务，生命周期 = 事务执行期 |
| 异常/中断/崩溃 | finally 保证减；崩溃由整体恢复处理（计数随进程消失） |
| 归零通知 | 递减到 0 时 notify 冻结者（同 _drain_cond） |
| 不一致检测 | 计数为负或超过事务数 → 隔离 |

## 8. 不变量

- I-ASW-1：field-aware CAS，禁止裸加。
- I-ASW-2：frozen==1 ⟹ gate==1 且 admit_count==0。
- I-ASW-3：admit_count 不溢出（≤32767）不下溢（≥0）。
- I-ASW-4：commit_count/receipt_write_count 在 admit 生命周期内增减。

## 9. 证据等级
原子状态字协议 DESIGN_PROVEN（基于 CAS 语义，模型级）；真实运行时 NOT_TESTED。
