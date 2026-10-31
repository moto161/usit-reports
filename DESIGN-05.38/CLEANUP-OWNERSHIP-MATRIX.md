# CLEANUP-OWNERSHIP-MATRIX.md — 资源所有权矩阵（05.38 修订）

证据等级：VERIFIED_STATIC
本文件为资源所有权、Fencing 支持与释放责任的权威定义。与 CLEANUP-FENCING-PROTOCOL.md 一致。

---

## 1. 资源清单（含 Fencing 能力）

| 资源 | 唯一所有者 | Fencing 支持 | 释放条件 | 幂等依据 | 接管策略 | 状态不明处理 |
|---|---|---|---|---|---|---|
| `_fork_coord_lock` | Fork 协调者 | 否（进程内锁） | Fork 完成/失败 | 锁句柄释放幂等 | 不跨进程，无需接管 | 随进程消失 |
| `_admit_state`（进程内） | 进程自身 | 否 | 进程退出 | 随进程消失 | — | GONE |
| `_admit_state`（跨进程共享 mmap） | 恢复者 | **取决于实现** | 旧写入者排除后 | CAS 重建 | 需 mmap 支持原子 CAS，否则 NOT_PROVEN | 旧写入者存活 → DENY |
| 子进程继承锁副本 | 子进程 | 否 | 子进程初始化 | 副本清理幂等 | 不跨进程 | uncertain → ISOLATED |
| Gate 状态 | 冻结者 | 否（进程内状态字） | 恢复 OPEN 前 | 状态字段幂等 | 不跨进程 | — |
| 持久化事务句柄 | 事务所有者 | **取决于文件锁** | 事务终态后 | 句柄 close 幂等 | flock 有 Token 语义则支持 | uncertain → ISOLATED |
| 临时文件/快照 | 快照者 | **否（unlink 无 Token）** | 快照提交后 | unlink 幂等 | **无法安全接管** → ISOLATED | uncertain → ISOLATED |
| 锁句柄（flock/fcntl） | 锁获取者 | **是（fd 绑定）** | 解锁后 | 重复解锁 no-op | flock 关闭 fd 释放，进程崩溃自动释放 | uncertain → ISOLATED |

---

## 2. Fencing 能力分级

| 级别 | 说明 | 资源示例 | 接管释放安全性 |
|---|---|---|---|
| FENCING_SUPPORTED | 资源端校验 Token，原子释放 | flock/fcntl（fd 绑定） | 安全（Token 拒绝旧操作） |
| FENCING_UNSUPPORTED | 无 Token 语义 | unlink 临时文件、裸 close | **NOT_PROVEN，禁止接管释放** |
| FENCING_UNKNOWN | 未验证 | 跨进程 mmap 状态字 | NOT_PROVEN，需证明原子性 |

---

## 3. 资源状态字段（05.38 扩展）

```python
class ResourceHandle:
    name: str
    owner_id: str
    ownership_version: int
    released: bool = False
    uncertain: bool = False
    retry_count: int = 0
    permanent_failed: bool = False
    MAX_RETRY: int = 3
    fencing_supported: bool      # 资源是否支持 Fencing Token
    fencing_token: object        # 资源端 Token（若支持）
    last_error: Exception = None
```

**新增字段**：`fencing_supported`、`fencing_token`。这是 05.37 缺失、导致 B37-R01 的根因。

---

## 4. 所有权获取与释放的原子边界

- 软件层所有权（owner_id/version）：进程内锁保护，线性化点 = 锁内赋值。
- 资源端释放：Fencing Token 保护，线性化点 = 资源端 `accept_fencing_token` + 原子释放。

**规则**：软件层校验通过 ≠ 资源已安全释放。资源释放由资源端 Fencing 决定（见 CLEANUP-FENCING-PROTOCOL.md §4）。

---

## 5. 不变量

1. 任一资源任一时刻最多一个有效 owner（owner_id + version）。
2. 同一资源不得被两个执行者同时**发起**释放（软件层校验）。
3. 同一资源不得被两个执行者同时**实际**释放（资源端 Fencing，若支持）。
4. Fencing 不支持的资源，禁止接管释放（NOT_PROVEN → ISOLATED）。
5. `released=True` 后不再释放（幂等）。
6. `uncertain=True` 后不释放（隔离）。

---

## 证据来源

- Fencing 协议：CLEANUP-FENCING-PROTOCOL.md
- 05.37 缺陷：IDEMPOTENT-CLEANUP-PROTOCOL.md（05.37 版）的 version 递增
- 证明等级：VERIFIED_STATIC + NOT_PROVEN（Fencing 不支持资源）
