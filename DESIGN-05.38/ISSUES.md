# ISSUES.md — 未解决问题、阻断项与功能禁用清单（05.38）

证据等级：见各条目

---

## 1. 本轮已闭合的问题

| 编号 | 问题 | 结论 | 证据 |
|---|---|---|---|
| B38-01 | version 递增不能阻止在途释放 | 已修复（资源端 Fencing） | CLEANUP-FENCING-PROTOCOL.md |
| B38-02 | 安全条件循环外检查 | 已修复（谓词纳入 CAS） | ATOMIC-CONDITION-CAS-PROTOCOL.md |
| B38-03 | 版本模型未定 + ABA + 递归 recover | 已修复（64 位合并原子字 + while 上限） | RECOVERY-VERSION-ATOMICITY.md |
| B38-04 | PASS=187/185 双口径 + 缺陷去重 | 已修复（统一口径 + UNIQUE_DEFECT） | HISTORICAL-COUNTEREXAMPLE-AUDIT.md |

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
| Fencing 不支持资源接管释放 | NOT_PROVEN（ISOLATED） | 无资源端 Token | 资源支持 Fencing Token |
| 跨进程状态字 CAS 原子性 | NOT_PROVEN（DENY） | HW CAS 未证明 | 平台 HW CAS + 共享内存证明 |
| 版本回绕 ABA 防护 | NOT_PROVEN（DENY） | 无持久化 epoch | 持久化 epoch 锚点 |

---

## 3. 阻断项

| 阻断项 | 影响 | 状态 |
|---|---|---|
| 锁外写入者无法完备枚举 | Fork/跨进程归零 DENY | 未解决（继承） |
| 身份防回滚无可信锚点 | 跨快照唯一性无法宣称 | 未解决（继承） |
| Receipt 认证未建立 | 事务恢复禁用 | 未解决（继承） |
| Break-glass 审计根未证明 | 紧急恢复禁用 | 未解决（继承） |
| Fencing 不支持资源 | 跨进程接管释放禁用 | 未解决（本轮新增明确） |
| 跨进程 HW CAS 未证明 | 跨进程恢复 DENY | 未解决（本轮新增明确） |

---

## 4. 风险提示

- 本工单 READ-ONLY 设计修订，全部结论静态推演，未运行时验证。
- 即使设计 PASS，不代表实现/运行时安全已验证。
- Fencing 与跨进程 CAS 原子性是本轮新增的诚实 NOT_PROVEN 边界，实现时必须先证明平台能力，否则不得放行。

---

## 5. 下一步

- IMPLEMENT-01：NOT_APPROVED
- A/B Merge：NOT_APPROVED
- Source Retire：DENY
- 等待 ChatGPT 独立复审。
