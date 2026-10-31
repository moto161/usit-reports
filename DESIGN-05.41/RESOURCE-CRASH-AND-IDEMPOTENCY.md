# RESOURCE-CRASH-AND-IDEMPOTENCY.md — 资源崩溃与幂等（05.41 权威）

证据等级：VERIFIED_STATIC（含 NOT_PROVEN 项）
目标：定义 operation / token 消费 / released 持久化之间的崩溃窗口与幂等语义。

---

## 1. 幂等释放语义

幂等键 = `(resource_id, token)`。返回值区分：

| 请求 | 状态 | 返回 |
|---|---|---|
| token 匹配，未 released | 执行释放 | OK |
| token 匹配，已 released | 不重复释放 | ALREADY_RELEASED |
| token 不匹配（被接管） | 拒绝 | FENCED_OUT |

**区分规则**：`released` 与 `token` 是独立判断。先判 token（不匹配→FENCED_OUT），再判 released（已释放→ALREADY_RELEASED）。

---

## 2. 崩溃窗口（operation / token 消费 / released 持久化）

`atomic_release` 在 `fence_lock` 内执行 operation → token 消费（current_token+1）→ released=True。三者的持久化若不能原子，存在崩溃窗口：

| 崩溃点 | 资源状态 | 持久化状态 | 重试处理 |
|---|---|---|---|
| operation 前 | 未释放 | 无变化 | 重试正常 |
| operation 中 | 部分释放 | released 未持久化 | 非幂等 → ISOLATED；幂等 → 重试 |
| operation 完成、token 消费前 | 已释放 | released 未持久化，token 未消费 | 需幂等键判定（见 §3） |
| token 消费 + released 后 | 已释放 | released 持久化 | 重试 → ALREADY_RELEASED |

---

## 3. 不确定状态的处理

若「operation 完成但 released/token 未持久化」（窗口 2/3），重试时**无法**仅凭 released 标志判断是否已释放。处理：

- **operation 幂等**：用幂等键 `(resource_id, token)` 重试，重复执行无害 → 可重试。
- **operation 非幂等**：状态不明 → **ISOLATED**，不自动重放，人工介入。

**不得声称整体原子**：若实际资源操作不能与 token 更新构成原子事务，必须明确标记该资源「operation 与 token 更新非原子」，崩溃后按不确定状态隔离，不得声称「校验+释放原子」（B41-04 第 7 条）。

---

## 4. 资源能力分级（继承，重新审定）

| 级别 | 条件 | 允许动作 |
|---|---|---|
| FENCING_SUPPORTED | 有 fence_lock + 单调 token + operation 幂等或可原子持久化 | 接管释放 |
| FENCING_PARTIAL | 有 fence_lock + token，但 operation 非幂等且无法原子持久化 | 接管释放时崩溃窗口 → ISOLATED |
| OS_LOCK_ONLY | flock/fcntl/fd，无 token | 禁止接管释放 |
| FENCING_UNSUPPORTED | 无任何机制 | ISOLATED |
| FENCING_UNKNOWN | 未验证 | NOT_PROVEN → 禁止接管 |

> **新增 FENCING_PARTIAL**：区分「有串行化边界但崩溃恢复不确定」的资源。这类资源允许单进程内接管释放，但崩溃后的重试必须 ISOLATED（不自动重放）。

---

## 5. 资源端拒绝旧 token / token 回滚 / 持久化失败

| 场景 | 处理 |
|---|---|
| 旧 token 迟到 | FENCED_OUT（token 单调） |
| token 回滚 | 禁止（单调，无回滚） |
| token 持久化失败 | NOT_PROVEN → DENY |
| 资源端重启 | token 持久化失败 → FENCING_UNKNOWN → ISOLATED |

---

## 6. 状态机

```
IDLE ──takeover(fence_lock)──▶ FENCED(new_token)
                                  │
                                  ├─ atomic_release(valid) ──▶ RELEASED
                                  ├─ atomic_release(old)    ──▶ FENCED_OUT
                                  ├─ atomic_release(repeat) ──▶ ALREADY_RELEASED
                                  └─ 崩溃（非幂等） ──▶ ISOLATED
```

---

## 7. NOT_PROVEN 边界

- operation 非幂等且无法原子持久化 → 崩溃后 ISOLATED。
- 跨进程 fence_lock → NOT_PROVEN → DENY。
- token/released 持久化 → NOT_PROVEN → DENY。

---

## 证据来源

- Fencing 协议：RESOURCE-FENCING-PROTOCOL.md
- 接管释放串行化：RESOURCE-TAKEOVER-RELEASE-SERIALIZATION.md
- 05.40 缺陷：RESOURCE-CRASH-AND-IDEMPOTENCY.md（05.40 版）未区分 FENCING_PARTIAL
- 证明等级：VERIFIED_STATIC（有 fence_lock）+ NOT_PROVEN（无原子原语）
