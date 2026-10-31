# UNIFIED-ATOMIC-STATE-WORD.md — 统一原子状态字（05.40 权威）

证据等级：VERIFIED_STATIC（单进程）+ NOT_PROVEN（跨进程/跨重启）
目标：修复 B39-R02——05.39 的 Token 协议用 128 位 CAS、版本协议用 64 位 CAS，二者共享同一状态字段，存在未定义的重叠并发修改。

---

## 1. 05.39 缺陷定位

05.39 存在两套互斥的原子宽度：

| 文档 | 状态字宽度 | 字段布局 |
|---|---|---|
| TOKEN-ATOMIC-TRANSACTION.md | 128 位 | bit0–31 状态 / bit32–63 version / bit64–95 token_seq / bit96–127 active_mask |
| RECOVERY-VERSION-ATOMICITY.md | 64 位 | bit0–31 状态 / bit32–63 version |

**矛盾**：`admit()`/`release()` 用 `cas128()` 修改整字，`cas_state_version()` 用 `cas64()` 只比较/修改低 64 位。同一状态字同时存在 64 位与 128 位两个 CAS 写入者，两者对 bit0–63 的重叠修改是**未定义行为**——128 位 CAS 会同时覆盖 version，64 位 CAS 会覆盖 state 而看不到 token 字段的变化。这是 B40-02 必须闭合的核心缺陷。

**根因**：05.39 未建立「唯一权威状态字布局 + 唯一原子访问宽度」，导致 Token 与版本两套协议各自定义了不同的字宽。

---

## 2. 唯一权威决策

**结论：所有共享状态字段统一到一个 128 位原子字，所有状态修改只通过同一条 128 位 `field_cas` 接口。64 位 CAS 路径废除。**

理由：
1. Token 身份模型（B40-01）需要 `generation` 字段，无法塞进 64 位。
2. 统一 128 位后，`state + version + token 账本` 处于同一原子提交边界，天然消除 64/128 位重叠修改。
3. 64 位不足以容纳「状态 + 版本 + Token 身份」的完整字段集合。

**唯一权威布局（128 位）**：

```
bit0–15    COUNT（15 位，COUNT_MASK=0xFFFE，COUNT_SHIFT=1，COUNT_MAX=32767）
bit16      GATE（1=OPEN，0=CLOSED）
bit17      DENIED（单调置位，不可清除）
bit18      FROZEN（1=冻结）
bit19      FREEZING（1=冻结中）
bit20–31   保留位（必须恒 0，非 0 判非法）
bit32–63   VERSION（32 位，单调递增，禁止回绕）
bit64–79   TOKEN_ACTIVE（16 位槽位位图，每 bit 对应一个槽位）
bit80–95   TOKEN_GENERATION（16 位，单调递增，槽位复用代际校验）
bit96–127  保留位（必须恒 0，非 0 判非法）
```

> 注意：bit20–31 与 bit96–127 为保留位。非法状态字判定见 §5。COUNT 字段沿用历史约束：`COUNT_MASK=0xFFFE`、`COUNT_SHIFT=1`、`COUNT_MAX=32767`（05.36 修正，不得放宽）。

---

## 3. 字段定义表（权威，全文档唯一）

| 字段 | 位段 | 掩码/取值 | 含义 | 非法值 |
|---|---|---|---|---|
| COUNT | bit1–15 | `0xFFFE` | 活跃准入计数（bit0 恒 0，因此 `& 0xFFFE`） | bit0=1 |
| GATE | bit16 | `0x10000` | 1=OPEN，0=CLOSED | — |
| DENIED | bit17 | `0x20000` | 1=已拒绝，单调置位 | 置位后清除=非法 |
| FROZEN | bit18 | `0x40000` | 1=冻结 | FROZEN=1 且 FREEZING=1 同现需按 §4 判定 |
| FREEZING | bit19 | `0x80000` | 1=冻结中 | — |
| 保留位 | bit20–31 | `0xFFF00000` | 必须 0 | 非 0 |
| VERSION | bit32–63 | `0xFFFFFFFF<<32` | 单调递增 | 回绕 |
| TOKEN_ACTIVE | bit64–79 | 16 位位图 | 每 bit = 一个槽位的活动状态 | 槽位越界 |
| TOKEN_GENERATION | bit80–95 | 16 位 | 单调递增，槽位复用代际 | 回绕 |
| 保留位 | bit96–127 | `0xFFFFFFFF<<96` | 必须 0 | 非 0 |

**唯一权威状态源**：本状态字是准入状态的唯一权威源，不得引入任何独立的安全判断计数器（继承 B35-01）。

---

## 4. 唯一原子访问接口

