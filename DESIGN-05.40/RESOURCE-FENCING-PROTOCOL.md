# RESOURCE-FENCING-PROTOCOL.md — 资源端 Fencing 协议（05.40 权威）

证据等级：VERIFIED_STATIC（含 NOT_PROVEN 项）
目标：修复 B39-R04——05.39 的 `release_with_fencing_token()` 仍是「比较 token 后调 release」，未证明比较与释放之间无竞态。

---

## 1. 05.39 缺陷定位

05.39 的唯一接口：

```python
def release_with_fencing_token(resource, token):
    if token != resource.current_token:   # ① 校验
        return 'FENCED_OUT'
    resource.release()                     # ② 释放
    return 'OK'
```

缺陷：① 校验与 ② 释放是**两步**，中间存在 TOCTOU 窗口——校验通过后、释放前，另一个执行者可能递增 `current_token` 并完成接管，旧执行者仍会执行 ② 释放。05.39 文档声明「比较与释放是同一个原子操作」，但伪代码实际是两步。这是 B39-R04 的核心。

---

## 2. 资源端原子释放的唯一权威接口

资源端必须提供一个**单一原子原语**，使「Token 校验 + 释放授权 + 资源状态变更 + Token 消费」在同一步完成，无中间窗口：

```python
# 资源端唯一权威接口（由资源端实现，非调用方拼接）
def resource_atomic_release(resource, token, operation):
    """
    资源端原子操作：仅当 token == resource.current_token 时，
    原子地执行 operation 并递增/消费 token。
    返回 'OK' / 'FENCED_OUT' / 'ALREADY_RELEASED'。
    这是资源端提供的单一原语，调用方不得拆分为「校验 + 释放」两步。
    """
```

**关键**：`token` 校验与 `operation` 执行由资源端在同一临界区/同一事务内完成。调用方（Owner）只调用这一个原语，不自行先校验再释放。

---

## 3. 资源端原子性如何实现（三种可证明方式）

资源端原子性只能通过以下三种之一证明，缺一不可：

| 方式 | 机制 | 证明条件 |
|---|---|---|
| A. 资源端单写者 + 原子原语 | 资源端内部用锁/CAS 保护 `current_token` 与释放动作 | 资源端保证「读 token → 比较 → 释放 → 消费 token」在同一临界区 |
| B. 资源端单调 Token + CAS | 资源端用 CAS 原子比较并更新 token，释放动作受同一 CAS 保护 | 释放动作与 token CAS 在同一原子事务 |
| C. 外部事务服务 | 分布式锁/租约服务提供原子 lease+fence | 服务证明其 fencing token 原子性，否则 NOT_PROVEN |

**禁止**：调用方先 `check_token(token)` 再 `release()` 的两步拼接，无论中间隔多短。

---

## 4. 接管 CAS 与旧 Owner 释放的线性化顺序

接管流程（资源端）：

```python
def takeover(resource, new_owner):
    # 资源端原子递增 token（单调）
    new_token = resource.current_token + 1
    if not resource.cas_token(resource.current_token, new_token):
        return 'TAKEOVER_CONFLICT'   # 资源端 token 已被并发更新
    return 'TAKEOVER_OK'
```

**线性化顺序**：

1. 接管 `cas_token` 成功是**线性化点**——从此刻起，旧 token 失效。
2. 旧 Owner 的任何 `resource_atomic_release(resource, old_token, op)` 在接管 CAS 之后执行时，`old_token != current_token` → `FENCED_OUT`，原子拒绝。
3. 旧 Owner 在接管 CAS 之前已进入 `resource_atomic_release` 的资源端临界区，则其释放先于接管线性化，旧 Owner 的释放是**合法**的（因为当时 token 仍有效），接管随后接管已释放资源。

**为何旧 Owner 无法绕过**：旧 Owner 只能通过 `resource_atomic_release` 这一个原语释放，而该原语在资源端原子比较 token。旧 Owner 无法在「校验通过后、释放前」插入自己的释放动作（因为校验与释放在资源端同一临界区，旧 Owner 无法抢占）。

---

## 5. 旧 Owner 复活 / 迟到请求 / 重复请求

| 场景 | 处理 |
|---|---|
| 旧 Owner 复活释放 | `old_token != current_token` → FENCED_OUT（资源端原子拒绝） |
| 旧 Token 迟到 | 同上，迟到旧 token 被资源端拒绝 |
| 重复释放 | 幂等：token 已消费，第二次返回 ALREADY_RELEASED |
| 释放与接管并发 | 资源端原子，唯一线性化点，一方胜出 |
| 网络延迟 | 资源端 token 单调，迟到旧 token 被拒 |

---

## 6. 资源能力分级（继承 05.39，重新审定）

| 级别 | 定义 | 允许动作 |
|---|---|---|
| FENCING_SUPPORTED | 资源端提供原子 `resource_atomic_release`（token 校验 + 释放同临界区） | 安全接管释放 |
| OS_LOCK_ONLY | 仅 OS 锁/fd 生命周期，无资源端原子 token 原语 | **禁止接管释放** |
| FENCING_UNSUPPORTED | 无任何 token/锁机制 | 禁止接管，ISOLATED |
| FENCING_UNKNOWN | 未验证 | NOT_PROVEN，禁止接管 |

**重新审定（B39-R04）**：即使资源端有 token 字段，但若 `resource_atomic_release` 不能证明「校验+释放」原子，则**降级为 FENCING_UNKNOWN**，禁止接管释放。仅有 token 字段不够，必须有原子原语证明。

---

## 7. 心跳超时的角色

- 心跳超时**只能触发接管调查**，不得单独作为「旧 Owner 已停止」的证明。
- 接管必须通过资源端 `cas_token` 原子递增 token 完成，且资源端必须支持 `resource_atomic_release` 原子原语。
- 资源端不支持原子原语 → 接管仅能触发调查，不得执行释放。

---

## 8. NOT_PROVEN 边界

- 资源端无原子 token 原语：NOT_PROVEN → 禁止接管释放 → ISOLATED。
- 资源端 token 持久化失败：NOT_PROVEN → DENY。
- 跨进程资源端原子性未证明：NOT_PROVEN → DENY。
- flock/fcntl/fd：不是资源端 Fencing（无 token 校验原语）→ 禁止接管释放。

---

## 证据来源

- 资源原子释放：RESOURCE-FENCING-ATOMIC-RELEASE.md
- 能力矩阵：RESOURCE-FENCING-CAPABILITY-MATRIX.md
- 05.39 缺陷：RESOURCE-FENCING-PROTOCOL.md（05.39 版）§3 的 `release_with_fencing_token` 两步 TOCTOU
- 证明等级：VERIFIED_STATIC（有原子原语的资源）+ NOT_PROVEN（无原子原语的资源）
