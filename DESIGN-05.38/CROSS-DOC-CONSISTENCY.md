# CROSS-DOC-CONSISTENCY.md — 跨文档一致性报告（05.38）

证据等级：VERIFIED_STATIC

---

## 1. 权威定义统一核对

| 协议项 | 权威文档 | 引用文档 | 一致性 |
|---|---|---|---|
| Fencing 三概念区分（软件校验/资源端Fencing/幂等） | CLEANUP-FENCING-PROTOCOL.md §2 | OWNERSHIP-MATRIX, IDEMPOTENT-CLEANUP | ✅ 一致 |
| Fencing 能力分级 | CLEANUP-OWNERSHIP-MATRIX.md §2 | FENCING-PROTOCOL, IDEMPOTENT-CLEANUP | ✅ 一致 |
| 谓词纳入 CAS（conditional_cas） | ATOMIC-CONDITION-CAS-PROTOCOL.md §2 | FIELD-OPS, OPERATION-MATRIX | ✅ 一致 |
| 准入令牌（release 防下溢） | ADMISSION-FIELD-OPERATIONS.md §5 | ATOMIC-CONDITION-CAS | ✅ 一致 |
| 状态+版本合并 64 位原子字 | RECOVERY-VERSION-ATOMICITY.md §1 | RECOVERY-CAS, CRASH-RECOVERY | ✅ 一致 |
| DENIED 单调语义（取并集） | RECOVERY-CAS-PROTOCOL.md §4 | DENIED 相关 | ✅ 一致 |
| COUNT_MASK=0xFFFE | ADMISSION-FIELD-OPERATIONS.md §1 | 全部 | ✅ 一致 |
| 重试上限 MAX_CAS_RETRY=8 | ATOMIC-CONDITION-CAS-PROTOCOL.md §5 | OPERATION-MATRIX | ✅ 一致 |

---

## 2. 05.37 冲突清单（本轮消除）

| 编号 | 05.37 冲突 | 05.38 修正 |
|---|---|---|
| 1 | version 递增不能阻止在途释放 | 引入资源端 Fencing |
| 2 | 安全条件循环外检查一次 | 谓词纳入 conditional_cas |
| 3 | release 无令牌防下溢 | 准入令牌 |
| 4 | 版本模型多选项未定 | 唯一 64 位合并原子字 |
| 5 | 陈旧 old_state / 递归 recover | 每轮重读 + while 上限 |
| 6 | PASS=187 vs 185 双口径 | 统一原始标注口径 |

---

## 3. 伪代码函数/状态/字段定义核对

所有伪代码引用的函数均有定义：`conditional_cas`、`field_set_count/gate/frozen/denied`、`cas_state_version`、`load_state_version`、`compute_allowed`、`build_new_state`、`claim_ownership`、`release_with_token`。无未定义符号。

---

## 4. 未解决的 NOT_PROVEN（如实保留，各文档一致）

- 跨快照身份唯一性：NOT_PROVEN。
- Receipt 创建者认证：NOT_PROVEN。
- Break-glass 审计信任根：NOT_PROVEN。
- 锁外写入者完备枚举：UNKNOWN。
- Fencing 不支持资源的跨进程接管释放：NOT_PROVEN。
- 跨进程共享状态字 64 位 CAS 原子性：NOT_PROVEN。
- 版本回绕 ABA 防护（无持久化 epoch）：NOT_PROVEN。

**关键检查**：每个 NOT_PROVEN 项在相关功能中都有明确 DENY/ISOLATED 动作（如 Fencing 不支持→ISOLATED、跨进程恢复→DENY），无「标 NOT_PROVEN 却继续执行」的遗漏。

---

## 5. 统计一致性

历史统计、主文档、执行报告、COUNTEREXAMPLE-RESULTS.md、HISTORICAL-COUNTEREXAMPLE-AUDIT.md 全部使用统一口径（见审计文档 §6），无数字冲突。

---

## 6. 结论

05.38 全部交付文档关键定义一致，05.37 的 6 处冲突全部消除，历史 NOT_PROVEN 约束完整继承且均有对应 DENY/ISOLATED 动作。

---

## 证据来源

- 各交付文档（本目录）
- 05.37 文档（DESIGN-05.37/）
- 证明等级：VERIFIED_STATIC
