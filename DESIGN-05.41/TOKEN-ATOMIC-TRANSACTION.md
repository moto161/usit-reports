# TOKEN-ATOMIC-TRANSACTION.md — Token 与 COUNT 原子事务（05.41 权威）

证据等级：VERIFIED_STATIC（单进程）+ NOT_PROVEN（跨进程/跨重启）
目标：在每槽位独立代际 + 独立 TRT 模型下，证明 Token 发放/消费与 COUNT 更新在同一原子事务内。

---

## 1. 事务模型总览

- 状态字（128 位）持有 COUNT + TOKEN_ACTIVE 位图 + 准入控制字段。
- TRT 持有每槽位 `{generation, owner_id, resource_id}`。
- 二者通过 `txn_lock` + 状态字 CAS 提交点保证原子一致（见 ATOMIC-TRANSACTION-BOUNDARY.md）。

---

## 2. admit（发放：COUNT+1 + 槽位占用 + TRT 写入）

```python
def admit(owner_id, resource_id):
    while True:
        txn = begin_txn()                       # 获取 txn_lock，读快照
        s = txn.state
        if get_denied(s):
            return 'DENIED'
        if not get_gate(s):
            return 'GATE_CLOSED'
        if get_count(s) >= COUNT_MAX:
            return 'OVERFLOW'
        if s.version == 0xFFFFFFFF:
            return 'VERSION_EXHAUSTED_DENY'
        slot = find_free_bit(s.token_active)
        if slot is None:
            return 'NO_SLOT'
        if txn.trt[slot].generation == 0xFFFF:
            return 'SLOT_GENERATION_EXHAUSTED_DENY'

        # 修改（锁内）
        txn.trt[slot].generation += 1
        txn.trt[slot].owner_id = owner_id
        txn.trt[slot].resource_id = resource_id
        s.token_active |= (1 << slot)
        s.count += 1
        s.version += 1

        r = commit_txn(txn)                     # 先写 TRT，再 CAS 状态字
        if r == 'OK':
            return ('OK', (slot, txn.trt[slot].generation))
        if r == 'CAS_FAILED':
            continue                             # 重读重算
        return r
```

**线性化点**：`commit_txn` 内的状态字 CAS 成功。COUNT+1、槽位占用、TRT 写入三者在 CAS 成功那一刻原子生效。

---

## 3. release（消费：校验 + COUNT−1 + 槽位清位 + TRT 代际失效）

```python
def release(token):    # token = (slot, gen)
    while True:
        txn = begin_txn()
        s = txn.state
        if not (s.token_active & (1 << token.slot)):
            return 'INVALID_TOKEN'
        if txn.trt[token.slot].generation != token.gen:
            return 'INVALID_TOKEN'               # 旧 Token 复活拒绝
        if get_count(s) == 0:
            return 'UNDERFLOW'
        if s.version == 0xFFFFFFFF:
            return 'VERSION_EXHAUSTED_DENY'

        txn.trt[token.slot].generation += 1      # 消费：代际+1，旧 token 失效
        s.token_active &= ~(1 << token.slot)
        s.count -= 1
        s.version += 1

        r = commit_txn(txn)
        if r == 'OK':
            return 'OK'
        if r == 'CAS_FAILED':
            continue
        return r
```

**关键**：消费时 `TRT[slot].generation += 1`，即使槽位 bit 清位后又被复用，旧 Token 的 `gen` 也因消费时已递增而无法再次匹配（消费本身也递增了代际，形成双重防护）。

---

## 4. 并发重复消费演算

```
线程A、B 并发 release(slot0, g5)：
A begin_txn → 校验通过（bit=1, gen=5）
B begin_txn（等待 txn_lock）
A commit：清位 + COUNT-1 + TRT[0].gen=6，CAS 成功，释放锁
B begin_txn → 读快照：bit=0 → INVALID_TOKEN
同一 Token 只消费一次 ✓
```

---

## 5. 崩溃窗口演算

| 崩溃点 | 状态 | 恢复 |
|---|---|---|
| admit 状态字 CAS 前崩溃 | TRT 已写（gen+1，owner 写入），状态字未写（COUNT 未+1，bit 未置位） | 恢复检测「TRT 有记录但 bit 未置位」→ 回滚 TRT（gen-1） |
| admit CAS 成功后崩溃 | COUNT+1 + bit 置位 + TRT 一致 | 事务完成，无需恢复 |
| release 状态字 CAS 前崩溃 | TRT gen+1 已写，状态字未写（bit 仍置位） | 恢复检测「TRT gen 与 bit 状态矛盾」→ 回滚 TRT gen |
| release CAS 成功后崩溃 | 清位 + COUNT-1 + TRT gen+1 一致 | 事务完成 |

**核心**：状态字 CAS 是唯一提交点。CAS 前崩溃 = TRT 领先，恢复时回滚 TRT；CAS 后 = 一致完成。详见 TOKEN-CRASH-RECOVERY.md。

---

## 6. 与 05.40 的差异

| 项 | 05.40 | 05.41 |
|---|---|---|
| Token 身份 | (slot, 全局 gen) | (slot, 每槽位 gen) |
| 原子机制 | 单 128 位 CAS | txn_lock + CAS 提交点 |
| 多 Token 并存 | 全局 gen 误杀 | 每槽位独立，互不影响 |

---

## 证据来源

- 原子边界：ATOMIC-TRANSACTION-BOUNDARY.md
- 每槽位代际：TOKEN-IDENTITY-AND-PER-SLOT-GENERATION.md
- 崩溃恢复：TOKEN-CRASH-RECOVERY.md
- 证明等级：VERIFIED_STATIC（单进程）+ NOT_PROVEN（跨进程/跨重启）
