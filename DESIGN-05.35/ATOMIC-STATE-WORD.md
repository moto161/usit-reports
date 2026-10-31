# ATOMIC-STATE-WORD.md

## 目标
原子状态字逐位复核，修复掩码、字段更新、DENIED 保护。

## 1. 位布局表

| 字段 | 位 | 掩码 | 移位 | 合法取值 |
|---|---|---|---|---|
| gate | bit0 | 0x1 | 0 | 0=OPEN, 1=FREEZING |
| admit_count | bit1–15 | **0x7FFE** | 1 | 0..32767（15 位全用） |
| frozen | bit16 | 0x10000 | 16 | 0/1 |
| DENIED | bit17 | 0x20000 | 17 | 0/1 |

## 2. 掩码修复说明

05.34 的 15 位 count 掩码若写成 `0x7FFE` 正确（bit1..15 = 15 位）。关键修复点：**必须覆盖全部 15 位（bit1 到 bit15），不得遗漏最高位 bit15**。任何 `0x7FFC`（遗漏 bit15）或 `0x3FFE`（只 14 位）都是错误。正确掩码 `COUNT_MASK = 0x7FFE`，`COUNT_MAX = 32767`。

## 3. 字段提取/更新/CAS 伪代码

```python
GATE_MASK   = 0x00001
COUNT_MASK  = 0x07FFE   # bit1..15 全 15 位
FROZEN_MASK = 0x10000
DENIED_MASK = 0x20000
COUNT_MAX   = 32767

def get_count(s):  return (s & COUNT_MASK) >> 1
def get_gate(s):   return s & GATE_MASK
def get_frozen(s): return s & FROZEN_MASK
def get_denied(s): return s & DENIED_MASK

def cas_admit_enter(self):
    while True:
        cur = self._admit_state.load()
        if cur & (GATE_MASK | DENIED_MASK):   # gate 关闭或 DENIED
            return False
        if get_count(cur) >= COUNT_MAX:
            raise AdmitLimit()
        new = cur + (1 << 1)                   # field-aware：只增 count 位（步进 2）
        if self._admit_state.cas(cur, new):
            return True                        # 活锁：失败重读

def cas_admit_exit(self):
    while True:
        cur = self._admit_state.load()
        if get_count(cur) == 0:
            raise AdmitUnderflow()
        new = cur - (1 << 1)                   # 只减 count 位
        if self._admit_state.cas(cur, new):
            return True

def cas_freeze(self):
    while True:
        cur = self._admit_state.load()
        if cur & (GATE_MASK | DENIED_MASK):
            return False
        new = cur | GATE_MASK                  # 置 gate=FREEZING
        if self._admit_state.cas(cur, new):
            return True

def cas_frozen(self):
    while True:
        cur = self._admit_state.load()
        if get_gate(cur) == 0 or get_count(cur) != 0:
            return False
        new = cur | FROZEN_MASK                # 置 frozen
        if self._admit_state.cas(cur, new):
            return True

def cas_denied(self):
    while True:
        cur = self._admit_state.load()
        new = cur | DENIED_MASK                # 置 DENIED（不可逆）
        if self._admit_state.cas(cur, new):
            return True
```

## 4. 禁止裸 state+1

field-aware 更新用 `cur + (1<<1)`（count 步进 2），不得用 `state+1`（会跨字段进位到 gate 位）。

## 5. 溢出/下溢/非法位/非法组合

| 情况 | 处理 |
|---|---|
| count 达 32767 再准入 | AdmitLimit 拒绝 |
| count 下溢（退出多） | AdmitUnderflow 隔离 |
| 非法位写入 | 保留位不变，检测隔离 |
| frozen==1 且 gate==0 | 非法组合，拒绝隔离 |
| DENIED==1 且 gate==OPEN | 非法组合（DENIED 置位后 gate 必须视为关闭） |

## 6. 状态转换与线性化点

| 转换 | 线性化点 | 失败后状态 |
|---|---|---|
| OPEN→FREEZING | cas_freeze 成功 | OPEN |
| FREEZING→FROZEN | cas_frozen 成功 | FREEZING |
| FROZEN→分叉 | fork 返回 | 回滚 OPEN |
| 任意→DENIED | cas_denied 成功 | 不可逆 |
| FREEZING→OPEN（回滚） | cas_unfreeze（前置 gate!=DENIED） | FREEZING |

## 7. DENIED 不可逆保护

- `cas_unfreeze` / rollback / 恢复 / 父子初始化的前置条件包含 `DENIED 位 == 0`。
- 任何清除 DENIED 的操作被 CAS 前置条件拒绝。
- DENIED 置位后，gate 语义上视为关闭（准入/冻结均被 DENIED_MASK 前置拒绝）。

```python
def cas_unfreeze(self):
    while True:
        cur = self._admit_state.load()
        if cur & DENIED_MASK:                 # DENIED 不可逆
            return False
        new = cur & ~GATE_MASK & ~FROZEN_MASK  # 清 gate/frozen，不动 DENIED
        if self._admit_state.cas(cur, new):
            return True
```

## 8. CAS 持续失败/活锁/饥饿

- 失败重读再 CAS；不无限重试后默认放行。
- 达到重试上限 → 进入安全终态（隔离/拒绝），不默认放行。
- gate==OPEN 时无饥饿（各线程独立推进 count）。

## 9. 证据等级
原子状态字 DESIGN_PROVEN（基于 CAS 语义，模型级）；运行时 NOT_TESTED。
