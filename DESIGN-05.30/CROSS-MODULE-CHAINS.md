# CROSS-MODULE-CHAINS.md

## 目标
25 条跨模块调用链，覆盖 DENIED×Receipt、Fork×事务、父子事务共享等。

## 调用链（25 条）

| # | 调用链 | 关键检查 |
|---|---|---|
| 1 | Fork请求 → 准入锁 → Gate检查 → FREEZING → 归零 → fork | Gate 与 in_flight 原子性 |
| 2 | 提交 → commit_lock → 写current.tmp → rename → 回执 | 提交与回执关联 |
| 3 | 身份分配 → identity_alloc_lock → next_seq → branch_path | 序号原子性 |
| 4 | Fork → 子重建身份 → branch_path 生成 | 子身份重建 |
| 5 | DENIED → 拒绝新事务 → 拒绝新Fork | DENIED 全阻断 |
| 6 | 事务 → txn_id 生成 → 回执 txn_id 绑定 | 事务身份 |
| 7 | 回执写入 → Receipt锁 → 排他创建 O_EXCL | 回执唯一性 |
| 8 | 恢复 → 读回执 → 认证 → 幂等判定 | 认证依赖 |
| 9 | Fork前 → 等待提交完成 → coord锁 | Fork×提交并发 |
| 10 | 父恢复 → close_fork_parent → 释放锁 → Gate恢复 | 父路径 |
| 11 | 子清理 → 重建锁 → 重建身份 → 恢复准入 | 子路径 |
| 12 | 崩溃恢复 → 磁盘证据 → rename结果确认 → 持久性确认 | 双维度 |
| 13 | EEXIST → 归属判定 → 隔离/幂等 | 回执归属 |
| 14 | 身份碰撞 → 拒绝 → 重生成 | 唯一性 |
| 15 | 外部替换 current → 哈希不匹配 → 隔离 | 外部写入者 |
| 16 | PID重用 → 不用PID判身份 → instance区分 | PID边界 |
| 17 | 快照恢复 → monotonic校验 → 拒绝重放 | 重放防护 |
| 18 | 多进程恢复同事务 → 排他锁 → 单恢复者 | 恢复竞争 |
| 19 | fork_event_id → branch_path → 树内唯一 | 结构唯一 |
| 20 | Receipt认证 → 密钥来源 → 轮换 | 认证边界 |
| 21 | DENIED → manual_reenable → 审计 | 人工解除 |
| 22 | 事务回执 → process_instance_id 绑定 | 实例绑定 |
| 23 | 子进程事务 → 独立txn_id → 不复用父txn | 事务隔离 |
| 24 | Fork时OUTCOME_UNKNOWN事务 → 隔离 → 不fork | 不确定态×fork |
| 25 | 身份×Receipt → fork_event_id 校验 → 归属 | 交叉绑定 |

## 2. 关键交叉结论

- Fork 前若存在 OUTCOME_UNKNOWN 事务 → 拒绝 fork（或先隔离该事务）。
- 子进程不复用父 txn_id，独立生成。
- 多进程恢复同一事务由排他锁串行化。
- DENIED 阻断事务提交与 fork 准入。

## 3. 证据等级
DESIGN_PROVEN（逻辑）；运行时：NOT_TESTED。
