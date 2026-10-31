# USIT-STAGE-B-DESIGN-05.29 交叉自审（CROSS-REVIEW.md）

> **声明：执行器自审，非独立审查。**

## 1. 状态名一致性

23 态状态机名称在 POSIX-COMMIT-STATE-MACHINE.md、POSIX-UNCERTAINTY.md、主文档 §3 中一致使用。Gate 七态一致。

## 2. 锁名一致性

`_admit_lock` / `_admit_cond` / `_fork_coord_lock` / `_commit_lock` / 业务资源锁 在 FORK-LOCK-OWNERSHIP.md、主文档 §1 中一致。

## 3. 身份字段一致性

process_instance_id / process_tree_id / process_generation / fork_epoch / fork_event_id / snapshot_id / parent_instance_id / parent_snapshot_id / fd_generation / txn_id / revision / pid 在 PROCESS-IDENTITY.md、主文档 §2 中一致。

## 4. 编号一致性

- Fork 异常时间线 FT1~FT30
- POSIX 崩溃 CR1~CR40
- rename 不确定 RU1~RU25
- 目录持久 DU1~DU25
- 外部写入者 EW1~EW20
- Fork×POSIX FX1~FX30
- 身份时间线 IT1~IT30
- 身份反例 CI1~CI30
- 回执冲突 CE-RC1~CE-RC20
- 幂等 CE-IDEM1~CE-IDEM20
- 身份×事务 CE-X1~CE-X30
- POSIX 反例 CE-P1~CE-P40
- 链路 L1~L30
- 不变量 I-01~I-80

## 5. 引用存在性

主文档引用的所有子文档、编号、状态名均存在。交叉引用无悬空。

## 6. 状态转移目标存在性

23 态状态机所有转移目标在表中定义，无引用不存在状态。

## 7. 自审结论

一致性检查通过。本自审不替代独立复审。
