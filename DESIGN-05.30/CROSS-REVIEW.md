# CROSS-REVIEW.md

## 目标
独立自我复核（非独立第三方复审，仅执行器交叉复核）。

## 1. 四组核心阻断复核

| # | 阻断 | 复核结论 | 证据文件 |
|---|---|---|---|
| B30-01 | DENIED 不可逆 | 已闭合：DENIED 不可逆，人工授权独立协议 | DENIED-STATE-MACHINE.md, PARENT-RESTORE-PROTOCOL.md |
| B30-02 | 身份结构唯一 | 已闭合：树内 branch_path 结构唯一，树间如实标概率 | PROCESS-TREE-IDENTITY.md, IDENTITY-UNIQUENESS-PROOF.md |
| B30-03 | Receipt 真实性 | 部分闭合：对象匹配与认证分离，认证层 UNKNOWN | RECEIPT-AUTHENTICITY.md, RECEIPT-IDEMPOTENCY.md |
| B30-04 | Fork 锁生命周期 | 已闭合：获取/所有权/释放/重建完整 | FORK-LOCK-LIFECYCLE.md, FORK-LOCK-OWNERSHIP.md |

## 2. 一致性复核

- 编号：不变量 I-001~I-080 连续，无重复/缺失（见机器统计）。
- 锁名称统一：_admit_lock / _fork_coord_lock / _commit_lock / 条件变量锁 / 业务资源锁 / 身份分配锁 / Receipt 写入锁。
- 身份字段统一：process_tree_id / branch_path / local_fork_sequence / process_instance_id / fork_event_id。
- Receipt 字段统一：13 字段。
- 状态名称统一：OPEN/FREEZING/FROZEN/RESTORING/RESTORE_FAILED/TERMINATING/DENIED。

## 3. 保留/修改/撤销

- 保留：05.29 的 Fork 业务计数分离、POSIX 双维度不确定态、23 态状态机骨架。
- 修改：DENIED 状态机、身份结构唯一性、Receipt 认证分层、Fork 锁生命周期。
- 撤销：05.29 的「DENIED→restore→OPEN」路径、「四元组证明创建者真实性」结论。
- 新增：branch_path 结构字段、六层 Receipt 证据、身份分配锁、Receipt 写入锁。

## 4. 遗留问题（如实）

- Receipt 创建者认证（D 层）依赖可信密钥基础设施，本只读阶段无法闭合 → UNKNOWN。
- 树间身份唯一性为概率唯一，全局严格唯一需可信注册表 → UNKNOWN。
- 所有运行时行为（fork/POSIX）NOT_TESTED。

## 5. 整体结论
PASS-WITH-ISSUES：四组核心协议闭合，认证边界 UNKNOWN 为隔离处理的非核心遗留。
