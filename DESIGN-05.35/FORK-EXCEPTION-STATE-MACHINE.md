# FORK-EXCEPTION-STATE-MACHINE.md

## 目标
统一异常处理状态机，覆盖全部异常路径。

## 1. 异常路径矩阵

| 异常 | 状态转换 | 责任人 | 资源所有权 | 终态 |
|---|---|---|---|---|
| 快照失败 | FROZEN→回滚 OPEN 或 DENIED | 冻结者 | 冻结者释放快照资源 | OPEN/DENIED |
| _do_fork 返回失败/异常 | SNAPSHOT_READY→回滚 | 冻结者（父） | 冻结者释放 | OPEN/DENIED |
| _do_fork 被中断 | 同上 | 冻结者 | 冻结者释放 | OPEN/DENIED |
| 父初始化失败 | 分叉→RESTORE_FAILED | 父协调器 | 父释放 | RESTORE_FAILED |
| 子初始化失败 | 子→安全终态 | 子进程 | 子释放 | 子退出 |
| 锁重建失败 | 子→安全终态 | 子进程 | 子回滚 | 子退出 |
| 身份重建失败 | 子→安全终态 | 子进程 | 子回滚 | 子退出 |
| Receipt 验证失败 | 冻结→回滚/隔离 | 冻结者 | 冻结者释放 | OPEN/隔离 |
| assert_commit_idle 失败 | 冻结→回滚 | 冻结者 | — | OPEN |
| assert_receipt_idle 失败 | 冻结→回滚 | 冻结者 | — | OPEN |
| 清理/回滚/恢复自身失败 | 保持原终态 | 清理者 | 记录附加异常 | 原终态 |
| 线程异常/取消 | 统一异常出口 | 各线程 | finally 释放 | 安全终态 |
| 进程崩溃/强制终止 | 崩溃恢复 | 恢复进程 | 磁盘证据 | 隔离 |
| 系统崩溃/重启 | 崩溃恢复 | 恢复进程 | 磁盘证据 | 隔离 |

## 2. 统一异常出口

```python
def _fork_coordinate(self):
    try:
        # 准入/冻结/等待/快照/fork
        ...
    except DrainTimeout:
        self._rollback_open()
    except ForkFailed:
        self._fork_failed_path()
    except SnapshotFailed:
        self._snapshot_failed_path()
    except Exception as e:
        self._enter_safe_terminal(e)   # 统一出口：清理 + 安全隔离
    finally:
        self._idempotent_cleanup()     # 幂等清理，不覆盖 DENIED
```

**证明**：所有异常通过 `_enter_safe_terminal` 或 `_idempotent_cleanup` 出口，冻结状态被清理或进入隔离，不绕过。

## 3. 父子初始化严格分离

- 父进程：恢复父锁/Gate/身份，**不执行子专属初始化**（不调用 `_child_*`/`_rebuild_*`）。
- 子进程：清理继承状态，重建子专属锁/身份/Gate。

## 4. 清理幂等

```python
def _idempotent_cleanup(self):
    if self._cleanup_done:
        return                       # 幂等：重复调用直接返回
    self._cleanup_done = True
    self._release_owned_resources()  # 每资源单一释放，释放后置空
    if self._admit_state.denied():
        return                       # DENIED 不覆盖
```

重复调用不重复释放、不覆盖 DENIED。

## 5. 清理失败处理

清理失败 → 进入明确隔离状态（RESTORE_FAILED / QUARANTINED），**不恢复 OPEN**。

## 6. 线程 finally vs 进程崩溃 vs 系统崩溃

| 场景 | 机制 | 覆盖 |
|---|---|---|
| 线程异常 | 线程级 finally | 清理线程资源 |
| 进程崩溃 | 进程退出，finally 不保证 | 崩溃恢复协议 |
| 系统崩溃 | 持久化 + 磁盘证据 | 重启恢复协议 |

禁止用线程 finally 覆盖进程/系统崩溃场景。

## 7. 证据等级
异常状态机 DESIGN_PROVEN（逻辑）；真实崩溃行为 NOT_TESTED。
