# PARENT-RESTORE-PROTOCOL.md

## 目标
重设计 `_close_fork_parent()` 全调用链，覆盖 11 种情况，明确异常优先级、最终状态写权限。

## 1. 全量伪代码（逐行）

```python
def _close_fork_parent(self):
    # 前置：父进程已从 fork() 返回，持有 _fork_coord_lock（协调器线程）
    primary_exc = None
    secondary_excs = []
    try:
        if self._fork_syscall_failed:
            # 情况 1：fork 系统调用失败
            primary_exc = self._fork_err
            self._enter_terminal('RESTORE_FAILED', primary_exc)   # 非 DENIED
            return
        if self._gate == 'RESTORING':
            # 情况 2：正常恢复 —— 仅 gate 仍为 RESTORING 才回 OPEN
            self._restore_gate_after_success()
        # 情况 3：gate 已为 DENIED —— 什么都不做，保持 DENIED
    except Exception as e:
        # 情况 4：恢复异常
        secondary_excs.append(e)
        if self._gate == 'DENIED':
            # 情况 5：DENIED 不可逆，绝不动
            return
        self._enter_terminal('RESTORE_FAILED', primary_exc or e)
    finally:
        # 情况 6：锁释放（只释放本线程持有的锁）
        try:
            self._release_owned_locks()
        except Exception as e:
            secondary_excs.append(e)   # 情况 7：锁释放失败，仅记录
    # 情况 8：finally 二次异常不覆盖 primary_exc，不改变 Gate 终态
    self._record_secondary_exceptions(secondary_excs)
```

## 2. 11 种情况覆盖表

| # | 情况 | 主异常 | Gate 终态 | 业务是否继续 |
|---|---|---|---|---|
| 1 | fork 系统调用失败 | fork_err | RESTORE_FAILED | 可重试后继续 |
| 2 | fork 成功，恢复成功 | 无 | OPEN | 是 |
| 3 | Freeze 失败（fork 前） | freeze_err | OPEN（回滚） | 是 |
| 4 | Parent Restore 失败 | restore_err | RESTORE_FAILED | 受控重试 |
| 5 | Gate 写入失败 | gate_err | DENIED | 否 |
| 6 | 锁释放失败 | lock_err（附加） | 保持原终态 | 视原终态 |
| 7 | finally 二次异常 | 附加，不覆盖 | 保持原终态 | 视原终态 |
| 8 | 子进程初始化失败 | 子进程侧，父进程无感 | 父进程 OPEN | 父进程是 |
| 9 | 父进程中断 | interrupt | TERMINATING 或 DENIED | 否 |
| 10 | 多异常同时发生 | 主异常优先级最高 | 视主异常 | — |
| 11 | gate 已 DENIED 后再恢复 | — | DENIED（不可逆） | 否 |

## 3. 异常优先级

主异常（原始业务/Fork 异常）> 恢复异常 > 清理异常。附加异常只记录，不覆盖，不影响终态。

## 4. 状态写权限归属

| 状态 | 唯一写入者 |
|---|---|
| OPEN/FREEZING/FROZEN | Gate 准入协调器 |
| RESTORING/RESTORE_FAILED | 父进程恢复协调器 |
| DENIED/TERMINATING | 终止协调器 |
| （人工解除 DENIED） | 管理授权器 `manual_reenable()` |

## 5. 父子恢复路径分离

- 父进程路径：`_close_fork_parent()` 恢复父侧锁与 Gate。
- 子进程路径：`_child_reinit()` 清理继承状态、重建锁与身份，失败进入子进程安全终态，**不回写父进程 Gate**。

## 6. 不变量

- I-PR-1：`_enter_terminal('DENIED')` 后任何路径不回写 OPEN。
- I-PR-2：finally 只释放锁，不写 Gate。
- I-PR-3：恢复成功仅当 gate == 'RESTORING' 时合法。
- I-PR-4：子进程不修改父进程 Gate 状态。

## 7. 证据等级
DESIGN_PROVEN（协议层）；NOT_TESTED（未运行真实 fork）。
