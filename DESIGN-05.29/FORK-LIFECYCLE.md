# USIT-STAGE-B-DESIGN-05.29 — Fork 完整生命周期（FORK-LIFECYCLE.md）

> B29-01 支撑文档。重建 Fork 生命周期，消除 05.28 的 in_flight 自锁（F01）。

## 1. 生命周期总览

```
REQUESTED → ADMISSION_CHECK → FREEZING → DRAINING → FROZEN
         → FORK_EXECUTION → PARENT_RESTORE / CHILD_REINIT → FINALIZED
失败分支：任意阶段 → FAILURE_CLASSIFICATION → DENIED / TERMINATING / RECOVERY_REQUIRED
```

## 2. 阶段定义

| 阶段 | 进入条件 | 退出条件 | 关键动作 | 业务准入 | fork 准入 |
|------|---------|---------|---------|---------|----------|
| REQUESTED | fork 调用到达 | 准入检查完成 | 置 `_fork_requested=True` | 仍 OPEN，允许 | 已请求 |
| ADMISSION_CHECK | 请求到达 | 线程安全证明通过 | 检查 Gate=OPEN + 线程安全 | 允许 | 检查中 |
| FREEZING | 准入通过 | 开始排空 | Gate=FREEZING，停新业务 | **禁止** | 冻结中 |
| DRAINING | FREEZING | 业务 in_flight==0 | 等待业务归零 | 禁止 | 冻结中 |
| FROZEN | 排空完成 | os.fork() 调用 | 生成快照 | 禁止 | 冻结完成 |
| FORK_EXECUTION | FROZEN | fork 返回 | os.fork() | 禁止 | 执行中 |
| PARENT_RESTORE | 父 fork 返回 | Gate 恢复/失败 | 恢复 Gate | 禁止直到恢复 | 恢复中 |
| CHILD_REINIT | 子 fork 返回 | 初始化完成/退出 | 重建锁+身份 | 禁止直到重建 | — |
| FINALIZED | 恢复/重建完成 | — | Gate=OPEN 或终态 | 恢复后允许 | 可再 fork |

## 3. 自锁消除（F01）

### 3.1 05.28 自锁路径

```
fork_entry → enter_op() → _in_flight += 1   (Fork 自身计入)
         → freeze → 等待 _in_flight == 0    (但 Fork 自己未 exit)
         → 死锁
```

### 3.2 消除方案：业务计数域与 Fork 协调分离

```python
_biz_in_flight = 0           # 只统计业务操作（FD/事务/快照读），Fork 不计入
_fork_in_progress = False    # Fork 协调独立标志
```

Fork 发起者**不调用** `enter_op()`，不登记 `_biz_in_flight`。`freeze()` 等待的是**业务** `_biz_in_flight` 归零，而 Fork 自身不在其中，故无自锁。

### 3.3 证明

1. Fork 进入 FREEZING 时，`_biz_in_flight` 只含真正业务操作。
2. Fork 发起者自身不贡献 `_biz_in_flight`。
3. `_biz_in_flight` 归零仅取决于业务操作退出，不依赖 Fork 退出。
4. 因此 Fork 等待 `_biz_in_flight==0` 不会等待自身。
5. **结论：自锁结构上不存在。**

## 4. 业务准入（enter_op 上下文管理器）

```python
def enter_op():
    with _admit_lock:
        if _gate != 'OPEN':
            raise GateClosed(_gate)
        _biz_in_flight += 1
    return _OpGuard()

class _OpGuard:
    def __enter__(self):
        return self
    def __exit__(self, exc_type, exc, tb):
        with _admit_lock:
            _biz_in_flight -= 1
            if _gate == 'FREEZING' and _biz_in_flight == 0:
                _admit_cond.notify_all()
        return False   # 不吞异常
```

- 进入：`with enter_op():` → 登记。
- 正常退出：`__exit__(None,None,None)` → 递减。
- 异常退出：`__exit__(exc_type,exc,tb)` → 递减 → 返回 False（异常继续传播）。
- 嵌套：每层独立登记/递减（计数 +N）。

## 5. Fork 协调（freeze，不登记业务）

```python
def freeze():
    with _admit_lock:
        if _gate != 'OPEN': raise GateClosed(_gate)
        _gate = 'FREEZING'
        _fork_in_progress = True
        while _biz_in_flight > 0:
            _admit_cond.wait(timeout=FREEZE_TIMEOUT)
            if _biz_in_flight > 0 and timed_out():
                _gate = 'OPEN'; _fork_in_progress = False
                raise FreezeTimeout()
        _gate = 'FROZEN'
```

## 6. 父子进程独立路径

- **PARENT_RESTORE**：父进程 fork 返回后，走 `_close_fork_parent()`（先 DENIED 后尝试恢复）。
- **CHILD_REINIT**：子进程 fork 返回后，走 `_child_reinit()`（重建锁对象 + 重建 instance_id + 初始化标志），失败 → os._exit(1)。

两条路径**独立**，不共享锁状态与运行时资源。

## 7. 失败分支统一规则

- FREEZING/DRAINING 失败 → 回滚 Gate 到 OPEN（可逆，因尚未 fork）。
- FORK_EXECUTION 后父失败 → DENIED（不可逆，见主文档 §1.6）。
- FORK_EXECUTION 后子失败 → os._exit(1)。
- 锁释放失败 → TERMINATING。
- 无法判定 → RECOVERY_REQUIRED（人工）。

## 8. 结论

生命周期完整覆盖 12 个阶段，业务与 Fork 计数域分离，自锁结构消除。协议层 DESIGN_PROVEN，运行时 NOT_TESTED。
