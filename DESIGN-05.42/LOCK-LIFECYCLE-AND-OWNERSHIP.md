# LOCK-LIFECYCLE-AND-OWNERSHIP.md — 锁生命周期与所有权（05.42 权威）

证据等级：VERIFIED_STATIC（单进程）+ NOT_PROVEN（跨进程/跨重启）
目标：修复 B41-R03——所有 txn_lock 获取/释放路径用 try/finally 保证释放，禁止提前返回遗留锁。

---

## 1. 05.41 缺陷定位

05.41 的 `admit()` / `release()` 在 `begin_txn()` 获取锁后，DENIED / GATE_CLOSED / NO_SLOT / INVALID_TOKEN / UNDERFLOW 等错误分支直接 `return`，未展示释放 `txn_lock`，导致一次普通拒绝就永久占住锁。

**根因**：锁释放只写在成功路径，未覆盖所有提前返回分支。

---

## 2. 唯一权威锁协议：try/finally

所有获取 `txn_lock` 的函数必须使用**事务上下文管理器**，`try/finally` 保证所有路径释放：

```python
def admit(owner_id, resource_id):
    with txn_guard():                 # ① 进入：获取 txn_lock
        # 所有校验与错误返回都在 finally 保护内
        if get_denied(state):
            return 'DENIED'           # ② 提前返回，finally 释放锁 ✓
        if not get_gate(state):
            return 'GATE_CLOSED'
        if get_count(state) >= COUNT_MAX:
            return 'OVERFLOW'
        slot = find_free_bit(active)
        if slot is None:
            return 'NO_SLOT'
        # ... 构造 WAL 记录并提交
        return run_txn(rec)           # ③ 正常返回，finally 释放锁 ✓
    # ④ finally 在此释放 txn_lock（无论 ②③ 哪条路径）
```

**关键**：`txn_guard()` 是上下文管理器，`__exit__` 无条件释放锁，覆盖所有 return 和异常。

---

## 3. 锁所有权规则

| 规则 | 要求 |
|---|---|
| 唯一所有者 | 同一时刻只有一个线程持有 txn_lock |
| 获取即释放 | 每个 `txn_guard()` 进入必配对一次释放 |
| 禁止重复释放 | 释放由 __exit__ 唯一执行，不重复 |
| 禁止释放他人锁 | 只释放本线程持有的锁（由 txn_guard 保证） |
| 异常后释放 | finally 覆盖异常路径 |

---

## 4. 所有函数锁路径表（逐条验证出口）

| 函数 | 提前返回分支 | 是否释放锁 |
|---|---|---|
| admit | DENIED / GATE_CLOSED / OVERFLOW / NO_SLOT / VERSION_EXHAUSTED / SLOT_GEN_EXHAUSTED | 是（finally） |
| release | INVALID_TOKEN / UNDERFLOW / VERSION_EXHAUSTED | 是（finally） |
| revoke | INVALID_TOKEN | 是（finally） |
| freeze | ALREADY_FROZEN / DENIED | 是（finally） |
| unfreeze | DENIED / NOT_FROZEN | 是（finally） |
| set_denied | —（幂等） | 是（finally） |
| recovery | DENIED / ISOLATED | 是（finally） |
| run_txn | ABORT / UNKNOWN / ERROR | 是（finally） |

**所有出口均释放锁**。禁止任何提前返回遗留锁。

---

## 5. CAS 失败重试与重试上限

- CAS 失败（或 WAL 冲突）→ 重新读完整状态（状态字 + TRT 快照），重新验证全部条件。
- 重试上限 16 次，耗尽 → CAS_LIVELOCK_DENY。
- 禁止外层无限 while 重试。
- 重试期间锁的获取/释放在 txn_guard 内，每次重试重新进入。

---

## 6. 锁重入 / 锁顺序 / 线程退出 / 进程崩溃

| 项 | 规则 |
|---|---|
| 锁重入 | txn_lock 非重入（避免死锁）；同一线程不得嵌套获取 |
| 锁顺序 | 单一 txn_lock，无多锁嵌套，无死锁 |
| 线程退出 | txn_guard 的 finally 在异常传播时释放（若线程被强制终止，锁由进程级清理处理） |
| 进程崩溃 | 锁随进程消失；跨进程锁需专门机制 → NOT_PROVEN → DENY |

---

## 7. 与 05.41 的差异

| 项 | 05.41 | 05.42 |
|---|---|---|
| 锁释放 | 仅成功路径（缺陷） | try/finally 全覆盖 |
| 提前返回 | 遗留锁 | 全部释放 |
| 锁路径表 | 无 | 逐函数审计 |

---

## 证据来源

- 提交协议：TRANSACTION-COMMIT-PROTOCOL.md
- 状态转换：STATE-INVARIANTS-AND-TRANSITIONS.md
- 证明等级：VERIFIED_STATIC（单进程）+ NOT_PROVEN（跨进程/跨重启）