所有状态修改**只允许**通过以下单一接口，禁止任何其它 CAS 宽度或无条件 `commit()`：

```python
# 唯一权威接口：128 位 field-aware CAS
def field_cas(expected: Word128, new_fields: dict) -> str:
    """
    原子比较整个 128 位 expected，与当前字一致时才提交 new_fields 指定的字段修改。
    返回 'OK' / 'CAS_FAILED' / 'VERSION_EXHAUSTED_DENY' / 'INVALID_STATE'。
    """
    new_word = expected.word
    for field, val in new_fields.items():
        new_word = field_set(new_word, field, val)   # 仅修改指定字段，不动其它位
    if not is_valid_state(new_word):
        return 'INVALID_STATE'
    return hardware_cas128(expected.word, new_word)
```

**禁止的路径（05.39 遗留）**：
- 任何 64 位 `hardware_cas64()` 对状态字 bit0–63 的写入 → **删除**。
- 任何无条件 `commit(state)` 整字覆盖 → **删除**。
- 任何 `new_word = (old + 1) & MASK` 的整字算术 → **删除**（字段修改必须走 field_set）。

**字段操作函数**（唯一，全文档引用同一组）：
- `field_get_count(w)` / `field_set_count(w, n)`：提取/设置 COUNT，掩码 `0xFFFE`。
- `field_get_gate(w)` / `field_set_gate(w, g)`。
- `field_get_denied(w)` / `field_set_denied(w)`（只置 1，无清 0）。
- `field_get_frozen(w)` / `field_set_frozen(w, f)`。
- `field_get_freezing(w)` / `field_set_freezing(w, f)`。
- `field_get_version(w)` / `field_set_version(w, v)`。
- `field_get_token_active(w)` / `field_get_token_generation(w)`。
- `field_slot_set_active(w, slot)` / `field_slot_clear_active(w, slot)`（槽位位图，见 TOKEN-IDENTITY-AND-BITMAP-MODEL.md）。

---

## 5. 非法状态字判定与默认 DENY

`is_valid_state(w)` 判定，任一不满足即 `INVALID_STATE`，默认不放行：

1. `COUNT & 1 != 0`（bit0 非法）→ 非法。
2. 保留位 bit20–31 非 0 → 非法。
3. 保留位 bit96–127 非 0 → 非法。
4. `FROZEN=1 且 FREEZING=1`：需按 FREEZE-STATE-MACHINE 判定。冻结完成（FROZEN=1）后 FREEZING 必须为 0；同现=非法中间态，DENY。
5. `DENIED=1 且 GATE=OPEN`：DENIED 置位后 GATE 必须 CLOSED；同现=非法，DENY（DENIED 优先）。
6. `TOKEN_ACTIVE` 槽位数与 COUNT 不一致：COUNT 应等于 `popcount(TOKEN_ACTIVE)`（每活动 Token 对应 COUNT+1）。不一致=账本损坏，DENY（不自动重建）。

---

## 6. 三个安全边界

| 边界 | 128 位 CAS 前提 | 状态 |
|---|---|---|
| 单进程 | 进程内 CMPXCHG16B（x86-64）或等价 | VERIFIED_STATIC |
| 跨进程 | 共享内存 + 16 字节对齐 + HW 128 位 CAS | NOT_PROVEN → DENY |
| 跨重启 | 128 位状态字持久化 + 恢复协议 | NOT_PROVEN → DENY |

**平台不支持 128 位原子 CAS 时**：整个准入/Token/Fencing 协议**整体 DENY**，不得退化为 64 位 CAS（因为 64 位无法容纳 Token 身份字段，退化会重新引入 B39-R02 的重叠修改）。

---

## 7. 与 05.39 的差异

| 项 | 05.39 | 05.40 |
|---|---|---|
| 状态字宽度 | 128 位（Token 文档）与 64 位（版本文档）并存 | 统一 128 位，64 位废除 |
| 原子接口 | cas128 / cas64 混用 | 唯一 field_cas（128 位） |
| Token 身份 | token_seq(32) + active_mask(32) | TOKEN_ACTIVE(16 槽位) + TOKEN_GENERATION(16) |
| 版本耗尽 | 仅 cas_state_version 检查 | 全部入口统一检查（见 VERSION-EXHAUSTION-ALL-ENTRYPOINTS.md） |

---

## 证据来源

- 05.39 缺陷：TOKEN-ATOMIC-TRANSACTION.md（128 位）与 RECOVERY-VERSION-ATOMICITY.md（64 位）的字宽矛盾
- 历史约束：COUNT_MASK=0xFFFE（05.36）、单一准入状态源（B35-01）
- 证明等级：VERIFIED_STATIC（单进程）+ NOT_PROVEN（跨进程/跨重启）
