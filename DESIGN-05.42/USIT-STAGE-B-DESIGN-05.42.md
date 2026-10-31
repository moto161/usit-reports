# USIT-STAGE-B-DESIGN-05.42 — 主文档

工单编号：USIT-STAGE-B-DESIGN-05.42
前置版本：DESIGN-05.41（只读基线，commit `a2b737a`）
工作模式：READ-ONLY ANALYSIS + DESIGN REVISION
目标：修复 05.41 独立复审发现的 B41-R01～R07，重构事务提交与恢复、COUNT 不变式、锁生命周期、Token 身份校验与资源幂等协议。

---

## 1. 七项核心缺陷与修复结论

| 编号 | 缺陷 | 修复结论 | 证据文档 |
|---|---|---|---|
| B42-01 | 两独立存储 mutex+CAS 冒充原子事务，恢复误清零历史 generation | WAL 预写日志 + COMMIT fsync 唯一提交点，恢复重放 WAL | WAL-OR-ATOMIC-STORAGE-DESIGN.md |
| B42-02 | revoke 主动破坏 COUNT 不变式 | revoke 同事务减 COUNT，删除 revoke_pending | STATE-INVARIANTS-AND-TRANSITIONS.md |
| B42-03 | 锁提前返回不释放 | try/finally 覆盖所有出口 | LOCK-LIFECYCLE-AND-OWNERSHIP.md |
| B42-04 | 幂等释放误返回 FENCED_OUT | consumed_records 独立幂等记录，幂等检查先于 fencing 检查 | RESOURCE-FENCING-PROTOCOL.md |
| B42-05 | Token 身份字段未全校验 | owner/resource 真实校验，state_version 拆为审计 | TOKEN-IDENTITY-VALIDATION.md |
| B42-06 | 版本检查依赖 txn.changed | 提交入口自动检测字段变化 | VERSION-MONOTONICITY-AND-EXHAUSTION.md |
| B42-07 | 提交顺序矛盾、release/slot_free 代际不一致 | 唯一提交时序、唯一 release 实现 | TRANSACTION-COMMIT-PROTOCOL.md；TOKEN-ALLOCATION-RELEASE-REUSE.md |

---

## 2. 事务模型（B42-01 核心）

- **三层存储**：WAL（唯一权威）+ 状态字（缓存）+ TRT（缓存）。
- **提交点唯一**：COMMIT 记录 fsync 成功。
- **提交顺序唯一**：PREPARE → COMMIT(fsync) → APPLY。
- **恢复靠重放 WAL**，不猜状态字/TRT 谁领先；历史 generation 由 WAL 记录保留，永不清零。
- 跨进程/跨重启 WAL 原子性 NOT_PROVEN → DENY。

---

## 3. COUNT 不变式（B42-02 核心）

- I-1：`COUNT == popcount(TOKEN_ACTIVE)` 严格成立，revoke 也在同事务减 COUNT。
- revoke_pending 删除；cleanup 只负责资源端释放，不补偿 COUNT。
- 不变式在 APPLY 阶段自动验证，不依赖调用方手动维护。

---

## 4. 锁生命周期（B42-03 核心）

所有 txn_lock 获取/释放用 try/finally，覆盖全部返回与异常路径。重试上限 16，非重入锁。

---

## 5. Token 身份（B42-04/05 核心）

- Token 身份 = (slot, generation, owner_id, resource_id)，owner/resource 真实参与授权校验；state_version 仅审计。
- release = token_consume = slot_free 唯一实现，generation 递增统一。

---

## 6. 资源幂等（B42-05 核心）

- consumed_records 独立幂等记录，幂等检查先于 fencing 检查，区分 ALREADY_RELEASED 与 FENCED_OUT。
- 接管与释放共用 fence_lock。

---

## 7. 反例与统计

- 本轮新增反例 **50 条**（TRANSACTION_WAL=15 / COUNT_INVARIANT=10 / LOCK_LIFECYCLE=10 / TOKEN_IDENTITY=5 / RESOURCE_IDEMPOTENCY=10）。
- 累计：RAW=460 / 唯一ID=460 / 唯一缺陷=458 / PASS=457 / FAIL=2 / NOT_PROVEN=1 / NOT_TESTED_RUNTIME=460。
- 详见 HISTORICAL-COUNTEREXAMPLE-AUDIT.md 与 COUNTEREXAMPLE-RESULTS.md。

---

## 8. 历史安全约束（继承，未放宽）

- 跨快照身份唯一性 / Receipt 认证 / Break-glass：NOT_PROVEN。
- 锁外写入者完备枚举：UNKNOWN。
- Commit/Receipt 覆盖与事务终态证明不足：DENY。
- DENIED 不可清除；Source Retire DENY；IMPLEMENT-01 / A/B Merge NOT_APPROVED。

---

## 9. 最终结论

- B42-01：PASS（单进程 VERIFIED_STATIC；跨进程/跨重启 NOT_PROVEN → DENY）
- B42-02：PASS（单进程 VERIFIED_STATIC）
- B42-03：PASS（单进程 VERIFIED_STATIC）
- B42-04：PASS（有 fence_lock + 持久化 consumed_records 的资源 VERIFIED_STATIC；否则 NOT_PROVEN → ISOLATED）
- B42-05：PASS（反例 50 条，ID 唯一）
- B42-06：PASS（跨文档一致，统计可复算）

**整体结论：PASS-WITH-ISSUES**（跨进程/跨重启 WAL 原子性与持久化未证明，对应功能 DENY/ISOLATED）。

- IMPLEMENT-01：NOT_APPROVED
- A/B Merge：NOT_APPROVED
- Source Retire：DENY

**STOP，等待 ChatGPT 独立复审。**
