# LOCK-OWNERSHIP-AND-CALL-GRAPH.md — 锁唯一所有权与调用图（05.43 权威）

工单：USIT-STAGE-B-DESIGN-05.43
工作包：B43-04（修复 B42-R04）
证据等级：VERIFIED_STATIC（单进程）+ NOT_PROVEN（跨进程/跨重启）
目标：消除 05.42 中「admit 用 `with txn_guard()` 持锁后又调 `run_txn`，而 run_txn 内部又通过上下文管理器再次获取非重入 txn_lock 导致自锁」的歧义。

---

## 1. B42-R04 缺陷（05.42 真实自锁）

05.42 存在两处矛盾的锁获取：

- `TOKEN-ALLOCATION-RELEASE-REUSE.md`：`with txn_guard(): ... return run_txn(rec)` —— 上层持锁。
- `TRANSACTION-COMMIT-PROTOCOL.md §4`：`run_txn` 内部通过事务上下文管理器持有锁。
- `LOCK-LIFECYCLE-AND-OWNERSHIP.md`：txn_lock 是**非重入锁**。

**后果**：若上层 `admit` 已持 txn_lock，再调 `run_txn`，而 run_txn 内部再次 `with txn_guard()` 获取同一个非重入锁，直接**自锁死**（死锁）。

---

## 2. 唯一所有权规则（05.43 修订）

**唯一持锁层级 = `run_txn`（最内层事务执行函数）。上层 `admit` / `release` / `revoke` / `freeze` / `unfreeze` / `set_denied` / `recovery` 一律不持有 txn_lock。**

规则：
1. txn_lock 只在 `run_txn` 内部获取和释放，全代码库唯一。
2. 上层入口函数只做参数校验、req_id 幂等查询、结果映射，不碰锁。
3. 禁止任何函数在持有 txn_lock 的情况下调用另一个也会获取 txn_lock 的函数（无嵌套持锁）。
4. 恢复流程使用**独立的恢复锁**，与 txn_lock 分离，恢复期间阻塞普通事务入口（见 §6）。

---

## 3. 调用链（唯一权威）

```
admit(owner_id, resource_id, req_id)
  └─ run_txn(req_id, 'ADMIT', ..., caller_owner)      ← 唯一持 txn_lock 处
       ├─ build_candidate()
       ├─ invariants_hold()
       ├─ wal_append_fsync(PREPARE)
       ├─ wal_append_fsync(COMMIT)
       ├─ publish()
       └─ wal_append(APPLY)

release(token, req_id)
  └─ run_txn(req_id, 'RELEASE', ..., caller_owner)    ← 唯一持 txn_lock 处

revoke(token, req_id)
  └─ run_txn(req_id, 'REVOKE', ..., caller_owner)

freeze() / unfreeze() / set_denied()
  └─ run_txn(req_id, op, ...)

recover()
  └─ recovery_guard()                                 ← 独立恢复锁，非 txn_lock
       ├─ group_by_txn_id + 幂等重放
       └─ publish
```

---

## 4. 锁获取/释放路径表（逐条审计所有出口）

| 函数 | 持锁点 | 释放点 | 出口清单 |
|---|---|---|---|
| `run_txn` | `with txn_guard():` | `with` 语句退出（try/finally 语义） | ABORT / UNKNOWN / COMMITTED_PENDING_APPLY / OK / 异常 → 全部经 finally 释放 |
| `admit` | 不持锁 | — | 只透传 run_txn 结果 |
| `release` | 不持锁 | — | 同上 |
| `revoke` | 不持锁 | — | 同上 |
| `freeze`/`unfreeze`/`set_denied` | 不持锁 | — | 同上 |
| `recover` | `with recovery_guard():` | `with` 退出 | OK / ISOLATED → finally 释放 |

---

## 5. 锁实现（try/finally 保证所有路径释放）

```python
import threading

txn_lock = threading.Lock()          # 非重入
recovery_lock = threading.Lock()     # 独立恢复锁

def txn_guard():
    # 用 try/finally 语义：任何 return / 异常都释放锁
    class _Guard:
        def __enter__(self):
            txn_lock.acquire()
            return self
        def __exit__(self, *exc):
            txn_lock.release()
            return False             # 不吞异常
    return _Guard()

def recovery_guard():
    class _Guard:
        def __enter__(self):
            recovery_lock.acquire()
            # 阻塞普通事务：置全局恢复标志
            global in_recovery
            in_recovery = True
            return self
        def __exit__(self, *exc):
            in_recovery = False
            recovery_lock.release()
            return False
    return _Guard()

# 普通事务入口检查恢复标志
def run_txn(...):
    if in_recovery:
        raise RecoveryInProgress        # 恢复期间拒绝新事务
    with txn_guard():
        ...
```

**关键**：`__exit__` 无论 `return` 还是异常都会执行 `release()`，且 `return False` 不吞异常，保证异常仍向上传播且锁已释放。**不存在提前返回遗留锁的路径**。

---

## 6. 恢复流程与普通事务的隔离

- 恢复使用 `recovery_lock` + 全局 `in_recovery` 标志。
- 恢复开始置 `in_recovery=True`，普通 `run_txn` 入口检查该标志，恢复期间直接拒绝（抛 `RecoveryInProgress`），不进入 txn_lock。
- 恢复完成清标志、释放 recovery_lock。
- 因此恢复与普通事务**不会并发改写状态**，`applied_txn_ids` 集合无并发污染（见 WAL-RECOVERY-IDEMPOTENCY.md §6）。

---

## 7. 锁重入、顺序、线程退出、进程崩溃

| 项 | 规则 |
|---|---|
| 重入 | txn_lock 非重入，禁止嵌套获取；任何嵌套即死锁，代码审查拒绝 |
| 锁顺序 | 单锁（txn_lock）+ 单恢复锁，无双锁顺序问题；恢复锁与 txn_lock 不同时持有 |
| 线程退出 | 线程持锁中崩溃 → 进程内锁可能遗留 → 跨线程未证明 → NOT_PROVEN |
| 进程崩溃 | 进程内锁自动消失，但 WAL 未刷盘部分 → 由恢复流程处理（NOT_PROVEN 跨重启） |

**结论**：单进程内锁生命周期 VERIFIED_STATIC；跨进程锁、锁继承、崩溃后锁恢复 → NOT_PROVEN，维持 DENY。

---

## 8. 与 05.42 的差异

| 项 | 05.42 | 05.43 |
|---|---|---|
| 持锁层级 | 上层 admit + run_txn 均持锁（矛盾） | 仅 run_txn 持锁 |
| 自锁风险 | 非重入锁嵌套自锁 | 消除（上层不持锁） |
| 恢复锁 | 未分离 | 独立 recovery_lock + in_recovery 标志 |
| 锁释放保证 | 文字声称，代码有提前 return 漏洞 | try/finally 语义，逐出口审计 |

---

## 9. 证据来源与证明边界

- 事务接口：TRANSACTION-PRECOMMIT-VALIDATION.md
- 恢复流程：WAL-RECOVERY-IDEMPOTENCY.md
- 崩溃矩阵：CRASH-AND-CONCURRENCY-MATRIX.md

**证明等级**：单进程锁所有权与出口审计 → VERIFIED_STATIC；跨进程/崩溃后锁状态 → NOT_PROVEN。
