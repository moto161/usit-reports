# USIT-STAGE-B-DESIGN-05.29 — Fork 锁所有权与锁顺序证明（FORK-LOCK-OWNERSHIP.md）

> B29-01 支撑文档。重建锁所有权模型（F03）与锁依赖证明。

## 1. 锁清单与所有权

| 锁 | 获取者 | 所有者 | 允许获取阶段 | 释放者 | 释放时机 | 获取失败 | 释放失败 | fork 后继承/重建 |
|----|--------|--------|-------------|--------|---------|---------|---------|----------------|
| `_admit_lock` | 业务入口 / freeze | 当前线程 | OPEN（业务）/ FREEZING（freeze）| 当前线程 | `with` 退出 | 抛异常，操作拒绝 | 进入异常态 | 子进程**重建** |
| `_admit_cond` | freeze（wait）| freeze 线程 | DRAINING | 自动（wait 释放/重获）| wait 返回 | 超时回滚 | — | 子进程重建 |
| `_fork_coord_lock` | fork 发起者 | fork 线程 | REQUESTED→FINALIZED | fork 线程 | 最外层 finally | fork 拒绝 | 终止 | 子进程重建 |
| `_commit_lock` | 事务提交者 | 提交线程 | 提交全程 | 提交线程 | S16 | 提交拒绝 | 终止/记录 | 子进程**不继承** |
| 业务资源锁 | 业务操作 | 业务线程 | 业务全程 | 业务线程 | 操作结束 | 操作失败 | 记录 | 子进程重建 |

**禁止**：未持锁释放、跨线程释放、重复释放。

## 2. 锁依赖有向图

```
业务路径：  _admit_lock ──> 业务资源锁 ──> _commit_lock
Fork 路径： _admit_lock ──> _fork_coord_lock
```

全局锁获取顺序（固定）：

```
1. _admit_lock（最外层准入）
2. 业务资源锁（业务操作内部）
3. _commit_lock（事务提交）
或
2'. _fork_coord_lock（fork 协调）
```

## 3. 锁获取/释放失败矩阵

| 场景 | 结果 | 终态 |
|------|------|------|
| `_admit_lock` 获取失败（业务）| 操作拒绝，异常传播 | 业务失败，Gate 不变 |
| `_admit_lock` 获取失败（freeze）| fork 拒绝 | OPEN |
| `_admit_cond.wait` 超时 | FreezeTimeout，回滚 | OPEN |
| `_fork_coord_lock` 获取失败 | fork 拒绝 | OPEN |
| `_fork_coord_lock` 释放失败 | 终止 | TERMINATING |
| `_commit_lock` 获取失败 | 提交拒绝 | PREPARED_VERIFIED |
| `_commit_lock` 释放失败 | 记录 + 终止判定 | TERMINATING/记录 |
| 业务资源锁获取失败 | 操作失败 | 业务异常 |

## 4. 死锁证明

### 4.1 环路检查

所有锁获取都遵循"先准入（_admit_lock）后具体锁（_commit_lock/_fork_coord_lock）"的顺序，**不存在反向路径**（没有 `_commit_lock → _admit_lock` 或 `_fork_coord_lock → _admit_lock`），故无环。

### 4.2 条件变量等待的死锁排除

`freeze()` 在 `_admit_lock` 内调用 `_admit_cond.wait()`，wait **会释放** `_admit_lock` 并阻塞，被 notify 后**重新获取**。业务 `__exit__` 递减计数并 `notify_all`。因此 freeze 等待期间不持有 `_admit_lock`，不形成循环等待。

### 4.3 Fork 与提交并发

Fork（`_fork_coord_lock`）与提交（`_commit_lock`）是两个独立锁，无交叉依赖；freeze 等待业务归零时，正在提交的事务属于业务（计入 `_biz_in_flight`），提交完成递减后 freeze 才能 FROZEN。无死锁。

### 4.4 结论

- 锁依赖有向图无环。
- 条件变量等待释放锁，无循环等待。
- 锁顺序固定，无死锁（静态推演）。
- **运行时并发行为 NOT_TESTED**（本机无 fork）。

## 5. fork 后锁的继承与重建

- `os.fork()` 后，子进程拥有父进程内存的**副本**，包括所有锁对象的**副本**。
- 锁的**持有状态**（谁持有、递归深度）在子进程中是不确定的（fork 只复制当前线程，其他线程的锁状态不复制）。
- 因此子进程**必须重建**所有锁：`_admit_lock`、`_admit_cond`、`_fork_coord_lock`、业务资源锁——赋值为**新对象**。
- `_commit_lock`：子进程**不继承**父进程的提交锁所有权，若子进程需要提交，使用自己重建的 `_commit_lock`。
- 子进程**不得** acquire/release 继承来的锁对象。

## 6. 结论

锁所有权、获取/释放责任、锁顺序、死锁排除、fork 后重建规则均已明确。协议层 DESIGN_PROVEN，运行时 NOT_TESTED。
