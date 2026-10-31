# RESOURCE-FENCING-PROTOCOL.md — 资源端 Fencing 协议（05.42 权威）

证据等级：VERIFIED_STATIC（含 NOT_PROVEN 项）
目标：修复 B41-R04——接管与释放共用 fence_lock，引入独立幂等请求记录，区分 ALREADY_RELEASED 与 FENCED_OUT。

---

## 1. 05.41 缺陷定位（B41-R04）

05.41 的释放顺序：校验 token → 释放 → `current_token += 1` → `released = True`。但第一次释放已递增 `current_token`，第二次携带原 token 进入时，首先因 `token != current_token` 返回 FENCED_OUT，**根本到不了 released 检查**。

**根因**：用「当前 token + released 标志」推断历史请求结果，但 token 递增后丢失了「该请求曾成功」的信息。

---

## 2. 唯一权威决策：独立幂等请求记录

资源端引入**独立幂等请求记录 `consumed_records`**，记录「已成功消费的 (token, 请求标识)」，与 `current_token` 分离。

```python
class Resource:
    fence_lock: Lock
    current_token: u64          # 单调递增，标识当前 owner 代际
    released: bool
    owner: id
    consumed_records: dict      # {(token) -> 请求结果记录}

    def atomic_release(self, token, req_id, operation):
        with self.fence_lock:
            # 1. 幂等检查：该 token 是否已成功消费
            if token in self.consumed_records:
                return 'ALREADY_RELEASED'        # 先判幂等，再判 fencing
            # 2. fencing 检查：token 是否仍是当前代际
            if token != self.current_token:
                return 'FENCED_OUT'
            # 3. 执行释放
            result = operation()
            self.current_token += 1              # 消费 token
            self.released = True
            self.consumed_records[token] = {     # 记录成功消费
                'req_id': req_id, 'status': 'CONSUMED'
            }
            return 'OK'
```

**关键**：
- **幂等检查在前**（token in consumed_records → ALREADY_RELEASED），**fencing 检查在后**（token != current_token → FENCED_OUT）。
- 这样成功释放后，重试同 token 先命中 `consumed_records` → ALREADY_RELEASED，不会错误走 FENCED_OUT。

---

## 3. 四种请求的区分

| 请求 | 判定顺序 | 返回 |
|---|---|---|
| 同一请求重复（已成功消费） | consumed_records 命中 | ALREADY_RELEASED |
| 旧 Owner 请求（被接管后） | token != current_token | FENCED_OUT |
| 新 Owner 首次请求 | token == current_token 且未消费 | OK |
| 被接管后的新请求（token 已是新值） | token == current_token | OK |

---

## 4. 幂等记录保存期限 / 持久化 / 恢复

| 项 | 规则 |
|---|---|
| 保存期限 | 直到资源被彻底回收/归档 |
| 持久化 | consumed_records 随资源端状态持久化；未证明 → NOT_PROVEN → DENY |
| 崩溃恢复 | consumed_records 持久化后，重试命中记录 → ALREADY_RELEASED |
| 重复请求去重 | 以 token 为幂等键 |

---

## 5. 唯一线性化点

线性化点 = 在 `fence_lock` 内完成「幂等检查 + fencing 检查 + 释放 + 记录」的时刻。获取锁本身**不是**操作完成，锁内的实际状态变更才是线性化点。

---

## 6. operation 与 token 持久化无法原子时

若 operation（实际资源操作）与 consumed_records 持久化无法原子，崩溃后：

- operation 完成但 consumed_records 未持久化 → 重试时 token 未命中记录，且 token 未递增（若 current_token 也未持久化）→ 可能重复释放。
- **处理**：operation 非幂等且无法原子持久化 → ISOLATED，不自动重放。

**不得声称整体原子**（B42-05 第 7 条）。无法原子时明确隔离。

---

## 7. flock/fcntl/fd 仍非 Fencing

无 fence_lock + current_token + consumed_records 的单调语义，继续 OS_LOCK_ONLY，禁止接管释放。

---

## 8. NOT_PROVEN 边界

- 跨进程 fence_lock：NOT_PROVEN → DENY。
- consumed_records / current_token / released 持久化：NOT_PROVEN → DENY。
- operation 非幂等且无法原子持久化：崩溃后 ISOLATED。

---

## 证据来源

- 幂等与恢复：RESOURCE-IDEMPOTENCY-AND-RECOVERY.md
- 接管释放串行化：RESOURCE-TAKEOVER-RELEASE-SERIALIZATION.md
- 05.41 缺陷：RESOURCE-FENCING-PROTOCOL.md（05.41 版）token 递增导致幂等失效
- 证明等级：VERIFIED_STATIC（有 fence_lock）+ NOT_PROVEN（无原子原语）
