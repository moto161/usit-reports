# IDEMPOTENT-CLEANUP-PROTOCOL.md — 清理幂等与所有权协议（05.38 修订）

证据等级：VERIFIED_STATIC
Fencing 协议见 CLEANUP-FENCING-PROTOCOL.md（权威）；资源矩阵见 CLEANUP-OWNERSHIP-MATRIX.md。

---

## 1. 与 05.37 的关键差异

05.37 依赖 `ownership_version += 1` 使旧 owner 失效，但**无法阻止旧执行者已发起的在途释放**（B37-R01）。05.38 引入资源端 Fencing，作为跨进程释放安全的唯一可靠依据。

---

## 2. 清理状态机（7 态，保持不变）

| 状态 | 语义 |
|---|---|
| CLEANUP_NOT_STARTED | 未开始 |
| CLEANUP_CLAIMED | 已取得执行所有权 |
| CLEANUP_IN_PROGRESS | 正在释放 |
| CLEANUP_PARTIAL | 部分完成（可重试） |
| CLEANUP_COMPLETED | 全部资源安全终态 |
| CLEANUP_FAILED | 有资源永久失败 |
| ISOLATED | 隔离（不可重试/状态不明） |

---

## 3. 执行所有权（软件层，进程内）

```python
def claim_ownership(self, executor_id):
    with self._claim_lock:
        if self._state in (CLAIMED, IN_PROGRESS):
            return False
        if self._state in (COMPLETED, ISOLATED):
            return self._state
        self._owner_id = executor_id
        self._ownership_version += 1
        self._state = CLAIMED
        return True
```

软件层所有权只决定「谁可以发起释放」，**不决定「资源端是否接受」**。

---

## 4. 资源释放（含 Fencing，修复 B37-R01）

```python
def release_resource(self, rname, executor_id, fencing_token):
    r = self._resources[rname]

    # (1) 软件层所有权校验
    if r.owner_id != executor_id:
        return 'OWNERSHIP_LOST'

    # (2) 资源状态检查
    if r.released:
        return 'NOOP'              # 幂等
    if r.uncertain:
        return 'UNCERTAIN'         # 隔离

    # (3) 资源端 Fencing（关键修复）
    if not r.fencing_supported:
        # 无法安全释放（可能被接管者并发释放）
        r.uncertain = True
        return 'FENCING_UNSUPPORTED'   # → ISOLATED

    try:
        # 资源端校验 Token 并原子释放（无 TOCTOU）
        r.resource.release_with_token(fencing_token)
        r.released = True
        r.last_error = None
        return 'RELEASED'
    except FencingRejected:
        return 'FENCED_OUT'        # 旧 Token 被拒，旧动作无效（安全）
    except UnknownStateError:
        r.uncertain = True
        return 'UNCERTAIN'
    except Exception as e:
        r.last_error = e
        r.retry_count += 1
        if r.retry_count >= r.MAX_RETRY:
            r.permanent_failed = True
            return 'PERMANENT_FAILED'
        return 'RETRYABLE'
```

**关键**：`release_with_token(fencing_token)` 是资源端原子操作——资源端在释放时校验 Token，Token 不匹配则拒绝（FENCED_OUT）。这消除了「软件层校验 → 外部释放」之间的 TOCTOU 窗口，也确保旧执行者复活后无法并发释放。

---

## 5. 接管协议（修复 B37-R01）

```python
def try_takeover(self, new_executor_id):
    with self._claim_lock:
        if self._state not in (CLAIMED, IN_PROGRESS, PARTIAL):
            return 'NO_ACTIVE_OWNER'
        if time.time() - self._heartbeat < OWNERSHIP_TIMEOUT:
            return 'OWNER_ALIVE'
        # 心跳超时 → 仅触发「接管调查」，非「接管释放」
        return self._takeover_investigate(new_executor_id)

def _takeover_investigate(self, new_executor_id):
    # 对每项资源做只读状态调查，验证 released 与否
    for r in self._resources.values():
        if not r.released:
            if r.fencing_supported:
                # 可安全接管：换 Token + 版本递增
                r.fencing_token = new_token()
                r.owner_id = new_executor_id
                r.ownership_version += 1
            else:
                # 不支持 Fencing → 无法安全接管释放
                r.uncertain = True   # 只标记不确定，不释放
                return 'FENCING_UNSUPPORTED'
    self._owner_id = new_executor_id
    return 'TAKEOVER_OK'
```

**核心规则**：
- 心跳超时 → 只做**调查**，不直接释放。
- Fencing 支持的资源：换新 Token + 版本递增，可安全接管释放。
- Fencing 不支持的资源：**不释放**，标记 uncertain → ISOLATED。

---

## 6. 重试与隔离

- `retry_count` 达 `MAX_RETRY=3` → permanent_failed → ISOLATED。
- 退避 `backoff` 指数增长到 `MAX_BACKOFF=8s`。
- `uncertain`/`FENCING_UNSUPPORTED` → 立即 ISOLATED，不重试。

---

## 7. 清理完成标志

`_cleanup_done = True` 仅在**每项资源都通过资源端 Fencing 确认 `released` 或 `verified-safe`** 时置位。任一资源 `uncertain` 或 `permanent_failed` 或 `FENCING_UNSUPPORTED`，保持 False。

---

## 8. 清理失败不得恢复 OPEN / 清除 DENIED / 允许 Fork

清理状态机与准入/Fork/DENIED 状态机正交。清理失败只影响清理本身，不改变已确定的 DENIED/FROZEN 终态。

---

## 证据来源

- Fencing 协议：CLEANUP-FENCING-PROTOCOL.md
- 资源矩阵：CLEANUP-OWNERSHIP-MATRIX.md
- 证明等级：VERIFIED_STATIC
