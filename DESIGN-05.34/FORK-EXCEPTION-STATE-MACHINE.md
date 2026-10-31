# FORK-EXCEPTION-STATE-MACHINE.md

## 目标
Fork 与快照异常路径闭合，明确状态转换、资源所有权、失败隔离、父子隔离。

## 1. os.fork() 真实异常行为契约

`os.fork()` 成功返回两次（父进程返回子 PID > 0，子进程返回 0）；失败抛 `OSError`（如资源耗尽 EAGAIN）。**不使用抽象的 `pid == -1` 作为正常失败机制**。封装层 `_do_fork()` 契约：成功返回 `(is_child: bool, child_pid)`；失败抛 `ForkFailed`。

## 2. 异常状态机

```
OPEN ──CAS freeze──► FREEZING ──drain──► FROZEN ──snapshot──► SNAPSHOT_READY ──fork──► 分叉
OPEN ◄──CAS unfreeze── FREEZING（超时/取消）
任意 ──snapshot 失败──► RESTORE_FAILED ──► OPEN（可恢复）或 DENIED（不可恢复）
SNAPSHOT_READY ──fork 异常──► FORK_FAILED ──► OPEN（回滚）或 DENIED
分叉 ──父初始化失败──► RESTORE_FAILED
分叉 ──子初始化失败──► 子安全终态（不影响父）
```

## 3. 完整伪代码（每步状态 + 清理责任）

```python
def _fork_coordinate(self):
    # 状态：OPEN
    with self._fork_coord_lock:
        if not self._admit_state.cas_freeze():
            raise FreezeRejected()            # 状态：保持 OPEN
    # 状态：FREEZING

    try:
        self._wait_admit_drain(timeout)       # 超时 → DrainTimeout → 回滚 OPEN
        self._assert_commit_idle()            # 失败 → 回滚
        self._assert_receipt_idle()           # 失败 → 回滚
    except DrainTimeout:
        self._rollback_open(); return         # 状态：OPEN

    # 状态：FROZEN
    try:
        self._take_snapshot()                 # 失败 → _snapshot_failed_path
    except Exception as e:
        self._snapshot_failed_path(e); return

    # 状态：SNAPSHOT_READY
    try:
        is_child, pid = self._do_fork()       # 抛 ForkFailed → _fork_failed_path
    except ForkFailed as e:
        self._fork_failed_path(e); return

    # 状态：分叉
    if is_child:
        self._child_path()                    # 子：清理继承 → 重建 → 独立初始化
    else:
        self._parent_path(pid)                # 父：恢复，不执行子初始化
```

## 4. 各失败路径清理责任

| 失败 | 责任主体 | 清理动作 | 终态 | 是否可恢复 |
|---|---|---|---|---|
| freeze 被拒 | 冻结者 | 无 | OPEN | 是 |
| drain 超时 | 冻结者 | CAS unfreeze | OPEN | 是 |
| commit/receipt 检查失败 | 冻结者 | CAS unfreeze | OPEN | 是 |
| snapshot 失败 | 冻结者 | CAS unfreeze + 释放快照资源 | RESTORE_FAILED→OPEN 或 DENIED | 视严重性 |
| fork 抛异常 | 冻结者（父） | CAS unfreeze | FORK_FAILED→OPEN 或 DENIED | 视严重性 |
| 父初始化失败 | 父协调器 | 释放父资源 | RESTORE_FAILED | 是 |
| 子初始化失败 | 子进程 | 回滚已建资源 | 子安全终态 | 否（子退出） |
| 锁重建失败 | 子进程 | 回滚已建锁 | 子安全终态 | 否 |
| 身份重建失败 | 子进程 | 回滚 | 子安全终态 | 否 |
| Receipt 检查失败 | 冻结者 | CAS unfreeze | OPEN 或隔离 | 是 |
| 清理自身再失败 | 清理者 | 记录附加异常，不改变已定终态 | 保持原终态 | — |

## 5. 资源所有权

- 父进程资源：父持有，父释放，不重复释放。
- 子进程资源：子持有，子释放。
- 快照资源：冻结者持有，快照失败时冻结者释放。
- 禁止父执行子专属初始化；禁止子释放父资源。

## 6. 关键规则

1. 清理失败不得错误恢复 OPEN（若已进入 DENIED 则保持 DENIED）。
2. 清理失败不得继续 Fork。
3. 重复释放：每个资源单一释放责任人，释放后置空引用。
4. 异常优先级：主异常 > 恢复异常 > 清理异常。

## 7. 证据等级
异常状态机 DESIGN_PROVEN（逻辑）；真实 fork 异常行为 NOT_TESTED。
