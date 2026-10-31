# STATE-INVARIANTS-AND-TRANSITIONS.md — 状态不变式与转换（05.41 权威）

证据等级：VERIFIED_STATIC（单进程）+ NOT_PROVEN（跨进程/跨重启）
目标：修复 B40-R02——统一 admit/slot_alloc/token_create/release/token_consume/token_revoke/freeze/unfreeze/recovery 的定义，确立 COUNT 与 TOKEN_ACTIVE 的不变式。

---

## 1. COUNT 与 TOKEN_ACTIVE 数学不变式（权威）

**不变式 I-1**：`COUNT == popcount(TOKEN_ACTIVE)`（活动 Token 数 = 准入计数）。

**不变式 I-2**：`TOKEN_ACTIVE 的每个置位 bit（槽位）对应 TRT 中一条 generation 匹配的活动记录`。

**不变式 I-3**：`DENIED 置位后，COUNT 不可再增加（准入关闭）`。

每次成功状态转换后必须重新验证 I-1、I-2；违反不变式的转换必须拒绝，不得依赖后续恢复补救。

---

## 2. 统一操作定义（B41-02 权威，全文档唯一）

| 规范名 | 别名（同一实现） | 前置条件 | 字段变化 | 线性化点 | 失败返回 |
|---|---|---|---|---|---|
| admit | — | DENIED=0 且 GATE=OPEN 且 COUNT<MAX 且有空槽位 且 版本/代际未耗尽 | COUNT+1, 槽位置位, TRT写入, VERSION+1 | 状态字 CAS 成功 | DENIED / GATE_CLOSED / OVERFLOW / NO_SLOT / VERSION_EXHAUSTED / SLOT_GEN_EXHAUSTED |
| slot_alloc | token_create | 同上（分配槽位） | 同 admit | 状态字 CAS 成功 | 同 admit |
| release | token_consume | 槽位活动 且 gen 匹配 且 COUNT>0 | 槽位清位, COUNT−1, TRT.gen+1, VERSION+1 | 状态字 CAS 成功 | INVALID_TOKEN / UNDERFLOW / VERSION_EXHAUSTED |
| token_revoke | —（唯一，非 release 别名） | 槽位活动 且 gen 匹配 | 槽位清位, TRT.gen+1, VERSION+1（**COUNT 不变**） | 状态字 CAS 成功 | INVALID_TOKEN |
| freeze | — | GATE=OPEN 且 DENIED=0 | GATE=CLOSED, FREEZING=1, VERSION+1 | 状态字 CAS 成功 | ALREADY_FROZEN / DENIED |
| unfreeze | — | FROZEN=1 且 DENIED=0 且 COUNT=0 | FROZEN=0, GATE=OPEN, VERSION+1 | 状态字 CAS 成功 | DENIED / NOT_FROZEN |
| recovery | — | 证据齐全 且 DENIED 未并发置位 | 按证据重建, VERSION+1 | 状态字 CAS 成功 | DENIED / 证据不足 DENY |

---

## 3. revoke / consume / release 的 COUNT 规则（唯一明确）

| 操作 | 减 COUNT？ | 说明 |
|---|---|---|
| release（= token_consume） | **是** | 正常消费：COUNT−1 |
| token_revoke | **否** | 撤销身份不清 COUNT，COUNT 由 cleanup/recovery 处理 |

> 全文档唯一。禁止出现「revoke 减 COUNT」或「consume 不减 COUNT」的冲突定义。

---

## 4. 每个操作的字段变化与不变式验证

每个操作在 `commit_txn` 提交前，必须验证 I-1（`新 COUNT == popcount(新 位图)`）：

| 操作 | COUNT 变化 | 位图变化 | I-1 验证 |
|---|---|---|---|
| admit | +1 | 置 1 bit | 新 COUNT = 旧+1 = popcount(新位图) ✓ |
| release | −1 | 清 1 bit | 新 COUNT = 旧−1 = popcount(新位图) ✓ |
| revoke | **0** | 清 1 bit | **违反 I-1**（COUNT 不变但 popcount 减 1）→ 见 §5 |
| freeze/unfreeze | 0 | 0 | 不变 ✓ |

---

## 5. revoke 的不变式处理（关键设计决策）

`token_revoke` 清位图但不减 COUNT，会**临时违反 I-1**。处理方式（唯一）：

**revoke 后立即进入 ISOLATED 状态**，该槽位的 COUNT 差值由 cleanup 或 recovery 补偿。在补偿完成前，**冻结（freeze）DENY**（因为 COUNT 未归零）。

```python
def revoke(txn, token):
    if not (txn.state.token_active & (1 << token.slot)):
        return 'INVALID_TOKEN'
    if txn.trt[token.slot].generation != token.gen:
        return 'INVALID_TOKEN'
    txn.trt[token.slot].generation += 1
    txn.state.token_active &= ~(1 << token.slot)
    txn.state.version += 1
    # COUNT 不变，位图清 1 bit → I-1 临时违反
    # 标记 REVOKE_PENDING，等待 cleanup/recovery 补偿 COUNT
    txn.revoke_pending += 1
    r = commit_txn(txn)
    return r
```

**补偿规则**：`revoke_pending` 计数记录「已撤销但未减 COUNT 的槽位数」。cleanup/recovery 处理这些槽位，补偿 COUNT（COUNT −= revoke_pending），恢复 I-1。补偿完成前冻结 DENY。

> 这是**唯一**允许 I-1 临时违反的路径，且必须显式用 `revoke_pending` 追踪，不得静默。

---

## 6. 重试行为

| 操作 | 重试上限 | 重试耗尽 |
|---|---|---|
| admit/release/revoke/freeze/unfreeze/recovery | 16 | CAS_LIVELOCK_DENY，终止 |

所有 CAS 失败重读重算，禁止复用旧状态；禁止外层无限循环。

---

## 7. 与 05.40 的差异

| 项 | 05.40 | 05.41 |
|---|---|---|
| slot_alloc vs token_create | 未明确 | 同一实现 |
| release vs token_consume | 未明确 | 同一实现 |
| revoke 是否减 COUNT | 未明确 | 明确：不减，走 revoke_pending 补偿 |
| I-1 不变式 | 未形式化 | 形式化 + revoke 例外显式追踪 |

---

## 证据来源

- 状态模型：UNIFIED-STATE-MODEL.md
- 原子边界：ATOMIC-TRANSACTION-BOUNDARY.md
- Token 生命周期：TOKEN-OWNERSHIP-AND-LIFECYCLE.md
- 证明等级：VERIFIED_STATIC（单进程）+ NOT_PROVEN（跨进程/跨重启）
