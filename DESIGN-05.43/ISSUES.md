# ISSUES.md — 未解决问题（05.43）

工单：USIT-STAGE-B-DESIGN-05.43
证据等级：STATIC_ANALYSIS

---

## 1. 未证明能力（继续 DENY / ISOLATED）

以下能力在 05.43 仍无法静态证明，维持禁用，不因本轮修复而放开：

| 编号 | 能力 | 状态 | 影响 | 禁用策略 | 重新启用条件 |
|---|---|---|---|---|---|
| U-01 | 跨进程 WAL 并发写 | NOT_PROVEN | 多进程共享 WAL 时恢复幂等无法保证 | DENY | 证明跨进程锁 + 记录原子追加 |
| U-02 | 跨重启恢复 | NOT_PROVEN | fsync 语义、半写页无法静态证明 | DENY | 运行时验证 fsync 原子性 |
| U-03 | 跨进程可信身份传递 | NOT_PROVEN | 跨主机 owner 授权无法防伪造 | DENY | 设计 mTLS/签名票据并证明不可伪造 |
| U-04 | operation 非幂等自动重放 | NOT_PROVEN | 业务副作用不可安全重放 | ISOLATED | 证明 operation 幂等，或引入补偿 |
| U-05 | 线程持锁崩溃后锁恢复 | NOT_PROVEN | 进程内锁可能遗留 | DENY | 引入可恢复锁机制并证明 |
| U-06 | 跨进程/跨重启 Fencing | NOT_PROVEN | 资源端单调推进无法跨进程保证 | DENY | 引入跨进程单调计数或资源服务器 |

---

## 2. 历史禁用功能（继续禁用，不受本轮影响）

| 功能 | 状态 | 说明 |
|---|---|---|
| Receipt 认证 | 禁用 | 历史 NOT_PROVEN，本轮未涉及 |
| Break-glass | 禁用 | 历史 NOT_PROVEN，本轮未涉及 |
| 跨快照身份 | 禁用 | 历史 NOT_PROVEN |
| 锁外写入者 | 禁用 | 历史 NOT_PROVEN |

---

## 3. 本轮新增但已闭合的问题

| 问题 | 状态 | 修复文档 |
|---|---|---|
| B42-R01 WAL 恢复重复执行 | 已修复 | WAL-RECOVERY-IDEMPOTENCY.md |
| B42-R02 COUNT 不变式校验时序矛盾 | 已修复 | TRANSACTION-PRECOMMIT-VALIDATION.md |
| B42-R03 COMMIT 后异常返回 ERROR | 已修复 | POST-COMMIT-ERROR-AND-RETRY.md |
| B42-R04 txn_lock 嵌套自锁 | 已修复 | LOCK-OWNERSHIP-AND-CALL-GRAPH.md |
| B42-R05 资源释放异常未隔离 | 已修复 | RESOURCE-FAILURE-AND-RECOVERY.md |
| B42-R06 资源端 Owner 校验缺失 | 已修复 | RESOURCE-OWNER-AUTHORIZATION.md |
| B42-R07 Token 身份字段矛盾 | 已修复 | TOKEN-IDENTITY-CANONICAL-MODEL.md |
| B42-R08 反例未覆盖实际漏洞 | 已修复 | COUNTEREXAMPLES.json（33 条含 FAIL） |
| B42-R09 编号映射不一致 | 已修复 | CROSS-DOC-CONSISTENCY.md |

---

## 4. 结论

九项 B42 缺陷全部闭合；六项跨进程/跨重启能力继续 DENY/ISOLATED。核心安全不变式（WAL 恢复幂等、提交前验证、锁所有权、Token 身份、Owner 授权、Fencing 串行化）在单进程内已闭合，跨进程/跨重启维持 NOT_PROVEN。
