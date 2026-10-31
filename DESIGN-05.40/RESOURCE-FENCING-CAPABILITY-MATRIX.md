# RESOURCE-FENCING-CAPABILITY-MATRIX.md — 资源端 Fencing 能力矩阵（05.40 权威）

证据等级：VERIFIED_STATIC（含 NOT_PROVEN 项）
目标：修复 B39-R04——重新审定资源能力分级，明确哪些资源能安全接管释放，哪些必须 ISOLATED。

---

## 1. 资源能力分级（权威，重新审定）

| 级别 | 判定条件（必须全部满足） | 允许动作 | 资源示例 |
|---|---|---|---|
| FENCING_SUPPORTED | ① 资源端有单调 token；② 资源端提供 `resource_atomic_release` 原子原语（校验+释放同临界区）；③ token 可持久化或跨进程可见性已证明 | 安全接管释放 | 提供 fencing token 语义的分布式锁/租约服务 |
| OS_LOCK_ONLY | 仅 OS 锁/fd 生命周期，无资源端 token 校验原语 | **禁止接管释放**（只能依赖进程退出自动释放） | flock / fcntl / fd 绑定 |
| FENCING_UNSUPPORTED | 无任何 token/锁机制 | 禁止接管，ISOLATED | unlink 临时文件 |
| FENCING_UNKNOWN | 有 token 字段但无法证明原子原语；或跨进程原子性未证明 | NOT_PROVEN，禁止接管 | 跨进程 mmap 上的 token 字段 |

**重新审定（B39-R04）**：仅有 `current_token` 字段不足以上升到 FENCING_SUPPORTED。必须同时具备「原子校验+释放原语」证明。凡缺少原子原语证明者，一律降级 FENCING_UNKNOWN。

---

## 2. 逐资源能力审定表

| 资源类型 | token 字段 | 原子释放原语 | 跨进程原子性 | 审定级别 | 允许动作 |
|---|---|---|---|---|---|
| 分布式锁（etcd/zk lease） | 有（lease 单调） | 有（服务端原子） | 服务端保证 | FENCING_SUPPORTED | 接管释放 |
| 数据库行锁 + fencing 列 | 有（需 CAS） | 视实现 | 视实现 | FENCING_UNKNOWN（未证明原子） | 禁止接管 |
| flock/fcntl | 无 token | 无 | 无 | OS_LOCK_ONLY | 禁止接管释放 |
| fd 绑定 | 无 token | 无 | 无 | OS_LOCK_ONLY | 禁止接管释放 |
| unlink 临时文件 | 无 | 无 | 无 | FENCING_UNSUPPORTED | ISOLATED |
| 跨进程 mmap token 字段 | 有（软件层） | 无证明 | NOT_PROVEN | FENCING_UNKNOWN | 禁止接管 |
| 本地文件锁 + 自建 token | 有（软件层） | 无证明 | 无 | FENCING_UNKNOWN | 禁止接管 |

---

## 3. 各能力等级的准入、接管、释放、失败策略

| 级别 | 准入 | 接管 | 释放 | 失败策略 |
|---|---|---|---|---|
| FENCING_SUPPORTED | 允许 | 允许（cas_token 原子） | 允许（resource_atomic_release） | 失败 → FENCED_OUT / ALREADY_RELEASED |
| OS_LOCK_ONLY | 允许（依赖 OS 锁） | 禁止 | 仅进程退出自动释放 | 接管请求 → 拒绝 |
| FENCING_UNSUPPORTED | 禁止 | 禁止 | 禁止 | ISOLATED |
| FENCING_UNKNOWN | 禁止 | 禁止 | 禁止 | NOT_PROVEN → ISOLATED |

---

## 4. 心跳超时与接管的关系

| 信号 | 作用 | 能否触发接管释放 |
|---|---|---|
| 心跳超时 | 触发接管**调查** | 否（仅调查） |
| 资源端 cas_token 成功 | 完成接管（token 递增） | 是（仅 FENCING_SUPPORTED） |
| resource_atomic_release 成功 | 完成释放 | 是（仅 FENCING_SUPPORTED） |

---

## 5. 不得误判的资源

以下机制**不是**资源端 Fencing（B39-R04 重申）：

- flock / fcntl：OS 锁，无 token 校验，进程退出才释放。
- fd 绑定 / 句柄生命周期：句柄关闭 ≠ 资源端拒绝旧 token。
- 软件层 owner_id / version：只阻止「未来发起」，不拒绝「已在途操作」。
- 进程内锁：不提供跨进程原子性。

---

## 证据来源

- Fencing 协议：RESOURCE-FENCING-PROTOCOL.md
- 原子释放：RESOURCE-FENCING-ATOMIC-RELEASE.md
- 崩溃幂等：RESOURCE-CRASH-AND-IDEMPOTENCY.md
- 05.39 缺陷：RESOURCE-FENCING-CAPABILITY-MATRIX.md（05.39 版）把 fd 绑定/锁自动释放归入 FENCING_SUPPORTED
- 证明等级：VERIFIED_STATIC（有原子原语）+ NOT_PROVEN（无原子原语）
