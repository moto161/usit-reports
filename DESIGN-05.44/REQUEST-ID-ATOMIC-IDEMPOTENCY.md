# REQUEST-ID-ATOMIC-IDEMPOTENCY.md — 请求 ID 原子幂等（05.44 权威）

工单：USIT-STAGE-B-DESIGN-05.44
工作包：B44-02（修复 B43-R02）
证据等级：VERIFIED_STATIC（单进程）+ NOT_PROVEN（跨进程/跨重启）
目标：修复 05.42/05.43 中 req_id 唯一性检查在事务锁之外、并发重复请求仍可能重复提交的缺陷。

---

## 1. B43-R02 缺陷（05.43 真实代码）

05.43 `POST-COMMIT-ERROR-AND-RETRY.md §5`：

```python
def run_txn(req_id, op, slot, owner_id, resource_id, caller_owner):
    existing = lookup_req(req_id)      # <-- 锁外查询
    if existing:
        return query_txn(existing.txn_id)
    ...
```

**缺陷**：两个线程同时收到相同 req_id 时：

- A 在锁外 `lookup_req`：不存在。
- B 在锁外 `lookup_req`：不存在。
- A 在锁内提交事务，req_id→txn_id 持久化。
- B 随后在锁内，未重新查询，二次提交。

---

## 2. 唯一权威规则（05.44 修订）

### 2.1 req_id 唯一性检查必须在事务串行化边界内

req_id 查询、占用、PREPARE、COMMIT 全部在**同一事务锁内**完成。

### 2.2 req_id 唯一键范围与持久化

- 唯一键范围：`req_id`，全局唯一，不允许重复。
- 持久化：req_id→txn_id 映射随 WAL COMMIT 记录持久化，崩溃后仍可查询。
- 生命周期：资源退役前不清理；清理须审计。

---

## 3. 事务接口伪代码（唯一权威）

```python
def run_txn(req_id, op, slot, owner_id, resource_id, caller_owner):
    with txn_guard():                       # 唯一事务串行化权
        # 锁内查询 req_id（B44-02 核心修订）
        existing = lookup_req_in_lock(req_id)
        if existing:
            return ('OK', existing.txn_id, 'ALREADY_PROCESSED')   # 幂等返回

        # 无重复：继续正常流程（见 TXN-ORDER-AND-WAL-REPLAY.md）
        ...
```

**关键**：
1. `lookup_req_in_lock` 在 txn_lock 内查询 req_id。
2. 若存在，直接返回 `OK` + 原 txn_id + `ALREADY_PROCESSED`，**不重复执行状态变更**。
3. 若不存在，继续构造候选、PREPARE、COMMIT、APPLY，保证唯一性。

---

## 4. 并发重复请求的结果

| 场景 | 线程 A | 线程 B | 结果 |
|---|---|---|---|
| 同时查询 req_id（锁外） | 不存在 | 不存在 | 分别进入锁 |
| A 先进入锁，查询 req_id | 不存在，提交 | 等待 | A 完成 |
| B 后进入锁，查询 req_id | — | 存在，返回 ALREADY_PROCESSED | B 幂等返回 |

**结论**：同一 req_id 并发请求只能有一个事务提交，其余返回同一 txn_id + ALREADY_PROCESSED。

---

## 5. req_id 占用与崩溃恢复

| 崩溃点 | 恢复动作 |
|---|---|
| PREPARE 前崩溃 | req_id 未占用，可重试 |
| PREPARE 后、COMMIT 前崩溃 | 从 WAL 恢复，req_id 已占用但未提交 → ISOLATED 或重放 |
| COMMIT 后崩溃 | req_id 已提交，查询返回 OK |
| APPLY 记录缺失 | 恢复续 APPLY，req_id 已占用 |

---

## 6. req_id 冲突规则

| 情况 | 结果 |
|---|---|
| req_id 已存在且操作类型相同 | ALREADY_PROCESSED |
| req_id 已存在但操作类型不同 | CONFLICT_OP_TYPE → ISOLATED |
| req_id 已存在但 owner 不同 | CONFLICT_OWNER → ISOLATED |
| req_id 已存在但 resource 不同 | CONFLICT_RESOURCE → ISOLATED |
| req_id 已存在但 payload 不同 | CONFLICT_PAYLOAD → ISOLATED |

---

## 7. 与 05.43 的差异

| 项 | 05.43 | 05.44 |
|---|---|---|
| req_id 查询位置 | 锁外 | 锁内 |
| 并发重复请求 | 可能重复提交 | 弹簧上锁，唯一提交 |
| req_id 唯一性 | 文字声称，代码未实现 | 锁内查询 + 持久化 |
| 冲突规则 | 无 | CONFLICT_xxx → ISOLATED |

---

## 8. 证据来源与证明边界

- 事务顺序：TXN-ORDER-AND-WAL-REPLAY.md
- 恢复屏障：RECOVERY-BARRIER-AND-ADMISSION.md
- 崩溃矩阵：CRASH-AND-CONCURRENCY-MATRIX.md

**证明等级**：单进程持锁内 req_id 唯一性 → VERIFIED_STATIC；跨进程 req_id 唯一性 → NOT_PROVEN，维持 DENY。
