# CROSS-DOC-CONSISTENCY.md — 跨文档一致性审查（05.42 权威）

证据等级：VERIFIED_STATIC
目标：逐项核对 05.42 全部协议文档、反例 JSON、统计之间的字段、状态、术语、数量一致性。

---

## 1. 事务协议一致性（B42-01）

| 检查项 | 结果 | 证据 |
|---|---|---|
| 提交点唯一（COMMIT fsync） | PASS | TRANSACTION-COMMIT-PROTOCOL.md §1 |
| 提交顺序唯一（PREPARE→COMMIT→APPLY） | PASS | TRANSACTION-COMMIT-PROTOCOL.md §1 |
| 恢复靠重放 WAL，不猜领先 | PASS | TRANSACTION-CRASH-RECOVERY.md §2 |
| 历史 generation 永不清零 | PASS | TRANSACTION-CRASH-RECOVERY.md §3 |
| WAL 是唯一权威，状态字/TRT 是缓存 | PASS | TRANSACTION-DATA-MODEL.md §1 |

---

## 2. 状态不变式一致性（B42-02）

| 检查项 | 结果 | 证据 |
|---|---|---|
| I-1 严格成立（revoke 也减 COUNT） | PASS | STATE-INVARIANTS-AND-TRANSITIONS.md §1/§2 |
| revoke_pending 删除 | PASS | REVOKE-AND-COMPENSATION-PROTOCOL.md §3 |
| release=consume=slot_free 唯一实现 | PASS | TOKEN-ALLOCATION-RELEASE-REUSE.md §2 |
| generation 递增统一 | PASS | TOKEN-ALLOCATION-RELEASE-REUSE.md §3 |

---

## 3. 锁生命周期一致性（B42-03）

| 检查项 | 结果 | 证据 |
|---|---|---|
| 所有出口 try/finally 释放 | PASS | LOCK-LIFECYCLE-AND-OWNERSHIP.md §2/§4 |
| 重试上限 16 | PASS | LOCK-LIFECYCLE-AND-OWNERSHIP.md §5 |
| 非重入锁 | PASS | LOCK-LIFECYCLE-AND-OWNERSHIP.md §6 |

---

## 4. Token 身份一致性（B42-04）

| 检查项 | 结果 | 证据 |
|---|---|---|
| owner/resource 参与授权校验 | PASS | TOKEN-IDENTITY-VALIDATION.md §2 |
| state_version 仅审计 | PASS | TOKEN-IDENTITY-VALIDATION.md §1 |
| 身份字段来源可信（WAL 重建） | PASS | TOKEN-IDENTITY-VALIDATION.md §4 |

---

## 5. 资源端 Fencing 一致性（B42-05）

| 检查项 | 结果 | 证据 |
|---|---|---|
| 幂等检查先于 fencing 检查 | PASS | RESOURCE-FENCING-PROTOCOL.md §2 |
| consumed_records 独立记录 | PASS | RESOURCE-FENCING-PROTOCOL.md §2 |
| ALREADY_RELEASED 与 FENCED_OUT 区分 | PASS | RESOURCE-IDEMPOTENCY-AND-RECOVERY.md §1 |
| 接管释放共用 fence_lock | PASS | RESOURCE-TAKEOVER-RELEASE-SERIALIZATION.md §1 |

---

## 6. 统计一致性

| 检查项 | 结果 | 证据 |
|---|---|---|
| RAW=460 / 唯一ID=460 / 唯一缺陷=458 | PASS | HISTORICAL-COUNTEREXAMPLE-AUDIT.md §3 |
| PASS=457 / FAIL=2 / NOT_PROVEN=1 | PASS | HISTORICAL-COUNTEREXAMPLE-AUDIT.md §3/§4 |
| NOT_TESTED_RUNTIME=460 | PASS | HISTORICAL-COUNTEREXAMPLE-AUDIT.md §3 |
| JSON 50 条 ID 唯一可解析 | PASS | COUNTEREXAMPLES.json（机器校验） |

---

## 7. NOT_PROVEN 约束与动作一致性

| NOT_PROVEN 项 | 动作 | 证据 |
|---|---|---|
| 跨进程 WAL/fence_lock/txn_lock | DENY | WAL-OR-ATOMIC-STORAGE-DESIGN.md §7；RESOURCE-FENCING-PROTOCOL.md §8 |
| 跨重启 WAL/TRT/consumed_records 持久化 | DENY | TRANSACTION-CRASH-RECOVERY.md §7；RESOURCE-IDEMPOTENCY-AND-RECOVERY.md §7 |
| operation 非幂等且无法原子 | ISOLATED | RESOURCE-IDEMPOTENCY-AND-RECOVERY.md §3 |

---

## 8. 遗留冲突

**本轮未发现跨文档关键冲突。** 05.41 的「提交顺序矛盾」「slot_free 与 release 代际不一致」已在本轮消除（提交顺序唯一、release 唯一实现）。

---

## 证据来源

- 全部协议文档：DESIGN-05.42/ 目录
- 反例 JSON：COUNTEREXAMPLES.json（50 条 ID 唯一）
- 证明等级：VERIFIED_STATIC
