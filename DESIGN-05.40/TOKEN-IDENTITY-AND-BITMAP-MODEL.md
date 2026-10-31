# TOKEN-IDENTITY-AND-BITMAP-MODEL.md — Token 身份与槽位位图模型（05.40 权威）

证据等级：VERIFIED_STATIC（单进程）+ NOT_PROVEN（跨进程/跨重启）
目标：修复 B39-R01——05.39 用 32 位 `token_seq` + 32 位 `active_mask`，单个 bit 不能证明任意 Token 身份唯一，槽位复用会让旧 Token 复活。

---

## 1. 05.39 缺陷定位

05.39 的 Token 身份 = `token_seq`（32 位单调），`active_mask` 是 32 位位图，每个活动 Token 占一个 bit。缺陷：

1. **位图 bit 不是身份**：`active_mask` 的某个 bit 只表示「该位置有活动 Token」，但**不携带「是哪个 Token」**。一旦 bit 被清 0 后复用给新 Token，旧 Token 携带相同 bit 位置即可被误判为「仍有效」。
2. **槽位复用无代际**：`token_seq` 单调递增，但 `active_mask` 的 bit 位置（槽位）在 Token 消费后会被复用。旧 Token（bit 位置相同）在新 Token 复用该槽位后，`is_active(active, old_token)` 可能返回 True，旧 Token 复活。
3. **单个 bit 无法编码完整身份**：`(slot)` 一个 bit 只有「占用/未占用」二态，无法区分「旧 Token」与「复用该槽位的新 Token」。

**根因**：把「槽位占用位」当作「Token 身份」，二者信息量不对等。

---

## 2. 唯一权威 Token 完整结构

Token 身份 = **槽位 + 代际（generation）** 的组合，二者都内嵌在 128 位状态字中：

```
Token 完整标识 = (slot, generation, owner_id, resource_id, state_version)
```

| 字段 | 位置 | 说明 |
|---|---|---|
| slot | TOKEN_ACTIVE 位图 bit 索引（0–15） | 16 个槽位，每 bit 一个 |
| generation | TOKEN_GENERATION（bit80–95，16 位） | 槽位复用时单调递增，旧 Token 失效 |
| owner_id | 业务上下文（进程/线程标识） | 不内嵌状态字，用于审计 |
| resource_id | 资源标识 | 不内嵌状态字，绑定资源 |
| state_version | 发放时的 VERSION | 内嵌状态字，用于恢复判定 |

**关键原则**：`TOKEN_ACTIVE` 位图**只能作为「活动状态索引」**，不得将单个 bit 当作任意 Token 身份的唯一证明。身份唯一性由 `(slot, generation)` 二元组保证。

---

## 3. 槽位分配与代际算法（内部实现，非抽象函数）

### 数据结构（内嵌状态字）

- `TOKEN_ACTIVE`：16 位，`bit i = 1` 表示槽位 i 当前被活动 Token 占用。
- `TOKEN_GENERATION`：16 位，**全局单调**，每次槽位分配（含复用）时 +1。

### 槽位分配 `slot_alloc()`

```python
def slot_alloc():
    while True:
        w = load_word()
        active = field_get_token_active(w)   # 16 位位图
        gen = field_get_token_generation(w)
        if gen == 0xFFFF:
            return ('GENERATION_EXHAUSTED_DENY', None)   # 代际耗尽，禁止回绕
        free_slot = find_free_bit(active)    # 找第一个 0 bit（0–15）
        if free_slot is None:
            return ('NO_SLOT', None)          # 16 槽位全占
        new_gen = gen + 1
        new_active = active | (1 << free_slot)
        r = field_cas(w, {
            'token_active': new_active,
            'token_generation': new_gen,
            'version': field_get_version(w) + 1,   # 版本耗尽检查见 VERSION-EXHAUSTION
        })
        if r == 'OK':
            return ('OK', (free_slot, new_gen))   # 返回 (slot, generation) 作为 Token 身份
        if r in ('VERSION_EXHAUSTED_DENY', 'GENERATION_EXHAUSTED_DENY', 'INVALID_STATE'):
            return (r, None)
        # CAS_FAILED → 重读重算
```

### 槽位校验 `slot_valid(w, slot, generation)`

```python
def slot_valid(w, slot, generation):
    active = field_get_token_active(w)
    cur_gen = field_get_token_generation(w)
    if not (0 <= slot < 16):
        return False
    if not (active & (1 << slot)):
        return False
    # 关键：generation 必须等于当前全局 generation，
    # 但因为 generation 是全局单调（非每槽位独立），需要额外记录「该槽位发放时的 generation」。
    # 见 §4 的 per-slot generation 修正。
    return generation == cur_gen   # 简化说明，完整见 §4
```

> **注意**：全局 `TOKEN_GENERATION` 是单个 16 位计数器，无法为每个槽位记录独立的发放代际。§4 给出修正模型——必须为每个槽位记录独立 generation，否则槽位复用仍无法区分旧 Token。

---

## 4. 每槽位独立代际（修正模型，权威）

