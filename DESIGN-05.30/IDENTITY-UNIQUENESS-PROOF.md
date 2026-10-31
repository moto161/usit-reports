# IDENTITY-UNIQUENESS-PROOF.md

## 目标
分别证明各类唯一性，严格区分 STRICT_STRUCTURAL / PROBABILISTIC / NOT_GUARANTEED / UNKNOWN。

## 1. 唯一性分类逐项

| # | 场景 | 结论 | 依据 |
|---|---|---|---|
| 1 | 单进程实例内唯一 | STRICT_STRUCTURAL | 内存计数器 + 锁 |
| 2 | 同父兄弟 fork 事件唯一 | STRICT_STRUCTURAL | branch_path 末位序号互异 |
| 3 | 同树多代 fork 事件唯一 | STRICT_STRUCTURAL | branch_path 前缀不同 |
| 4 | 不同树之间唯一 | PROBABILISTIC | tree_id 128-bit 随机 |
| 5 | 同主机不同进程唯一 | PROBABILISTIC | tree_id + instance 随机 |
| 6 | PID 重用区分 | NOT_GUARANTEED（依赖随机分量） | 不依赖 PID |
| 7 | 进程崩溃后身份 | PROBABILISTIC | 快照恢复 |
| 8 | 系统重启后身份 | PROBABILISTIC | tree_id 重生成 |
| 9 | 随机源失效 | UNKNOWN → DENY | 无随机则拒绝建树 |
| 10 | 身份复制/恢复/导入 | NOT_GUARANTEED（需校验） | 校验失败隔离 |

## 2. 结构唯一性证明（树内）

**定理**：同一 `process_tree_id` 下，任意两次 fork 事件的 `branch_path` 互不相同。

**证明**（归纳）：
- 基例：根进程第 1 次 fork 分配 `[1]`，第 2 次分配 `[2]`，末位由原子 `local_fork_sequence` 递增，互异。
- 归纳步：设深度 k 的某进程 P 的 branch_path 为 `p=[s1..sk]`。P 的第 j 次 fork 分配 `p+[j]`。不同兄弟 j 不同 → 路径末位不同；不同祖先 → 路径前缀不同。故全树路径两两不同。
- 原子性：`local_fork_sequence` 的递增由身份分配锁 `_identity_alloc_lock` 保护，多线程/父子并发串行化，无重复分配。

结论：树内 fork_event_id 唯一性为 **STRICT_STRUCTURAL**。

## 3. 树间唯一性（如实标注）

`process_tree_id` 为 128-bit 随机，碰撞概率 ≈ 2^-128。这是**概率唯一**，非严格唯一。故：
- 树间 fork_event_id：**PROBABILISTIC**。
- 任何需要全局严格唯一的下游断言，必须引入可信全局注册表（超出本设计边界，标 UNKNOWN）。

## 4. 禁止项

- 禁止将概率唯一写成严格唯一。
- 禁止用随机后缀替代结构性唯一性证明。
- 禁止将 PID 作为唯一身份。
- 禁止在随机源失效时静默沿用旧身份。

## 5. 降级路径

| 条件 | 降级动作 |
|---|---|
| 随机源不可用 | 拒绝建树（DENY），不沿用旧 tree_id |
| branch_path 损坏 | 校验失败 → 隔离 |
| local_fork_sequence 溢出 | 拒绝继续 fork（DENY） |
| 快照序号倒退 | 拒绝恢复（重放防护） |

## 6. 证据等级
树内结构唯一：DESIGN_PROVEN；树间唯一：PROBABILISTIC；全局严格唯一：UNKNOWN（需可信注册表）。
