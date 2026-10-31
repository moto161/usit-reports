# ATOMIC-STATE-WORD.md — 原子状态字完整修复（05.36）

证据等级：VERIFIED_STATIC（位运算数学验证）
适用边界：Fork 冻结协议、准入/退出、DENIED 不可逆保护
修订基线：05.35 全文档使用 `COUNT_MASK=0x7FFE`（错误，遗漏 bit15）

---

## 1. 错误定位与根因

05.35 全文档将 bit1–15 的 15 位计数器掩码定义为 `0x7FFE`，并在主文档、`ATOMIC-STATE-WORD.md`、`STATE-TRANSITION-MATRIX.md`、`CROSS-DOC-CONSISTENCY.md`、`EXECUTION-REPORT.md` 及 `COUNTEREXAMPLES.json`（`B35-MASK-001` 及其后续条目）中一致地传播了该错误。

### 数学事实（已验证）

```
bit1..bit15 全置 1 的掩码 = Σ(1<<i for i in 1..15) = 0xFFFE = 65534
0x7FFE = 0b0111_1111_1111_1110  → bit15 = 0（仅 14 位，遗漏 bit15）
0xFFFE = 0b1111_1111_1111_1110  → bit15 = 1（完整 15 位）
```

### 错误触发场景（真实计数 = 16384）

```
真实计数 = 16384
存储字段 = 16384 << 1 = 0x8000
0x7FFE 提取 → (0x8000 & 0x7FFE) >> 1 = 0      ← 错误：非零计数被识别为 0
0xFFFE 提取 → (0x8000 & 0xFFFE) >> 1 = 16384  ← 正确
```

后果：任何 `count ∈ [16384, 32767]` 的准入计数，在 05.35 的 `0x7FFE` 掩码下，其 bit15 被掩掉，导致 `get_count()` 返回的值偏小；当 count=16384 时直接归零，`_wait_admit_drain()` 会在仍有 16384 个准入未退出时误判「已排空」，进而允许 Fork 在业务未停止时进行。这是**安全关键缺陷**。

---

## 2. 修正后的位布局（权威定义）

| 字段 | 位 | 掩码 | 移位 | 合法取值 |
|---|---|---|---|---|
| gate | bit0 | `0x0001` | 0 | 0=CLOSED, 1=OPEN |
| admit_count | bit1–15 | **`0xFFFE`** | 1 | 0..32767（15 位全用） |
| frozen | bit16 | `0x10000` | 16 | 0/1 |
| DENIED | bit17 | `0x20000` | 17 | 0/1（置位后不可逆） |

```python
GATE_MASK   = 0x00001
COUNT_MASK  = 0x0FFFE   # bit1..15 全 15 位（05.36 修正）
COUNT_SHIFT = 1
COUNT_MAX   = 32767
FROZEN_MASK = 0x10000
DENIED_MASK = 0x20000

def get_gate(s):   return (s & GATE_MASK) != 0
def get_count(s):  return (s & COUNT_MASK) >> COUNT_SHIFT
def get_frozen(s): return (s & FROZEN_MASK) != 0
def get_denied(s): return (s & DENIED_MASK) != 0
```

**位布局不变，唯一修正点是 `COUNT_MASK` 从 `0x7FFE` → `0xFFFE`。** 其余字段掩码、移位与取值不变。

---

## 3. 字段操作规范（field-aware CAS）

禁止裸 `state + 1` / `state - 1`。所有跨字段安全操作必须走 field-aware CAS：

```python
def cas(old, new):
    """原子比较交换。返回 True 表示成功。"""
    ...

def field_cas(expected, *, gate=None, count_delta=0, frozen=None, denied=False):
    """构造 field-aware 新值，只改变指定字段，其余字段原样保留。"""
    while True:
        cur = load()
        if cur != expected:
            return False  # 状态已变，由调用方重试或降级
        nv = cur
        if gate is not None:
            nv = (nv & ~GATE_MASK) | (GATE_MASK if gate else 0)
        if count_delta != 0:
            c = (nv & COUNT_MASK) >> COUNT_SHIFT
            nc = c + count_delta
            if nc < 0 or nc > COUNT_MAX:
                return 'OVERFLOW' if nc > COUNT_MAX else 'UNDERFLOW'
            nv = (nv & ~COUNT_MASK) | (nc << COUNT_SHIFT)
        if frozen is not None:
            nv = (nv & ~FROZEN_MASK) | (FROZEN_MASK if frozen else 0)
        if denied:
            nv |= DENIED_MASK
        if cas(cur, nv):
            return True
```

关键点：
- 计数步进单位为 `1 << COUNT_SHIFT`（=2），不是 `+1`。
- 字段更新先做**掩码清除**再做**或运算写入**，保证不污染 gate / frozen / DENIED。
- CAS 失败后**重新读取 `cur`** 再判断，不得用陈旧 expected 继续循环造成活锁（见 §5）。

---

## 4. 各操作的线性化点

