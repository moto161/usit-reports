# FORK-LOCK-ORDER-UNIFIED.md

## 目标
统一锁顺序与生命周期，建立唯一权威定义，消除文档间冲突。

## 1. 权威锁顺序（唯一定义，其余文档引用此处）

```
1. 身份分配锁 (_identity_alloc_lock)
2. 准入锁 (_admit_lock)
3. 提交锁 (_commit_lock)
4. 业务资源锁 (_business_resource_lock)
5. Receipt 写入锁 (_receipt_write_lock)
```

**Fork 协调锁 `_fork_coord_lock`**：独立串行化 fork 准入，**不与上述业务锁嵌套**。其获取发生在 `_admit_lock` 释放之后、`_commit_lock` 短持有检查之前。

## 2. 锁获取/释放顺序（各执行路径）

| 路径 | 获取顺序 | 释放顺序（逆序） |
|---|---|---|
| 普通业务操作 | admit → 业务资源 | 业务资源 → admit |
| 事务提交 | commit → 业务资源 → receipt | receipt → 业务资源 → commit |
| Fork 协调 | coord（独立）→ 短取 admit（置 admit_closed）→ 短取 commit（检查） | commit → admit → coord |
| 身份分配 | identity_alloc（单锁） | identity_alloc |
| 回执写入 | commit（已持）→ receipt | receipt → commit |

## 3. Fork 发生时锁持有状态处理

| 锁 | Fork 时状态 | 父进程处理 | 子进程处理 |
|---|---|---|---|
| _fork_coord_lock | 协调线程持有 | 父释放自己的副本 | 子重建 |
| _admit_lock | admit_closed 置位后释放 | 恢复 admit_closed | 子重建 |
| _commit_lock | 短持有已释放 | 无 | 子重建 |
| _identity_alloc_lock | fork 前已释放 | 无 | 子重建 |
| _receipt_write_lock | 可能被事务线程持有 | 父释放自己 | 子重建 |

## 4. 父/子锁所有权

- 父进程：持有 fork 前已获取的锁；fork 后仅释放**自己**持有的锁。
- 子进程：不继承父持有的锁所有权；重建**全部**进程本地锁。
- OS 级 advisory lock：子进程不继承，需重取。

## 5. 异常退出/超时/恢复时的锁释放

| 情况 | 规则 |
|---|---|
| 正常退出 | finally 逆序释放 |
| 异常退出 | finally 保证释放，释放失败仅记录 |
| 超时 | 超时后 finally 释放，进入回滚或安全终态 |
| 恢复 | 恢复协调器释放协调锁，业务锁由持有线程释放 |
| 锁释放失败 | 记录附加异常，不改变已确定 Gate 终态 |

## 6. 死锁反例与证明（20 条，见 COUNTEREXAMPLES.json 死锁类）

核心消除规则：
1. `_fork_coord_lock` 不与任何业务锁嵌套（禁止 coord↔admit、coord↔commit 交叉）。
2. 全局锁顺序唯一（禁止逆序获取）。
3. 条件变量 wait 前释放关联锁。
4. 子进程重建全部锁，不继承父持锁状态。
5. 多锁获取失败回滚已获锁。

## 7. 权威性声明

本文件是锁顺序的**唯一权威定义**。其他文档（FORK-LOCK-LIFECYCLE、FORK-LOCK-OWNERSHIP、LOCK-ORDER-PROOF）引用本文件，不得重复产生冲突规则。若发现冲突，以本文件为准并记录到 CROSS-DOCUMENT-CONSISTENCY。

## 8. 证据等级
锁顺序协议 DESIGN_PROVEN（逻辑）；真实运行时死锁 NOT_TESTED。
