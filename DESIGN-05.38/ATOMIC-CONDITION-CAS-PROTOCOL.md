# ATOMIC-CONDITION-CAS-PROTOCOL.md — 原子条件 CAS 协议（05.38 权威）

证据等级：VERIFIED_STATIC
目标：修复 05.37 B37-R02——安全前置条件只在 CAS 循环外检查一次，提交点无原子保障。

---

## 1. 05.37 缺陷定位

05.37 的 `admit()`/`cas_frozen()` 等在 CAS 循环外检查 DENIED/GATE/COUNT，然后进入 `field_cas`。问题：

```python
def admit():                      # 05.37 缺陷
    while True:
        cur = load()
        if get_denied(cur): return 'DENIED'     # 循环外检查
        if not get_gate(cur): return 'GATE_CLOSED'
        if get_count(cur) >= COUNT_MAX: return 'OVERFLOW'
        r = field_cas(cur, count_delta=+1)       # field_cas 只改 count，不重查 DENIED/GATE
        ...
```

`field_cas` 内部只校验 `cur == expected`，**不校验 DENIED/GATE**。因此：读 `cur`（DENIED=0, GATE=OPEN）→ 并发置 DENIED=1 → `field_cas` 仍可能成功（若 count 字段未变），导致 DENIED 被绕过准入。

---

## 2. 修复原则：谓词纳入 CAS

安全前置条件必须作为**谓词**纳入原子 CAS，禁止「CAS 循环外检查一次 + 无条件 field 更新」。

统一接口：

```python
def conditional_cas(predicate, transform):
    """原子执行：仅当 predicate(state) 为真时，才提交 transform(state)。

    谓词与变换在同一个 CAS 内原子评估，杜绝「检查→提交」之间的竞争窗口。
    返回 'OK' / 'PREDICATE_FALSE' / 'CAS_FAIL' / 'LIVELOCK' / 'TIMEOUT'
    """
    retries = 0
    while True:
        cur = load()
        if not predicate(cur):
            return 'PREDICATE_FALSE'      # 前置条件不满足（含 DENIED/GATE/COUNT）
        new_state = transform(cur)        # 纯函数，基于最新 cur 计算
        if cas(cur, new_state):
            return 'OK'
        retries += 1
        if retries >= MAX_CAS_RETRY:
            return 'LIVELOCK'
        # CAS 失败 → 重新读 cur，重新评估 predicate（不复用旧检查结果）
```

**关键**：`predicate(cur)` 和 `transform(cur)` 都基于**同一次 `load()` 读到的 `cur`**，且 `cas(cur, new_state)` 校验状态未被并发修改。若并发修改使 CAS 失败，下一轮重新 `load()` 并重新评估谓词——**谓词结果永不过期**。

---

## 3. 各操作的谓词与变换

### admit（准入）

```python
def admit(token):
    def predicate(s):
        return (not get_denied(s)) and get_gate(s) and (get_count(s) < COUNT_MAX)
    def transform(s):
        return field_set_count(s, get_count(s) + 1)   # 掩码隔离，禁止整字加
    return conditional_cas(predicate, transform)
```

提交时保证：DENIED=0 且 GATE=OPEN 且 COUNT<MAX，三者原子成立。

### release（退出，含准入令牌）

```python
def release(token):
    def predicate(s):
        return token_valid(token) and get_count(s) > 0
    def transform(s):
        return field_set_count(s, get_count(s) - 1)
    return conditional_cas(predicate, transform)
```

**准入令牌**：每个成功 `admit()` 返回唯一 token，`release()` 必须携带有效 token。防止任意调用者减计数导致下溢（不能仅凭 COUNT>0 就允许任意 release）。

### cas_frozen（冻结完成）

```python
def cas_frozen():
    def predicate(s):
        return (not get_denied(s)) and (not get_gate(s)) and (get_count(s) == 0)
    def transform(s):
        return s | FROZEN_MASK
    return conditional_cas(predicate, transform)
```

提交时保证：DENIED=0 且 GATE=CLOSED 且 COUNT=0。

### cas_unfreeze（解冻）

```python
def cas_unfreeze():
    def predicate(s):
        return (not get_denied(s)) and get_frozen(s)
    def transform(s):
        return s & ~FROZEN_MASK
    return conditional_cas(predicate, transform)
```

DENIED 一旦置位，predicate 恒假，解冻不可清除或绕过 DENIED。

---

## 4. 非法状态字处理（默认拒绝）

```python
def is_valid_state(s):
    # 未知保留位必须为 0
    if (s & ~(GATE_MASK | COUNT_MASK | FROZEN_MASK | DENIED_MASK)) != 0:
        return False
    # COUNT 字段合法范围
    if get_count(s) > COUNT_MAX:
        return False
    # DENIED 与其他状态冲突检测（DENIED 可与任何合法态叠加，但 gate/frozen 必须 0/1）
    if (s & GATE_MASK) not in (0, GATE_MASK):
        return False
    if (s & FROZEN_MASK) not in (0, FROZEN_MASK):
        return False
    return True

# conditional_cas 前置：state 非法 → 默认拒绝（不得放行）
```

非法状态字：返回 `INVALID_STATE`，操作拒绝，不修改状态。

---

## 5. 重试终止与降级

- `MAX_CAS_RETRY`（建议 8）：连续 CAS 失败达到上限 → `LIVELOCK`，**终止当前操作**，安全降级（准入拒绝 / 冻结失败 / 解冻拒绝）。
- 超时（可选 `TIMEOUT` 参数）：超时返回 `TIMEOUT`，终止。
- **禁止外层 `while True` 无限重试**。所有重试由 `conditional_cas` 内部统一控制上限。

```python
# 错误示范（05.37）：外层无限重试
while True:
    r = field_cas(...)
    if r == 'OK': break
    # 无上限 → 活锁

# 正确：conditional_cas 内部统一上限，外层只调用一次
r = conditional_cas(predicate, transform)
```

---

## 6. 原子 CAS 实现前提

`conditional_cas` 声称的原子性依赖以下前提，**不满足时不得声称跨进程安全**：

| 前提 | 要求 | 不满足后果 |
|---|---|---|
| 状态字宽度 | 与平台原子类型匹配（如 32/64 位） | 非原子读写 → 撕裂 |
| 对齐 | 自然对齐 | 非对齐 → 非原子 |
| 内存序 | 至少 acquire-release | 重排 → 竞态 |
| 共享范围 | 明确是进程内/跨进程 | 跨进程需共享内存 + 原子指令 |
| 跨进程原子性 | 需 HW CAS（如 CMPXCHG）+ 共享内存 | 无 HW 支持 → NOT_PROVEN |

**本设计明确**：状态字为进程内原子变量时，CAS 原子性成立；跨进程共享场景需额外证明 HW 原子性，未证明则 NOT_PROVEN（见 RECOVERY-VERSION-ATOMICITY.md）。

---

## 7. 与 ADMISSION-FIELD-OPERATIONS.md 的关系

- `ADMISSION-FIELD-OPERATIONS.md` 定义字段掩码与 field 更新原语（`field_set_count` 等）。
- 本文件定义「谓词 + 变换」的原子 CAS 框架，**取代** 05.37 的「循环外检查 + field_cas」模式。
- 二者一致：`transform` 内部只能调用 field 原语，禁止整字算术。

---

## 证据来源

- 字段原语：ADMISSION-FIELD-OPERATIONS.md（05.38 修订）
- 05.37 缺陷：ADMISSION-FIELD-OPERATIONS.md（05.37 版）的 admit/release 循环外检查
- 证明等级：VERIFIED_STATIC
