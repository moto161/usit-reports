# PROCESS-TREE-IDENTITY.md

## 目标
修复 05.29「概率分量冒充结构唯一」。从结构上重新设计身份关系，引入 `branch_path`。

## 1. 身份八层分离

| 层 | 定义 | 唯一性来源 | 等级 |
|---|---|---|---|
| A. Process Instance Identity | 单个 OS 进程实例 | 随机 + PID + 启动时间 | PROBABILISTIC |
| B. Process Tree Identity | 树根身份 | 128-bit 随机 | PROBABILISTIC |
| C. Process Branch Identity | 根到进程的结构路径 | 纯结构性序号序列 | STRICT_STRUCTURAL（树内） |
| D. Fork Event Identity | 一次 fork 事件 | tree_id + branch_path | 树内 STRICT，树间 PROB |
| E. Snapshot Identity | 一次 fork 快照 | 随机 | PROBABILISTIC |
| F. Transaction Identity | 事务 | 随机 | PROBABILISTIC |
| G. Generation Identity | 代际 | 结构序号 | STRICT（树内） |
| H. PID Identity | OS 进程号 | OS 分配，可重用 | NOT_GUARANTEED |

**禁止用单一随机 ID 同时承担多个职责。**

## 2. 结构性唯一候选

```
fork_event_id = (process_tree_id, branch_path)
branch_path    = [s1, s2, ..., sk]   # 从根到当前进程，每层父内分配的序号
local_fork_sequence = sk（当前分支内单调递增）
```

`branch_path` 不含随机分量；`[1,3,2]` 表示根第1子 → 该子第3子 → 该子第2子。

## 3. 14 项证明要点

| # | 问题 | 结论 |
|---|---|---|
| 1 | 根进程身份创建 | 首次生成 process_tree_id（随机），branch_path=[] |
| 2 | 树间防碰撞 | 仅概率（128-bit 随机），标 PROBABILISTIC |
| 3 | 子分支路径生成 | 父在 child_path = parent_path + [next_seq] |
| 4 | 兄弟不同序号 | 身份分配锁原子 next_seq |
| 5 | 多代路径不同 | 路径前缀不同即不同 |
| 6 | 父子并发 fork | 身份分配锁串行化 |
| 7 | 多线程原子分配 | 身份分配锁保护 |
| 8 | 崩溃时序号 | 序号持久化到快照，恢复读取 |
| 9 | 重启旧身份 | 快照恢复 branch_path，不新生成 |
| 10 | 快照恢复防重放 | 快照含 monotonic 序号，拒绝倒退 |
| 11 | PID 重用 | 不依赖 PID 判身份 |
| 12 | 根身份复制 | 复制需重生成 tree_id 或标记冲突 |
| 13 | 外部导入 | 校验 tree_id + branch_path 一致性 |
| 14 | 身份损坏 | 校验失败 → DENY / 隔离 |

## 4. 进程身份转换表

| 阶段 | process_tree_id | branch_path | process_instance_id | parent_instance_id | fork_event_id |
|---|---|---|---|---|---|
| 根进程创建 | 新生成 | [] | 新生成 | — | — |
| Fork 前（父） | 保留 | parent_path | 保留 | 保留 | — |
| Fork 返回父进程 | 保留 | parent_path | 保留 | 保留 | 新分配 child 路径 |
| Fork 返回子进程 | 继承 | child_path（重建） | 重建 | 父 instance_id（历史） | 新生成 |
| 子进程重建前 | 继承 | 未定 | 未定 | — | 未定 |
| 子进程重建后 | 继承 | child_path | 新生成 | 保留历史 | 新生成 |
| 子进程再次 Fork | 继承 | child_path | 保留 | 保留 | 新分配孙路径 |
| 进程重启恢复 | 从快照恢复 | 从快照恢复 | 新生成 | 快照历史 | 从快照恢复 |

字段标记：继承 / 重建 / 保留历史值 / 新生成 / 禁止继续。

## 5. 关键结论

- `process_instance_id`：子进程**重建**（新随机），不继承父值；父值保留为 `parent_instance_id` 历史证据。
- `fork_event_id`：树内严格唯一靠 branch_path；树间靠 tree_id 概率唯一。
- PID 永不作为唯一身份。

## 6. 证据等级
结构唯一性：DESIGN_PROVEN（树内）；树间：PROBABILISTIC（如实）。运行时：NOT_TESTED。