| 操作 | 线性化点 | 前置条件（同一次 CAS 内检查） |
|---|---|---|
| 准入 `admit()` | `cas(gate==OPEN 且 count<COUNT_MAX → count+1)` 成功 | gate=OPEN, count<COUNT_MAX |
| 退出 `release()` | `cas(count→count-1)` 成功 | count>0 |
| 冻结 `freeze()` | `cas(gate=OPEN→CLOSED 且 frozen=0→1)` 成功 | gate=OPEN |
| 解冻 `unfreeze()` | `cas(frozen=1→0)` 成功 | **DENIED_MASK==0**（否则拒绝） |
| 设置 DENIED | `cas(denied=1)` 成功 | 无（DENIED 可与任意合法态叠加） |

**准入的原子性**：`admit()` 必须在**同一个 CAS 内**同时完成「检查 gate==OPEN」和「count+1」，不允许先读 gate、后单独加 count。这是 05.33 已确立、05.36 继续继承的线性化点约束。

---

## 5. 边界值验证（十六进制 + 二进制）

| 状态转换 | 计数 | 存储字段（count<<1） | `0xFFFE` 提取 |
|---|---|---|---|
| 0 → 1 | 1 | 0x0002 | 1 ✓ |
| 16382 → 16383 | 16383 | 0x7FFE | 16383 ✓ |
| 16383 → 16384 | 16384 | 0x8000 | 16384 ✓（05.35 此处返回 0 ✗） |
| 16384 → 16385 | 16385 | 0x8002 | 16385 ✓ |
| 32766 → 32767 | 32767 | 0xFFFE | 32767 ✓ |
| 32767 → 新准入 | — | — | 拒绝（count==COUNT_MAX） |
| 1 → 0 | 0 | 0x0000 | 0 ✓ |
| 0 → 再次退出 | — | — | 拒绝（UNDERFLOW） |

**结论：修复后（`0xFFFE`）任何非零准入计数都不可能被提取为 0。** `get_count()` 返回 0 当且仅当 bit1–15 全为 0，即真实计数确为 0。

---

## 6. 溢出 / 下溢 / 非法组合 / CAS 失败

- **溢出**：`count == COUNT_MAX(32767)` 时 `admit()` 返回 `OVERFLOW`，拒绝新准入，Gate 可继续维持 OPEN 或由冻结者关闭，**不得回绕**。
- **下溢**：`count == 0` 时 `release()` 返回 `UNDERFLOW`，记录异常，**不得借位污染 frozen/DENIED**。
- **非法位**：gate/frozen/DENIED 字段取值超出 {0,1}、或 count 字段出现非法组合（如 count>COUNT_MAX），一律进入 ISOLATED，拒绝 Fork。
- **CAS 持续失败**：采用「重新读 `cur` → 重新判断前置条件 → 有限重试」策略。重试上限 `MAX_CAS_RETRY`（建议 8 次）。超过上限**不得默认放行**，必须返回 `CAS_LIVELOCK` 并进入安全降级（保持当前 Gate 状态，冻结者视为失败）。

---

## 7. 与 05.35 反例的修正关系

`COUNTEREXAMPLES.json` 中 `B35-MASK-001`（05.35 编号，值 16384 掩码提取）在 05.35 被错误标为 PASS。05.36 将其**重新判定为 FAIL（05.35 设计存在漏洞）**，并以修正后的 `0xFFFE` 重新验证为 PASS。不得沿用 05.35 的错误 PASS 结论——详见 `COUNTEREXAMPLE-RESULTS.md` 的「历史反例复核」章节。

---

## 8. 状态存储与生命周期（为 B36-04 铺底）

状态字 `_admit_state` 的存储介质、作用域、生命周期，其权威定义在本文件的 §8，并与 `CRASH-RECOVERY-PROTOCOL.md` 保持一致：

| 场景 | 介质 | 作用域 | 进程退出后 | 恢复规则 |
|---|---|---|---|---|
| 单进程单线程 | 进程内原子变量 | 进程内 | 随进程消失 | 由 `CRASH-RECOVERY-PROTOCOL.md` 定义，见该文档 |
| 单进程多线程 | 进程内原子变量（共享） | 进程内 | 随进程消失 | 同上 |
| 多进程（Fork） | 写时复制副本 | 每进程独立 | 各自消失 | 子进程继承副本，父进程不共享 |
| 跨进程共享 | 共享内存 / mmap | 跨进程 | **可能存活** | 见 B36-04，不得仅凭「某进程退出」推定归零 |

**强制规则**：跨进程共享场景下，进程退出**不**自动清零共享状态字；必须由恢复者证明「所有旧写入者已停止」后才能重建或归零，否则 DENY。此规则在 `CRASH-RECOVERY-PROTOCOL.md` 中展开并证明。

---

## 证据来源

- 位运算数学验证：本机 Python 3.12 执行（见 EXECUTION-REPORT.md §B36-01）
- 05.35 错误位置：`ATOMIC-STATE-WORD.md` 第 11/17/23 行、`COUNTEREXAMPLES.json` `B35-MASK-001`
- 证明等级：VERIFIED_STATIC（静态数学推演，非运行时测试）
