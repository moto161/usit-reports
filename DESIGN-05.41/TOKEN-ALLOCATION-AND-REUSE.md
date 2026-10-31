# TOKEN-ALLOCATION-AND-REUSE.md — Token 槽位分配与复用（05.41 权威）

证据等级：VERIFIED_STATIC（单进程）+ NOT_PROVEN（跨进程/跨重启）
目标：给出每槽位独立代际下的完整分配、释放、复用、并发演算。

---

## 1. 槽位模型

- 槽位数 16（TOKEN_ACTIVE 16 位位图，bit0–15）。
- 槽位状态：bit=1 占用（活动 Token），bit=0 空闲。
- 每槽位代际 `TRT[slot].generation`（16 位），仅该槽位复用时 +1。

---

## 2. 分配算法（txn_lock 内，完整）

```python
def slot_alloc(txn, owner_id, resource_id):
    slot = find_free_bit(txn.state.token_active)   # 第一个 0 bit
    if slot is None:
        return 'NO_SLOT'                            # 16 槽位全占
    if txn.state.version == 0xFFFFFFFF:
        return 'VERSION_EXHAUSTED_DENY'
    if txn.trt[slot].generation == 0xFFFF:
        return 'SLOT_GENERATION_EXHAUSTED_DENY'     # 该槽位代际耗尽

    txn.trt[slot].generation += 1                   # 复用/分配：该槽位代际 +1
    txn.trt[slot].owner_id = owner_id
    txn.trt[slot].resource_id = resource_id
    txn.state.token_active |= (1 << slot)
    txn.state.count += 1
    txn.state.version += 1
    txn.commit()                                    # 单一提交点
    return ('OK', (slot, txn.trt[slot].generation))
```

> 注：首次分配一个从未用过的槽位，`generation` 从 0 递增到 1；复用释放过的槽位，`generation` 在旧值上 +1。两种情况的语义一致——「该槽位每次发放 generation 严格递增」。

---

## 3. 释放算法（txn_lock 内）

```python
def slot_free(txn, token):    # token = (slot, gen)
    if not (txn.state.token_active & (1 << token.slot)):
        return 'INVALID_TOKEN'
    if txn.trt[token.slot].generation != token.gen:
        return 'INVALID_TOKEN'                       # 旧 Token 复活拒绝
    if txn.state.count == 0:
        return 'UNDERFLOW'
    txn.state.token_active &= ~(1 << token.slot)
    txn.state.count -= 1
    txn.state.version += 1
    txn.commit()
    return 'OK'
```

---

## 4. 并发演算（完整状态演算）

### 场景 A：多 Token 并存 + 独立消费

```
初始：槽位0(g5)活动、槽位1(g7)活动，COUNT=2
线程A slot_alloc → 槽位2(g1)，COUNT=3，槽位0/1 不受影响
线程B release(slot0,g5) → 槽位0清位，COUNT=2，槽位1 仍活动
线程C release(slot1,g7) → 槽位1清位，COUNT=1
最终：仅槽位2 活动，COUNT=1 ✓
```

### 场景 B：16 槽位全活动

```
16 个 slot_alloc 依次分配槽位0–15，COUNT=16
每个 release(slot_i, g_i) 独立成功，COUNT 递减
COUNT 始终 == popcount(TOKEN_ACTIVE) ✓
```

### 场景 C：并发分配同槽位

```
线程A、B 同时 find_free_bit → 都读到 slot=5
A txn 提交成功（槽位5 占用，g+1）
B txn 提交时发现 token_active 已变（txn 冲突检测）→ 重试，重新找 free bit
不会出现两个 Token 同槽位 ✓
```

### 场景 D：并发消费同 Token

```
线程A、B 同时 slot_free(slot0,g5)
A txn 提交成功（清位，COUNT-1）
B txn 冲突重试 → token_active bit=0 → INVALID_TOKEN
同一 Token 只消费一次 ✓
```

---

## 5. 槽位复用与旧 Token 拒绝

| 情况 | 判定 |
|---|---|
| 槽位0 复用：TRT[0].generation 5→6 | 旧 Token(slot0,g5) 释放时 gen=6!=5 → INVALID_TOKEN |
| 槽位0 未复用，旧 Token(slot0,g5) | gen==5 → 有效（正常消费） |
| 槽位0 已释放（bit=0） | INVALID_TOKEN |

---

## 6. 序列号耗尽 / 崩溃

| 场景 | 处理 |
|---|---|
| 某槽位 generation==0xFFFF | SLOT_GENERATION_EXHAUSTED_DENY（仅该槽位，不影响他槽位） |
| 分配中途崩溃 | txn 未 commit，无变化；commit 后崩溃，状态一致 |
| 释放中途崩溃 | txn 原子，要么完成要么无变化 |
| 进程崩溃 | TRT + 状态字一起恢复，见 TOKEN-CRASH-RECOVERY.md |

---

## 7. 与 05.40 的差异

| 项 | 05.40 | 05.41 |
|---|---|---|
| 分配时 generation | 全局 +1（误杀他槽位） | 仅该槽位 +1 |
| 代际存储 | 状态字 bit80–95 | TRT[slot].generation |
| 多 Token 并存 | 缺陷（互相失效） | 每槽位独立，互不影响 |

---

## 证据来源

- 状态模型：UNIFIED-STATE-MODEL.md
- 每槽位代际：TOKEN-IDENTITY-AND-PER-SLOT-GENERATION.md
- 原子边界：ATOMIC-TRANSACTION-BOUNDARY.md
- 证明等级：VERIFIED_STATIC（单进程）+ NOT_PROVEN（跨进程/跨重启）
