# IDEMPOTENT-CLEANUP-PROTOCOL.md — 清理幂等性与部分失败恢复（05.36）

证据等级：VERIFIED_STATIC
目标：修复 05.35 的「`_cleanup_done` 在资源释放前即置 True，释放失败后第二次调用直接返回导致资源泄漏」问题。

---

## 1. 05.35 缺陷定位

05.35 存在如下反模式：

```python
if self._cleanup_done:
    return
self._cleanup_done = True              # ← 过早标记完成
self._release_owned_resources()        # ← 若此处抛异常，资源泄漏且不可重试
```

问题：`_cleanup_done` 在资源**实际清理完成前**置为 True。若 `_release_owned_resources()` 抛异常或部分资源释放失败，第二次调用会因 `_cleanup_done==True` 直接 return，导致残留资源永久泄漏，且无法进入隔离态。

---

## 2. 清理状态机（06 态）

| 状态 | 语义 | 可进入的后续状态 |
|---|---|---|
| CLEANUP_NOT_STARTED | 未开始 | IN_PROGRESS |
| CLEANUP_IN_PROGRESS | 进行中（持有清理所有权） | PARTIAL / COMPLETED / FAILED |
| CLEANUP_PARTIAL | 部分完成（有资源未释放） | IN_PROGRESS（重试剩余）/ FAILED / ISOLATED |
| CLEANUP_COMPLETED | 全部完成 | （终态） |
| CLEANUP_FAILED | 清理失败（可重试资源） | IN_PROGRESS（重试）/ ISOLATED |
| ISOLATED | 隔离（不可重试或状态不明资源） | （终态，不可逆） |

**关键不变量**：
1. `_cleanup_done` 只能在 `CLEANUP_COMPLETED` 时置 True。
2. 每项资源有独立所有权标识 + 独立释放状态，不共享单个布尔标记。
3. `CLEANUP_PARTIAL` / `CLEANUP_FAILED` 时 `_cleanup_done` 保持 False，允许重试。
4. 存在「状态不明」资源时，进入 ISOLATED，禁止重试（防止重复释放或破坏）。

---

## 3. 资源清理矩阵

| 资源 | 所有者 | 释放条件 | 幂等依据 | 失败动作 | 最终状态 |
|---|---|---|---|---|---|
| _fork_coord_lock | Fork 协调者 | Fork 完成或失败后 | 锁句柄释放天然幂等（已释放则 no-op） | 记录失败，不阻塞其他资源 | COMPLETED/FAILED |
| _admit_state（进程内） | 进程自身 | 进程退出 | 随进程消失，无显式释放 | — | COMPLETED |
| 子进程继承锁副本 | 子进程 | 子进程初始化清理 | 副本清理幂等 | 失败 → 子进程 ISOLATED | COMPLETED/ISOLATED |
| Gate 状态 | 冻结者 | 恢复 OPEN 前 | 状态字段幂等写入 | 失败 → 保持 CLOSED | COMPLETED/FAILED |
| 持久化事务句柄 | 事务所有者 | 事务终态后 | 句柄 close 幂等 | 失败 → ISOLATED（状态不明） | COMPLETED/ISOLATED |
| 临时文件 / 快照 | 快照者 | 快照提交后 | unlink 幂等（不存在则 no-op） | 失败 → 保留并记录 | COMPLETED/FAILED |

**规则**：某项资源释放失败，不得阻止其他**独立安全**清理操作继续，但必须保留失败记录。仅当资源「状态不明」（无法确定是否已释放/是否已提交）时，才进入 ISOLATED 并停止重试。

---

## 4. 修正后的清理伪代码

```python
class CleanupManager:
    def __init__(self):
        self._state = CLEANUP_NOT_STARTED
        self._resources = {}   # name -> {owner, released: bool, uncertain: bool}

    def cleanup(self):
        while True:
            if self._state == CLEANUP_COMPLETED:
                return 'DONE'
            if self._state == ISOLATED:
                return 'ISOLATED'
            # 单线程所有权：并发调用通过锁串行化
            with self._cleanup_lock:
                if self._state == CLEANUP_COMPLETED:
                    return 'DONE'
                if self._state == ISOLATED:
                    return 'ISOLATED'
                self._state = CLEANUP_IN_PROGRESS
                # 只处理尚未完成且非 uncertain 的资源
                pending = [r for r in self._resources.values()
                           if not r.released and not r.uncertain]
                if not pending:
                    self._state = CLEANUP_COMPLETED
                    self._cleanup_done = True   # 仅此处置 True
                    return 'DONE'
            # 逐个释放（在锁外执行，避免长临界区）
            failed_any = False
            for r in pending:
                try:
                    if not r.released:
                        r.release()          # 幂等释放
                        r.released = True
                except UnknownStateError:
                    r.uncertain = True       # 状态不明 → 隔离
                    self._enter_isolated()
                    return 'ISOLATED'
                except Exception as e:
                    r.last_error = e
                    failed_any = True        # 保留失败记录，继续其他资源
            # 锁内更新状态
            with self._cleanup_lock:
                if failed_any:
                    self._state = CLEANUP_PARTIAL   # 允许重试剩余
                else:
                    self._state = CLEANUP_COMPLETED
                    self._cleanup_done = True
            if self._state == CLEANUP_PARTIAL:
                continue   # 重试尚未 released 的资源

    def _enter_isolated(self):
        with self._cleanup_lock:
            self._state = ISOLATED
            self._cleanup_done = False   # 隔离态绝不标记完成
```

**要点**：
- `_cleanup_done = True` 只在 `CLEANUP_COMPLETED` 分支出现，且仅当 `pending` 为空（所有非 uncertain 资源已 released）。
- 部分失败（`CLEANUP_PARTIAL`）时继续循环重试剩余资源，不提前标记完成。
- `UnknownStateError`（状态不明）→ 立即 ISOLATED，停止重试，防止重复释放。
- 清理幂等：`r.release()` 对已释放资源必须 no-op，重复调用安全。

---

## 5. 并发清理安全

- 多个线程并发调用 `cleanup()`：通过 `_cleanup_lock` 串行化，同一时刻只有一个线程处于 IN_PROGRESS。
- 所有权唯一：每项资源在矩阵中标注唯一 owner，release 只由 owner 执行。
- 清理不得清除或覆盖 DENIED：清理逻辑不触碰状态字 DENIED 位；DENIED 的不可逆性由 STATE-TRANSITION-MATRIX.md §3 保证，与清理正交。

---

## 6. 清理函数自身异常 / 超时 / 取消 / 进程退出

| 事件 | 处理 |
|---|---|
| release() 抛普通异常 | 记录 `last_error`，`failed_any=True`，继续其他资源，最终 CLEANUP_PARTIAL |
| release() 抛 UnknownStateError | 该资源 uncertain，进入 ISOLATED，停止 |
| 清理超时 | 中断当前 release，进入 CLEANUP_FAILED，保留失败记录，可安全重试资源标记为 released 的继续 |
| 清理被取消 | 同超时，进入 CLEANUP_FAILED |
| 进程退出 | 进程内资源随进程消失；持久化资源由 CRASH-RECOVERY-PROTOCOL.md 处理 |

---

## 证据来源

- 05.35 缺陷反模式：05.35 工单引用的伪代码片段
- 清理状态机：本文件 §2
- 证明等级：VERIFIED_STATIC
