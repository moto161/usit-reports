# TOKEN-ATOMIC-TRANSACTION.md — Token 与 COUNT 原子事务（05.40 权威）

证据等级：VERIFIED_STATIC（单进程）+ NOT_PROVEN（跨进程/跨重启）
目标：修复 B39-R01——Token 发放/消费与 COUNT 更新必须在同一原子提交边界，消除崩溃窗口与并发重复消费。

---

## 1. 05.39 缺陷定位（B39-R01 完整）

05.39 的 Token 身份用 32 位 `token_seq` + 32 位 `active_mask`，缺陷：

1. **崩溃窗口**：`COUNT+1` 与 `token_seq+1`/`active_mask` 更新在 128 位 CAS 内，但 `active_mask` 单个 bit 不携带身份，崩溃后无法区分「哪个 Token 对应哪个 bit」。
2. **并发重复消费**：`is_active(active, token)` 只查 bit 是否置位，不查 token 身份（因为 bit 不编码身份）。两个线程用不同 token 值指向同一 bit 时，都可能通过检查。
3. **槽位复用**：bit 复用后，旧 token 携带同 bit 位置即可被误判有效。

05.40 修正：Token 身份改为 `(slot, generation)`，generation 单调递增且内嵌状态字，使 Token 发放/消费与 COUNT 更新在同一 128 位 CAS 内原子完成，且身份唯一可验证。

---

## 2. 唯一权威事务模型

**结论：Token 与 COUNT 全部内嵌 128 位状态字，单条 `field_cas` 完成「身份发放/消费 + COUNT 更新」的原子提交。无独立账本，无 WAL。**

05.39 的「方案 B（独立账本 + WAL）」因 128 位状态字已能容纳 Token 身份而**废弃**。不再存在 `admit_tokens` 字典或任何独立于状态字的 Token 账本。

---

## 3. admit（原子：COUNT+1 + 槽位占用 + generation+1）

```python
def admit():
    while True:
        w = load_word()
        state_fields = decode(w)          # count/gate/denied/frozen/freezing
        active = field_get_token_active(w)
        gen = field_get_token_generation(w)
        ver = field_get_version(w)

        # 前置条件（提交点原子校验，见 §5）
        if get_denied(state_fields):
            return 'DENIED'
        if not get_gate(state_fields):
            return 'GATE_CLOSED'
        if get_count(state_fields) >= COUNT_MAX:
            return 'OVERFLOW'
        if ver == 0xFFFFFFFF:
            return 'VERSION_EXHAUSTED_DENY'
        if gen == 0xFFFF:
            return 'GENERATION_EXHAUSTED_DENY'

        free = find_free_bit(active)
        if free is None:
            return 'NO_SLOT'

        r = field_cas(w, {
            'count':            get_count(state_fields) + 1,
            'token_active':     active | (1 << free),
            'token_generation': gen + 1,
            'version':          ver + 1,
        })
        if r == 'OK':
            return ('OK', (free, gen + 1))   # 线性化点：COUNT+1、槽位占用、gen+1 原子完成
        if r in ('VERSION_EXHAUSTED_DENY', 'INVALID_STATE'):
            return r
        # CAS_FAILED → 重读重算
```

**关键**：`COUNT+1`、槽位 `free` 置位、`generation+1`、`version+1` 四者**同一条 128 位 CAS** 提交。不存在「COUNT 已改但 Token 未入账」或「Token 已发但 generation 未递增」的中间态。

---

## 4. release（原子：身份校验 + COUNT−1 + 槽位清位）

```python
def release(token):    # token = (slot, gen)
    while True:
        w = load_word()
        active = field_get_token_active(w)
        gen = field_get_token_generation(w)
        count = get_count(decode(w))

        # 身份校验（原子，非独立检查）
        if not (active & (1 << token.slot)):
            return 'INVALID_TOKEN'
        if gen != token.gen:
            return 'INVALID_TOKEN'          # 旧 Token 复活拒绝
        if count == 0:
            return 'UNDERFLOW'
        if field_get_version(w) == 0xFFFFFFFF:
            return 'VERSION_EXHAUSTED_DENY'

        r = field_cas(w, {
            'token_active': active & ~(1 << token.slot),
            'count':        count - 1,
            'version':      field_get_version(w) + 1,
        })
        if r == 'OK':
            return 'OK'                     # 线性化点：校验+COUNT−1+清位原子完成
        if r == 'VERSION_EXHAUSTED_DENY':
            return r
        # CAS_FAILED → 重读重算
```

**关键**：`(active & (1<<slot))` 与 `gen == token.gen` 两个校验，加上 `COUNT−1` 与 `clear bit`，全部在同一 CAS 内。并发重复消费：

- 线程 A CAS 成功：槽位清位，COUNT−1。
- 线程 B 重读 `w`：`active & (1<<slot) == 0` → `INVALID_TOKEN`。
- **同一 Token 只能消费一次**。∎

---

## 5. 前置条件纳入原子 CAS（B39-R02 修复）

05.38 的缺陷是「在 CAS 循环外检查一次前置条件，再 CAS」。05.40 修正：**所有前置条件在 field_cas 提交点原子校验**。

实现方式：`field_cas` 的 `expected.word` 就是调用者刚读到的 `w`。CAS 比较整个 128 位 `w`，只有当前状态字仍等于 `w`（即所有前置条件依赖的字段未变）才提交。因此：

- DENIED/GATE/COUNT/version/generation/active 在提交点与 `w` 一致时才提交。
- 任何字段在 CAS 前被并发修改，CAS 失败，调用者重读重算。
- 禁止复用旧 `w` 的字段值（旧 DENIED 检查结果）继续提交。

---

## 6. 线性化点汇总

| 操作 | 线性化点 |
|---|---|
| admit | 128 位 field_cas 成功（COUNT+1 + 槽位占用 + gen+1） |
| release | 128 位 field_cas 成功（身份校验 + COUNT−1 + 清位） |
| token_create | 128 位 field_cas 成功（槽位占用 + gen+1） |
| token_consume | 128 位 field_cas 成功（身份校验 + 清位） |
| token_revoke | 128 位 field_cas 成功（身份校验 + 清位） |

---

## 7. 崩溃与恢复

内嵌模型下，Token 账本在状态字内，崩溃后随状态字一起持久化/恢复，无「COUNT 与 Token 分离」问题。恢复规则见 TOKEN-CRASH-RECOVERY.md。

- 跨重启：状态字 128 位持久化未证明 → generation 可能回退 → 旧 Token 复活风险 → NOT_PROVEN → DENY。

---

## 证据来源

- Token 身份：TOKEN-IDENTITY-AND-BITMAP-MODEL.md
- 槽位复用：TOKEN-SLOT-ALLOCATION-AND-REUSE.md
- 状态字布局：UNIFIED-ATOMIC-STATE-WORD.md
- 05.39 缺陷：TOKEN-ATOMIC-TRANSACTION.md（05.39 版）§3 active_mask 无代际
- 证明等级：VERIFIED_STATIC（单进程）+ NOT_PROVEN（跨进程/跨重启）
