# LOCK-WAIT-FOR-GRAPH.md

## 目标
锁等待图及死锁环证明。

## 1. Wait-for Graph

节点 = 锁；有向边 A→B 表示「持有 A 的线程等待 B」。

依据 GLOBAL-LOCK-ORDER.md 的权威顺序，合法等待边只有：

```
identity_alloc → gate_state → admit → commit → 业务资源 → receipt_write
```

Fork 协调锁 `coord` 是**独立节点**，与上述任何节点**无入边、无出边**（不嵌套）。

## 2. 图结构

```
[coord]         (孤立节点，无等待边)

identity_alloc ──► gate_state ──► admit ──► commit ──► 业务资源 ──► receipt_write
```

## 3. 死锁环检查

合法等待边构成**全序**（total order），不产生环：

- 若所有线程按全局顺序获取锁，任意时刻等待边方向一致（从低序锁指向高序锁），不可能成环。
- `coord` 孤立，不参与任何环。

结论：**无死锁环**。

## 4. 潜在环反例（已消除）

| 反例 | 环 | 消除规则 |
|---|---|---|
| 线程 A 持 admit 等 coord，线程 B 持 coord 等 admit | admit↔coord 环 | coord 与 admit 不嵌套 |
| 线程 A 持 commit 等 coord，线程 B 持 coord 等 commit | commit↔coord 环 | coord 与 commit 不嵌套 |
| 线程 A 持 commit 等业务，线程 B 持业务等 commit | commit↔业务环 | 统一顺序 commit→业务 |

## 5. 死锁消除规则（唯一权威）

1. 全局锁顺序唯一，禁止逆序获取。
2. coord 与所有业务锁不嵌套。
3. 条件变量 wait 前释放关联锁。
4. 子进程重建全部锁，不继承父持锁状态。
5. 多锁获取失败回滚已获锁。
6. 禁止通过超时/重试/异常捕获掩盖死锁。

## 6. 锁不可用/损坏/状态未知降级

| 情况 | 降级 |
|---|---|
| 锁不可用 | 拒绝操作，记录，进入安全终态 |
| 锁损坏 | 拒绝操作，隔离，需人工 |
| 锁状态未知 | 拒绝操作，NOT_PROVEN，进入安全终态 |

## 7. 反例
20 条锁竞争与死锁反例见 COUNTEREXAMPLES.json（B32-LOCK-021 ~ 040）。

## 8. 证据等级
Wait-for Graph 无环：DESIGN_PROVEN（静态推演）；真实运行时死锁：NOT_TESTED。
