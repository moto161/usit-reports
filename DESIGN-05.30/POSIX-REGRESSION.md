# POSIX-REGRESSION.md

## 目标
回归复核 05.29 POSIX 协议，确认不确定态与并发边界未退化。

## 1. 回归矩阵（10 项）

| # | 检查项 | 05.29 状态 | 05.30 回归结论 |
|---|---|---|---|
| 1 | rename 操作结果与持久性分离 | 已分离 | 未退化，维持双维度 |
| 2 | OUTCOME_UNKNOWN 盲目重试 | 禁止 | 未退化，禁止 |
| 3 | DURABILITY_UNKNOWN → DURABLE | 禁止 | 未退化，禁止 |
| 4 | advisory lock 误认强制锁 | 已纠正 | 未退化，明确非强制 |
| 5 | 外部写入者边界 | 已明确 UNKNOWN | 未退化 |
| 6 | current 外部替换 | 已建模 | 未退化 |
| 7 | inode/dev+哈希仅作对象匹配 | 已限定 | 未退化 |
| 8 | Receipt 承担提交真实性 | 已移除 | 已修复（认证分离） |
| 9 | 恢复仅依赖磁盘证据 | 已要求 | 未退化 |
| 10 | 不确定态统一隔离 | 已要求 | 未退化 |

## 2. 外部写入者边界（如实）

- advisory lock（flock/fcntl）是**进程级建议锁**，仅约束遵守协议的写入者。
- 不能约束：不遵守协议的进程、管理员直接修改、文件系统异常。
- 两次读取一致 / inode 一致 / 哈希一致均**不能证明无锁外写入者**。
- 本设计无法建立排他写入权威 → 无锁外写入者前提标 **UNKNOWN**。

## 3. 双维度不确定态（维持）

维度 A 操作结果：NOT_STARTED / IN_PROGRESS / CONFIRMED_APPLIED / CONFIRMED_NOT_APPLIED / OUTCOME_UNKNOWN。
维度 B 持久性：NOT_DURABLE / DURABILITY_PENDING / DURABLE_CONFIRMED / DURABILITY_UNKNOWN。

组合态合法恢复：
- (CONFIRMED_APPLIED, DURABLE_CONFIRMED) → 提交成功。
- (OUTCOME_UNKNOWN, *) → 隔离，禁止重试。
- (*, DURABILITY_UNKNOWN) → 目录 fsync 重试一次，仍未知则隔离。
- (CONFIRMED_NOT_APPLIED, *) → 可重试提交。

## 4. 禁止项（维持）

- 操作是否发生不明却标记提交成功。
- 操作已发生却因 fsync 失败认定未发生。
- 目录 fsync 失败后盲目重复 rename。
- 不确定态直接跳 DURABLE。
- 仅凭进程内变量恢复磁盘事务。
- 仅凭文件名判断对象身份。

## 5. 新发现问题
无新回归问题；B30-03 的 Receipt 认证问题已在本轮单独修复。

## 6. 证据等级
POSIX 协议：DESIGN_PROVEN（逻辑，继承 05.29 复核）；运行时：NOT_TESTED。
