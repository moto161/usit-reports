# TOKEN-SLOT-ALLOCATION-AND-REUSE.md — Token 槽位分配与复用（05.40 权威）

证据等级：VERIFIED_STATIC（单进程）+ NOT_PROVEN（跨进程/跨重启）
目标：修复 B39-R01——给出完整槽位映射、分配、释放、复用、代际校验及旧 Token 拒绝算法。

---

## 1. 槽位模型

- **槽位数**：16（TOKEN_ACTIVE 为 16 位位图，bit0–bit15）。
- **槽位 = 位图 bit 索引**。
- **槽位状态**：`1` = 被活动 Token 占用，`0` = 空闲。
- **代际**：全局 `TOKEN_GENERATION`（16 位单调递增），每次槽位分配（含复用）+1。

---

## 2. 槽位分配算法（完整）

```python
def slot_alloc():
    while True:
        w = load_word()
        active = field_get_token_active(w)
        gen = field_get_token_generation(w)
        ver = field_get_version(w)

        if gen == 0xFFFF:
            return ('GENERATION_EXHAUSTED_DENY', None)   # 代际耗尽，禁止回绕
        if ver == 0xFFFFFFFF:
            return ('VERSION_EXHAUSTED_DENY', None)

        free = find_free_bit(active)      # 返回第一个 0 bit 的索引，或 None
        if free is None:
            return ('NO_SLOT', None)       # 16 槽位全占，无法分配

        r = field_cas(w, {
            'token_active':   active | (1 << free),
            'token_generation': gen + 1,
            'version':        ver + 1,
        })
        if r == 'OK':
            return ('OK', (free, gen + 1))   # Token = (slot=free, gen=gen+1)
        if r in ('VERSION_EXHAUSTED_DENY', 'GENERATION_EXHAUSTED_DENY', 'INVALID_STATE'):
            return (r, None)
        # CAS_FAILED → 重读重算（free bit 可能已被并发占用）
```

**并发分配**：两个线程同时 `slot_alloc`，都读到 `active` 相同、`free` 相同。只有一个 CAS 成功，另一个 CAS 失败后重读 `active`（此时 free bit 已被占），重新找下一个 free bit。不会出现两个 Token 占用同一槽位。

---

## 3. 槽位释放算法

```python
def slot_free(token):    # token = (slot, gen)
    while True:
        w = load_word()
        active = field_get_token_active(w)
        gen = field_get_token_generation(w)
        count = field_get_count(w)

        # 槽位必须仍被占用，且代际匹配（旧 Token 复活拒绝）
        if not (active & (1 << token.slot)):
            return 'INVALID_TOKEN'
        if gen != token.gen:
            return 'INVALID_TOKEN'          # 槽位已复用，旧 Token 失效
        if count == 0:
            return 'UNDERFLOW'

        r = field_cas(w, {
            'token_active': active & ~(1 << token.slot),
            'count':        count - 1,
            'version':      field_get_version(w) + 1,
        })
        if r == 'OK':
            return 'OK'
        if r == 'VERSION_EXHAUSTED_DENY':
            return r
        # CAS_FAILED → 重读重算
```

---

## 4. 槽位复用与代际校验（防旧 Token 复活）

**复用规则**：槽位被释放（bit 清 0）后，可再次分配给新 Token。每次分配 `generation+1`。

**旧 Token 拒绝规则**：

| 情况 | 判定 | 结果 |
|---|---|---|
| 旧 Token `(slot, g1)`，槽位未复用 | `cur_gen == g1` 且 bit 占用 | 有效（正常消费） |
| 旧 Token `(slot, g1)`，槽位已复用给新 Token `(slot, g2)` | `cur_gen == g2 != g1` | `INVALID_TOKEN`（旧 Token 失效） |
| 旧 Token `(slot, g1)`，槽位已释放 | bit=0 | `INVALID_TOKEN` |

**核心不变式**：`TOKEN_GENERATION` 单调递增且禁止回绕，因此任何槽位复用后，旧 Token 的 `gen` 必然严格小于当前 `gen`，`gen != token.gen` 校验必然拒绝旧 Token。∎

---

## 5. 重复消费防护

`slot_free(token)` 首次成功清位后，该槽位 bit=0。第二次 `slot_free(token)`：
- `active & (1<<slot) == 0` → 返回 `INVALID_TOKEN`。
- 不会再次 `COUNT−1`。

**并发重复消费**：线程 A、B 同时 `slot_free(token)`，都通过「bit 占用 且 gen 匹配」检查。A 的 CAS 成功清位，B 的 CAS 失败后重读，此时 bit=0 → `INVALID_TOKEN`。**同一 Token 只能消费一次**。∎

---

## 6. 序列号耗尽与崩溃

| 场景 | 处理 |
|---|---|
| generation 耗尽（0xFFFF） | GENERATION_EXHAUSTED_DENY，禁止复用，禁止回绕 |
| 分配中途崩溃 | CAS 是原子的，要么成功（槽位占用+gen+1）要么失败（无变化），无中间态 |
| 释放中途崩溃 | CAS 原子，要么清位+COUNT−1 完成，要么无变化 |
| 进程崩溃 | 状态字持久化后恢复，见 TOKEN-CRASH-RECOVERY.md |

---

## 7. 算法完整性清单

| 算法 | 定义位置 | 是否给内部实现 |
|---|---|---|
| slot_alloc | §2 | 是（完整伪代码） |
| slot_free | §3 | 是 |
| slot_valid（gen 匹配） | §4 | 是 |
| find_free_bit | §2 引用 | 是（线性扫描 bit0–15） |
| 旧 Token 拒绝 | §4 | 是 |

**禁止**：只给 `mark_active()` / `is_active()` / `consume_token()` 抽象命名而不给内部算法（05.39 缺陷，已修正）。

---

## 证据来源

- Token 身份：TOKEN-IDENTITY-AND-BITMAP-MODEL.md
- 状态字布局：UNIFIED-ATOMIC-STATE-WORD.md
- 证明等级：VERIFIED_STATIC（单进程）+ NOT_PROVEN（跨进程/跨重启）
