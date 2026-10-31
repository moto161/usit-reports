# RESOURCE-RELEASE-ATOMICITY.md — 资源释放原子性（05.39）

证据等级：VERIFIED_STATIC
Fencing 协议见 RESOURCE-FENCING-PROTOCOL.md。

---

## 1. 唯一权威释放接口

资源释放只能通过**一个**接口，禁止「先校验 Token、再独立释放」的两步 TOCTOU 路径：

```python
def release_with_fencing_token(resource, token):
    """资源端原子：Token 校验与释放一步完成。"""
    # 资源端内部（原子，无 TOCTOU 窗口）：
    if token != resource.current_token:
        return 'FENCED_OUT'        # 旧 Token 被原子拒绝
    resource.release()             # 原子释放
    return 'OK'
```

**禁止的 05.38 模式**：

```python
# 禁止（两步，TOCTOU）
if not r.resource.accept_fencing_token(fencing_token):
    return 'FENCED_OUT'
r.resource.release(fencing_token)   # ← 与 accept 之间有窗口
```

---

## 2. 原子性的证明前提

`release_with_fencing_token` 的原子性依赖资源端实现：

| 前提 | 要求 | 不满足后果 |
|---|---|---|
| Token 比较与释放原子 | 资源端单步完成 | 两步 → TOCTOU → NOT_PROVEN |
| Token 单调 | 资源端 Token 单调递增 | 回绕 → 旧 Token 复活 |
| Token 持久化 | 资源端 Token 跨重启持久 | 重启丢失 → Token 回滚 |

---

## 3. 线性化点

| 场景 | 线性化点 |
|---|---|
| 正常释放 | `release_with_fencing_token` 的资源端原子比较+释放完成 |
| 接管 | 资源端 `cas_token` 成功（Token 递增） |
| 旧 Token 拒绝 | 资源端 Token 比较失败（FENCED_OUT） |

---

## 4. 资源端拒绝旧 Token 失败的处理

若资源端「拒绝旧 Token」本身失败（如资源端崩溃、Token 状态损坏）：
- 无法确认旧 Token 是否已被拒绝 → FENCING_UNKNOWN。
- 释放结果不确定 → ISOLATED。
- 不自动接管释放。

---

## 5. 与软件层 Owner 检查的关系

软件层 Owner 检查（进程内 owner_id/version）是**第一道防线**（是否允许发起），资源端 Token 是**第二道防线**（是否接受）。两道都必须通过：

```python
def release(resource, executor_id, token):
    if not software_owner_check(resource, executor_id):
        return 'OWNERSHIP_LOST'        # 软件层拒绝
    return release_with_fencing_token(resource, token)   # 资源端原子
```

软件层检查失败 → 不发起；资源端 Token 不匹配 → FENCED_OUT。二者缺一不可，且资源端是跨进程安全的最终依据。

---

## 证据来源

- Fencing 协议：RESOURCE-FENCING-PROTOCOL.md
- 能力矩阵：RESOURCE-FENCING-CAPABILITY-MATRIX.md
- 05.38 缺陷：两步 accept+release 的 TOCTOU
- 证明等级：VERIFIED_STATIC + NOT_PROVEN（资源端原子性未证明）
