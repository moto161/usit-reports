# FORK-PARENT-CHILD-PATHS.md

## 目标
父子执行路径图，证明父进程与子进程完全隔离。

## 1. 执行路径图

```
                    ┌─────────────┐
                    │ 入口: OPEN   │
                    └──────┬──────┘
                           │ 获取 _fork_coord_lock (唯一所有者:协调线程)
                           ▼
                    ┌─────────────┐
                    │ FREEZING    │ 置 admit_closed (短持有 _admit_lock)
                    │ 归零        │ 检查提交 (短持有 _commit_lock)
                    │ 快照        │
                    └──────┬──────┘
                           │ os.fork()
                           ▼
                 ┌─────────┼─────────┐
            pid<0 │         │ pid>0   │ pid==0
                 ▼         ▼         ▼
        ┌──────────┐ ┌──────────┐ ┌──────────────────┐
        │ 失败路径  │ │ 父进程路径│ │ 子进程路径         │
        │ 回滚Gate │ │ 恢复Gate │ │ 丢弃继承锁         │
        │ 回滚admit│ │ 不释放coord│ │ 重建锁/身份/Gate   │
        └──────────┘ │ 不触碰子锁│ │ 恢复业务准入       │
                     └──────────┘ └──────────────────┘
```

## 2. 父子路径职责对照

| 职责 | 父进程 `_parent_path` | 子进程 `_child_path` |
|---|---|---|
| 释放 coord 锁 | **否**（with 已释放） | **否**（丢弃副本） |
| 恢复 Gate | 是（RESTORING→OPEN） | 重建（→OPEN） |
| 恢复 admit_closed | 是 | 重建（open） |
| 重建锁 | **否** | 是（全部） |
| 重建身份 | **否** | 是 |
| 触碰子锁 | **否** | — |
| 触碰父锁 | — | **否**（丢弃副本） |

## 3. 隔离不变量

- I-PC-1：父进程 `_parent_path` 不引用任何 `_child_*` / `_rebuild_*` 函数。
- I-PC-2：子进程 `_child_path` 不引用父恢复函数 `_close_fork_parent` / `_restore_gate_after_success`（子走独立重建路径）。
- I-PC-3：父子锁实例独立，互不释放。
- I-PC-4：fork 后父进程 Gate/身份/锁不受子进程任何操作影响。

## 4. 不变量证明

**证明 I-PC-1/I-PC-2**：`_parent_path` 与 `_child_path` 由 `pid` 值互斥分支（pid>0 与 pid==0 互斥），且两函数体不交叉引用。

**证明 I-PC-3**：fork 语义保证父子进程内存独立；子进程 `_discard_inherited_locks()` 丢弃的是**子进程内存中的副本**，不触碰父进程内存中的锁对象。

**证明 I-PC-4**：父子内存独立 → 子进程对自身 Gate/身份/锁的写不反映到父进程。

结论：DESIGN_PROVEN（协议层）。

## 5. Fork 失败路径

- pid<0：`_fork_failed_path()` 回滚 Gate 到 OPEN、admit_closed 到 False。
- coord 锁已在 with 退出释放，无残留。
- 不产生子进程，无子路径。

## 6. 证据等级
协议层 DESIGN_PROVEN；真实 fork 行为 NOT_TESTED。
