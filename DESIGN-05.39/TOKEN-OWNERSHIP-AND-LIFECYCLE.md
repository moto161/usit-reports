# TOKEN-OWNERSHIP-AND-LIFECYCLE.md — Token 所有权与生命周期（05.39）

证据等级：VERIFIED_STATIC
原子事务见 TOKEN-ATOMIC-TRANSACTION.md（权威）。

---

## 1. Token 唯一性

- Token 由 `token_seq`（32 位单调递增）生成，`admit()` 的 CAS 原子递增。
- 无回绕：`token_seq == 0xFFFFFFFF` 时 DENY（见 VERSION-EXHAUSTION-PROTOCOL.md）。
- 唯一性不依赖随机数，依赖单调序列号 + 无回绕证明。

---

## 2. Token 所有权

| 属性 | 值 |
|---|---|
| 发放者 | `admit()` 的 CAS（状态字内） |
| 持有者 | 完成准入的线程/进程 |
| 消费权 | 持有者一次（`release(token)` 消费后失效） |
| 所有权转移 | **不允许**（Token 绑定持有者，不转让） |

---

## 3. Token 生命周期状态机

```
NOT_ISSUED ──admit CAS──▶ ACTIVE ──release CAS──▶ CONSUMED
                              │
                              ├─ 业务异常退出（未 release）──▶ LEAKED（见 §5）
                              └─ 进程崩溃 ──▶ 恢复判定（见 TOKEN-CRASH-RECOVERY.md）
```

- `NOT_ISSUED`：token_seq 尚未分配。
- `ACTIVE`：active 位图置位，COUNT 计入。
- `CONSUMED`：active 位图清除，COUNT 减回。
- `LEAKED`：业务异常退出未 release，Token 仍 active，COUNT 多计 → 由清理/恢复处理（见 §5）。

---

## 4. 生成 / 持久化 / 消费 / 恢复规则

| 操作 | 规则 |
|---|---|
| 生成 | `token_seq` 单调递增，CAS 原子 |
| 持久化 | 内嵌状态字，随状态字持久化（跨重启需持久化证明，否则 NOT_PROVEN） |
| 消费 | `release(token)` CAS 原子清除 active 位 |
| 恢复 | 崩溃后按状态字 active 位图判定，见 TOKEN-CRASH-RECOVERY.md |

---

## 5. Token 泄漏处理

业务异常退出但未 `release(token)`（LEAKED）时：
- active 位图仍置位，COUNT 仍多计。
- 冻结等待 COUNT 归零时，LEAKED token 会导致冻结卡死。
- **处理**：业务异常必须由线程级清理（finally）调用 `release(token)`；若线程崩溃无法 release，由恢复协议（TOKEN-CRASH-RECOVERY.md）根据「业务是否已提交」判定：
  - 已提交 → 事务完成，token 视为可回收（COUNT−1 + 清 active）。
  - 未提交/不确定 → 隔离，不自动回收，COUNT 保持，冻结 DENY。

---

## 6. 跨线程 / 跨进程

- **跨线程**：内嵌状态字 + 128 位 CAS，原子性成立（同进程）。
- **跨进程**：需共享内存 + HW 128 位 CAS，未证明 → NOT_PROVEN → DENY。
- Token 不跨进程转让；跨进程场景下 Token 功能整体 DENY。

---

## 证据来源

- 原子事务：TOKEN-ATOMIC-TRANSACTION.md
- 崩溃恢复：TOKEN-CRASH-RECOVERY.md
- 证明等级：VERIFIED_STATIC（单进程）+ NOT_PROVEN（跨进程）
