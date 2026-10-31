# RESOURCE-FENCING-PROTOCOL.md — 资源端 Fencing 协议（05.39 权威）

证据等级：VERIFIED_STATIC（含 NOT_PROVEN 项）
目标：修复 B38-R01/R08——Fencing Token 无资源端注册/更新机制、flock/fcntl 被误归类、TOCTOU 未闭合。

---

## 1. 05.38 缺陷定位

1. `try_takeover()` 的 `r.fencing_token = new_token()` 是**进程内对象赋值**，未说明资源端如何原子接受新 Token / 拒绝旧 Token。
2. `flock/fcntl` 被误归类为 FENCING_SUPPORTED——fd 绑定或锁释放 ≠ 资源端单调 Token 拒绝旧执行者。
3. §4 仍保留 `accept_fencing_token()` → `release()` 两步，中间有 TOCTOU。

---

## 2. 五个必须区分的概念

| 概念 | 作用 | 作用域 | 能否阻止旧执行者在途操作 |
|---|---|---|---|
| 软件层 Owner 检查 | 是否允许发起 | 进程内 | 否（不撤销在途） |
| OS 锁/句柄生命周期 | 锁自动释放 | 进程/OS | 部分（进程退出释放锁，但不拒绝旧 Token） |
| 资源端 Fencing Token | 资源端拒绝旧操作 | 资源端 | 是（资源端校验 Token） |
| 资源端原子释放 | 校验+释放一步完成 | 资源端 | 是 |
| 幂等操作 | 重复操作无害 | 资源本身 | 是（仅幂等释放） |

**硬性规则**：`flock`/`fcntl`/fd 绑定/锁自动释放，属于「OS 锁/句柄生命周期」，**不是**资源端 Fencing。它们不能让资源端基于单调 Token 拒绝旧执行者的在途释放。

---

## 3. 资源端 Token 权威存储与机制

资源端 Fencing 必须满足以下机制，缺一不可：

1. **权威存储**：Token 的权威版本存储在**资源端**（而非进程内）。进程内只是缓存。
2. **单调递增**：Token 版本单调递增，旧 Token 永远小于新 Token。
3. **比较规则**：资源端释放操作比较「请求携带的 Token」与「资源端当前 Token」。
4. **拒绝旧 Token**：携带旧 Token 的释放请求被资源端**原子拒绝**。
5. **原子释放**：Token 校验与释放是**同一个原子操作**，无中间 TOCTOU 窗口。

```python
# 唯一权威接口（资源端实现）
def release_with_fencing_token(resource, token):
    """资源端原子：比较 token 与资源端当前 Token，
    相等则释放并返回 OK，不等则拒绝 FENCED_OUT。"""
    # 资源端内部（原子）：
    if token != resource.current_token:
        return 'FENCED_OUT'
    resource.release()
    return 'OK'
```

**禁止**：先 `accept_fencing_token(token)` 再独立 `release()` 的两步路径（TOCTOU）。

---

## 4. Token 注册与更新（接管时）

接管流程必须走资源端：

```python
def takeover(resource, new_owner):
    # 1. 资源端原子更新 Token（单调递增）
    new_token = resource.current_token + 1
    if not resource.cas_token(resource.current_token, new_token):
        return 'TAKEOVER_CONFLICT'   # 资源端 Token 已被并发更新
    # 2. 旧 Token 立即失效（资源端比较规则拒绝旧 Token）
    return 'TAKEOVER_OK'
```

**关键**：Token 更新在**资源端**原子完成（`cas_token`），旧 Token 从此刻起被资源端拒绝。进程内 `owner_id`/`version` 只是软件层辅助，不是 Fencing 权威。

---

## 5. 旧 Owner 复活为何无法释放

接管后，旧 Owner 若复活并尝试释放：
- 旧 Owner 携带的 Token 是旧值（接管前的 token）。
- 资源端当前 Token 已递增。
- `release_with_fencing_token(resource, old_token)` → `old_token != current_token` → `FENCED_OUT`。
- 旧 Owner 的在途释放被资源端原子拒绝。

**这是资源端 Fencing 与软件层 version 的本质区别**：version 递增只阻止「未来发起」，资源端 Token 拒绝「已在途操作」。

---

## 6. 资源能力分级（权威）

| 级别 | 定义 | 允许动作 | 资源示例 |
|---|---|---|---|
| FENCING_SUPPORTED | 资源端有单调 Token + 原子释放 | 安全接管释放 | 支持 Token 语义的分布式锁/租约服务 |
| OS_LOCK_ONLY | 仅 OS 锁/fd 生命周期，无资源端 Token | **禁止接管释放**（只能依赖进程退出自动释放） | flock/fcntl/fd |
| FENCING_UNSUPPORTED | 无任何 Token/锁机制 | 禁止接管，ISOLATED | unlink 临时文件 |
| FENCING_UNKNOWN | 未验证 | NOT_PROVEN，禁止接管 | 跨进程 mmap |

**修正 05.38**：`flock/fcntl` 从 FENCING_SUPPORTED **降级为 OS_LOCK_ONLY**，不再声称资源端 Fencing。

---

## 7. 各种异常场景

| 场景 | 处理 |
|---|---|
| 旧 Owner 迟到请求 | 资源端拒绝旧 Token（FENCED_OUT） |
| 重复请求 | 幂等：Token 已消费则拒绝 |
| 网络延迟 | 资源端 Token 单调，迟到旧 Token 被拒 |
| 资源端重启 | Token 持久化失败 → FENCING_UNKNOWN → ISOLATED |
| Token 回滚 | 禁止（Token 单调，无回滚） |
| Token 持久化失败 | NOT_PROVEN → DENY |
| 释放与接管同时 | 资源端原子，唯一线性化点，一方胜出 |

---

## 8. NOT_PROVEN 边界

- 不支持资源端 Token 的资源（flock/fcntl/unlink）：**禁止自动接管释放** → ISOLATED。
- 资源端 Token 持久化失败：NOT_PROVEN → DENY。
- 心跳超时只触发调查，不单独作为「旧 Owner 已停止」证明。

---

## 证据来源

- 资源能力矩阵：RESOURCE-FENCING-CAPABILITY-MATRIX.md
- 资源释放原子性：RESOURCE-RELEASE-ATOMICITY.md
- 05.38 缺陷：CLEANUP-FENCING-PROTOCOL.md（05.38 版）
- 证明等级：VERIFIED_STATIC + NOT_PROVEN（无 Token 资源）
