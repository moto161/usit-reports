# ISSUES.md — 未解决问题、阻断项与功能禁用清单（05.37）

证据等级：见各条目

---

## 1. 本轮已闭合的问题

| 编号 | 问题 | 结论 | 证据 |
|---|---|---|---|
| B37-01 | 三套冻结描述不一致 | 已统一 | FREEZE-STATE-MACHINE.md |
| B37-02 | 清理锁外释放无所有权、无限重试 | 已修复 | IDEMPOTENT-CLEANUP-PROTOCOL.md, CLEANUP-OWNERSHIP-MATRIX.md |
| B37-03 | 恢复 commit(state) 无条件覆盖、allowed 未定义 | 已修复 | RECOVERY-CAS-PROTOCOL.md, CRASH-RECOVERY-PROTOCOL.md |
| B37-04 | 准入整字算术 cur+(1<<1) | 已修复 | ADMISSION-FIELD-OPERATIONS.md |
| B37-05 | 反例累计数量计算错误 | 已核验 | HISTORICAL-COUNTEREXAMPLE-AUDIT.md |

---

## 2. 继承的功能禁用清单（NOT_PROVEN / UNKNOWN）

| 功能 | 状态 | 禁用理由 | 重新启用条件 |
|---|---|---|---|
| 跨快照身份唯一性宣称 | NOT_PROVEN | 无可信防回滚锚点 | 可信外部单调计数器/独立持久化锚点证明 |
| Receipt 事务恢复 + 幂等成功确认 | NOT_PROVEN（禁用） | 创建者认证未建立 | 可验证的创建者认证机制 |
| 未认证 Receipt | QUARANTINED | 认证未建立 | 同上 |
| Break-glass 紧急恢复 | NOT_PROVEN（禁用） | 审计信任根未证明 | 可信外部审计根 + 双人审批 + 防篡改证明 |
| 跨进程共享计数器归零 | NOT_PROVEN（DENY） | 旧写入者排除证明不完整 | 完备枚举并证明锁外写入者受控 |
| 锁外写入者 | UNKNOWN | 无法完备枚举 | 完整写入入口清单逐项证明 |

---

## 3. 阻断项

| 阻断项 | 影响 | 状态 |
|---|---|---|
| 锁外写入者无法完备枚举 | Fork 在共享介质场景保持 DENY | 未解决（继承） |
| 身份防回滚无可信锚点 | 跨快照唯一性无法宣称 | 未解决（继承） |
| Receipt 创建者认证未建立 | 事务恢复功能禁用 | 未解决（继承） |
| Break-glass 审计信任根未证明 | 紧急恢复禁用 | 未解决（继承） |

---

## 4. 风险提示

- 本工单 READ-ONLY 设计修订，全部结论为静态推演，未运行时验证（NOT_TESTED_RUNTIME）。
- 掩码修复（0xFFFE）为安全关键：实现时若沿用 05.35 错误掩码将导致 count≥16384 准入排空误判。
- 即使设计审查 PASS，也不代表实现或运行时安全已验证。

---

## 5. 下一步

- IMPLEMENT-01：NOT_APPROVED
- A/B Merge：NOT_APPROVED
- Source Retire：DENY
- 等待 ChatGPT 独立复审。
