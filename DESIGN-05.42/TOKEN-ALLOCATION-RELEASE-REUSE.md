# TOKEN-ALLOCATION-RELEASE-REUSE.md — Token 分配/释放/复用（05.42 权威）

证据等级：VERIFIED_STATIC（单进程）+ NOT_PROVEN（跨进程/跨重启）
目标：修复 B41-R07——统一 release/token_consume/slot_free 的唯一实现，统一 generation 递增时机。

---

## 1. 05.41 缺陷定位

05.41 的 `slot_free()`（TOKEN-ALLOCATION-AND-REUSE.md）不递增 generation，而 `release()`（TOKEN-ATOMIC-TRANSACTION.md）递增。同一释放操作两个实现，generation 规则不一致。

**根因**：文档拆成多份，各自写了不同的释放伪代码，没有唯一权威实现。

---

## 2. 唯一权威操作实现

### admit（分配）

```python
def admit(owner_id, resource_id):
    with txn_guard():
        if get_denied(state): return 'DENIED'
        if not get_gate(state): return 'GATE_CLOSED'
        if get_count(state) >= COUNT_MAX: return 'OVERFLOW'
        if version_exhausted(): return 'VERSION_EXHAUSTED_DENY'
        slot = find_free_bit(active)
        if slot is None: return 'NO_SLOT'
        if trt[slot].generation == 0xFFFF: return 'SLOT_GENERATION_EXHAUSTED_DENY'
        rec = build_record(op='ADMIT', slot=slot,
                           old_gen=trt[slot].generation,
                           new_gen=trt[slot].generation + 1,
                           owner_id=owner_id, resource_id=resource_id,
                           count_delta=+1, active_bit=+1, version_delta=+1)
        return run_txn(rec)   # PREPARE → COMMIT → APPLY
```

### release（= token_consume = slot_free，唯一实现）

```python
def release(token, caller_owner, caller_resource):   # token = (slot, gen)
    with txn_guard():
        v = validate_token(token, caller_owner, caller_resource)   # 含 owner/resource 校验
        if v != 'OK': return v
        if get_count(state) == 0: return 'UNDERFLOW'
        if version_exhausted(): return 'VERSION_EXHAUSTED_DENY'
        rec = build_record(op='RELEASE', slot=token.slot,
                           old_gen=token.gen, new_gen=token.gen + 1,
                           owner_id=caller_owner, resource_id=caller_resource,
                           count_delta=-1, active_bit=-1, version_delta=+1)
        return run_txn(rec)
```

**关键统一**：
- `release`、`token_consume`、`slot_free` 是**同一实现**（release）。
- **generation 递增时机统一**：release 消费时 `new_gen = old_gen + 1`，与 admit 分配时一致。
- 所有释放路径（release/consume/slot_free/revoke）都递增 generation，无例外。

---

## 3. generation 变化规则（统一）

| 操作 | generation 变化 |
|---|---|
| 首次分配（admit） | 0 → 1（或旧值 +1） |
| 复用分配（admit） | 旧值 +1 |
| 释放（release/consume/slot_free） | 旧值 +1 |
| 撤销（revoke） | 旧值 +1 |

**统一规则**：**任何涉及槽位的操作（分配、释放、撤销）都使该槽位 generation +1**。这样旧 Token（携带旧 gen）在槽位状态变化后必然失效。

---

## 4. 多 Token 并存演算

```
初始：slot0(g1) 活动，slot1(g1) 活动，COUNT=2
admit slot2 → slot2(g1)，COUNT=3，slot0/slot1 不变
release(slot0,g1) → slot0 清位，gen→2，COUNT=2，slot1 不受影响
release(slot1,g1) → slot1 清位，gen→2，COUNT=1
最终：slot2 活动，COUNT=1，I-1 成立 ✓
```

---

## 5. 旧 Token 拒绝

```
slot0 释放后复用：TRT[0].generation 从 2 → 3
旧 Token(slot0,g2) 释放 → TRT[0].generation=3 != 2 → INVALID_TOKEN ✓
```

---

## 6. 与 05.41 的差异

| 项 | 05.41 | 05.42 |
|---|---|---|
| release 实现 | 多份（slot_free 与 release 不同） | 唯一 release |
| generation 递增 | slot_free 不递增，release 递增 | 统一都递增 |
| 别名 | 未统一 | release=consume=slot_free |

---

## 证据来源

- Token 身份校验：TOKEN-IDENTITY-VALIDATION.md
- 状态转换：STATE-INVARIANTS-AND-TRANSITIONS.md
- 提交协议：TRANSACTION-COMMIT-PROTOCOL.md
- 证明等级：VERIFIED_STATIC（单进程）+ NOT_PROVEN（跨进程/跨重启）
