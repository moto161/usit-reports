# RESOURCE-CRASH-AND-IDEMPOTENCY.md — 资源崩溃与幂等（05.40 权威）

证据等级：VERIFIED_STATIC（含 NOT_PROVEN 项）
目标：修复 B39-R04——定义资源端崩溃、响应丢失、重复释放、部分释放、资源端拒绝旧 token 的完整处理与幂等语义。

---

## 1. 幂等释放语义

幂等键 = `(resource_id, token)`。`resource_atomic_release` 的幂等保证：

| 调用次数 | 结果 |
|---|---|
| 第一次（token 匹配，未 released） | 执行释放，`released=True`，token 消费，返回 OK |
| 第二次（token 匹配，已 released） | ALREADY_RELEASED（不重复释放） |
| 任何次（token 不匹配） | FENCED_OUT |

---

## 2. 崩溃与响应丢失

| 场景 | 状态 | 处理 |
|---|---|---|
| 释放完成，响应丢失 | released=True，token 已消费 | 调用方重试 → ALREADY_RELEASED（幂等，无害） |
| 释放中途崩溃（operation 未完成） | released 未置位 | 重试（若 operation 幂等）或 ISOLATED（若不幂等/不确定） |
| 释放中途崩溃（operation 已完成，released 未置位） | 状态不明 | 不确定 → ISOLATED，不自动重放 |
| 资源端重启，token 丢失 | token 持久化失败 | FENCING_UNKNOWN → ISOLATED |
| 资源端重启，token 持久化成功 | token 恢复 | 可继续，按单调 token 判定 |

---

## 3. 部分释放

| 情况 | 处理 |
|---|---|
| operation 部分完成（如释放了 A 子资源但 B 未释放） | 状态不明 → ISOLATED，不自动继续 |
| operation 可证明幂等 | 可重试，幂等键去重 |
| operation 不可幂等 | ISOLATED，人工介入 |

**规则**：释放操作必须可验证地到达安全终态才标记 `released=True`。部分释放（不确定是否完成）不得标记完成。

---

## 4. 资源端拒绝旧 token

| 情况 | 处理 |
|---|---|
| 旧 Owner 用旧 token 释放 | FENCED_OUT（资源端原子拒绝） |
| 旧 token 因网络延迟迟到 | FENCED_OUT（token 单调，旧值必被拒） |
| token 回滚（非法） | 禁止（token 单调，无回滚路径） |
| token 持久化失败 | NOT_PROVEN → DENY |

---

## 5. 接管者崩溃 / 旧 Owner 复活 / 重复接管

| 场景 | 处理 |
|---|---|
| 接管者崩溃（cas_token 已成功） | token 已递增，旧 Owner 失效；新接管者需重新 cas_token 再递增 |
| 旧 Owner 复活释放 | FENCED_OUT（token 已递增） |
| 重复接管 | 第二次 cas_token 再递增，第一次接管者失效（若它未完成释放） |
| 接管与释放并发 | 资源端原子，唯一线性化点，一方胜出 |

---

## 6. 状态机（资源端）

```
IDLE ──takeover cas_token──▶ FENCED(new_token)
                                │
                                ├─ resource_atomic_release(valid token) ──▶ RELEASED
                                ├─ resource_atomic_release(old token)   ──▶ FENCED_OUT（保持 FENCED）
                                └─ 崩溃/重启 ──▶ UNKNOWN（token 持久化失败 → ISOLATED）
```

---

## 7. NOT_PROVEN 边界

- operation 幂等性未证明 → 崩溃后 ISOLATED，不自动重试。
- 资源端 token 持久化未证明 → 重启后 ISOLATED。
- 跨进程资源端原子性未证明 → NOT_PROVEN → DENY。

---

## 证据来源

- Fencing 协议：RESOURCE-FENCING-PROTOCOL.md
- 原子释放：RESOURCE-FENCING-ATOMIC-RELEASE.md
- 能力矩阵：RESOURCE-FENCING-CAPABILITY-MATRIX.md
- 05.39 缺陷：RESOURCE-RELEASE-ATOMICITY.md（05.39 版）未定义响应丢失/部分释放的幂等语义
- 证明等级：VERIFIED_STATIC（有原子原语）+ NOT_PROVEN（无原子原语）
