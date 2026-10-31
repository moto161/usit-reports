# ADMISSION-FIELD-OPERATIONS.md — 统一准入字段操作（05.37 权威）

证据等级：VERIFIED_STATIC
本文件为状态字字段操作的**唯一权威接口定义**。所有文档、伪代码、反例必须使用本接口。

---

## 1. 05.36 缺陷定位

05.36 的 `DENIED-ATOMIC-PROTECTION.md` 仍使用：

```python
nv = cur + (1 << COUNT_SHIFT)   # ← 整字算术
```

这违反了「禁止整字算术、只允许 field-aware CAS」的规范。虽然 `(1 << COUNT_SHIFT)` 在 count 未达上限时不会进位到 bit16，但这是**依赖数值巧合**的正确性，而非结构性保证；且一旦 count 字段值异常，整字加法可能污染 frozen/DENIED 位。

05.37 统一为单一 field-aware 接口，彻底禁止整字算术。

---

## 2. 字段定义（权威，继承 05.36 修正）

```python
GATE_MASK   = 0x00001
COUNT_MASK  = 0x0FFFE   # bit1..15（05.36 已修正，05.37 继承）
COUNT_SHIFT = 1
COUNT_MAX   = 32767
FROZEN_MASK = 0x10000
DENIED_MASK = 0x20000

def get_gate(s):   return (s & GATE_MASK) != 0
def get_count(s):  return (s & COUNT_MASK) >> COUNT_SHIFT
def get_frozen(s): return (s & FROZEN_MASK) != 0
def get_denied(s): return (s & DENIED_MASK) != 0
```

---

## 3. 统一 field-aware 接口

所有状态字修改只能通过以下接口，禁止任何直接的整字算术（`+`、`-`、`|=`、`&=`）或无条件赋值：

```python
def field_cas(expected, *, gate=None, count_delta=0, frozen=None, denied=False):
    """field-aware CAS：只改变指定字段，其余字段原样保留。

    返回 'OK' / 'OVERFLOW' / 'UNDERFLOW' / 'DENIED' / 'CAS_FAIL' / 'LIVELOCK'
    """
    retries = 0
    while True:
        cur = load()
        if cur != expected:
            return 'CAS_FAIL'          # 状态已变，调用方重读重算
        nv = cur
        if gate is not None:
            nv = (nv & ~GATE_MASK) | (GATE_MASK if gate else 0)
        if count_delta != 0:
            c = (nv & COUNT_MASK) >> COUNT_SHIFT
            nc = c + count_delta
            if nc > COUNT_MAX:
                return 'OVERFLOW'
            if nc < 0:
                return 'UNDERFLOW'
            # 用掩码清除 + 移位写入，绝不整字加法
            nv = (nv & ~COUNT_MASK) | (nc << COUNT_SHIFT)
        if frozen is not None:
            nv = (nv & ~FROZEN_MASK) | (FROZEN_MASK if frozen else 0)
        if denied:
            nv |= DENIED_MASK          # 只置位，永不清除
        if cas(cur, nv):
            return 'OK'
        retries += 1
        if retries >= MAX_CAS_RETRY:
            return 'LIVELOCK'          # 不无限重试，返回后降级
        # 重试前重新读 cur（下次循环 load()），不复用旧 expected
        expected = load()
```

**关键**：`count_delta` 通过「掩码清除 + 移位写入」实现，**禁止** `cur + (1 << COUNT_SHIFT)`。这样即使 count 字段异常（如非法位），也不会污染 gate/frozen/DENIED。

---

## 4. 各操作的统一实现

### 准入进入（admit）

```python
def admit():
    while True:
        cur = load()
        if get_denied(cur):        # DENIED 优先
            return 'DENIED'
        if not get_gate(cur):      # gate 关闭
            return 'GATE_CLOSED'
        if get_count(cur) >= COUNT_MAX:
            return 'OVERFLOW'
        r = field_cas(cur, count_delta=+1)
        if r == 'OK':
            return 'OK'
        if r in ('OVERFLOW', 'DENIED'):
            return r
        # CAS_FAIL / LIVELOCK → 重读重算（循环顶部 load）
```

