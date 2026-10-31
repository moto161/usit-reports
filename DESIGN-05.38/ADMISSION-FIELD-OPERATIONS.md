# ADMISSION-FIELD-OPERATIONS.md — 准入字段操作（05.38 修订）

证据等级：VERIFIED_STATIC
本文件为状态字字段原语与掩码的权威定义。原子条件 CAS 框架见 ATOMIC-CONDITION-CAS-PROTOCOL.md。

---

## 1. 字段定义（继承 05.36 修正，05.38 复核）

```python
GATE_MASK   = 0x00001
COUNT_MASK  = 0x0FFFE   # bit1..15（15 位全用）
COUNT_SHIFT = 1
COUNT_MAX   = 32767
FROZEN_MASK = 0x10000
DENIED_MASK = 0x20000
KNOWN_MASK  = GATE_MASK | COUNT_MASK | FROZEN_MASK | DENIED_MASK  # 0x3FFFF

def get_gate(s):   return (s & GATE_MASK) != 0
def get_count(s):  return (s & COUNT_MASK) >> COUNT_SHIFT
def get_frozen(s): return (s & FROZEN_MASK) != 0
def get_denied(s): return (s & DENIED_MASK) != 0
```

复核结论：`COUNT_MASK=0xFFFE` 正确（bit1–15 全 15 位）；`COUNT_SHIFT=1`；`COUNT_MAX=32767`。禁止整字加减与无条件状态覆盖。

---

## 2. 字段原语（纯函数，仅被 transform 调用）

```python
def field_set_count(s, c):
    """设置 count 字段，掩码隔离，禁止整字加减。"""
    return (s & ~COUNT_MASK) | ((c << COUNT_SHIFT) & COUNT_MASK)

def field_set_gate(s, g):
    return (s & ~GATE_MASK) | (GATE_MASK if g else 0)

def field_set_frozen(s, f):
    return (s & ~FROZEN_MASK) | (FROZEN_MASK if f else 0)

def field_set_denied(s):
    return s | DENIED_MASK            # 只置位，永不清除
```

**规则**：任何字段更新必须通过上述原语，禁止 `s + 1`、`s - 1`、`s + (1 << COUNT_SHIFT)` 等整字算术。

---

## 3. 字段隔离性证明（复核）

掩码两两无公共位（GATE=0x1、COUNT=0xFFFE、FROZEN=0x10000、DENIED=0x20000），因此：
- `field_set_count` 用 `& ~COUNT_MASK` + `| (c<<1)`，不触碰 gate/frozen/DENIED。
- `field_set_gate`/`field_set_frozen`/`field_set_denied` 同理互不干扰。
- `c` 的最大值 32767，`32767 << 1 = 0xFFFE` 恰在 COUNT_MASK 内，不进位到 FROZEN(bit16)。

∎

---

## 4. 各操作的谓词与变换（统一到 conditional_cas）

见 ATOMIC-CONDITION-CAS-PROTOCOL.md §3。此处列出操作矩阵：

| 操作 | 谓词 predicate(s) | 变换 transform(s) |
|---|---|---|
| admit(token) | ¬denied ∧ gate ∧ count<MAX | count+1（token 发放） |
| release(token) | token_valid ∧ count>0 | count−1（token 消耗） |
| cas_frozen | ¬denied ∧ ¬gate ∧ count==0 | frozen=1 |
| cas_unfreeze | ¬denied ∧ frozen | frozen=0 |

---

## 5. 准入令牌（修复 release 下溢）

05.37 的 `release()` 只检查 `count > 0` 就允许任意调用者减计数，存在「重复释放导致下溢」风险。05.38 引入准入令牌：

```python
admit_tokens = {}   # token -> count 份额（进程内，或跨进程共享）

def admit():
    r = conditional_cas(admit_predicate, admit_transform)
    if r == 'OK':
        token = new_token()
        admit_tokens[token] = 1
        return ('OK', token)
    return r

def release(token):
    def predicate(s):
        return (token in admit_tokens) and get_count(s) > 0
    def transform(s):
        return field_set_count(s, get_count(s) - 1)
    r = conditional_cas(predicate, transform)
    if r == 'OK':
        del admit_tokens[token]   # 令牌一次性消耗
    return r
```

- 每个 `admit()` 成功发放唯一 token。
- `release(token)` 校验 token 有效（一次性），防止重复释放/任意调用者减计数。
- token 无效或已消耗 → `release` 拒绝，COUNT 不下溢。

---

## 6. 非法状态字处理

`conditional_cas` 前置调用 `is_valid_state(s)`（见 ATOMIC-CONDITION-CAS-PROTOCOL.md §4）：
- 未知保留位非 0 → INVALID_STATE。
- COUNT > 32767 → INVALID_STATE。
- gate/frozen 字段非 0/1 → INVALID_STATE。
- 默认拒绝，不修改状态。

---

## 7. 原子 CAS 实现前提（跨进程）

进程内状态字：原子变量，CAS 成立。跨进程共享：需 HW CAS + 共享内存 + 内存序证明，否则 NOT_PROVEN（见 RECOVERY-VERSION-ATOMICITY.md §4）。

---

## 证据来源

- 字段掩码：继承 05.36，本轮复核
- 原子条件 CAS：ATOMIC-CONDITION-CAS-PROTOCOL.md
- 05.37 缺陷：release 无令牌、循环外检查
- 证明等级：VERIFIED_STATIC
