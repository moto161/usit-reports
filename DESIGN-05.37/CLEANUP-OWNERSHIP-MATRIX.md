# CLEANUP-OWNERSHIP-MATRIX.md — 资源所有权矩阵（05.37 权威）

证据等级：VERIFIED_STATIC
本文件为资源所有权与释放责任的**唯一权威定义**。IDEMPOTENT-CLEANUP-PROTOCOL.md 引用本矩阵。

---

## 1. 资源清单与所有权

| 资源 | 唯一所有者 | 释放条件 | 幂等依据 | 失败动作 | 状态不明时的处理 | 最终状态 |
|---|---|---|---|---|---|---|
| `_fork_coord_lock` | Fork 协调者 | Fork 完成/失败后 | 锁句柄释放天然幂等 | 记录失败，重试≤3 | uncertain → ISOLATED | RELEASED/ISOLATED |
| `_admit_state`（进程内） | 进程自身 | 进程退出 | 随进程消失 | — | — | GONE |
| `_admit_state`（跨进程共享） | 恢复者（经接管） | 旧写入者排除后 | CAS 重建，非释放 | 证据不足 → DENY | 旧写入者存活 → DENY | REBUILT/DENY |
| 子进程继承锁副本 | 子进程 | 子进程初始化清理 | 副本清理幂等 | 失败 → 子进程 ISOLATED | uncertain → ISOLATED | RELEASED/ISOLATED |
| Gate 状态 | 冻结者 | 恢复 OPEN 前 | 状态字段幂等写入 | 失败 → 保持 CLOSED | — | RELEASED/CLOSED |
| 持久化事务句柄 | 事务所有者 | 事务终态后 | 句柄 close 幂等 | 失败 → ISOLATED | uncertain → ISOLATED | RELEASED/ISOLATED |
| 临时文件/快照 | 快照者 | 快照提交后 | unlink 幂等（不存在则 no-op） | 失败 → 保留并记录 | uncertain → ISOLATED | RELEASED/ISOLATED |
| 锁句柄（flock/fcntl） | 锁获取者 | 解锁后 | 重复解锁 no-op | 失败 → 记录 | uncertain → ISOLATED | RELEASED/ISOLATED |

---

## 2. 所有权字段（每项资源）

```python
class ResourceHandle:
    name: str
    owner_id: str              # 唯一所有者标识
    released: bool = False     # 是否已释放
    uncertain: bool = False    # 状态是否不明（禁止重试）
    retry_count: int = 0       # 已重试次数
    permanent_failed: bool = False  # 永久失败（禁止再试）
    MAX_RETRY: int = 3         # 重试上限
    last_error: Exception = None
```

---

## 3. 所有权获取的线性化点

- 清理执行所有权：`claim_ownership()` 在 `_claim_lock` 内原子检查+赋值（线性化点 = owner_id 被写下的那一刻）。
- 资源级释放权：`release_resource()` 校验 `owner_id == 当前执行者` 且 `released == False`，在资源状态检查与 `r.release()` 之间，通过 `_claim_lock` 串行化（同一执行者持有所有权期间独占）。

---

## 4. 所有权不变量

1. 任一资源在任何时刻最多一个 `owner_id`。
2. 同一资源不得被两个执行者同时释放（所有权独占 + 版本校验）。
3. `released=True` 后不再释放（幂等）。
4. `uncertain=True` 后不释放（隔离）。
5. `permanent_failed=True` 后不重试（隔离）。
6. 所有权转移（接管）必须 `ownership_version += 1`，旧 owner 失效。

---

## 5. 资源状态 → 清理动作映射

| 资源状态 | 清理动作 |
|---|---|
| released=False, uncertain=False, permanent_failed=False | 尝试释放 |
| released=True | no-op（幂等） |
| uncertain=True | 隔离（不释放） |
| permanent_failed=True | 隔离（不释放） |

---

## 证据来源

- 清理状态机：IDEMPOTENT-CLEANUP-PROTOCOL.md
- 证明等级：VERIFIED_STATIC
