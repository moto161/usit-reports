# POST-COMMIT-ERROR-AND-RETRY.md — COMMIT 后异常与调用方重试（05.43 权威）

工单：USIT-STAGE-B-DESIGN-05.43
工作包：B43-03（修复 B42-R03）
证据等级：VERIFIED_STATIC（单进程）+ NOT_PROVEN（跨进程/跨重启）
目标：消除 05.42 中 `except Exception: return 'ERROR'` 在 COMMIT 已 fsync 后 APPLY 抛异常时诱导调用方盲目重试、造成重复 ADMIT/RELEASE 的缺陷。

---

## 1. B42-R03 缺陷（05.42 真实代码）

05.42 `TRANSACTION-COMMIT-PROTOCOL.md §2`：

```python
try:
    ...
    if not wal_append_fsync(rec, status='COMMIT'):
        return 'UNKNOWN'
    apply_to_state(rec)
    apply_to_trt(rec)
    wal_append(rec, status='APPLY')
    return 'OK'
except Exception:
    return 'ERROR'          # 缺陷：COMMIT 已成功后 APPLY 抛异常也返回 ERROR
```

**缺陷**：COMMIT 已 fsync 成功（事务已提交），随后 `apply_to_state` 抛异常。此时返回 `ERROR`，调用方会认为事务失败而重新提交同一逻辑操作，导致 COUNT +2、重复 ADMIT/RELEASE。调用方无法区分「真的失败」与「已提交但应用未完成」。

---

## 2. 事务结果状态机（唯一权威）

| 状态 | 含义 | 调用方动作 |
|---|---|---|
| `ABORT` | PREPARE 前/中失败，事务未提交 | 可安全重试（重新提交） |
| `UNKNOWN` | COMMIT fsync 结果不确定 | 必须查询 `txn_id` 结果，禁止盲目重试 |
| `COMMITTED_PENDING_APPLY` | COMMIT 成功，APPLY 未完成/失败 | 不重试，等待恢复流程续 APPLY |
| `OK` | 已提交且已应用 | 完成 |

**核心原则**：
1. **COMMIT 成功后的任何异常，绝不返回 `ERROR`**，而是 `COMMITTED_PENDING_APPLY`。
2. **COMMIT 结果不确定返回 `UNKNOWN`**，调用方必须先查询。
3. 只有 `ABORT` 才允许调用方安全重试。

---

## 3. 提交协议伪代码（区分四态）

```python
def run_txn(op, slot, owner_id, resource_id, caller_owner):
    txn_id = next_txn_id()
    with txn_guard():
        state_new, trt_new = build_candidate(...)
        if not invariants_hold(state_new, trt_new):
            return ('ABORT', txn_id, None)          # 未提交，可重试
        if version_exhausted(state_new):
            return ('ABORT', txn_id, 'VERSION_EXHAUSTED_DENY')

        rec = build_record(txn_id, ...)

        if not wal_append_fsync(rec, status='PREPARE'):
            return ('ABORT', txn_id, None)          # 未提交

        commit_ok = wal_append_fsync(rec, status='COMMIT')
        if not commit_ok:
            return ('UNKNOWN', txn_id, None)        # 提交不确定，禁止盲目重试

        # COMMIT 已成功，从此禁止返回 ERROR
        try:
            publish(state_new, trt_new)             # APPLY
        except Exception:
            mark_committed_pending_apply(txn_id)    # 持久化标记，供恢复续 APPLY
            return ('COMMITTED_PENDING_APPLY', txn_id, None)

        wal_append(rec, status='APPLY')
        return ('OK', txn_id, None)
```

**关键**：`try/except` 只包裹 COMMIT 之后的 APPLY 段，且异常分支返回 `COMMITTED_PENDING_APPLY` 而非 `ERROR`。

---

## 4. txn_id 查询接口

调用方在收到 `UNKNOWN` 或 `COMMITTED_PENDING_APPLY` 后，通过查询接口确定事务最终状态：

```python
def query_txn(txn_id):
    recs = scan_wal_for(txn_id)
    if any(r.status == 'COMMIT' for r in recs):
        if any(r.status == 'APPLY' for r in recs):
            return 'OK'                       # 已提交已应用
        return 'COMMITTED_PENDING_APPLY'      # 已提交待应用
    if any(r.status == 'PREPARE' for r in recs):
        return 'UNKNOWN'                      # 仅 PREPARE，提交不确定
    return 'NOT_FOUND'
```

**语义**：
- 存在 COMMIT + APPLY → 事务完成。
- 存在 COMMIT 无 APPLY → 已提交待应用，恢复流程会续 APPLY，调用方不应重试。
- 只有 PREPARE → 提交不确定，调用方等待或按恢复策略处理。

---

## 5. 重复调用的稳定幂等键

同一逻辑请求不得生成两笔有效事务。05.43 引入**请求幂等键**：

- `req_id`：由调用方提供的稳定业务请求 ID（例如 `owner_id + resource_id + 操作类型 + 业务序列`）。
- `run_txn` 入口先查 `committed_req_ids`（持久化集合），若该 `req_id` 已存在对应成功事务，直接返回该事务结果，不重复执行。
- `req_id → txn_id` 映射随 WAL 持久化（写入 COMMIT 记录），崩溃后仍可查询。

```python
def run_txn(req_id, op, slot, owner_id, resource_id, caller_owner):
    existing = lookup_req(req_id)
    if existing:
        return query_txn(existing.txn_id)     # 幂等返回，不重复执行
    ...
    # 成功提交后记录 req_id → txn_id
```

---

## 6. 异常路径覆盖矩阵

| 故障点 | 返回状态 | 调用方动作 |
|---|---|---|
| 不变式不满足 | ABORT | 可重试（业务层面修正后） |
| PREPARE fsync 失败 | ABORT | 可重试 |
| COMMIT fsync 失败（未落盘） | UNKNOWN | 查询 txn_id |
| COMMIT fsync 结果不确定（无法确认） | UNKNOWN | 查询 txn_id，等待恢复 |
| COMMIT 成功、APPLY 抛异常 | COMMITTED_PENDING_APPLY | 不重试，等恢复续 APPLY |
| COMMIT 成功、APPLY 成功、响应丢失 | OK（但调用方未收到） | 按 req_id 幂等查询 |
| 进程崩溃（任意点） | 恢复后按 WAL 判定 | 恢复流程处理 |

---

## 7. 与 05.42 的差异

| 项 | 05.42 | 05.43 |
|---|---|---|
| 异常返回 | `except: return 'ERROR'` | 按故障点返回 ABORT/UNKNOWN/COMMITTED_PENDING_APPLY |
| 四态区分 | 无 | ABORT/UNKNOWN/COMMITTED_PENDING_APPLY/OK |
| txn 查询接口 | 无 | `query_txn` |
| 幂等键 | 无 | `req_id → txn_id` 持久化映射 |
| COMMIT 后异常 | 诱导重试 → 重复提交 | COMMITTED_PENDING_APPLY，恢复续 APPLY |

---

## 8. 证据来源与证明边界

- 提交时序：TRANSACTION-PRECOMMIT-VALIDATION.md
- 恢复续 APPLY：WAL-RECOVERY-IDEMPOTENCY.md
- 崩溃矩阵：CRASH-AND-CONCURRENCY-MATRIX.md

**证明等级**：单进程、txn_lock 内四态区分 + req_id 幂等键 → VERIFIED_STATIC；跨进程 req_id 映射一致性、fsync 不确定性判定 → NOT_PROVEN，维持 DENY。
