# GLOBAL-LOCK-ORDER.md

## 目标
统一锁协议（唯一权威），检查所有调用链含间接锁获取。

## 1. 唯一权威锁清单

| 锁 | 用途 | 嵌套 |
|---|---|---|
| _admit_state | 原子字：Gate + admit_count 合并（非锁，用 CAS） | — |
| _commit_count | 原子提交计数（非锁） | — |
| _receipt_write_count | 原子回执计数（非锁） | — |
| _fork_coord_lock | 串行化冻结发起者 | **禁止嵌套任何业务锁** |
| _commit_lock | 保护提交临界区 | 可嵌套业务资源 |
| _identity_alloc_lock | 保护身份序号 | 叶子锁 |
| _receipt_write_lock | 保护回执写入 | 在 commit 内 |
| 业务资源锁 | 保护业务数据 | 在 commit 内 |

## 2. 锁获取/释放矩阵

| 路径 | 获取顺序 | 嵌套 |
|---|---|---|
| 业务准入 | CAS(admit_state) 原子 | 无锁 |
| 业务退出 | CAS(admit_state) 原子 | 无锁 |
| 事务提交 | commit → 业务资源 → receipt | 固定顺序 |
| 身份分配 | identity_alloc | 单锁 |
| Fork 冻结 | coord（串行化）→ CAS(admit_state) freeze | coord 不嵌套业务锁 |
| Fork 等待归零 | 无锁（条件变量 wait） | 无锁 |
| 快照 | 无锁（冻结态业务归零） | 无锁 |

## 3. 完整调用链及间接锁获取

| 函数 | 直接获取锁 | 间接获取锁 | 违规？ |
|---|---|---|---|
| _fork_freeze | _fork_coord_lock | 无 | 否 |
| _close_admit（旧） | _admit_lock | — | **已废除，改用原子字** |
| _assert_no_commit（旧） | _commit_lock | — | **已废除，改用原子计数** |
| _assert_receipt_idle | 无（读原子计数） | 无 | 否 |
| _take_snapshot | 无 | 无 | 否 |

**关键修复**：05.32 的 `_close_admit()`（获取 admit_lock）、`_assert_no_commit()`（获取 commit_lock）在 coord 内被调用，违反锁顺序。05.33 改用**原子字 CAS**，彻底移除 coord 内业务锁获取。

## 4. Wait-for Graph（与伪代码一致）

```
coord ──（串行化冻结者，无业务锁边）
冻结者等待：admit_count==0（条件变量，非锁等待）
```

coord 节点与业务锁节点之间**无锁等待边**。冻结者等待的是「原子计数归零」这一条件，而非「获取某把业务锁」。因此不形成锁环。

## 5. 死锁环检测

合法锁顺序全序：identity → commit → 业务资源 → receipt。coord 独立。无环。

## 6. 禁止项

- 只检查直接调用遗漏间接调用 → 已检查完整调用链。
- 只改权威不改引用 → 引用文档已同步（见 CROSS-DOCUMENT-CONSISTENCY）。
- Wait-for Graph 与伪代码不一致 → 已对齐。
- 把等待业务归零误判为锁依赖 → 已区分「条件等待」与「锁等待」。
- 用「理论上不会发生」代替证明 → 全部给出证明。

## 7. 证据等级
锁协议 DESIGN_PROVEN（逻辑）；运行时 NOT_TESTED。
