# RESOURCE-FENCING-ATOMIC-RELEASE.md — 资源端 Fencing 原子释放（05.40 权威）

证据等级：VERIFIED_STATIC（含 NOT_PROVEN 项）
目标：修复 B39-R04——给出资源端「Token 校验 + 释放 + 消费」的原子释放协议，消除 TOCTOU 窗口。

---

## 1. 原子释放的唯一原语

资源端必须实现单一原子原语，调用方不得拆分：

```python
# 资源端内部（唯一原子原语）
def resource_atomic_release(resource, token, operation):
    """资源端原子：token 校验、释放授权、资源状态变更、token 消费 在同一临界区。"""
    with resource.internal_lock:            # 资源端单写者锁（或等价 CAS 事务）
        if resource.current_token != token:
            return 'FENCED_OUT'              # 旧 token 被拒绝
        if resource.released:
            return 'ALREADY_RELEASED'        # 幂等：已释放
        result = operation(resource)         # 执行实际释放动作
        resource.current_token += 1          # 消费 token（单调递增，释放后旧 token 失效）
        resource.released = True
        return 'OK'
```

**关键**：`token` 比较、`operation` 执行、`current_token` 递增、`released` 置位，四者都在 `internal_lock` 临界区内，无中间 TOCTOU 窗口。

---

## 2. 原子性证明（三种资源端模型）

| 模型 | 临界区机制 | 证明条件 | 状态 |
|---|---|---|---|
| 单写者锁 | 资源端 `internal_lock` | 锁正确实现且无重入死锁 | VERIFIED_STATIC（单进程） |
| 资源端 CAS | 资源端 `cas_token` 保护释放动作 | 释放动作与 token CAS 同事务 | 视资源端实现，未证明则 NOT_PROVEN |
| 外部事务服务 | 分布式锁/租约的 fence token | 服务证明其 fencing 原子性 | 未证明 → NOT_PROVEN |

**跨进程**：资源端锁必须是跨进程锁（如共享内存锁/分布式锁），普通进程内锁不提供跨进程原子性 → NOT_PROVEN → DENY。

---

## 3. 接管 CAS 与释放的线性化点

- **接管线性化点**：`resource.cas_token(old, new)` 成功。
- **释放线性化点**：`resource_atomic_release` 进入临界区并比较 token 通过的时刻。

两个线性化点通过资源端 `current_token` 的全序关系确定先后：

```
时间线：
  旧 Owner:  resource_atomic_release(old_token) ──▶ 比较 old_token==current? 
  接管者:   cas_token(old→new) 成功 ──▶ current_token=new
```

- 若旧 Owner 的比较发生在 cas_token 之前：`old_token == current` 成立，释放合法。
- 若旧 Owner 的比较发生在 cas_token 之后：`old_token != current` → FENCED_OUT。

**没有**「比较通过但释放被接管打断」的窗口，因为比较与释放都在同一临界区，接管者必须等临界区释放后才能 cas_token。

---

## 4. 崩溃 / 响应丢失 / 资源端重启

| 场景 | 处理 |
|---|---|
| 释放完成但响应丢失 | 幂等：第二次 `resource_atomic_release` 返回 ALREADY_RELEASED |
| 释放中途崩溃 | 资源端临界区未完成，`released` 未置位 → 下次可重试（若 operation 幂等）或 ISOLATED（若不确定） |
| 资源端重启 | token 持久化失败 → FENCING_UNKNOWN → ISOLATED |
| Token 回滚 | 禁止（token 单调递增，无回滚） |
| 持久化失败 | NOT_PROVEN → DENY |

---

## 5. 幂等释放键

- 幂等键 = `(resource_id, token)`。
- `resource_atomic_release` 用 `released` 标志 + `current_token` 判断：
  - token 匹配且未 released → 执行释放。
  - token 匹配但已 released → ALREADY_RELEASED（幂等）。
  - token 不匹配 → FENCED_OUT。

---

## 6. 禁止清单（05.39 遗留必须删除）

1. `if token != resource.current_token: return FENCED_OUT; resource.release()` 两步拼接。
2. `accept_fencing_token(token)` 后再独立 `release()`。
3. 用 `atomic=True` / `transaction()` 命名而不给实际临界区/事务边界。
4. 将 flock/fcntl/fd 绑定当作资源端 token 校验原语。

---

## 7. NOT_PROVEN 边界

- 资源端无法证明「校验+释放」原子 → 禁止接管释放 → ISOLATED。
- 跨进程资源端原子性（跨进程锁/CAS）未证明 → NOT_PROVEN → DENY。
- 资源端 token 持久化失败 → NOT_PROVEN → DENY。

---

## 证据来源

- Fencing 协议：RESOURCE-FENCING-PROTOCOL.md
- 能力矩阵：RESOURCE-FENCING-CAPABILITY-MATRIX.md
- 崩溃幂等：RESOURCE-CRASH-AND-IDEMPOTENCY.md
- 05.39 缺陷：RESOURCE-FENCING-PROTOCOL.md（05.39 版）§3 两步 TOCTOU
- 证明等级：VERIFIED_STATIC（有原子原语的资源）+ NOT_PROVEN（无原子原语）
