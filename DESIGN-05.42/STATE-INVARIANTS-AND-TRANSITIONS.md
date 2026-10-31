# STATE-INVARIANTS-AND-TRANSITIONS.md — 状态不变式与转换（05.42 权威）

证据等级：VERIFIED_STATIC（单进程）+ NOT_PROVEN（跨进程/跨重启）
目标：修复 B41-R02——revoke 也减 COUNT，使 COUNT==popcount(TOKEN_ACTIVE) 严格成立，消除 revoke_pending 未定义存储。

---

## 1. 不变式（严格，revoke 也满足）

**不变式 I-1**：`COUNT == popcount(TOKEN_ACTIVE)`（**严格成立，任何成功提交后均满足**）。

**不变式 I-2**：`TOKEN_ACTIVE 每个置位 bit 对应 TRT 中一条 generation 匹配的活动记录`。

**不变式 I-3**：`DENIED 置位后 COUNT 不可再增加`。

**关键修正（B41-R02）**：05.41 的 revoke 清位不减 COUNT，主动破坏 I-1，靠 `revoke_pending` 事后补偿。05.42 改为** revoke 在同一事务内完成「COUNT−1 + 清位 + generation 失效」**，I-1 严格成立，无需 `revoke_pending` 这个未定义存储。

---

## 2. 统一操作定义（唯一权威）

| 操作 | 别名 | COUNT 变化 | 位图变化 | TRT generation | VERSION |
|---|---|---|---|---|---|
| admit | — | +1 | 置位 | 该槽位 +1 | +1 |
| slot_alloc | token_create（同实现） | +1 | 置位 | +1 | +1 |
| release | token_consume（同实现） | −1 | 清位 | +1 | +1 |
| token_revoke | — | **−1** | 清位 | +1 | +1 |
| freeze | — | 0 | 0 | 0 | +1 |
| unfreeze | — | 0 | 0 | 0 | +1 |
| set_denied | — | 0 | 0 | 0 | +1 |
| recovery | — | 按重放 | 按重放 | 按重放 | 按重放 |

**关键统一（B41-R02）**：`release = token_consume`，`slot_alloc = token_create`，均同一实现。`revoke` 与 `release` 的**唯一区别**是语义意图（撤销 vs 消费），但**字段变化完全相同**（都减 COUNT、清位、generation+1）。因此：

- revoke 与 release 在数据层面**等价**，都维护 I-1。
- 区别仅在于调用方意图与审计标记，不影响状态不变式。

---

## 3. 每次成功提交后自动验证不变式

WAL 的 APPLY 阶段，在更新状态字 + TRT 后，**自动**验证 I-1：

```python
def apply_and_verify(rec):
    apply_to_state(state, rec)
    apply_to_trt(trt, rec)
    # 自动验证不变式（不依赖调用方手动维护）
    if get_count(state) != popcount(get_token_active(state)):
        return 'INVARIANT_VIOLATION'   # 拒绝，回滚 APPLY（事务未提交）
    return 'OK'
```

**关键**：不变式验证在**提交点（COMMIT fsync）之前**完成（在 APPLY 前的校验阶段），若违反则事务不进入 COMMIT，直接 ABORT。这样「违反不变式的转换」根本不会被提交。

---

## 4. revoke 与 release 的并发交错

| 交错 | 结果 |
|---|---|
| revoke 与 release 并发同 slot | txn_lock 串行，一方先成功（清位+COUNT−1+gen+1），另一方重读 bit=0 → INVALID_TOKEN |
| revoke 与 cleanup 并发 | cleanup 只处理「已撤销但资源未释放」的资源，不重复减 COUNT |
| revoke 与 freeze 并发 | revoke 减 COUNT 后 I-1 仍成立，freeze 正常 |
| revoke 与 recovery 并发 | 重放 WAL 幂等，revoke 记录只 APPLY 一次 |

---

## 5. 与 05.41 的差异

| 项 | 05.41 | 05.42 |
|---|---|---|
| revoke 是否减 COUNT | 否（revoke_pending 补偿） | **是**（同事务减 COUNT） |
| revoke_pending 存储 | 未定义 | **删除** |
| I-1 是否严格 | 否（revoke 破坏） | 是（严格成立） |
| 不变式验证 | 依赖调用方 | APPLY 阶段自动验证 |

---

## 证据来源

- 数据模型：TRANSACTION-DATA-MODEL.md
- 提交协议：TRANSACTION-COMMIT-PROTOCOL.md
- revoke 补偿：REVOKE-AND-COMPENSATION-PROTOCOL.md
- 证明等级：VERIFIED_STATIC（单进程）+ NOT_PROVEN（跨进程/跨重启）
