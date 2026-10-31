# ISSUES.md

## 目标
未解决问题与功能禁用清单。

## 1. 未解决问题（阻断，如实）

| # | 问题 | 等级 | 功能状态 |
|---|---|---|---|
| 1 | 身份防回滚唯一性 | NOT_PROVEN | 跨快照身份功能禁用 |
| 2 | Receipt 创建者认证 | NOT_PROVEN | 事务恢复/幂等确认禁用 |
| 3 | Break-glass 审计信任根 | NOT_PROVEN | 紧急恢复禁用 |
| 4 | 锁外写入者存在性 | UNKNOWN | 边界标注 |
| 5 | 所有运行时行为 | NOT_TESTED | — |

## 2. 功能禁用清单

| 功能 | 禁用原因 | 重新启用条件 |
|---|---|---|
| 跨快照防回滚身份唯一性 | NOT_PROVEN | 可信外部单调计数器 |
| Receipt 幂等成功确认 | 认证未建立 | 认证设施 + 验签 |
| 事务提交恢复（凭回执判成功） | 认证未建立 | 同上 |
| QUARANTINED 自动恢复 | 认证未建立 | 认证 + 人工审批 |
| 回执跨进程信任 / 自动清理 | 认证未建立 | 认证建立 |
| Break-glass 紧急恢复 | 审计信任根 NOT_PROVEN | 可信外部审计服务 |

## 3. 本轮新增的协议级结论（已闭合）

- 准入原子性：DESIGN_PROVEN（合并原子字 + CAS）。
- Fork 冻结协议：DESIGN_PROVEN（coord 与业务锁解耦）。
- 快照一致性：DESIGN_PROVEN（前置条件归零）。
- 父子事务所有权：DESIGN_PROVEN（branch_path 空间不相交）。

## 4. 阻断项

- IMPLEMENT-01：NOT_APPROVED（维持）。
- 身份防回滚、Receipt 认证、Break-glass 审计信任根三项未闭合。

## 5. 下一轮建议

- 引入可信单调计数器 / 密钥存储 / 外部审计服务闭合三项 NOT_PROVEN。
