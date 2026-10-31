# RESOURCE-FAILURE-AND-RECOVERY.md — 资源端释放异常与恢复（05.43 权威）

工单：USIT-STAGE-B-DESIGN-05.43
工作包：B43-05（修复 B42-R05）
证据等级：VERIFIED_STATIC（单进程）+ NOT_PROVEN（跨进程/跨重启）
目标：把 05.42 只在文字里声称「ISOLATED」却未落到伪代码的资源释放异常隔离，落实为可执行的提交边界、崩溃矩阵和恢复入口。

---

## 1. B42-R05 缺陷（05.42 真实问题）

05.42 `RESOURCE-FENCING-PROTOCOL.md` 的实际顺序：

```
operation() → current_token += 1 → consumed_records 写入
```

问题：
1. `operation()` 可能**部分成功**后抛异常（副作用已发生，但状态未记录）。
2. `operation()` 成功后 `consumed_records` 写入失败（已释放但无幂等记录）。
3. 其他文档说「这种情况进入 ISOLATED」，但**伪代码没有完整的异常捕获、持久化状态标记和恢复入口**。
4. `operation` 与 `consumed_records` 不能原子提交，仅凭 `fence_lock` 宣称原子，是伪原子。

---

## 2. 真实提交边界（05.43 修订）

明确 `operation`、`current_token` 更新、`consumed_records` 写入三者的**真实提交边界**：

| 步骤 | 是否可回滚 | 持久化 | 崩溃后果 |
|---|---|---|---|
| 1. 校验 token + owner（授权） | 是（无副作用） | 无 | 无 |
| 2. 执行 `operation()` | **否**（业务副作用，可能非幂等） | 取决于业务 | 可能部分成功 |
| 3. `current_token += 1`（Fencing 单调推进） | 否 | 随 WAL | 已推进 |
| 4. 写 `consumed_records`（幂等记录） | 否 | 随 WAL | 已记录 |

**核心结论**：`operation`（业务副作用）与 `consumed_records`（幂等记录）**无法在同一原子事务内提交**——业务副作用是外部世界状态，无法用 WAL 回滚。因此：

- **能证明 operation 幂等** → 允许恢复时按幂等键重放。
- **不能证明 operation 幂等** → 进入 **ISOLATED**，禁止自动重放，必须人工介入。

---

## 3. 资源端释放伪代码（唯一权威，含异常隔离）

```python
def atomic_release(token, req_id, trusted_caller_owner, operation):
    # 步骤 1：授权（无副作用）
    if not validate_token(token, trusted_caller_owner, resource):
        return 'FENCED_OUT'                    # 或 OWNER_MISMATCH / STALE_TOKEN

    # 幂等查询：本请求已成功过？
    if req_id in consumed_records:
        return 'ALREADY_RELEASED'              # 幂等返回，不重复 operation

    # 步骤 2：执行 operation（可能非幂等）
    try:
        op_result = operation()                # 业务副作用
    except Exception:
        # operation 抛异常：副作用状态未知
        return isolate_release(token, req_id, 'OP_FAILED')

    # 步骤 3 + 4：Fencing 推进 + 幂等记录（随 WAL 事务）
    txn_result = run_txn(req_id, 'RESOURCE_RELEASE',
                         token, trusted_caller_owner,
                         op_record=op_result)
    if txn_result.status == 'OK':
        return 'OK'
    elif txn_result.status == 'UNKNOWN':
        return 'UNKNOWN'                       # 查询 txn_id
    elif txn_result.status == 'COMMITTED_PENDING_APPLY':
        return 'COMMITTED_PENDING_APPLY'
    else:
        return isolate_release(token, req_id, 'TXN_ABORT_AFTER_OP')


def isolate_release(token, req_id, reason):
    # 明确的 ISOLATED 入口：持久化隔离标记，禁止自动重放
    persist_isolation_record(token, req_id, reason)
    return 'ISOLATED'
```

**关键修正**：
1. **授权失败不执行 operation**（见 RESOURCE-OWNER-AUTHORIZATION.md）。
2. **幂等查询在 operation 之前**：已成功释放的请求重试，直接 `ALREADY_RELEASED`，不重复 operation。
3. **operation 之后、WAL 事务之前的异常** → `isolate_release` 持久化隔离标记，返回 `ISOLATED`。
4. **`run_txn` 内部**承担 `current_token += 1` + `consumed_records` 写入，二者在同一 WAL 事务内（见 TRANSACTION-PRECOMMIT-VALIDATION.md），**这一部分是真正原子的**。
5. **operation 本身与 WAL 事务不原子**，因此 operation 抛异常必须隔离，不自动重放。

---

## 4. 崩溃矩阵（资源释放视角）

| 崩溃点 | 恢复动作 |
|---|---|
| 授权校验通过后、operation 前 | 未执行 operation，无副作用；恢复后按 req_id 无记录 → 可重试 |
| operation 执行中 | 副作用部分发生；恢复后 operation 幂等性未知 → 若可证明幂等则重放，否则 ISOLATED |
| operation 成功后、WAL COMMIT 前 | 副作用已发生，无 consumed_records；恢复后查 WAL：无 COMMIT → 该 req_id 无成功记录 → 需判断 operation 是否幂等，幂等则重放，否则 ISOLATED |
| WAL COMMIT 后、APPLY 前 | 已提交待应用；恢复续 APPLY（见 WAL-RECOVERY-IDEMPOTENCY.md） |
| consumed_records 写入后 | 已成功；重试返回 ALREADY_RELEASED |

---

## 5. consumed_records 的持久化与清理

| 项 | 规则 |
|---|---|
| 唯一键 | `req_id`（业务请求幂等键） |
| 持久化 | 随 WAL 事务（COMMIT 记录）持久化 |
| 保存期限 | 无上限（至少在资源生命周期内保留），防止旧请求重放 |
| 清理规则 | 仅当资源被彻底退役且所有历史请求不可能再到达时，才允许归档清理；清理本身也须有审计记录 |
| 恢复 | 恢复时从 WAL 重放 consumed_records，重建幂等集合 |

---

## 6. 与 05.42 的差异

| 项 | 05.42 | 05.43 |
|---|---|---|
| 异常隔离 | 仅文字声称 ISOLATED | `isolate_release` 落到伪代码，持久化隔离标记 |
| 幂等查询时机 | 未明确 | operation 之前查询 req_id |
| operation 与记录原子性 | fence_lock 伪原子 | 明确 operation 不可回滚，WAL 内部分真原子 |
| 崩溃矩阵 | 未完整 | 五窗口逐项恢复动作 |

---

## 7. 证据来源与证明边界

- 授权：RESOURCE-OWNER-AUTHORIZATION.md
- 幂等键：POST-COMMIT-ERROR-AND-RETRY.md
- Fencing 串行化：RESOURCE-TAKEOVER-RELEASE-SERIALIZATION（05.42，保留）
- 崩溃矩阵：CRASH-AND-CONCURRENCY-MATRIX.md

**证明等级**：单进程授权 + 幂等查询 + WAL 内原子 → VERIFIED_STATIC；operation 幂等性、跨进程 Fencing、跨重启隔离记录 → NOT_PROVEN，维持 ISOLATED/DENY。
