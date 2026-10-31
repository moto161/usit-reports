# CROSS-DOC-CONSISTENCY.md — 跨文档一致性报告（05.37）

证据等级：VERIFIED_STATIC

---

## 1. 权威定义统一核对

| 协议项 | 权威文档 | 引用文档 | 一致性 |
|---|---|---|---|
| 冻结三阶段 | FREEZE-STATE-MACHINE.md §3 | DENIED-ATOMIC-PROTECTION.md, 主文档 | ✅ 一致 |
| cas_frozen 检查 gate/count/DENIED | FREEZE-STATE-MACHINE.md §4 | DENIED-ATOMIC-PROTECTION.md | ✅ 一致 |
| DENIED 优先级 DENIED>FROZEN>FREEZING>OPEN | FREEZE-STATE-MACHINE.md §5/§6 | DENIED-ATOMIC-PROTECTION.md §6 | ✅ 一致 |
| 清理 7 态 + 所有权 | IDEMPOTENT-CLEANUP-PROTOCOL.md §2/§3 | CLEANUP-OWNERSHIP-MATRIX.md | ✅ 一致 |
| 恢复版本校验 CAS | RECOVERY-CAS-PROTOCOL.md §3 | CRASH-RECOVERY-PROTOCOL.md | ✅ 一致 |
| 字段操作唯一接口 field_cas | ADMISSION-FIELD-OPERATIONS.md §3 | FREEZE/DENIED/CLEANUP | ✅ 一致 |
| COUNT_MASK=0xFFFE | ADMISSION-FIELD-OPERATIONS.md §2 | 全部（继承 05.36） | ✅ 一致 |

---

## 2. 05.36 冲突清单（本轮消除）

| 编号 | 05.36 冲突 | 05.37 修正 |
|---|---|---|
| 1 | 三套冻结描述不一致 | 统一为 FREEZE-STATE-MACHINE.md |
| 2 | 矩阵 T4 DENIED 通配 `*` vs 伪代码 DENIED=0 | 权威：DENIED 必须 =0 |
| 3 | 清理锁外释放无所有权证明 | 增加执行所有权 + 版本 |
| 4 | 永久失败资源无限重试 | 增加 retry 上限 + 退避 |
| 5 | 恢复 `commit(state)` 无条件覆盖 | 改为 cas_with_version |
| 6 | `allowed`/`denied_evidence` 未定义 | 明确证据清单 + compute_allowed |
| 7 | 准入 `cur + (1<<COUNT_SHIFT)` 整字算术 | 统一 field_cas |

---

## 3. 关键不变量（跨文档一致）

1. 非零准入计数不可能提取为 0（COUNT_MASK=0xFFFE）。
2. DENIED 一旦置位不可清除（无写回 0 路径）。
3. 冻结三阶段：关 Gate → 等归零 → 设 FROZEN，cas_frozen 原子检查三项。
4. 清理执行所有权唯一，失败不无限重试。
5. 恢复用版本校验 CAS，不覆盖并发 DENIED。
6. 字段操作统一 field_cas，无整字算术。

---

## 4. 未解决的 NOT_PROVEN（如实保留）

- 跨快照身份唯一性：NOT_PROVEN。
- Receipt 创建者认证：NOT_PROVEN。
- Break-glass 审计信任根：NOT_PROVEN。
- 锁外写入者完备枚举：UNKNOWN。

在各文档中一致标注，无「一文档 PASS、另一文档 NOT_PROVEN」矛盾。

---

## 5. 结论

05.37 全部交付文档关键定义一致，05.36 的 7 处冲突/缺陷全部消除，历史 NOT_PROVEN 约束完整继承。

---

## 证据来源

- 各交付文档（本目录）
- 05.36 文档（DESIGN-05.36/）
- 证明等级：VERIFIED_STATIC
