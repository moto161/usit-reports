# RESOURCE-IDEMPOTENCY-AND-RECOVERY.md — 资源幂等与恢复（05.42 权威）

证据等级：VERIFIED_STATIC（含 NOT_PROVEN 项）
目标：修复 B41-R04——通过 consumed_records 实现真正的幂等释放，区分 ALREADY_RELEASED 与 FENCED_OUT。

---

## 1. 幂等键与请求结果记录

幂等键 = `token`。`consumed_records[token]` 记录「该 token 已成功消费」。

| 请求 | 判定 | 返回 |
|---|---|---|
| token 在 consumed_records | 已消费 | ALREADY_RELEASED |
| token 不在记录，token != current_token | 被接管 | FENCED_OUT |
| token 不在记录，token == current_token | 首次 | 执行释放 → OK |

**关键顺序**：幂等检查（consumed_records）**先于** fencing 检查（current_token）。这样成功释放后重试命中幂等记录，返回 ALREADY_RELEASED，而非 FENCED_OUT。

---

## 2. 崩溃窗口（operation / token 消费 / 记录持久化）

`atomic_release` 在 fence_lock 内：operation → current_token+1 → consumed_records 写入。三者的持久化若不能原子：

| 崩溃点 | 状态 | 重试处理 |
|---|---|---|
| operation 前 | 无变化 | 正常重试 |
| operation 中 | 部分释放 | 非幂等 → ISOLATED；幂等 → 重试 |
| operation 完成、token 消费前 | 已释放，token 未递增 | 需幂等键判定 |
| token 消费 + 记录后 | 已释放，记录已持久化 | 重试 → ALREADY_RELEASED |

**关键崩溃窗口**：operation 完成但 consumed_records 未持久化。重试时 token 未命中记录，且若 current_token 也未持久化，可能重复释放。

---

## 3. 不确定状态隔离

- operation **幂等**：用幂等键重试，重复执行无害 → 可重试。
- operation **非幂等**：状态不明 → ISOLATED，不自动重放，人工介入。
- **不得声称整体原子**：若 operation 与记录持久化无法原子，明确标记「非原子」，崩溃后隔离。

---

## 4. 幂等记录持久化 / 保存期限

| 项 | 规则 |
|---|---|
| 持久化 | consumed_records 随资源端状态持久化 |
| 保存期限 | 直到资源彻底回收/归档 |
| 崩溃恢复 | 持久化后重试命中记录 → ALREADY_RELEASED |
| 去重 | 以 token 为幂等键 |

---

## 5. 资源端重启

| 场景 | 处理 |
|---|---|
| consumed_records 持久化成功 | 重启后重试命中记录 → ALREADY_RELEASED |
| consumed_records 持久化失败 | FENCING_UNKNOWN → ISOLATED |
| current_token 持久化失败 | FENCING_UNKNOWN → ISOLATED |

---

## 6. 状态机

```
IDLE ──takeover(fence_lock)──▶ FENCED(token=T)
                                 │
                                 ├─ atomic_release(T) 首次 ──▶ RELEASED（consumed_records[T]=CONSUMED）
                                 ├─ atomic_release(T) 重试 ──▶ ALREADY_RELEASED
                                 ├─ atomic_release(old<T) ──▶ FENCED_OUT
                                 └─ 崩溃（非幂等） ──▶ ISOLATED
```

---

## 7. NOT_PROVEN 边界

- consumed_records / current_token / released 持久化：NOT_PROVEN → DENY。
- operation 非幂等且无法原子持久化：崩溃后 ISOLATED。
- 跨进程 fence_lock：NOT_PROVEN → DENY。

---

## 证据来源

- Fencing 协议：RESOURCE-FENCING-PROTOCOL.md
- 接管释放串行化：RESOURCE-TAKEOVER-RELEASE-SERIALIZATION.md
- 05.41 缺陷：RESOURCE-CRASH-AND-IDEMPOTENCY.md（05.41 版）幂等语义与实际伪代码矛盾
- 证明等级：VERIFIED_STATIC（有 fence_lock）+ NOT_PROVEN（无原子原语）
