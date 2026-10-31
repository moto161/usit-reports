# CROSS-DOC-CONSISTENCY.md — 跨文档一致性报告（05.39）

证据等级：VERIFIED_STATIC

---

## 1. 权威定义统一核对

| 协议项 | 权威文档 | 引用文档 | 一致性 |
|---|---|---|---|
| Token 内嵌状态字（128 位 CAS） | TOKEN-ATOMIC-TRANSACTION.md §3 | OWNERSHIP, CRASH-RECOVERY, CONCURRENCY-MATRIX | ✅ |
| 方案 B（WAL 事务）仅退化用 | TOKEN-ATOMIC-TRANSACTION.md §6 | CRASH-RECOVERY | ✅ |
| 资源端 Fencing 唯一接口 release_with_fencing_token | RESOURCE-FENCING-PROTOCOL.md §3 | RELEASE-ATOMICITY, CAPABILITY-MATRIX | ✅ |
| flock/fcntl 降级 OS_LOCK_ONLY | RESOURCE-FENCING-CAPABILITY-MATRIX.md §2 | FENCING-PROTOCOL | ✅ |
| 版本耗尽 DENY 禁止回绕 | VERSION-EXHAUSTION-PROTOCOL.md §2 | RECOVERY-VERSION-ATOMICITY | ✅ |
| 移除 epoch（或进 CAS） | VERSION-EXHAUSTION-PROTOCOL.md §5 | RECOVERY-VERSION-ATOMICITY | ✅ |
| 历史统计统一口径 | HISTORICAL-COUNTEREXAMPLE-AUDIT.md §6 | RESULTS, 主文档, 执行报告 | ✅ |

---

## 2. 05.38 冲突清单（本轮消除）

| 编号 | 05.38 冲突 | 05.39 修正 |
|---|---|---|
| 1 | Token 独立账本非原子 | Token 内嵌状态字 |
| 2 | flock/fcntl 误标 FENCING_SUPPORTED | 降级 OS_LOCK_ONLY |
| 3 | 两步 accept+release TOCTOU | 统一 release_with_fencing_token |
| 4 | 版本静默回绕 vs 文字 DENY | 前置 VERSION_EXHAUSTED_DENY |
| 5 | epoch 三重判定未进 CAS | 移除 epoch（或进 CAS） |
| 6 | 185/188/218 矛盾 | 统一 220/218/273 |

---

## 3. 禁止的跨文档矛盾（逐项确认）

- [x] 无「一文档要求原子提交、另一文档分步提交」。
- [x] 无「一文档声明支持接管、另一文档要求 ISOLATED」（统一按能力分级）。
- [x] 无「文字禁止回绕、伪代码允许回绕」（已前置检查）。
- [x] 无反例标 PASS 但伪代码存在安全漏洞。
- [x] NOT_PROVEN 功能在各文档均默认禁用。

---

## 4. NOT_PROVEN 项的 DENY/ISOLATED 动作核对

| NOT_PROVEN 项 | DENY/ISOLATED 动作 | 位置 |
|---|---|---|
| 跨进程 Token | DENY | TOKEN-OWNERSHIP-AND-LIFECYCLE.md §6 |
| 跨进程状态字 CAS | DENY | RECOVERY-VERSION-ATOMICITY.md §5 |
| 版本回绕 ABA | DENY（禁止回绕） | VERSION-EXHAUSTION-PROTOCOL.md §2 |
| Fencing 不支持资源 | ISOLATED | RESOURCE-FENCING-PROTOCOL.md §8 |
| 身份防回滚/Receipt/Break-glass/锁外写入者 | 禁用 | ISSUES.md |

---

## 5. 结论

05.39 全部交付文档关键定义一致，05.38 的 6 处冲突全部消除，NOT_PROVEN 约束均有对应 DENY/ISOLATED 动作。

---

## 证据来源

- 各交付文档（本目录）
- 05.38 文档（DESIGN-05.38/）
- 证明等级：VERIFIED_STATIC
