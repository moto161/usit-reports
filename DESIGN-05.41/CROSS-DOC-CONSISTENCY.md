# CROSS-DOC-CONSISTENCY.md — 跨文档一致性审查（05.41 权威）

证据等级：VERIFIED_STATIC
目标：逐项核对 05.41 全部协议文档、反例 JSON、统计之间的字段、状态、术语、数量一致性。

---

## 1. 状态模型一致性

| 检查项 | 结果 | 证据 |
|---|---|---|
| 状态字 128 位，TOKEN_GENERATION 从状态字删除 | PASS | UNIFIED-STATE-MODEL.md §2 |
| Token 记录表 TRT 每槽位独立 generation | PASS | UNIFIED-STATE-MODEL.md §3；TOKEN-IDENTITY-AND-PER-SLOT-GENERATION.md §3 |
| COUNT_MASK=0xFFFE / SHIFT=1 / MAX=32767 | PASS | UNIFIED-STATE-MODEL.md §2 |
| DENIED 单调置位不可清除 | PASS | UNIFIED-STATE-MODEL.md §2 |

---

## 2. Token 身份模型一致性

| 检查项 | 结果 | 证据 |
|---|---|---|
| Token = (slot, generation, owner, resource, state_version) | PASS | TOKEN-IDENTITY-AND-PER-SLOT-GENERATION.md §1 |
| 每槽位独立代际，他槽位不影响 | PASS | TOKEN-IDENTITY-AND-PER-SLOT-GENERATION.md §3 |
| 旧 Token 拒绝（gen 不匹配） | PASS | TOKEN-ALLOCATION-AND-REUSE.md §5 |
| 多 Token 并存互不失效 | PASS | TOKEN-ALLOCATION-AND-REUSE.md §4 |

---

## 3. 原子事务边界一致性

| 检查项 | 结果 | 证据 |
|---|---|---|
| COUNT + TRT 通过 txn_lock + CAS 提交点 | PASS | ATOMIC-TRANSACTION-BOUNDARY.md §2 |
| 状态字 CAS 是唯一提交点 | PASS | ATOMIC-TRANSACTION-BOUNDARY.md §4 |
| 崩溃窗口（TRT 领先）恢复回滚 | PASS | TOKEN-CRASH-RECOVERY.md §2 |
| 并发重复消费防护 | PASS | TOKEN-ATOMIC-TRANSACTION.md §4 |

---

## 4. 操作定义一致性（B41-02）

| 检查项 | 结果 | 证据 |
|---|---|---|
| release = token_consume（同一实现，减 COUNT） | PASS | TOKEN-OWNERSHIP-AND-LIFECYCLE.md §5/§6 |
| slot_alloc = token_create（同一实现） | PASS | TOKEN-OWNERSHIP-AND-LIFECYCLE.md §6 |
| token_revoke 不减 COUNT，走补偿 | PASS | STATE-INVARIANTS-AND-TRANSITIONS.md §5 |
| COUNT == popcount(TOKEN_ACTIVE) 不变式 | PASS | STATE-INVARIANTS-AND-TRANSITIONS.md §1 |

---

## 5. 版本单调性一致性（B41-03）

| 检查项 | 结果 | 证据 |
|---|---|---|
| field_cas 内执行版本递增检查 | PASS | VERSION-MONOTONICITY-AND-EXHAUSTION.md §2 |
| 回退/跳跃/静默不递增/回绕均拒绝 | PASS | VERSION-MONOTONICITY-AND-EXHAUSTION.md §2 |
| version 与 generation 独立 | PASS | VERSION-MONOTONICITY-AND-EXHAUSTION.md §4 |
| 白名单（只读/资源端/cleanup） | PASS | VERSION-MONOTONICITY-AND-EXHAUSTION.md §3 |

---

## 6. 资源端 Fencing 一致性（B41-04）

| 检查项 | 结果 | 证据 |
|---|---|---|
| 接管与释放共用 fence_lock | PASS | RESOURCE-FENCING-PROTOCOL.md §2 |
| flock/fcntl/fd 非 Fencing | PASS | RESOURCE-FENCING-PROTOCOL.md §7 |
| ALREADY_RELEASED 与 FENCED_OUT 区分 | PASS | RESOURCE-CRASH-AND-IDEMPOTENCY.md §1 |
| 非幂等崩溃 ISOLATED | PASS | RESOURCE-CRASH-AND-IDEMPOTENCY.md §3 |

---

## 7. 统计一致性

| 检查项 | 结果 | 证据 |
|---|---|---|
| RAW=410 / 唯一ID=410 / 唯一缺陷=408 | PASS | HISTORICAL-COUNTEREXAMPLE-AUDIT.md §3 |
| PASS=407 / FAIL=2 / NOT_PROVEN=1 | PASS | HISTORICAL-COUNTEREXAMPLE-AUDIT.md §3/§4 |
| NOT_TESTED_RUNTIME=410 | PASS | HISTORICAL-COUNTEREXAMPLE-AUDIT.md §3 |
| JSON 60 条 ID 唯一可解析 | PASS | COUNTEREXAMPLES.json（机器校验） |

---

## 8. NOT_PROVEN 约束与动作一致性

| NOT_PROVEN 项 | 动作 | 证据 |
|---|---|---|
| 跨进程 txn_lock / fence_lock | DENY | ATOMIC-TRANSACTION-BOUNDARY.md §6；RESOURCE-FENCING-PROTOCOL.md §8 |
| 跨重启 TRT/状态字持久化 | DENY | TOKEN-CRASH-RECOVERY.md §7 |
| 跨重启 generation 单调 | DENY | TOKEN-IDENTITY-AND-PER-SLOT-GENERATION.md §8 |
| 资源端非幂等崩溃 | ISOLATED | RESOURCE-CRASH-AND-IDEMPOTENCY.md §3 |

---

## 9. 遗留冲突

**本轮未发现跨文档关键冲突。** 若后续复审发现，记录于 ISSUES.md。

---

## 证据来源

- 全部协议文档：DESIGN-05.41/ 目录
- 反例 JSON：COUNTEREXAMPLES.json（60 条 ID 唯一）
- 证明等级：VERIFIED_STATIC