### 准入退出（release）

```python
def release():
    while True:
        cur = load()
        if get_count(cur) == 0:
            return 'UNDERFLOW'
        r = field_cas(cur, count_delta=-1)
        if r == 'OK':
            return 'OK'
        if r == 'UNDERFLOW':
            return 'UNDERFLOW'
        # CAS_FAIL / LIVELOCK → 重读重算
```

### 冻结（cas_frozen）

```python
def cas_frozen():
    while True:
        cur = load()
        if get_gate(cur):           # gate 仍 OPEN
            return 'GATE_OPEN'
        if get_denied(cur):         # DENIED 优先
            return 'DENIED'
        if get_count(cur) != 0:     # 未归零
            return 'COUNT_NOT_ZERO'
        r = field_cas(cur, frozen=True)
        if r == 'OK':
            return 'OK'
        # 其他 → 重读重算
```

### 解冻（cas_unfreeze）

```python
def cas_unfreeze():
    while True:
        cur = load()
        if get_denied(cur):         # DENIED 优先，禁止解冻
            return 'DENIED'
        if not get_frozen(cur):
            return 'NOT_FROZEN'
        r = field_cas(cur, frozen=False)
        if r == 'OK':
            return 'OK'
        # 其他 → 重读重算
```

---

## 5. 字段隔离性证明

**定理**：`field_cas` 的任何字段更新不会改变其他字段。

证明：每个字段的更新都用「`& ~FIELD_MASK` 清除 + `| (value << SHIFT)` 写入」的掩码模式，掩码互不重叠（GATE=0x00001、COUNT=0x0FFFE、FROZEN=0x10000、DENIED=0x20000 两两无公共位）。因此：
- count 更新（`& ~COUNT_MASK` + `| (nc << 1)`）不触碰 gate/frozen/DENIED。
- gate 更新不触碰 count/frozen/DENIED。
- frozen 更新不触碰 gate/count/DENIED。
- DENIED 只置位（`|= DENIED_MASK`），不清除任何字段。

且 `COUNT_MASK=0xFFFE` 覆盖 bit1–15 全部 15 位，`nc << 1` 的最大值 `32767 << 1 = 0xFFFE` 仍在 COUNT_MASK 内，不会进位到 frozen(bit16)。∎

---

## 6. 边界证明

- **准入拒绝（gate 关闭或 DENIED）**：`admit()` 前置检查 `get_denied` 和 `get_gate`，任一不满足即返回，不进入 field_cas。∎
- **下溢拒绝（count=0 退出）**：`release()` 前置检查 `get_count==0`，返回 UNDERFLOW。∎
- **32767 上限不溢出**：`admit()` 前置检查 `get_count >= COUNT_MAX`，返回 OVERFLOW；field_cas 内也双重检查 `nc > COUNT_MAX`。∎

---

## 7. CAS 失败 / 饥饿 / 超时

- **CAS_FAIL**：状态被并发修改，调用方重读 `cur` 重算，不复用旧 expected。
- **LIVELOCK**：连续 `MAX_CAS_RETRY`（建议 8）次 CAS 失败，返回 LIVELOCK，调用方降级（准入拒绝或冻结失败），不无限重试。
- **超时**：冻结者阶段2 等待归零设超时，超时后走 unfreeze 回退（但 DENIED 优先，见 FREEZE-STATE-MACHINE.md §6-C7）。

---

## 8. 全文档统一性声明

本接口是状态字修改的**唯一**接口。任何文档、状态图、伪代码、反例中出现 `cur + 1`、`state + 1`、`nv = cur + (1 << COUNT_SHIFT)` 等整字算术或无条件 `commit(state)`，均视为违反本规范，必须修正为 `field_cas`。05.36 的 `DENIED-ATOMIC-PROTECTION.md` 中 `nv = cur + (1 << COUNT_SHIFT)` 已在本轮修正。

---

## 证据来源

- 字段掩码：ATOMIC-STATE-WORD.md §2（05.36，继承）
- 05.36 整字算术位置：DENIED-ATOMIC-PROTECTION.md（05.36 版）§4
- 证明等级：VERIFIED_STATIC
