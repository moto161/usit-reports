# ISSUES.md — 未解决问题、阻断项与功能禁用清单（05.36）

证据等级：见各条目

---

## 1. 本轮已闭合的问题

| 编号 | 问题 | 结论 | 证据 |
|---|---|---|---|
| B36-01 | 计数器掩码 0x7FFE 遗漏 bit15 | 已修复（0xFFFE） | ATOMIC-STATE-WORD.md §1/§2 |
| B36-02 | cas_frozen 未完整检查 DENIED | 已修复（同 CAS 检查） | DENIED-ATOMIC-PROTECTION.md §2 |
| B36-03 | _cleanup_done 提前置位导致泄漏 | 已修复（仅 COMPLETED 置位） | IDEMPOTENT-CLEANUP-PROTOCOL.md §4 |
| B36-04 | 崩溃恢复计数器重置未经证明 | 已闭合（作用域矩阵 + 旧写入者排除 + DENY） | CRASH-RECOVERY-PROTOCOL.md |

---

## 2. 继承的功能禁用清单（NOT_PROVEN / UNKNOWN，不得升级为 PASS）

| 功能 | 状态 | 禁用理由 | 重新启用条件 |
|---|---|---|---|
| 跨快照身份唯一性宣称 | NOT_PROVEN | 无可信防回滚锚点 | 提供可信外部单调计数器/独立持久化锚点证明 |
| Receipt 事务恢复 + 幂等成功确认 | NOT_PROVEN（禁用） | 创建者认证未建立 | 建立可验证的创建者认证机制 |
| 未认证 Receipt | QUARANTINED | 认证未建立 | 同上 |
| Break-glass 紧急恢复 | NOT_PROVEN（禁用） | 审计信任根未证明 | 提供可信外部审计根 + 双人审批 + 防篡改证明 |
| 跨进程共享计数器归零 | NOT_PROVEN（DENY） | 旧写入者排除证明不完整（锁外写入者 UNKNOWN） | 完备枚举并证明所有锁外写入者受控 |
| 锁外写入者 | UNKNOWN | 无法完备枚举 | 建立完整写入入口清单并逐项证明受冻结约束 |

---

## 3. 阻断项

| 阻断项 | 影响 | 状态 |
|---|---|---|
| 锁外写入者无法完备枚举 | Fork 在共享介质场景保持 DENY | 未解决（继承 05.33） |
| 身份防回滚无可信锚点 | 跨快照唯一性无法宣称 | 未解决（继承 05.33） |
| Receipt 创建者认证未建立 | 事务恢复功能禁用 | 未解决（继承 05.33） |
| Break-glass 审计信任根未证明 | 紧急恢复禁用 | 未解决（继承 05.31） |

---

## 4. 风险提示

- 本工单为 READ-ONLY 设计修订，所有结论均为静态推演，**未经过运行时验证**（NOT_TESTED_RUNTIME）。
- 即使设计审查 PASS，也不代表实现或运行时安全已验证（继承历轮验收原则）。
- 掩码修复（0x7FFE→0xFFFE）是**安全关键修正**：若实现时沿用 05.35 的错误掩码，将导致 count≥16384 时准入排空误判，Fork 在业务未停止时进行。

---

## 5. 下一步

- IMPLEMENT-01：NOT_APPROVED。
- A/B Merge：NOT_APPROVED。
- Source Retire：DENY。
- 等待 ChatGPT 独立复审。
