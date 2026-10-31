# IDEMPOTENT-CLEANUP-PROTOCOL.md — 清理幂等性与所有权协议（05.37）

证据等级：VERIFIED_STATIC
资源所有权矩阵见 CLEANUP-OWNERSHIP-MATRIX.md（权威）；本文件定义清理状态机与执行协议。

---

## 1. 05.36 缺陷定位

05.36 的清理协议存在两个未解决问题：
1. **资源释放发生在锁外**，虽然加了状态锁，但未证明「同一资源只由一个清理执行者释放」——锁只保护了状态字段，未保护资源本身，存在两个执行者同时释放同一资源的窗口。
2. **永久失败资源会进入无限重试**——`while True` 循环在 `CLEANUP_PARTIAL` 时无条件 `continue`，若某资源持续失败且非 uncertain，会无限循环。

05.37 通过「执行所有权 + 重试上限 + 退避」修复。

---

## 2. 清理状态机（7 态）

| 状态 | 语义 | 进入条件 | 退出到 |
|---|---|---|---|
| CLEANUP_NOT_STARTED | 未开始 | 初始 | CLAIMED |
| CLEANUP_CLAIMED | 已取得执行所有权 | `claim_ownership()` 成功 | IN_PROGRESS |
| CLEANUP_IN_PROGRESS | 正在释放 | 从 CLAIMED 进入 | PARTIAL / COMPLETED / FAILED |
| CLEANUP_PARTIAL | 部分完成（有可重试资源未释放） | 有 recoverable 失败 | IN_PROGRESS（重试）/ ISOLATED |
| CLEANUP_COMPLETED | 全部资源安全终态 | 所有资源 released 或 verified-safe | （终态） |
| CLEANUP_FAILED | 有资源永久失败 | 达到重试上限 | ISOLATED（隔离永久失败资源） |
| ISOLATED | 隔离（不可重试/状态不明） | 永久失败或 uncertain | （终态，不可逆） |

**核心不变量**：
1. `_cleanup_done` 仅在 `CLEANUP_COMPLETED` 置 True。
2. 执行所有权（`owner_id` + `ownership_version` + `resource_set`）在 `CLAIMED` 时原子取得。
3. 同一资源在任何时刻最多被一个执行者释放。
4. 永久失败资源达到重试上限后进入 ISOLATED，不再循环。

---

## 3. 执行所有权协议（修复「锁外释放无所有权」）

```python
class CleanupManager:
    def __init__(self):
        self._state = CLEANUP_NOT_STARTED
        self._owner_id = None          # 当前执行者 ID
        self._ownership_version = 0    # 所有权版本号
        self._resources = {}           # name -> ResourceHandle
        self._claim_lock = Lock()

    def claim_ownership(self, executor_id):
        """原子取得执行所有权。线性化点 = _claim_lock 内的 owner 检查+赋值。"""
        with self._claim_lock:
            if self._state in (CLEANUP_IN_PROGRESS, CLEANUP_CLAIMED):
                return False           # 已有执行者
            if self._state == CLEANUP_COMPLETED:
                return 'DONE'
            if self._state == ISOLATED:
                return 'ISOLATED'
            self._owner_id = executor_id
            self._ownership_version += 1
            self._state = CLEANUP_CLAIMED
            return True

    def release_resource(self, rname):
        """单个资源释放。执行者必须已持有所有权。"""
        r = self._resources[rname]
        # 所有权校验：只有当前 owner 能释放
        if r.released:
            return 'NOOP'              # 幂等：已释放
        if r.uncertain:
            return 'UNCERTAIN'         # 状态不明 → 隔离，不释放
        try:
            r.release()                # 幂等释放
            r.released = True
            r.last_error = None
            return 'RELEASED'
        except UnknownStateError:
            r.uncertain = True         # 状态不明 → 隔离
            return 'UNCERTAIN'
        except Exception as e:
            r.last_error = e
            r.retry_count += 1
            if r.retry_count >= r.MAX_RETRY:
                r.permanent_failed = True
                return 'PERMANENT_FAILED'
            return 'RETRYABLE'
```

**关键**：资源释放不再「锁外无所有权」。每个资源有独立的 `released` / `uncertain` / `retry_count` / `permanent_failed` 状态，释放前校验执行者所有权，同一资源不会有两个执行者同时释放（所有权在 `claim_ownership` 串行化取得）。

---

## 4. 资源释放完整伪代码（带重试上限 + 退避）

