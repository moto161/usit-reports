# TOKEN-IDENTITY-AND-PER-SLOT-GENERATION.md — Token 身份与每槽位独立代际（05.41 权威）

证据等级：VERIFIED_STATIC（单进程）+ NOT_PROVEN（跨进程/跨重启）
目标：修复 B40-R01——全局 generation 使其他活动 Token 失效；改为每槽位独立代际。

---

## 1. 05.40 缺陷定位

05.40 用**单个全局** `TOKEN_GENERATION`，导致：

```
slot_alloc(槽位2) → 全局 generation 6
release(slot0, gen=5) → cur_gen=6 != 5 → INVALID_TOKEN  ← 槽位0 Token 被误杀
```

**根因**：全局代际把「所有槽位的分配」串进同一个递增序列，任何分配都改变所有 Token 的校验基准。

---

## 2. 四个必须区分的概念

| 概念 | 存储 | 语义 |
|---|---|---|
| 全局发放序列 | 无（废除） | 05.40 的 TOKEN_GENERATION，已删除 |
| 每槽位代际 | TRT[slot].generation（16 位） | 该槽位复用时的代际，独立递增 |
| 活动位图 | 状态字 TOKEN_ACTIVE（16 位） | 每 bit = 槽位是否活动 |
| Token 完整身份 | (slot, generation, owner, resource, state_version) | 独立可验证 |

---

## 3. 每槽位独立代际（权威）

- `TRT[slot].generation`：16 位，**仅在槽位 slot 被复用分配时 +1**。
- 其他槽位的 generation 不受影响。
- 每个 Token 独立验证：`TRT[slot].generation == token.gen`。

### 多 Token 并存（关键演算）

```
初始：TRT[0].generation=5（Token0 活动），TRT[1].generation=7（Token1 活动）
slot_alloc 分配槽位 2：
  → TRT[2].generation = 0（新槽位从 0 或递增初始值起）
  → TRT[0].generation 仍=5，TRT[1].generation 仍=7
release(slot0, gen=5) → TRT[0].generation==5 → 校验通过 ✓
release(slot1, gen=7) → TRT[1].generation==7 → 校验通过 ✓
```

**结论**：槽位 2 的分配不影响槽位 0/1 的 Token。满足「任意槽位新分配不导致其他 ACTIVE Token 失效」。

---

## 4. 旧 Token 拒绝（槽位复用）

槽位 0 释放后复用：

```
TRT[0].generation: 5 →（释放，槽位0空闲）→（复用分配）→ 6
旧 Token(slot0, gen=5) 再释放：
  → TRT[0].generation=6 != 5 → INVALID_TOKEN ✓
```

**为何其他槽位变化不影响它**：旧 Token 校验只读 `TRT[slot].generation`，不读任何全局计数或他槽位 generation。槽位 1 的 generation 变化与槽位 0 的旧 Token 无关。

---

## 5. generation 存储 / 位宽 / 耗尽 / 持久化 / 恢复 / 回绕

| 项 | 规则 |
|---|---|
| 存储 | TRT[slot].generation，独立于状态字 |
| 位宽 | 16 位（0–65535） |
| 耗尽 | 某槽位 generation==0xFFFF 时，该槽位拒绝再分配 → SLOT_GENERATION_EXHAUSTED_DENY |
| 持久化 | TRT 随状态字一起持久化；未证明 → NOT_PROVEN → DENY |
| 恢复 | 崩溃后按 TRT[slot].generation 判定，见 TOKEN-CRASH-RECOVERY.md |
| 回绕 | 禁止（16 位耗尽即 DENY，不回绕） |

---

## 6. 每槽位代际的原子更新

`TRT[slot].generation` 的更新与 COUNT/TOKEN_ACTIVE 的更新必须在**同一事务锁内**（见 ATOMIC-TRANSACTION-BOUNDARY.md）。任何槽位 generation 的变更不得逃逸事务边界。

```python
# 槽位复用分配（在 txn_lock 内）
def slot_alloc(txn, owner_id, resource_id):
    slot = find_free_bit(txn.state.token_active)
    if slot is None:
        return 'NO_SLOT'
    if txn.trt[slot].generation == 0xFFFF:
        return 'SLOT_GENERATION_EXHAUSTED_DENY'
    # 复用：generation+1；首次分配：generation 从 0 起（或递增初始值）
    txn.trt[slot].generation += 1
    txn.trt[slot].owner_id = owner_id
    txn.trt[slot].resource_id = resource_id
    txn.state.token_active |= (1 << slot)
    txn.state.count += 1
    txn.commit()   # 单一提交点（见 ATOMIC-TRANSACTION-BOUNDARY.md）
    return ('OK', (slot, txn.trt[slot].generation))
```

---

## 7. 唯一性证明

Token 身份 = `(slot, generation)`。每槽位 generation 独立单调、禁止回绕。任意槽位 slot 的两次发放，generation 必不同（复用 +1，不回绕）。因此 `(slot, generation)` 二元组全局唯一。∎

---

## 8. NOT_PROVEN 边界

- 跨进程 TRT 共享 + 跨进程 txn_lock：NOT_PROVEN → DENY。
- 跨重启 TRT 持久化 + generation 单调：NOT_PROVEN → DENY。
- generation 回绕（无持久化 epoch）：NOT_PROVEN → 耗尽 DENY。

---

## 证据来源

- 状态模型：UNIFIED-STATE-MODEL.md
- 原子边界：ATOMIC-TRANSACTION-BOUNDARY.md
- 05.40 缺陷：TOKEN-IDENTITY-AND-BITMAP-MODEL.md（05.40 版）§4 全局 generation
- 证明等级：VERIFIED_STATIC（单进程）+ NOT_PROVEN（跨进程/跨重启）
