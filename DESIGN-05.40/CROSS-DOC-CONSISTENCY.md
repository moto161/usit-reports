# CROSS-DOC-CONSISTENCY.md — 跨文档一致性审查（05.40 权威）

证据等级：VERIFIED_STATIC
目标：逐项核对 05.40 全部协议文档、矩阵、反例 JSON 与统计之间的字段、状态、术语、数量一致性。

---

## 1. 状态字字段一致性

| 检查项 | 结果 | 证据 |
|---|---|---|
| 状态字宽度唯一（128 位） | PASS | UNIFIED-ATOMIC-STATE-WORD.md §2；ATOMIC-ACCESS-WIDTH-PROTOCOL.md §2 |
| COUNT_MASK=0xFFFE / SHIFT=1 / MAX=32767 | PASS | UNIFIED-ATOMIC-STATE-WORD.md §2/§3 |
| DENIED 单调置位不可清除 | PASS | UNIFIED-ATOMIC-STATE-WORD.md §3；STATE-TRANSITION-AND-CAS-MATRIX.md §1 |
| TOKEN_ACTIVE 16 位槽位 / TOKEN_GENERATION 16 位 | PASS | UNIFIED-ATOMIC-STATE-WORD.md §2；TOKEN-IDENTITY-AND-BITMAP-MODEL.md §2 |
| 保留位 bit20–31 / bit96–127 恒 0 | PASS | UNIFIED-ATOMIC-STATE-WORD.md §2/§5 |

---

## 2. 原子访问宽度一致性

| 检查项 | 结果 | 证据 |
|---|---|---|
| 唯一 field_cas（128 位），64 位 CAS 废除 | PASS | ATOMIC-ACCESS-WIDTH-PROTOCOL.md §2；UNIFIED-ATOMIC-STATE-WORD.md §4 |
| 无无条件 commit / 整字算术 | PASS | ATOMIC-ACCESS-WIDTH-PROTOCOL.md §7 |
| CAS 失败重读重算 | PASS | ATOMIC-ACCESS-WIDTH-PROTOCOL.md §3 |
| 重试上限 16 + 终止状态 | PASS | ATOMIC-ACCESS-WIDTH-PROTOCOL.md §4 |

---

## 3. Token 身份模型一致性

| 检查项 | 结果 | 证据 |
|---|---|---|
| Token = (slot, generation) | PASS | TOKEN-IDENTITY-AND-BITMAP-MODEL.md §2；TOKEN-SLOT-ALLOCATION-AND-REUSE.md §4 |
| generation 单调禁止回绕 | PASS | TOKEN-IDENTITY-AND-BITMAP-MODEL.md §4；VERSION-EXHAUSTION-ALL-ENTRYPOINTS.md §5 |
| 位图仅作活动索引，不作身份 | PASS | TOKEN-IDENTITY-AND-BITMAP-MODEL.md §1/§2 |
| 旧 Token 复活拒绝（gen 不匹配） | PASS | TOKEN-SLOT-ALLOCATION-AND-REUSE.md §4；TOKEN-ATOMIC-TRANSACTION.md §4 |

---

## 4. 版本耗尽一致性

| 检查项 | 结果 | 证据 |
|---|---|---|
| 耗尽检查在 field_cas 提交路径内 | PASS | VERSION-EXHAUSTION-ALL-ENTRYPOINTS.md §2 |
| 全入口覆盖（admit/release/slot_alloc/consume/revoke/freeze/unfreeze/DENIED/recovery） | PASS | VERSION-EXHAUSTION-ALL-ENTRYPOINTS.md §3 |
| 禁止回绕 / 快照回滚 / 复用 | PASS | VERSION-EXHAUSTION-ALL-ENTRYPOINTS.md §4 |
| version 与 generation 独立耗尽 | PASS | VERSION-EXHAUSTION-ALL-ENTRYPOINTS.md §5 |

---

## 5. 资源端 Fencing 一致性

| 检查项 | 结果 | 证据 |
|---|---|---|
| 唯一原子原语 resource_atomic_release | PASS | RESOURCE-FENCING-PROTOCOL.md §2；RESOURCE-FENCING-ATOMIC-RELEASE.md §1 |
| flock/fcntl/fd 不归入 Fencing | PASS | RESOURCE-FENCING-CAPABILITY-MATRIX.md §1/§2 |
| 不支持原子原语的资源 ISOLATED | PASS | RESOURCE-FENCING-PROTOCOL.md §6/§8 |
| 心跳超时仅触发调查 | PASS | RESOURCE-FENCING-PROTOCOL.md §7 |

---

## 6. 统计一致性

| 检查项 | 结果 | 证据 |
|---|---|---|
| RAW=350 / 唯一ID=350 / 唯一缺陷=348 | PASS | HISTORICAL-COUNTEREXAMPLE-AUDIT.md §3；COUNTEREXAMPLE-RESULTS.md §2 |
| PASS=347 / FAIL=2 / NOT_PROVEN=1 | PASS | HISTORICAL-COUNTEREXAMPLE-AUDIT.md §3/§4 |
| NOT_TESTED_RUNTIME=350 | PASS | HISTORICAL-COUNTEREXAMPLE-AUDIT.md §3 |
| JSON 75 条 ID 唯一可解析 | PASS | COUNTEREXAMPLES.json（机器校验） |

---

## 7. NOT_PROVEN 约束与 DENY/ISOLATED 动作一致性

| NOT_PROVEN 项 | 动作 | 证据 |
|---|---|---|
| 跨进程 128 位 HW CAS | DENY | ATOMIC-ACCESS-WIDTH-PROTOCOL.md §6 |
| 跨重启状态字持久化 | DENY | ATOMIC-ACCESS-WIDTH-PROTOCOL.md §6；TOKEN-CRASH-RECOVERY.md §5 |
| 跨重启 generation 持久化 | DENY | TOKEN-CRASH-RECOVERY.md §5 |
| 资源端无原子原语 | ISOLATED | RESOURCE-FENCING-PROTOCOL.md §8 |
| 资源端 token 持久化失败 | DENY | RESOURCE-FENCING-PROTOCOL.md §8 |

---

## 8. 遗留冲突（若发现）

**本轮未发现跨文档关键冲突。** 所有字段、接口、统计、NOT_PROVEN 动作在各文档间一致。若后续复审发现冲突，记录于 ISSUES.md。

---

## 证据来源

- 全部协议文档：DESIGN-05.40/ 目录
- 反例 JSON：COUNTEREXAMPLES.json（机器核验 75 条 ID 唯一）
- 证明等级：VERIFIED_STATIC
