# CROSS-DOC-CONSISTENCY.md — 跨文档一致性报告（05.36）

证据等级：VERIFIED_STATIC
目标：检查 05.36 全部交付文档是否存在关键冲突，并复核与 05.35 的差异。

---

## 1. 关键定义统一核对

| 协议项 | 权威定义文档 | 引用文档 | 一致性 |
|---|---|---|---|
| COUNT_MASK=0xFFFE | ATOMIC-STATE-WORD.md §2 | STATE-TRANSITION-MATRIX.md, DENIED-ATOMIC-PROTECTION.md, 主文档 | ✅ 一致 |
| COUNT_SHIFT=1, COUNT_MAX=32767 | ATOMIC-STATE-WORD.md §2 | 同上 | ✅ 一致 |
| DENIED 位=bit17=0x20000 | ATOMIC-STATE-WORD.md §2 | STATE-TRANSITION-MATRIX.md, DENIED-ATOMIC-PROTECTION.md | ✅ 一致 |
| frozen=bit16=0x10000 | ATOMIC-STATE-WORD.md §2 | STATE-TRANSITION-MATRIX.md | ✅ 一致 |
| FREEZING 移除（用 CLOSED+frozen=0 表达） | STATE-TRANSITION-MATRIX.md §1 | DENIED-ATOMIC-PROTECTION.md | ✅ 一致 |
| DENIED 不可逆 | STATE-TRANSITION-MATRIX.md §3 | DENIED-ATOMIC-PROTECTION.md, CRASH-RECOVERY-PROTOCOL.md | ✅ 一致 |
| 清理 6 态状态机 | IDEMPOTENT-CLEANUP-PROTOCOL.md §2 | 主文档 | ✅ 一致 |
| 共享计数器归零 DENY | CRASH-RECOVERY-PROTOCOL.md §4/§5 | ATOMIC-STATE-WORD.md §8 | ✅ 一致 |
| 单一准入状态源 | ATOMIC-STATE-WORD.md（继承 B35-01） | 全部 | ✅ 一致（无第二套计数器） |

---

## 2. 与 05.35 的冲突清单（本轮消除）

| 编号 | 05.35 内容 | 05.36 修正 | 状态 |
|---|---|---|---|
| 1 | COUNT_MASK=0x7FFE（遗漏 bit15） | COUNT_MASK=0xFFFE | 已消除 |
| 2 | cas_frozen 未完整检查 DENIED | cas_frozen 同 CAS 检查 DENIED/gate/count | 已消除 |
| 3 | _cleanup_done 提前置 True | 仅在 CLEANUP_COMPLETED 置 True | 已消除 |
| 4 | FREEZING 与 FROZEN 冗余并存 | 移除 FREEZING，统一为 frozen 位 | 已消除 |
| 5 | B35-MASK-001 错误 PASS | 修正为 FAIL（历史缺陷）+ 修复后 PASS | 已消除 |

---

## 3. 关键不变量（跨文档一致）

1. 非零准入计数不可能被提取为 0（COUNT_MASK=0xFFFE 保证）。
2. DENIED 一旦置位，任何普通路径不可清除（无写回 0 的操作）。
3. 准入与冻结使用单一状态字，无双源。
4. 清理 `_cleanup_done` 仅随 CLEANUP_COMPLETED 置位。
5. 共享介质计数器归零必须有旧写入者排除证明，否则 DENY。

---

## 4. 未解决的 NOT_PROVEN（如实保留，不升级）

- 跨快照身份唯一性：NOT_PROVEN。
- Receipt 创建者认证：NOT_PROVEN。
- Break-glass 审计信任根：NOT_PROVEN。
- 锁外写入者完备枚举：UNKNOWN。

这些不构成文档间冲突，而是**明确的功能禁用边界**，在各文档中一致标注，未出现「一文档宣称 PASS、另一文档标注 NOT_PROVEN」的矛盾。

---

## 5. 结论

05.36 全部交付文档关键定义一致，无未解释的协议冲突。05.35 的掩码、DENIED 检查、清理幂等三处缺陷已在 05.36 全部消除，且历史 NOT_PROVEN 约束完整继承。

---

## 证据来源

- 各交付文档（本目录）
- 05.35 文档（`usit_p0_05.35_reports/`）
- 证明等级：VERIFIED_STATIC
