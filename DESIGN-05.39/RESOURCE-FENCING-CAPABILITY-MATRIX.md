# RESOURCE-FENCING-CAPABILITY-MATRIX.md — 资源 Fencing 能力矩阵（05.39 权威）

证据等级：VERIFIED_STATIC
Fencing 协议见 RESOURCE-FENCING-PROTOCOL.md。

---

## 1. 资源能力分级（权威，修正 05.38）

| 资源 | 能力级别 | 是否有资源端 Token | 原子释放 | 接管释放 | 证据 |
|---|---|---|---|---|---|
| 分布式锁/租约服务（带 Token） | FENCING_SUPPORTED | 是 | 是 | 安全 | 资源端协议支持单调 Token |
| flock（文件锁） | OS_LOCK_ONLY | 否 | 否（依赖进程退出） | **禁止** | 仅 OS 锁生命周期 |
| fcntl（记录锁） | OS_LOCK_ONLY | 否 | 否 | **禁止** | 同上 |
| fd 绑定句柄 | OS_LOCK_ONLY | 否 | 否 | **禁止** | fd 身份 ≠ Token |
| unlink 临时文件 | FENCING_UNSUPPORTED | 否 | 否 | **禁止** | 无 Token 语义 |
| 裸 close/句柄释放 | FENCING_UNSUPPORTED | 否 | 否 | **禁止** | 无 Token |
| 跨进程 mmap 状态字 | FENCING_UNKNOWN | 未验证 | 未验证 | NOT_PROVEN | HW CAS 未证明 |
| 持久化事务句柄（带幂等键） | FENCING_UNKNOWN | 部分 | 未验证 | NOT_PROVEN | 需幂等键证明 |

---

## 2. 关键修正（B38-R08）

05.38 将 `flock/fcntl/fd 绑定` 归入 FENCING_SUPPORTED，是**错误分类**。原因：

- fd 绑定只保证「进程退出时 fd 关闭、锁释放」，是 OS 层的生命周期管理。
- 它**不能让资源端基于单调 Token 拒绝旧执行者的在途释放**。
- 因此 flock/fcntl/fd 降级为 OS_LOCK_ONLY，禁止接管释放。

---

## 3. 各能力级别的允许动作

| 级别 | 允许 | 禁止 |
|---|---|---|
| FENCING_SUPPORTED | 接管释放（资源端 Token） | — |
| OS_LOCK_ONLY | 依赖进程退出自动释放；接管只做只读调查 | 接管释放 |
| FENCING_UNSUPPORTED | 无 | 接管释放，标记 ISOLATED |
| FENCING_UNKNOWN | 无 | 接管释放，NOT_PROVEN → DENY |

---

## 4. 接管策略映射

| 资源级别 | 心跳超时后动作 |
|---|---|
| FENCING_SUPPORTED | 资源端 cas_token 接管，换新 Token，安全释放 |
| OS_LOCK_ONLY | 只读调查；等待旧 Owner 进程退出自动释放，不主动释放 |
| FENCING_UNSUPPORTED | 标记 uncertain → ISOLATED |
| FENCING_UNKNOWN | NOT_PROVEN → DENY |

---

## 证据来源

- Fencing 协议：RESOURCE-FENCING-PROTOCOL.md
- 05.38 错误分类：CLEANUP-OWNERSHIP-MATRIX.md（05.38 版）§2
- 证明等级：VERIFIED_STATIC
