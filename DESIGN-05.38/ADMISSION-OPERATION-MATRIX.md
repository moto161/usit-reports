# ADMISSION-OPERATION-MATRIX.md — 准入操作矩阵（05.38）

证据等级：VERIFIED_STATIC
本文件为四个准入相关操作的谓词、变换、线性化点、失败模式的权威矩阵。

---

## 1. 操作矩阵

| 操作 | 谓词 predicate(s) | 变换 transform(s) | 线性化点 | 成功条件 |
|---|---|---|---|---|
| admit(token) | ¬denied ∧ gate ∧ count<MAX | count+1 | cas(cur→new) 成功 | 全部谓词真 |
| release(token) | token_valid ∧ count>0 | count−1 | cas(cur→new) 成功 | 全部谓词真 |
| cas_frozen | ¬denied ∧ ¬gate ∧ count==0 | frozen=1 | cas(cur→new) 成功 | 全部谓词真 |
| cas_unfreeze | ¬denied ∧ frozen | frozen=0 | cas(cur→new) 成功 | 全部谓词真 |

---

## 2. 失败模式

| 操作 | 失败返回 | 触发条件 | 是否修改状态 |
|---|---|---|---|
| admit | DENIED | denied=1 | 否 |
| admit | GATE_CLOSED | gate=0 | 否 |
| admit | OVERFLOW | count>=MAX | 否 |
| admit | LIVELOCK | CAS 连续失败 8 次 | 否 |
| release | INVALID_TOKEN | token 无效/已消耗 | 否 |
| release | UNDERFLOW | count==0 | 否 |
| release | LIVELOCK | CAS 失败 8 次 | 否 |
| cas_frozen | DENIED | denied=1 | 否 |
| cas_frozen | GATE_OPEN | gate=1 | 否 |
| cas_frozen | COUNT_NOT_ZERO | count>0 | 否 |
| cas_frozen | LIVELOCK | CAS 失败 8 次 | 否 |
| cas_unfreeze | DENIED | denied=1 | 否 |
| cas_unfreeze | NOT_FROZEN | frozen=0 | 否 |
| cas_unfreeze | LIVELOCK | CAS 失败 8 次 | 否 |

**所有失败模式均不修改状态字**，且 DENIED 置位时所有操作的 predicate 恒假，操作一律拒绝。

---

## 3. DENIED 并发不可绕过性

**定理**：任何 CAS 重试都不能绕过 DENIED 并发置位。

证明：每个操作的 `predicate(s)` 都包含 `¬get_denied(s)`，且 `predicate` 在**每一轮 CAS 循环**都基于最新 `load()` 重新评估。若 DENIED 在任一轮被置位：
- 该轮的 `predicate` 返回 False → 返回 DENIED/PREDICATE_FALSE。
- 即使 DENIED 在「读 cur」和「cas(cur,new)」之间被并发置位，cas 会因 `cur` 被改变而失败，下一轮重新 `load()` 读到 DENIED=1，predicate 恒假。

因此不存在「DENIED 置位后被 CAS 绕过」的路径。∎

---

## 4. 重试终止

所有操作的重试由 `conditional_cas` 内部统一控制：`MAX_CAS_RETRY=8` 次后返回 LIVELOCK，操作终止并降级。禁止外层 `while True` 无限重试。

---

## 5. 与各文档的关系

- 谓词/变换定义：ATOMIC-CONDITION-CAS-PROTOCOL.md §3。
- 字段原语：ADMISSION-FIELD-OPERATIONS.md §2。
- 本矩阵是操作层面的汇总，不重复定义，引用上述权威文档。

---

## 证据来源

- 原子条件 CAS：ATOMIC-CONDITION-CAS-PROTOCOL.md
- 字段原语：ADMISSION-FIELD-OPERATIONS.md
- 证明等级：VERIFIED_STATIC
