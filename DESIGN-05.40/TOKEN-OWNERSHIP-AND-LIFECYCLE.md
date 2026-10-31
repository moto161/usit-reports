# TOKEN-OWNERSHIP-AND-LIFECYCLE.md — Token 所有权与生命周期（05.40 权威）

证据等级：VERIFIED_STATIC（单进程）+ NOT_PROVEN（跨进程/跨重启）
原子事务见 TOKEN-ATOMIC-TRANSACTION.md（权威）。

---

## 1. Token 唯一性（修订 B39-R01）

- Token 身份 = `(slot, generation)`，二者内嵌 128 位状态字。
- `slot`：16 槽位位图的 bit 索引（0–15）。
- `generation`：全局 16 位单调递增，禁止回绕（耗尽 DENY）。
- **唯一性证明**：generation 单调且无回绕，任意两次发放的 `(slot, generation)` 二元组不同（即使 slot 相同，generation 必不同）。∎
- 唯一性**不依赖随机数**，依赖「单调 generation + 禁止回绕」的静态可证性质。

---

## 2. Token 完整结构

| 字段 | 位置 | 说明 |
|---|---|---|
| token_id = slot | 位图 bit 索引 | 槽位标识 |
| generation | 发放时全局 generation | 代际，防旧 Token 复活 |
| owner_id | 业务上下文 | 审计用，不内嵌状态字 |
| resource_id | 资源标识 | 绑定资源，不内嵌状态字 |
| state_version | 发放时 VERSION | 恢复判定用，内嵌状态字 |

---

## 3. Token 所有权

| 属性 | 值 |
|---|---|
| 发放者 | `slot_alloc()` 的 field_cas（状态字内） |
| 持有者 | 完成准入的线程/进程 |
| 消费权 | 持有者一次（`release(token)` 消费后槽位清位） |
| 所有权转移 | **不允许**（Token 绑定持有者，不转让） |

---

## 4. Token 生命周期状态机

```
NOT_ISSUED ──slot_alloc CAS──▶ ACTIVE(slot, gen)
                                   │
                                   ├─ release CAS ──▶ CONSUMED（槽位清位，可复用）
                                   ├─ token_revoke CAS ──▶ REVOKED（槽位清位）
                                   ├─ 业务异常未 release ──▶ LEAKED（见 §6）
                                   └─ 进程崩溃 ──▶ 恢复判定（TOKEN-CRASH-RECOVERY.md）
```

---

## 5. 生成 / 持久化 / 消费 / 撤销 / 过期 / 恢复

| 操作 | 规则 |
|---|---|
| 生成 | `slot_alloc()`：槽位占用 + generation+1，field_cas 原子 |
| 持有 | ACTIVE 状态，COUNT 计入 |
| 消费 | `release(token)`：身份校验 + 槽位清位 + COUNT−1，field_cas 原子 |
| 撤销 | `token_revoke(token)`：槽位清位（不改变 COUNT，仅撤销身份） |
| 过期 | 无基于时间的过期；仅通过消费/撤销/恢复终止 |
| 恢复 | 崩溃后按状态字槽位位图 + generation 判定，见 TOKEN-CRASH-RECOVERY.md |

> `token_revoke` 与 `release` 的区别：`release` 同时 COUNT−1（正常消费），`revoke` 仅撤销身份不清 COUNT（由清理/恢复后续处理 COUNT）。二者在状态字层面均清槽位位，故「旧 Token 复活」均被 `gen` 校验拒绝。

---

## 6. Token 泄漏（LEAKED）处理

业务异常退出但未 `release(token)`：

- 槽位仍占用，COUNT 仍多计。
- 冻结等待 COUNT 归零时，LEAKED token 会导致冻结卡死。
- **处理**：
  - 业务异常必须由线程级清理（finally）调用 `release(token)`。
  - 线程崩溃无法 release 时，由恢复协议根据「业务是否已提交」判定：
    - 已提交 → 事务完成，token 可回收（槽位清位 + COUNT−1）。
    - 未提交/不确定 → ISOLATED，不自动回收，COUNT 保持，冻结 DENY。

---

## 7. 跨线程 / 跨进程

- **跨线程**：内嵌状态字 + 128 位 field_cas，原子性成立（同进程）。VERIFIED_STATIC。
- **跨进程**：需共享内存 + HW 128 位 CAS，未证明 → NOT_PROVEN → DENY。
- Token 不跨进程转让；跨进程场景下 Token 功能整体 DENY。

---

## 证据来源

- Token 身份：TOKEN-IDENTITY-AND-BITMAP-MODEL.md
- 原子事务：TOKEN-ATOMIC-TRANSACTION.md
- 崩溃恢复：TOKEN-CRASH-RECOVERY.md
- 证明等级：VERIFIED_STATIC（单进程）+ NOT_PROVEN（跨进程/跨重启）
