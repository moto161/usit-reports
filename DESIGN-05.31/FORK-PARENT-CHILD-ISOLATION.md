# FORK-PARENT-CHILD-ISOLATION.md

## 目标
严格隔离父子执行路径，父进程不执行任何子进程专属初始化。

## 1. 核心原则

- 父进程路径：保留父进程原有锁、身份、Gate 状态及资源所有权，**不重建、不覆盖**自身锁/身份/Gate。
- 子进程路径：清理继承的锁状态，重建子进程专属锁、身份、Gate。
- 所有子进程专属操作必须处于明确子进程分支内。

## 2. 完整伪代码（显式分支，非缩进表达安全边界）

```python
def _fork_coordinate(self):
    # ---- 公共前置：父进程侧，协调线程 ----
    if self._gate != 'OPEN':
        raise ForkRejected()
    with self._fork_coord_lock:
        self._gate = 'FREEZING'
        with self._admit_lock:
            self._admit_closed = True
        self._wait_biz_drain()
        self._assert_no_commit_in_progress()
        self._pre_fork_snapshot()
        pid = self._do_fork()

    # ============ 显式分叉 ============
    if pid < 0:
        # ---- Fork 失败路径（父进程） ----
        self._fork_failed_path()
    elif pid == 0:
        # ---- 子进程路径（唯一子进程专属操作区） ----
        self._child_path()
    else:
        # ---- 父进程路径 ----
        self._parent_path(pid)

def _fork_failed_path(self):
    # 父进程，未产生子进程
    with self._fork_coord_lock:
        self._admit_closed = False
        self._gate = 'OPEN'          # 回滚到 OPEN（未进 DENIED）
    self._record_fork_error()

def _parent_path(self, pid):
    # 父进程：仅释放协调锁 + 恢复 Gate，不重建任何子进程资源
    self._close_fork_parent()        # 见 PARENT-RESTORE-PROTOCOL
    # 禁止：不调用 _rebuild_* 任何子进程重建函数

def _child_path(self):
    # 子进程：清理继承 → 重建专属资源
    self._child_cleanup_inherited()  # 清继承锁状态
    self._rebuild_fork_coord_lock()  # 子重建协调锁
    self._rebuild_admit_lock()
    self._rebuild_commit_lock()
    self._rebuild_identity()         # 子重建身份（branch_path 续接）
    self._gate = 'OPEN'
    self._admit_closed = False
```

## 3. 父子路径状态转换表

| 阶段 | 执行进程 | 锁 | 身份 | Gate | 资源所有权 |
|---|---|---|---|---|---|
| Fork 前 | 父 | 父持 coord/admit | 父 identity | FREEZING | 父 |
| Fork 返回 pid<0 | 父 | 释放协调锁 | 父保留 | OPEN（回滚） | 父 |
| Fork 返回 pid>0 | 父 | 父释放自己锁 | 父保留 | 恢复 OPEN | 父 |
| Fork 返回 pid==0 | 子 | 子重建全部锁 | 子重建 | 子重建 OPEN | 子独立 |
| 子清理继承 | 子 | 清继承状态 | 未定 | 未定 | 子 |
| 子重建完成 | 子 | 子专属锁 | 子 identity | OPEN | 子 |
| 子重建失败 | 子 | 释放已建 | 未定 | 安全终态 | 子 |

## 4. 未知 PID / 异常退出 / 部分初始化失败

| 情况 | 处理 |
|---|---|
| pid < 0（fork 失败） | `_fork_failed_path()` 回滚 OPEN |
| pid == 0 但子重建异常 | 子进入安全终态（TERMINATING/DENIED），不写父 Gate |
| 子部分初始化失败 | 回滚已建资源，进入安全终态 |
| 父等待子退出异常 | 父记录，不影响父 Gate 恢复 |
| 未知 PID（os 异常） | 标记 UNKNOWN，进入安全终态，不盲目恢复 |

## 5. 父进程不受子进程初始化影响的不变量证明

**不变量 I-FPC-1**：父进程在 `_parent_path()` 中只调用 `_close_fork_parent()`，该函数（见 PARENT-RESTORE-PROTOCOL）仅释放父持有的锁与恢复父 Gate，不触碰子进程资源。

**证明**：
1. `_parent_path` 分支与 `_child_path` 分支由 `pid` 值互斥（pid>0 与 pid==0 不可能同时为真）。
2. `_parent_path` 内不引用任何 `_rebuild_*` / `_child_*` 函数。
3. fork 后父子内存独立，子进程对自身锁/身份/Gate 的修改不影响父进程副本。
4. 故父进程锁/身份/Gate 不被子进程初始化影响。

**结论**：DESIGN_PROVEN（协议层）。

## 6. 与 PARENT-RESTORE-PROTOCOL 一致性

- `_parent_path` 委托 `_close_fork_parent()`，与 05.30 PARENT-RESTORE-PROTOCOL 的 11 种情况覆盖一致。
- 无冲突：父路径不新增子进程操作，子路径不调用父恢复函数。

## 7. 证据等级
协议层 DESIGN_PROVEN；真实 fork 行为 NOT_TESTED。
