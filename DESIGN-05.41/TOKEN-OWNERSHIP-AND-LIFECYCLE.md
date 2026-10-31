# TOKEN-OWNERSHIP-AND-LIFECYCLE.md — Token 所有权与生命周期（05.41 权威）

证据等级：VERIFIED_STATIC（单进程）+ NOT_PROVEN（跨进程/跨重启）
原子事务见 TOKEN-ATOMIC-TRANSACTION.md（权威）。

---

## 1. Token 完整身份

Token 身份 = `(slot, generation, owner_id, resource_id, state_version)`：

| 字段 | 存储 | 说明 |
|---|---|---|
| slot | 状态字 TOKEN_ACTIVE bit 索引 | 0–15 |
| generation | TRT[slot].generation | 每槽位独立 16 位 |
| owner_id | TRT[slot].owner_id | 持有者 |
| resource_id | TRT[slot].resource_id | 资源绑定 |
| state_version | 发放时 VERSION | 恢复判定 |

---

## 2. Token 唯一性

- 身份 = `(slot, generation)`，每槽位 generation 独立单调、禁止回绕。
- 任意槽位两次发放 generation 必不同（复用 +1）。
- **独立验证**：每个 Token 只依赖 `TRT[slot].generation`，不依赖全局计数，故他槽位变化不影响它。∎

---

## 3. Token 所有权

| 属性 | 值 |
|---|---|
| 发放者 | `admit()` 事务（txn_lock 内） |
| 持有者 | 完成准入的线程/进程 |
| 消费权 | 持有者一次 |
| 所有权转移 | **不允许** |

---

## 4. Token 生命周期状态机

```
NOT_ISSUED ──admit(槽位占用, TRT写入)──▶ ACTIVE(slot, gen)
                                            │
                                            ├─ release ──▶ CONSUMED（清位+代际+1）
                                            ├─ revoke ──▶ REVOKED（清位，见 §5）
                                            ├─ 业务异常未 release ──▶ LEAKED
                                            └─ 崩溃 ──▶ 恢复判定
```

---

## 5. release / consume / revoke 的唯一规则（B41-02 关键）

| 操作 | 是否减 COUNT | 是否清位图 bit | 是否递增 TRT 代际 | 定义 |
|---|---|---|---|---|
| release | **是** | 是 | 是 | 正常消费：COUNT−1 + 清位 + 代际失效 |
| consume | **是**（release 的别名，见 §6） | 是 | 是 | 与 release 完全等价 |
| revoke | **否** | 是 | 是 | 撤销身份：清位 + 代际失效，**COUNT 由清理/恢复处理** |

> **唯一明确规则**：`token_consume` 与 `release` 是**同一操作的两个名称**，均减 COUNT。`token_revoke` 是**独立操作**，只撤销身份不清 COUNT。COUNT 的清理由 `cleanup`/`recovery` 完成。此规则全文档唯一，禁止多个名称指向不同实现。

---

## 6. 操作别名统一（B41-02）

| 规范名称 | 别名 | 是否同一实现 |
|---|---|---|
| admit | — | 唯一 |
| slot_alloc | token_create | **同一实现**（都是分配槽位） |
| release | token_consume | **同一实现**（都减 COUNT 消费） |
| token_revoke | — | 唯一（只撤销身份） |

> `slot_alloc` 与 `token_create` 是同一操作；`release` 与 `token_consume` 是同一操作。禁止多个名称指向不同实现。

---

## 7. 生成 / 持久化 / 消费 / 撤销 / 过期 / 恢复

| 操作 | 规则 |
|---|---|
| 生成 | admit（txn_lock 内，COUNT+1 + 槽位占用 + TRT 写入） |
| 持久化 | TRT + 状态字一起持久化，未证明 → NOT_PROVEN → DENY |
| 消费 | release/consume（COUNT−1 + 清位 + 代际+1） |
| 撤销 | revoke（清位 + 代际+1，COUNT 不变） |
| 过期 | 无时间过期，仅通过消费/撤销/恢复终止 |
| 恢复 | 见 TOKEN-CRASH-RECOVERY.md |

---

## 8. Token 泄漏（LEAKED）

业务异常未 release：
- 槽位仍占用，COUNT 多计。
- 冻结等待 COUNT 归零时卡死。
- 处理：线程级 finally 调 release；线程崩溃由恢复判定（已提交→回收；未提交/不确定→ISOLATED）。

---

## 9. NOT_PROVEN 边界

- 跨进程 TRT 共享 + txn_lock：NOT_PROVEN → DENY。
- 跨重启 TRT 持久化：NOT_PROVEN → DENY。

---

## 证据来源

- 每槽位代际：TOKEN-IDENTITY-AND-PER-SLOT-GENERATION.md
- 原子事务：TOKEN-ATOMIC-TRANSACTION.md
- 状态转换：STATE-INVARIANTS-AND-TRANSITIONS.md
- 证明等级：VERIFIED_STATIC（单进程）+ NOT_PROVEN（跨进程/跨重启）
