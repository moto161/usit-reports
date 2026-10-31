# FORK-LOCK-LIFECYCLE.md

## 目标
修复 05.29 锁获取路径缺失、父子锁状态未区分问题。提供 15 步完整伪代码。

## 1. 15 步 Fork 协调锁完整伪代码

每步标记：`执行进程｜当前线程｜持有锁｜锁所有者｜Gate｜in_flight｜身份状态｜异常处理`

```python
def _fork_coordinate(self):
    # 步骤1：Fork 请求进入（父进程，调用线程）
    #   持锁：无｜所有者：无｜Gate：OPEN｜in_flight：正常｜身份：父｜异常：入口拒绝记日志
    if self._gate != 'OPEN':
        raise ForkRejected('gate not open')

    # 步骤2：获取 _fork_coord_lock（父进程，调用线程）
    with self._fork_coord_lock:
        # 步骤3：检查 Gate（父进程，协调线程）
        #   持锁：_fork_coord_lock｜Gate：OPEN｜异常：Gate 非 OPEN → 拒绝
        self._assert_gate_open()

        # 步骤4：禁止新的普通准入（父进程，协调线程）
        #   持锁：_fork_coord_lock + _admit_lock｜Gate：FREEZING
        self._gate = 'FREEZING'
        with self._admit_lock:
            self._admit_closed = True

        # 步骤5：等待已有普通操作结束（父进程，协调线程）
        #   持锁：_fork_coord_lock｜等待 _biz_in_flight == 0（条件变量）
        self._wait_biz_drain(timeout=self._freeze_timeout)

        # 步骤6：检查提交状态（父进程，协调线程）
        #   持锁：_fork_coord_lock + _commit_lock（短持有）
        self._assert_no_commit_in_progress()

        # 步骤7：执行 Fork 前准备（父进程，协调线程）
        self._pre_fork_snapshot()

        # 步骤8：调用 fork（父进程，协调线程）—— 关键分叉点
        pid = self._do_fork()

        # 步骤9：父进程释放/恢复协调资源（父进程，协调线程）
        if pid > 0:
            self._close_fork_parent()   # 见 PARENT-RESTORE-PROTOCOL.md

        # 步骤10：子进程清理继承状态（子进程，fork 后单线程）
        else:
            self._child_cleanup_inherited()

        # 步骤11：子进程重建协调锁（子进程）
        self._rebuild_fork_coord_lock()

        # 步骤12：子进程重建其他进程本地锁（子进程）
        self._rebuild_admit_lock(); self._rebuild_commit_lock()

        # 步骤13：子进程重建身份（子进程）
        self._rebuild_identity()

        # 步骤14：子进程恢复业务准入（子进程）
        self._gate = 'OPEN'; self._admit_closed = False

        # 步骤15：异常进入安全终态（任一步骤异常）
        #   任一失败 → _enter_terminal('DENIED' 或 'TERMINATING')
```

## 2. 关键点

- 步骤 8 fork 前，协调线程持有 `_fork_coord_lock`；fork 后父子各有独立副本。
- 步骤 9 父进程释放**自己**持有的锁；步骤 10-12 子进程**重建**（不释放父锁）。
- 子进程不得释放父进程持有的锁；父进程释放后子副本不同步。

## 3. 异常终态

任一协调步骤失败 → 进入明确安全终态（DENIED 不可逆，TERMINATING 退出），不静默继续。

## 4. 证据等级
锁生命周期协议：DESIGN_PROVEN（逻辑）；真实 fork 行为：NOT_TESTED（平台无 fork）。
