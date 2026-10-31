# FORK-TRANSACTION-OWNERSHIP.md

## 目标
父子进程事务所有权，防止双重拥有。

## 1. 事务所有权模型

事务唯一身份 `txn_id`，所有权绑定 `(process_tree_id, branch_path)`。

- 父事务：`(tree_id, parent_branch_path)`。
- 子事务：`(tree_id, child_branch_path = parent_branch_path + [local_seq])`。

父子 branch_path 不同 → txn 空间不相交。

## 2. 所有权表

| 阶段 | 父 | 子 |
|---|---|---|
| Fork 前 | 父拥有全部进行中事务 | 无 |
| Fork 返回父 | 保留 | 无 |
| Fork 返回子 | 保留（父侧） | 丢弃继承，重建为空 |
| 子重建后 | 父事务 | 子独立事务空间 |
| 子再次 Fork | 父事务 | 子 + 孙事务 |
| 父退出 | 父事务失效 | 子独立继续 |

## 3. txn_id 生成

- 父：`(tree_id, parent_branch_path, parent_txn_seq)`。
- 子：`(tree_id, child_branch_path, child_txn_seq=0)`。

## 4. 不变量

- I-TO-1：child_branch_path != parent_branch_path。
- I-TO-2：txn_id 含 branch_path → 父子不相交。
- I-TO-3：子不继承父 txn_seq（重建为 0）。
- I-TO-4：子不恢复父未完成事务。

## 5. 证明

父子 txn_id 含不同 branch_path → 永不相交 → 不存在双重拥有。DESIGN_PROVEN（结构性，依赖 branch_path 唯一性；跨快照防回滚仍 NOT_PROVEN）。

## 6. 与崩溃恢复的关系

崩溃后，事务归属通过磁盘元数据中的 `(tree_id, branch_path, txn_id)` 识别。无法识别归属 → 隔离。

## 7. 证据等级
单次运行内 DESIGN_PROVEN；跨快照 NOT_PROVEN；运行时 NOT_TESTED。
