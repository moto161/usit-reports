# FORK-LOCK-LIFECYCLE.md

## 目标
彻底闭合 Fork 锁生命周期，消除重复释放、漏释放、跨进程错误释放。

## 1. 核心原则（唯一权威）

1. **每把锁只有一个所有者**（owner）。
2. **每次获取对应唯一一个释放责任人**（release owner）。
3. 区分三种释放机制：
   - 自动释放（`with` 退出 / 上下文管理器 `__exit__`）。
   - 显式释放（明确 `release()`）。
   - 子进程清理继承状态（不是释放父锁，而是丢弃子副本）。
4. 父进程与子进程持有**独立锁实例**，互不释放对方锁。

## 2. 05.31 问题修复

05.31 的错误结构：
```python
with self._fork_coord_lock:      # 退出自动释放
    ...
    pid = os.fork()
if pid > 0:
    self._parent_path(pid)        # 内部再次释放 → 重复释放
```

修复：`_fork_coord_lock` 的获取与释放**只由协调函数本身负责一次**，父路径 `_parent_path()` 不触碰 coord 锁。

## 3. 锁获取/释放责任矩阵

| 锁 | 所有者 | 获取者 | 释放责任人 | 释放机制 | 父进程处理 | 子进程处理 |
|---|---|---|---|---|---|---|
| _fork_coord_lock | 协调器线程 | 协调器线程 | **协调器线程**（唯一） | with 自动 | 不释放（with 已处理） | 丢弃副本 + 重建 |
| _gate_state_lock | Gate 协调器 | Gate 写者 | Gate 写者 | with 自动 | 不释放 | 丢弃 + 重建 |
| _admit_lock | 业务线程 | 业务线程 | 业务线程 | with 自动 | 恢复 admit_closed（不释放子锁） | 丢弃 + 重建 |
| _commit_lock | 事务线程 | 事务线程 | 事务线程 | with 自动 | 不释放子锁 | 丢弃 + 重建 |
| _identity_alloc_lock | 身份线程 | 身份线程 | 身份线程 | with 自动 | 不释放子锁 | 丢弃 + 重建 |
| _receipt_write_lock | 事务线程 | 事务线程 | 事务线程 | with 自动 | 不释放子锁 | 丢弃 + 重建 |

**关键**：父进程在 `_parent_path()` 中**不释放任何子进程锁**；子进程在 `_child_path()` 中**不释放父进程锁**，而是丢弃继承副本并重建。

## 4. 完整伪代码（唯一释放责任）

```python
def _fork_coordinate(self):
    # coord 锁由本函数唯一获取/释放
    with self._fork_coord_lock:
        self._set_gate('FREEZING')       # 通过 _gate_state_lock 保护
        self._close_admit()              # 通过 _admit_lock 短持有置 admit_closed
        self._wait_biz_drain()           # 等待业务归零（条件变量）
        self._assert_no_commit()         # 通过 _commit_lock 短持有检查
        self._pre_fork_snapshot()
        pid = self._do_fork()            # fork 后父子各有独立锁副本
    # coord 锁在此释放，唯一释放点

    if pid < 0:
        self._fork_failed_path()          # 只回滚 Gate/admit_closed，不碰 coord
    elif pid == 0:
        self._child_path()                # 丢弃继承锁副本 + 重建
    else:
        self._parent_path(pid)            # 只恢复 Gate，不释放 coord（已释放）

def _parent_path(self, pid):
    # 不释放任何锁；coord 已在 with 退出释放
    self._restore_gate_after_success()   # 仅当 gate==RESTORING 才回 OPEN
    # 子进程相关锁由子进程自行重建，父不触碰

def _child_path(self):
    # 丢弃继承的锁副本（不调用 release，因为那是父的锁）
    self._discard_inherited_locks()
    self._rebuild_all_locks()            # 重建子专属锁
    self._rebuild_identity()
    self._set_gate('OPEN')
    self._open_admit()
```

## 5. Fork 各阶段锁状态

| 阶段 | coord | admit | commit | identity | receipt | 说明 |
|---|---|---|---|---|---|---|
| Fork 调用前 | 协调线程持有 | 已释放 | 已释放 | 已释放 | 可能事务持有 | 短持有检查后释放 |
| Fork 调用中 | 协调线程持有 | 未持 | 未持 | 未持 | 视事务 | fork 原子复制内存 |
| Fork 返回父 | 协调线程持有 | 未持 | 未持 | 未持 | 父副本 | with 退出释放 |
| Fork 返回子 | 子副本（父持有状态） | 子副本 | 子副本 | 子副本 | 子副本 | 全部丢弃重建 |

## 6. 异常/失败/信号中断清理

| 情况 | 清理 |
|---|---|
| Fork 失败（pid<0） | `_fork_failed_path()` 回滚 Gate/admit_closed，coord 已在 with 退出释放 |
| 父异常 | finally 保证 coord 释放（with），不触碰子锁 |
| 子异常 | 子丢弃继承锁，重建失败进入安全终态 |
| 信号中断 | 同异常处理，finally 释放 |
| 部分初始化失败 | 回滚已建资源，进入安全终态 |

## 7. 禁止项检查

- 重复释放：coord 只有 with 一个释放点 → 无。
- 漏释放：with 保证 → 无。
- 错误释放：父不释放子锁，子不释放父锁 → 无。
- 跨进程释放：父子锁实例独立 → 无。
- 锁永久占用：with + finally → 无。

## 8. 反例
20 条锁生命周期反例见 COUNTEREXAMPLES.json（B32-LOCK-001 ~ 020）。

## 9. 证据等级
锁生命周期协议：DESIGN_PROVEN（逻辑）；真实 fork 行为：NOT_TESTED。
