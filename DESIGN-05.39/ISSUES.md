# ISSUES.md — 未解决问题、阻断项与功能禁用清单（05.39）

证据等级：见各条目

---

## 1. 本轮已闭合的问题

| 编号 | 问题 | 结论 | 证据 |
|---|---|---|---|
| B39-01 | Token 与 COUNT 非原子 | 已修复（内嵌 128 位 CAS） | TOKEN-ATOMIC-TRANSACTION.md |
| B39-02 | flock/fcntl 误标 Fencing、两步 TOCTOU | 已修复（能力分级 + 单接口） | RESOURCE-FENCING-PROTOCOL.md |
| B39-03 | 版本静默回绕 | 已修复（VERSION_EXHAUSTED_DENY） | VERSION-EXHAUSTION-PROTOCOL.md |
| B39-04 | 185/188/218 矛盾 + HASH 错误 | 已修复（统一口径 + blob 哈希） | HISTORICAL-COUNTEREXAMPLE-AUDIT.md |

---

## 2. 继承的功能禁用清单（NOT_PROVEN / UNKNOWN）

| 功能 | 状态 | 禁用理由 | 重新启用条件 |
|---|---|---|---|
| 跨快照身份唯一性 | NOT_PROVEN | 无防回滚锚点 | 可信外部单调计数器/持久化锚点 |
| Receipt 事务恢复/幂等确认 | NOT_PROVEN（禁用） | 创建者认证未建立 | 可验证认证机制 |
| 未认证 Receipt | QUARANTINED | 认证未建立 | 同上 |
| Break-glass 紧急恢复 | NOT_PROVEN（禁用） | 审计信任根未证明 | 可信审计根 + 双人审批 |
| 跨进程共享计数器归零 | NOT_PROVEN（DENY） | 旧写入者排除不完整 | 完备枚举锁外写入者 |
| 锁外写入者 | UNKNOWN | 无法完备枚举 | 完整写入入口清单 |
| 跨进程 Token 功能 | NOT_PROVEN（DENY） | 128 位跨进程 CAS 未证明 | HW CAS + 共享内存证明 |
| 跨进程状态字 CAS | NOT_PROVEN（DENY） | HW CAS 未证明 | 同上 |
| 版本回绕 ABA / 跨重启 | NOT_PROVEN（DENY） | 无持久化 epoch | 持久化 epoch 锚点 |
| Fencing 不支持资源接管 | NOT_PROVEN（ISOLATED） | 无资源端 Token | 资源支持单调 Token |

---

## 3. 阻断项

| 阻断项 | 影响 | 状态 |
|---|---|---|
| 锁外写入者无法完备枚举 | Fork/跨进程归零 DENY | 未解决（继承） |
| 身份防回滚无可信锚点 | 跨快照唯一性无法宣称 | 未解决（继承） |
| Receipt 认证未建立 | 事务恢复禁用 | 未解决（继承） |
| Break-glass 审计根未证明 | 紧急恢复禁用 | 未解决（继承） |
| 跨进程 HW CAS 未证明 | 跨进程 Token/恢复 DENY | 未解决（本轮明确） |
| Fencing 不支持资源 | 跨进程接管释放禁用 | 未解决（本轮明确） |

---

## 4. 风险提示

- 本工单 READ-ONLY 设计修订，全部结论静态推演，未运行时验证。
- Token 内嵌方案依赖 128 位 CAS；若平台不支持 128 位，退化为方案 B（WAL），跨进程安全 NOT_PROVEN。
- 即使设计 PASS，不代表实现/运行时安全已验证。

---

## 5. 下一步

- IMPLEMENT-01：NOT_APPROVED
- A/B Merge：NOT_APPROVED
- Source Retire：DENY
- 等待 ChatGPT 独立复审。

---

## 6. 功能禁用清单（汇总）

| 禁用功能 | 动作 |
|---|---|
| 跨进程 Token | DENY |
| 跨进程恢复 | DENY |
| 版本回绕续期 | DENY |
| Fencing 不支持资源接管 | ISOLATED |
| 身份唯一性宣称 | 禁用 |
| Receipt 事务恢复 | 禁用 |
| Break-glass 紧急恢复 | 禁用 |