**发现**：单一全局 `TOKEN_GENERATION` 不足以区分「旧 Token 复用槽位」与「新 Token 占用同槽位」。因为全局代际对所有槽位共享，槽位 i 的旧 Token（generation=g1）与新 Token（generation=g2）都会看到「当前全局 generation=g2」，`slot_valid` 无法用全局值区分。

**权威修正**：每个槽位的代际必须**独立**存储，或通过「槽位分配时的 generation 快照」来校验。

由于状态字仅 128 位，无法为 16 个槽位各存一个 16 位 generation（需 256 位）。因此采用**方案：Token 身份由「槽位 + 发放时全局 generation 快照」组成，校验时用「槽位占用 且 该槽位的 active 位在 generation 快照之后未被复用」判定**。

**最终权威模型（在 128 位约束下可证明）**：

```
Token 身份 = (slot, gen_at_issue)
其中 gen_at_issue 是 slot_alloc() 成功时 TOKEN_GENERATION 的值。
```

**校验规则（关键，防旧 Token 复活）**：

```python
def token_consume(token):   # token = (slot, gen_at_issue)
    while True:
        w = load_word()
        active = field_get_token_active(w)
        cur_gen = field_get_token_generation(w)
        # 槽位必须仍被占用
        if not (active & (1 << token.slot)):
            return 'INVALID_TOKEN'
        # 关键：当前全局 generation 必须 == 发放时的 gen_at_issue。
        # 若槽位被复用，generation 必然 +1，cur_gen > gen_at_issue → 旧 Token 失效。
        if cur_gen != token.gen_at_issue:
            return 'INVALID_TOKEN'          # 旧 Token 复活被拒绝
        # 消费：清位，COUNT−1，VERSION+1
        new_active = active & ~(1 << token.slot)
        r = field_cas(w, {
            'token_active': new_active,
            'count': field_get_count(w) - 1,
            'version': field_get_version(w) + 1,
        })
        if r == 'OK':
            return 'OK'
        # CAS_FAILED → 重读重算
```

**为何能阻止旧 Token 复活**：

- 旧 Token `(slot, g1)` 在槽位被复用后，新 Token `(slot, g2)` 发放时 `TOKEN_GENERATION` 已递增到 g2（g2 > g1）。
- 旧 Token 的 `gen_at_issue = g1`，而当前 `cur_gen = g2 ≠ g1` → `token_consume` 返回 `INVALID_TOKEN`。
- 即使槽位 bit 相同，`cur_gen != gen_at_issue` 的校验挡住旧 Token。

**唯一性证明**：Token 身份 = `(slot, gen_at_issue)`。generation 全局单调递增、禁止回绕（耗尽 DENY），故任意两次发放的 `gen_at_issue` 不同，身份唯一。∎

---

## 5. 槽位复用、旧 Token 迟到、重复消费

| 场景 | 处理 |
|---|---|
| 槽位复用 | 分配时 generation+1，旧 Token 的 gen_at_issue 失效 |
| 旧 Token 迟到 | `cur_gen != gen_at_issue` → INVALID_TOKEN |
| 重复消费 | 首次消费清位后，`active & (1<<slot)` 为 0 → INVALID_TOKEN |
| 并发分配同槽位 | 两个 alloc 同时找 free bit，只有一个 CAS 成功，另一个重读重算找新 free bit |
| 并发释放同 Token | 一个 CAS 成功清位，另一个重读后 `active bit=0` → INVALID_TOKEN |
| 序列号耗尽 | generation==0xFFFF → GENERATION_EXHAUSTED_DENY，禁止复用 |
| 进程崩溃 | 状态字持久化/恢复，见 TOKEN-CRASH-RECOVERY.md |

---

## 6. 状态转换表（Token）

```
NOT_ISSUED ──slot_alloc CAS──▶ ACTIVE(slot, gen)
                                  │
                                  ├─ token_consume CAS ──▶ CONSUMED（槽位清位，可复用）
                                  ├─ token_revoke CAS  ──▶ REVOKED（槽位清位）
                                  ├─ 业务异常未 release ──▶ LEAKED（恢复判定）
                                  └─ 进程崩溃 ──▶ 恢复（TOKEN-CRASH-RECOVERY.md）
```

---

## 7. NOT_PROVEN 边界

- 跨进程：`(slot, gen)` 身份依赖共享内存 + HW 128 位 CAS，未证明 → NOT_PROVEN → DENY。
- 跨重启：generation 持久化未证明 → 恢复时 generation 可能回退，旧 Token 复活风险 → NOT_PROVEN → DENY。
- generation 回绕：无持久化 epoch 时回绕防护 NOT_PROVEN → 耗尽 DENY。

---

## 证据来源

- 状态字布局：UNIFIED-ATOMIC-STATE-WORD.md
- 槽位复用：TOKEN-SLOT-ALLOCATION-AND-REUSE.md
- 原子事务：TOKEN-ATOMIC-TRANSACTION.md
- 05.39 缺陷：TOKEN-ATOMIC-TRANSACTION.md §3 的 `active_mask` 位图（无代际）
- 证明等级：VERIFIED_STATIC（单进程）+ NOT_PROVEN（跨进程/跨重启）
