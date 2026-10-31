# USIT-STAGE-B-DESIGN-05.41 — 主文档

工单编号：USIT-STAGE-B-DESIGN-05.41
前置版本：DESIGN-05.40（只读基线，commit `d26626b`）
工作模式：READ-ONLY ANALYSIS + DESIGN REVISION
目标：修复 05.40 独立复审发现的 B40-R01～R04，重新闭合底层数据模型、原子事务边界、状态转换和资源端并发协议。

---

## 1. 四项核心缺陷与修复结论

| 编号 | 缺陷 | 修复结论 | 证据文档 |
|---|---|---|---|
| B41-01 | 全局 TOKEN_GENERATION 使其他活动 Token 失效 | 每槽位独立 generation + 独立 TRT 记录表 | TOKEN-IDENTITY-AND-PER-SLOT-GENERATION.md |
| B41-02 | revoke/consume/release COUNT 定义冲突 | 统一操作定义 + COUNT==popcount 不变式 | STATE-INVARIANTS-AND-TRANSITIONS.md |
| B41-03 | field_cas 版本递增靠注释未执行 | commit_txn 内强制单调性检查 | VERSION-MONOTONICITY-AND-EXHAUSTION.md |
| B41-04 | 接管（cas_token）与释放（internal_lock）无共同串行化 | 统一 fence_lock 串行化 | RESOURCE-FENCING-PROTOCOL.md |

---

## 2. 统一状态模型（B41-01 核心）

- **状态字（128 位）**：COUNT / GATE / DENIED / FROZEN / FREEZING / VERSION / TOKEN_ACTIVE（16 槽位位图）。`TOKEN_GENERATION` 从状态字**删除**。
- **Token 记录表 TRT（独立存储）**：每槽位 `{generation(16位), owner_id, resource_id}`。
- **每槽位独立代际**：`TRT[slot].generation` 仅该槽位复用时 +1，他槽位不受影响。
- **COUNT 与 TRT 原子一致性**：通过 `txn_lock` + 状态字 CAS 提交点保证（见 ATOMIC-TRANSACTION-BOUNDARY.md）。

**Token 完整身份** = `(slot, generation, owner_id, resource_id, state_version)`。

---

## 3. COUNT 与 TOKEN_ACTIVE 不变式（B41-02 核心）

- 不变式 I-1：`COUNT == popcount(TOKEN_ACTIVE)`。
- 统一操作定义：`release = token_consume`（减 COUNT）；`slot_alloc = token_create`（同一实现）；`token_revoke` 独立（清位不减 COUNT，走 revoke_pending 补偿）。
- 每次转换后重验 I-1；违反则拒绝。

---

## 4. 版本单调性（B41-03 核心）

`commit_txn` 内部强制：版本 +1（禁止回退/跳跃/静默不递增/回绕），generation 与 version 独立单调。全入口经同一提交路径。

---

## 5. 资源端 Fencing（B41-04 核心）

接管与释放共用 `fence_lock`（单一串行化边界），消除 05.40 的「锁 + 独立 CAS」分离。幂等释放区分 ALREADY_RELEASED 与 FENCED_OUT。

---

## 6. 反例与统计

- 本轮新增反例 **60 条**（TOKEN_MULTI_ACTIVE=15 / TOKEN_SLOT_GENERATION=15 / COUNT_STATE_INVARIANT=10 / VERSION_MONOTONICITY=10 / RESOURCE_FENCING=10）。
- 累计：RAW=410 / 唯一ID=410 / 唯一缺陷=408 / PASS=407 / FAIL=2 / NOT_PROVEN=1 / NOT_TESTED_RUNTIME=410。
- 详见 HISTORICAL-COUNTEREXAMPLE-AUDIT.md 与 COUNTEREXAMPLE-RESULTS.md。

---

## 7. 历史安全约束（继承，未放宽）

- 跨快照身份唯一性：NOT_PROVEN。
- Receipt Creator Auth：NOT_PROVEN。
- Break-glass 审计信任根：NOT_PROVEN。
- 锁外写入者完备枚举：UNKNOWN。
- Commit/Receipt 覆盖与事务终态证明不足：DENY。
- DENIED 不可清除；Source Retire DENY；IMPLEMENT-01 / A/B Merge NOT_APPROVED。

---

## 8. 最终结论

- B41-01：PASS（单进程 VERIFIED_STATIC；跨进程/跨重启 NOT_PROVEN → DENY）
- B41-02：PASS（单进程 VERIFIED_STATIC）
- B41-03：PASS（单进程 VERIFIED_STATIC；跨重启 NOT_PROVEN → DENY）
- B41-04：PASS（有 fence_lock 的资源 VERIFIED_STATIC；无原子原语 NOT_PROVEN → ISOLATED）
- B41-05：PASS（反例 60 条，ID 唯一）
- B41-06：PASS（跨文档一致，统计可复算，HASH 核验）

**整体结论：PASS-WITH-ISSUES**（跨进程/跨重启原子性与持久化未证明，对应功能 DENY/ISOLATED）。

- IMPLEMENT-01：NOT_APPROVED
- A/B Merge：NOT_APPROVED
- Source Retire：DENY

**STOP，等待 ChatGPT 独立复审。**
