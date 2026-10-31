# USIT-STAGE-B-DESIGN-05.40 — 主文档

工单编号：USIT-STAGE-B-DESIGN-05.40
前置版本：DESIGN-05.39（只读基线，commit `f25be69`）
工作模式：READ-ONLY ANALYSIS + DESIGN REVISION
目标：修复 05.39 独立复审发现的 B39-R01～R04，闭合四条安全链（Token 身份、原子提交、版本单调性、资源端 Fencing）。

---

## 1. 四项核心缺陷与修复结论

| 编号 | 缺陷 | 修复结论 | 证据文档 |
|---|---|---|---|
| B40-01 | 32 位 token_seq + 32 位 active_mask 无法证明 Token 身份唯一 | Token 身份改为 `(slot, generation)`，generation 单调内嵌状态字，槽位复用带代际校验 | TOKEN-IDENTITY-AND-BITMAP-MODEL.md |
| B40-02 | Token 协议 128 位 CAS 与版本协议 64 位 CAS 共享同一状态字 | 统一唯一 128 位状态字，唯一 field_cas 接口，64 位 CAS 废除 | UNIFIED-ATOMIC-STATE-WORD.md |
| B40-03 | 版本耗尽检查仅在 cas_state_version，admit/release 等入口绕过 | 版本耗尽检查下沉到 field_cas 提交路径内，全入口覆盖 | VERSION-EXHAUSTION-ALL-ENTRYPOINTS.md |
| B40-04 | release_with_fencing_token 仍是「比较+释放」两步 TOCTOU | 资源端唯一原子原语 resource_atomic_release，校验+释放同临界区 | RESOURCE-FENCING-PROTOCOL.md |

---

## 2. 唯一权威状态字布局（128 位）

```
bit0–15    COUNT（COUNT_MASK=0xFFFE, SHIFT=1, MAX=32767）
bit16      GATE（1=OPEN, 0=CLOSED）
bit17      DENIED（单调置位，不可清除）
bit18      FROZEN
bit19      FREEZING
bit20–31   保留位（恒 0）
bit32–63   VERSION（单调递增，禁止回绕）
bit64–79   TOKEN_ACTIVE（16 槽位位图）
bit80–95   TOKEN_GENERATION（单调递增，槽位复用代际）
bit96–127  保留位（恒 0）
```

所有状态修改仅通过唯一 `field_cas`（128 位）接口，禁止 64 位 CAS、无条件 commit、整字算术。

---

## 3. Token 身份模型（B40-01 核心）

- Token 完整标识 = `(slot, generation, owner_id, resource_id, state_version)`。
- `slot`：16 槽位位图 bit 索引（0–15）。
- `generation`：全局 16 位单调递增，禁止回绕，每次槽位分配（含复用）+1。
- **旧 Token 复活拒绝**：槽位复用后 generation 递增，旧 Token 的 `gen` 严格小于当前 `gen`，`gen != token.gen` 校验拒绝旧 Token。
- 唯一性证明：generation 单调无回绕 → 任意两次发放 `(slot, generation)` 二元组不同。

---

## 4. 版本耗尽全入口覆盖（B40-03 核心）

版本耗尽检查**下沉到 `field_cas` 内部提交路径**，任何入口只要请求 `version+1`，耗尽时统一返回 `VERSION_EXHAUSTED_DENY`。全入口覆盖：admit / release / slot_alloc / token_consume / token_revoke / freeze / unfreeze / set_denied / recovery。

---

## 5. 资源端 Fencing 原子释放（B40-04 核心）

资源端唯一原子原语 `resource_atomic_release(resource, token, operation)`：token 校验、释放授权、资源状态变更、token 消费在同一临界区完成。flock/fcntl/fd 不归入 Fencing；不支持原子原语的资源 ISOLATED。

---

## 6. 反例与统计

- 本轮新增反例 **75 条**（TOKEN_IDENTITY=20 / ATOMIC_WIDTH=20 / VERSION_EXHAUST=15 / FENCING=20）。
- 累计：RAW_ENTRY_TOTAL=350 / UNIQUE_ID_TOTAL=350 / UNIQUE_DEFECT_TOTAL=348 / PASS=347 / FAIL=2 / NOT_PROVEN=1 / NOT_TESTED_RUNTIME=350。
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

- B40-01：PASS（单进程 VERIFIED_STATIC；跨进程/跨重启 NOT_PROVEN → DENY）
- B40-02：PASS（单进程 VERIFIED_STATIC；跨进程/跨重启 NOT_PROVEN → DENY）
- B40-03：PASS（单进程 VERIFIED_STATIC；跨重启 NOT_PROVEN → DENY）
- B40-04：PASS（有原子原语的资源 VERIFIED_STATIC；无原子原语 NOT_PROVEN → ISOLATED）
- B40-05：PASS（跨文档一致，统计可复算）

**整体结论：PASS-WITH-ISSUES**（跨进程/跨重启原子性与持久化未证明，对应功能 DENY/ISOLATED）。

- IMPLEMENT-01：NOT_APPROVED
- A/B Merge：NOT_APPROVED
- Source Retire：DENY

**STOP，等待 ChatGPT 独立复审。**
