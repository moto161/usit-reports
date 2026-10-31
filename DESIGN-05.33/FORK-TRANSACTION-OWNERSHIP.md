# FORK-TRANSACTION-OWNERSHIP.md

## 目标
父子进程事务所有权表，证明不存在父子双重拥有同一事务。

## 1. 事务所有权模型

事务唯一身份：`txn_id`。事务所有权绑定到 `(process_tree_id, branch_path)` 二元组。

- 父进程事务：归属 `(tree_id, parent_branch_path)`。
- 子进程事务：归属 `(tree_id, child_branch_path)`，其中 `child_branch_path = parent_branch_path + [local_seq]`。

由于 `child_branch_path != parent_branch_path`，父子事务所有权空间不相交。

## 2. 父子事务所有权表

| 阶段 | 父进程事务所有权 | 子进程事务所有权 |
|---|---|---|
| Fork 前 | 父拥有全部进行中事务 | 无（未存在） |
| Fork 返回父 | 保留父事务 | 无 |
| Fork 返回子 | 保留父事务（父侧） | 丢弃继承事务引用，重建为空 |
| 子重建后 | 父事务（父侧） | 子独立事务空间 |
| 子再次 Fork | 父事务 | 子事务 + 孙事务 |
| 父退出 | 父事务失效 | 子事务独立继续 |

## 3. txn_id 生成规则

- 父进程 txn_id：`(tree_id, parent_branch_path, parent_txn_seq)`。
- 子进程 txn_id：`(tree_id, child_branch_path, child_txn_seq)`，`child_txn_seq` 从 0 重新开始。
- 由于 branch_path 不同，即使 txn_seq 相同，完整 txn_id 也不同。

## 4. 防止双重拥有的不变量

- I-TO-1：`child_branch_path != parent_branch_path`（fork 必追加序号）。
- I-TO-2：txn_id 含 branch_path 分量 → 父子 txn_id 空间不相交。
- I-TO-3：子进程不继承父进程的 txn_seq 计数器（重建为 0）。
- I-TO-4：子进程不恢复父进程的未完成事务（不继承父事务引用）。

## 5. 证明

**定理**：不存在父子进程同时认为自己拥有同一事务。

**证明**：
1. 事务所有权由 txn_id 标识，txn_id 含 branch_path。
2. 父事务 txn_id 含 parent_branch_path，子事务 txn_id 含 child_branch_path。
3. fork 时 child_branch_path = parent_branch_path + [local_seq]，故两者不同。
4. 因此父子 txn_id 永不相交，不可能拥有同一事务。

结论：DESIGN_PROVEN（结构性，依赖 branch_path 唯一性；跨快照防回滚仍 NOT_PROVEN）。

## 6. 与 Receipt/快照的关系

- 子进程快照不含父进程未完成事务（I-TO-4）。
- 子进程不恢复父进程的 QUARANTINED/半成品 Receipt（隔离独立）。

## 7. 证据等级
事务所有权（单次运行内）DESIGN_PROVEN；跨快照依赖身份防回滚 NOT_PROVEN；运行时 NOT_TESTED。