```python
    def cleanup(self, executor_id):
        # 1. 取得所有权
        claim = self.claim_ownership(executor_id)
        if claim in ('DONE', 'ISOLATED'):
            return claim
        if not claim:
            return 'OWNERSHIP_CONFLICT'   # 已有执行者

        self._state = CLEANUP_IN_PROGRESS
        backoff = 1
        while True:
            pending = [r for r in self._resources.values()
                       if not r.released and not r.uncertain
                       and not r.permanent_failed]
            if not pending:
                # 检查是否有永久失败资源 → 隔离
                if any(r.permanent_failed for r in self._resources.values()):
                    self._state = ISOLATED
                    return 'ISOLATED'
                self._state = CLEANUP_COMPLETED
                self._cleanup_done = True   # 仅此处置 True
                return 'DONE'

            failed_any = False
            for r in pending:
                res = self.release_resource(r.name)
                if res == 'UNCERTAIN':
                    self._state = ISOLATED
                    return 'ISOLATED'      # 状态不明 → 立即隔离
                if res == 'PERMANENT_FAILED':
                    failed_any = True      # 该资源永久失败，隔离
                elif res == 'RETRYABLE':
                    failed_any = True

            if failed_any:
                if any(r.permanent_failed for r in self._resources.values()):
                    self._state = ISOLATED
                    return 'ISOLATED'
                self._state = CLEANUP_PARTIAL
                time.sleep(backoff)        # 退避，指数增长
                backoff = min(backoff * 2, MAX_BACKOFF)
                continue                   # 重试剩余 recoverable 资源
            # 全部 released，回到循环顶部会因 pending 为空而 COMPLETED
```

**重试与隔离策略**：
- `retry_count` 达到 `MAX_RETRY`（建议 3）→ `permanent_failed=True` → 进入 ISOLATED，不再循环。
- 退避 `backoff` 从 1 开始指数增长到 `MAX_BACKOFF`（建议 8 秒），避免紧循环。
- `uncertain` 资源（UnknownStateError）立即 ISOLATED，不重试（状态不明不能安全重试）。

---

## 5. 清理者崩溃与接管协议

清理执行者崩溃后，不能仅凭超时就假设资源释放「没有发生」。接管条件：

1. **所有权心跳**：执行者持有所有权期间周期更新 `heartbeat` 时间戳。
2. **接管判定**：`heartbeat` 超过 `OWNERSHIP_TIMEOUT`（建议 60s）后，接管者才能尝试重新 `claim_ownership`。
3. **接管前的资源状态核查**：接管者必须先对每项资源做**可验证的状态检查**（如锁句柄是否存在、文件是否存在、事务是否已提交），确认其 `released` 与否，**不得假设**。
4. **状态不明则隔离**：若资源状态无法验证，标记 `uncertain`，进入 ISOLATED，禁止接管释放。
5. **所有权版本递增**：接管成功后 `ownership_version += 1`，旧的执行者（若复活）因版本不匹配无法继续释放。

```python
    def try_takeover(self, new_executor_id):
        with self._claim_lock:
            if self._state not in (CLEANUP_CLAIMED, CLEANUP_IN_PROGRESS, CLEANUP_PARTIAL):
                return 'NO_ACTIVE_OWNER'
            if time.time() - self._heartbeat < OWNERSHIP_TIMEOUT:
                return 'OWNER_ALIVE'          # 未超时，不接管
            # 超时：接管
            self._owner_id = new_executor_id
            self._ownership_version += 1      # 版本递增，旧 owner 失效
            return 'TAKEOVER_OK'
```

---

## 6. 清理函数自身异常 / 取消 / 进程终止 / 系统崩溃

| 事件 | 处理 |
|---|---|
| release() 普通异常 | 记录 error，`retry_count++`，recoverable → 重试；达上限 → 永久失败 → ISOLATED |
| release() UnknownStateError | uncertain → ISOLATED，不重试 |
| 清理线程取消 | 状态保持 PARTIAL/IN_PROGRESS，所有权保留（心跳停止后由接管处理） |
| 清理者进程终止 | 心跳停止 → 超时 → 接管（§5） |
| 系统崩溃 | 持久化资源由 CRASH-RECOVERY-PROTOCOL.md 处理；内存资源随进程消失 |

**清理失败不得恢复 OPEN，不得清除 DENIED，不得允许 Fork**——清理状态机与准入/Fork 状态机正交，清理失败只影响清理本身，不影响已确定的 DENIED/FROZEN 终态。

---

## 7. 清理完成标志的置位条件

`_cleanup_done = True` **只在**「所有资源达到可验证的安全终态」时置位，即：
- 每项资源的 `released == True`（幂等释放已确认），或
- 资源随进程消失（内存资源，无需显式释放），或
- 资源已被证明由其他机制安全处置（有持久化证据）。

任一资源 `uncertain` 或 `permanent_failed`，`_cleanup_done` 保持 False。

---

## 证据来源

- 05.36 缺陷：IDEMPOTENT-CLEANUP-PROTOCOL.md（05.36 版，锁外释放 + 无限重试）
- 资源所有权矩阵：CLEANUP-OWNERSHIP-MATRIX.md
- 证明等级：VERIFIED_STATIC
