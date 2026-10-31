# FORK-FREEZE-PROTOCOL.md

## 目标
重构 Fork 前冻结流程，coord 不与任何业务锁嵌套。

## 1. 核心原则

1. `_fork_coord_lock` 只用于**串行化冻结发起者**（同一时刻只有一个冻结者）。
2. coord 作用域内**不获取** `_admit_lock` / `_commit_lock` / Receipt 锁 / `_gate_state_lock`，也**不间接调用**任何会获取这些锁的函数（禁止通过封装/回调/上下文管理器隐藏嵌套）。
3. Gate 与准入计数合并为**单一原子状态字** `_admit_state`（原子 CAS 更新），消除分离窗口。
4. 冻结操作唯一所有者 = 冻结协调器；明确状态机、超时、失败回滚。
5. 冻结失败/超时/异常/Fork 失败 → 确定安全状态。

## 2. 原子状态字设计

```
_admit_state 编码：
  bit0        : Gate（0=OPEN, 1=FREEZING）
  bit1..15    : 准入计数 admit_count（进行中的业务请求数）
  bit16       : 冻结中标志 frozen
```

准入线性化点：`CAS(_admit_state, gate==OPEN ? state+1 : FAIL)`。
冻结线性化点：`CAS(_admit_state, gate==OPEN ? (set FREEZING bit) : FAIL)`。

> 准入与冻结共享同一原子字，CAS 语义保证：冻结置位 FREEZING 成功后，后续任何准入 CAS（前置 gate==OPEN）必然失败。

## 3. 完整伪代码（每步注明锁/状态/失败处理/责任主体）

```python
# 责任主体：冻结协调器（唯一冻结者）
def _fork_freeze(self):
    # [步骤1] FREEZING 申请
    #   持有锁：_fork_coord_lock（串行化冻结者）
    #   未持有：任何业务锁
    #   状态：尝试原子置 FREEZING
    with self._fork_coord_lock:
        if not self._admit_state.cas_freeze():     # 线性化点
            raise FreezeRejected('gate not open or already freezing')

    # [步骤2] 关闭新准入（已由 CAS 原子完成，无需 admit_lock）
    #   Gate 已置 FREEZING，新准入 CAS 必然失败

    # [步骤3] 等待已有业务退出（不持有任何业务锁）
    #   等待条件：admit_count == 0（条件变量，wait 前不持锁）
    self._wait_admit_drain(timeout=self._freeze_timeout)

    # [步骤4] 确认提交归零（独立检查，不持有 commit_lock 嵌套）
    self._assert_commit_idle()          # 读原子提交计数，不获取锁

    # [步骤5] 确认 Receipt 写入归零
    self._assert_receipt_idle()         # 读原子 receipt 计数，不获取锁

    # [步骤6] 快照（在冻结态下，业务已归零）
    self._take_snapshot()

    # [步骤7] Fork
    pid = self._do_fork()

    # [步骤8] 父子分支
    if pid < 0:
        self._fork_failed_path()         # 回滚冻结，恢复 OPEN
    elif pid == 0:
        self._child_path()
    else:
        self._parent_path(pid)

def _fork_failed_path(self):
    # 责任主体：冻结协调器
    # 持有锁：无（coord 已释放）
    self._admit_state.cas_unfreeze()     # 原子回滚 FREEZING -> OPEN
    self._record_error()

def _parent_path(self, pid):
    # 不释放 coord（with 已处理），不重建子资源
    self._admit_state.cas_unfreeze()     # 恢复 OPEN

def _child_path(self):
    # 丢弃继承的锁/原子状态副本，重建子进程专属状态
    self._discard_inherited()
    self._rebuild_locks()
    self._rebuild_identity()
    self._admit_state = AdmitState.OPEN  # 子进程独立初始
```

## 4. 冻结状态机

```
OPEN ──(CAS freeze 成功)──► FREEZING ──(admit_count==0)──► FROZEN
OPEN ──(CAS freeze 失败)──► 拒绝（保持 OPEN）
FREEZING ──(超时)──► FREEZE_TIMEOUT ──(CAS unfreeze)──► OPEN
FROZEN ──(fork 成功)──► 父子分支恢复
FROZEN ──(fork 失败)──► CAS unfreeze → OPEN
任意 ──(异常/中断)──► 安全终态（回滚或 DENIED）
```

## 5. 锁状态时间线

| 阶段 | coord | admit | commit | receipt | gate_state | 说明 |
|---|---|---|---|---|---|---|
| 冻结申请 | 持（串行化） | 未持 | 未持 | 未持 | 未持（用原子字） | CAS freeze |
| 关闭新准入 | 已释放 | 未持 | 未持 | 未持 | 未持 | 原子完成 |
| 等待业务归零 | 未持 | 未持 | 未持 | 未持 | 未持 | 条件变量 wait |
| 确认提交/回执归零 | 未持 | 未持 | 未持 | 未持 | 未持 | 读原子计数 |
| 快照 | 未持 | 未持 | 未持 | 未持 | 未持 | 冻结态下业务归零 |
| Fork | 未持 | 未持 | 未持 | 未持 | 未持 | fork 原子复制 |

**结论：coord 作用域内从不获取业务锁；等待业务归零不持有任何锁。**

## 6. 不变量

- I-FF-1：coord 与任何业务锁不同时持有。
- I-FF-2：冻结 CAS 成功后新准入 CAS 必失败。
- I-FF-3：快照仅在 admit_count==0 且 commit_idle 且 receipt_idle 时生成。
- I-FF-4：冻结失败/超时/Fork 失败原子回滚到 OPEN。
- I-FF-5：父不释放子锁，子不释放父锁。

## 7. 证据等级
冻结协议 DESIGN_PROVEN（逻辑）；真实 fork 行为 NOT_TESTED。
