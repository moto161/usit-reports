# ISSUES.md — 未解决问题（05.45）

工单：USIT-STAGE-B-DESIGN-05.45
证据等级：STATIC_ANALYSIS

---

## 1. 未证明能力（继续 DENY / ISOLATED）

以下能力在 05.45 仍无法静态证明，维持禁用：

| 编号 | 能力 | 状态 | 影响 | 禁用策略 | 重新启用条件 |
|---|---|---|---|---|---|
| U-01 | 跨进程 WAL 并发写 | NOT_PROVEN | 多进程共享 WAL 时恢复幂等无法保证 | DENY | 证明跨进程锁 + 记录原子追加 |
| U-02 | 跨重启恢复 | NOT_PROVEN | fsync 语义、半写页无法静态证明 | DENY | 运行时验证 fsync 原子性 |
| U-03 | 跨进程可信身份传递 | NOT_PROVEN | 跨主机 owner 授权无法防伪造 | DENY | 设计 mTLS/签名票据并证明不可伪造 |
| U-04 | operation 非幂等自动重放 | NOT_PROVEN | 业务副作用不可安全重放 | ISOLATED | 证明 operation 幂等，或引入预留+确认机制 |
| U-05 | 线程持锁崩溃后锁恢复 | NOT_PROVEN | 进程内锁可能遗留 | DENY | 引入可恢复锁机制并证明 |
| U-06 | 跨进程资源锁一致性 | NOT_PROVEN | 资源端释放跨进程串行化无法保证 | ISOLATED | 引入跨进程资源锁机制 |
| U-07 | 跨进程恢复屏障 | NOT_PROVEN | 跨进程恢复屏障一致性无法保证 | DENY | 证明跨进程恢复屏障一致性 |
| U-08 | 跨进程 Fencing | NOT_PROVEN | 资源端单调推进无法跨进程保证 | DENY | 引入跨进程单调计数或资源服务器 |
| U-09 | 跨进程预留/确认 | NOT_PROVEN | 跨进程预留/确认一致性无法保证 | ISOLATED | 引入跨进程预留/确认机制 |
| U-10 | 跨进程检查点一致性 | NOT_PROVEN | 跨进程检查点快照/索引一致性无法保证 | DENY | 证明跨进程检查点一致性 |

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
| B44-R03 恢复屏障死锁（Lock.wait） | 已修复 | RECOVERY-BARRIER-AND-ADMISSION.md |
| B44-R02 req_id 仅资源锁冲突 | 已修复 | REQUEST-ID-ATOMIC-IDEMPOTENCY.md |
| B44-R05 operation 预留/确认未接入真实释放 | 已修复 | RESOURCE-OPERATION-CRASH-RECOVERY.md |
| B44-R01 PREPARE 参与排序/commit_seq 耗尽/检查点无快照 | 已修复 | TXN-ORDER-AND-WAL-REPLAY.md |
| B44-R05 publish 非原子 | 已修复 | POST-COMMIT-FAILURE-CLOSURE.md |
| B44-R06 query_txn 未校验一致性 | 已修复 | POST-COMMIT-FAILURE-CLOSURE.md |
| B44-R08 WAL 截断未定义 | 已修复 | WAL-INTEGRITY-AND-CHECKPOINT.md（05.45 保留，检查点含快照+索引+摘要） |
| B44-R09 隔离记录持久化失败未定义 | 已修复 | POST-COMMIT-FAILURE-CLOSURE.md |

---

## 4. 结论

六项 B44 缺陷全部闭合；十项跨进程/跨重启能力继续 DENY/ISOLATED。核心安全不变式（恢复屏障、req_id 原子幂等、预留/确认协议、事务顺序、原子发布）在单进程内已闭合，跨进程/跨重启维持 NOT_PROVEN。
