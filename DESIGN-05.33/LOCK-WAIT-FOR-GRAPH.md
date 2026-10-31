# LOCK-WAIT-FOR-GRAPH.md

## 目标
锁等待图及死锁分析，与伪代码一致。

## 1. 节点与边

节点：
- 锁节点：coord、commit、identity_alloc、receipt_write、业务资源
- 原子节点（非锁）：admit_state、commit_count、receipt_write_count

边（A→B = 持 A 等 B）：
- 合法锁等待边（全序）：identity_alloc → commit → 业务资源 → receipt_write
- coord：**无锁等待边**（冻结者等待条件变量，非锁）

## 2. 图结构

```
[coord]                 （孤立于锁等待图；冻结者等条件变量，不参与锁环）

identity_alloc ──► commit ──► 业务资源 ──► receipt_write   （全序，无环）
```

## 3. 与 05.32 的差异修正

05.32 将 coord 描述为「孤立节点，无任何等待边」，但伪代码中冻结者实际等待业务归零，导致证明与伪代码不一致。

05.33 修正：
- coord 在**锁等待图**中仍孤立（无锁等待边）。
- 冻结者等待的是**条件变量**（admit_count==0），不是锁。
- 明确区分「条件等待」与「锁等待」，两者不同图。

## 4. 死锁环检测

1. 锁等待边构成全序 → 无环。
2. coord 无锁等待边 → 不参与环。
3. 条件变量等待：wait 前释放所有锁，唤醒后重新获取 → 不持锁等待 → 不形成环。

结论：**无未解释死锁环**。

## 5. 条件等待与锁等待的严格区分

| 类型 | 对象 | 是否持锁 | 是否可成环 |
|---|---|---|---|
| 锁等待 | 锁 | 否（wait 前释放） | 全序无环 |
| 条件等待 | 条件变量（计数归零） | 否 | 无环 |
| 冻结者等待业务归零 | 条件变量 | 否 | 无环 |

## 6. 潜在环反例（已消除）

| 反例 | 环 | 消除 |
|---|---|---|
| coord 持 admit 等待（旧设计） | coord↔admit | 原子字替代，无 admit 锁 |
| coord 持 commit 等待（旧设计） | coord↔commit | 原子计数替代，无 commit 锁 |
| 冻结者持锁等待归零 | 条件等待持锁 | wait 前释放锁 |

## 7. 证据等级
Wait-for Graph 无环：DESIGN_PROVEN（静态）；运行时死锁：NOT_TESTED。
