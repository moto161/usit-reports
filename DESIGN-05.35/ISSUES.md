# ISSUES.md

## 目标
未解决问题与功能禁用清单。

## 1. 未解决问题（阻断，如实）

| # | 问题 | 等级 | 功能状态 |
|---|---|---|---|
| 1 | 跨快照身份唯一性 | NOT_PROVEN | 跨快照身份功能禁用 |
| 2 | Receipt 创建者认证 | NOT_PROVEN | 事务恢复/幂等确认禁用 |
| 3 | Break-glass 审计信任根 | NOT_PROVEN | 紧急恢复禁用 |
| 4 | 锁外写入者 | UNKNOWN/NOT_PROVEN | Fork 在无法排除锁外写入者时 DENY |
| 5 | Commit/Receipt 写入入口未全覆盖 | NOT_PROVEN | Fork DENY |
| 6 | 所有运行时行为 | NOT_TESTED | — |

## 2. 功能禁用清单

| 功能 | 禁用原因 | 重新启用条件 |
|---|---|---|
| 跨快照防回滚身份唯一性 | NOT_PROVEN | 可信外部单调计数器 |
| Receipt 幂等成功确认 | 认证未建立 | 认证设施 + 验签 |
| 事务提交恢复（凭回执判成功） | 认证未建立 | 同上 |
| QUARANTINED 自动恢复 | 认证未建立 | 认证 + 人工审批 |
| Break-glass 紧急恢复 | 审计信任根 NOT_PROVEN | 可信外部审计服务 |
| Fork（存在锁外写入者/未覆盖入口） | NOT_PROVEN | 封闭部署或证明无锁外写入者 |

## 3. 本轮协议级闭合项

- 准入状态源统一：DESIGN_PROVEN（单一原子字）。
- 状态字掩码/DENIED 保护：DESIGN_PROVEN（field-aware CAS + DENIED 位不可逆）。
- 异常/崩溃恢复：DESIGN_PROVEN（状态机 + 磁盘证据协议）。

## 4. 阻断项

- IMPLEMENT-01：NOT_APPROVED。
- A/B Merge：NOT_APPROVED。
- Source Retire：DENY。

## 5. 下一轮建议
引入可信单调计数器 / 密钥存储 / 外部审计服务闭合三项 NOT_PROVEN；锁外写入者与写入入口需部署级隔离或完整证明。
