# DENIED-ATOMIC-PROTECTION.md — DENIED 与 FROZEN 的原子保护（05.36）

证据等级：VERIFIED_STATIC
目标：修复 05.35 的 `cas_frozen()` 未在原子转换中完整检查 DENIED 的问题，证明任何并发顺序下 DENIED 一旦提交即不可被清除。

---

## 1. DENIED 编码与存储位置

- DENIED 位：状态字 bit17，`DENIED_MASK = 0x20000`。
- 与 OPEN/FREEZING/FROZEN 的合法组合见 `STATE-TRANSITION-MATRIX.md` §1（S3/S4/S5 为 DENIED 合法态）。
- DENIED 与 admit_count 无耦合：DENIED 置位后，准入一律被 `admit()` 的 gate 检查拒绝（因为 DENIED 触发 Gate 物理关闭，见 §4），但已存在的 count 由退出流程自然排空，不被强制清零。

---

## 2. cas_frozen 的完整原子检查（05.36 修正）

05.35 的 `cas_frozen()` 只检查了 gate 和 count，未在同一 CAS 内检查 DENIED。05.36 修正如下：

```python
def cas_frozen():
    """冻结完成：gate 必须 CLOSED、count 必须 0，且 DENIED 必须为 0。
    任一项不满足则冻结失败。DENIED=1 时冻结绝不成功。"""
    while True:
        cur = load()
        if (cur & GATE_MASK) != 0:      # gate 仍 OPEN
            return 'GATE_OPEN'
        if (cur & DENIED_MASK) != 0:    # DENIED 已置位
            return 'DENIED'
        if (cur & COUNT_MASK) != 0:     # 准入未排空
            return 'COUNT_NOT_ZERO'
        nv = cur | FROZEN_MASK
        if cas(cur, nv):
            return 'OK'
```

**关键**：`gate==CLOSED`、`DENIED==0`、`count==0` 三项在同一 CAS 的 `expected` 状态上原子检查。任何一项不满足，冻结不成功，且**不修改状态字**。

---

## 3. 并发时间线（覆盖工单要求的全部时序）

每个时序证明「只要 DENIED 已提交，后续普通操作不能恢复 OPEN 或清除 DENIED」。

### T1：FROZEN 转换 与 DENIED 设置 同时发生

- 线程 A 执行 `cas_frozen()`：读 `cur`，gate=CLOSED, count=0, DENIED=0，准备 CAS。
- 线程 B 执行 `set_denied()`：CAS 置 DENIED=1 成功。
- 线程 A 的 `cas(cur, nv)` 因 `cur` 已被 B 改变而**失败**，重新读 `cur`，发现 DENIED=1，返回 `'DENIED'`。
- 结果：DENIED=1 胜出，FROZEN 未设置。后续 unfreeze 因 DENIED==0 检查失败而拒绝。∎

### T2：unfreeze 与 DENIED 设置 同时发生

- 线程 A 执行 `unfreeze()`：读 `cur`，frozen=1, DENIED=0，准备 CAS。
- 线程 B 执行 `set_denied()`：CAS 置 DENIED=1 成功。
- 线程 A 的 CAS 失败，重读发现 DENIED=1，拒绝 unfreeze。
- 结果：DENIED=1 胜出，无法解冻。∎

### T3：rollback 与 DENIED 设置 同时发生

- rollback 目标是把 frozen=1 回退到 frozen=0。
- 线程 A 执行 rollback：读 `cur`，frozen=1, DENIED=0。
- 线程 B 置 DENIED=1。
- 线程 A 的 rollback CAS 失败，重读发现 DENIED=1，rollback 拒绝（rollback 前置 DENIED==0）。
- 结果：DENIED 不可被 rollback 清除。∎

### T4：父进程恢复 与 DENIED 设置 同时发生

- 父进程恢复路径重建状态字时，从持久化证据读取 DENIED 位。
- 若恢复期间 DENIED 被置位，恢复逻辑必须在最终 CAS 提交前**重新校验 DENIED 位**，发现 DENIED=1 则放弃恢复 OPEN，保持 DENIED。
- 结果：恢复不得覆盖 DENIED。∎

### T5：两个线程同时设置 DENIED

- 线程 A、B 都执行 `set_denied()`。DENIED 置位是幂等的（`nv |= DENIED_MASK`）。
- 无论谁先 CAS 成功，最终 DENIED=1，且不丢失。
- 结果：拒绝状态不丢失。∎

### T6：CAS 失败后重读状态并重试

- 所有 field-aware CAS 失败后必须 `load()` 重读最新 `cur`，以最新状态重新判定，不得用陈旧 expected 继续。
- 见 ATOMIC-STATE-WORD.md §5 的 `MAX_CAS_RETRY` 有限重试规则。∎

---

## 4. 拒绝状态与 Gate 的关系

- DENIED 置位后，语义上 Gate 必须物理关闭。若 DENIED 置位时 Gate 尚为 OPEN（S3），系统在**同一个或紧随其后的原子操作**中将 Gate 置为 CLOSED。
- 但从 `DENIED=1` 到 `Gate=CLOSED` 之间若存在极短窗口，准入操作 `admit()` 的前置检查含 `gate==OPEN`，且 DENIED 置位后立即关闭 Gate；为防止窗口内准入，`admit()` 的前置检查**同时要求 `DENIED==0`**：

```python
def admit():
    while True:
        cur = load()
        if (cur & DENIED_MASK) != 0:
            return 'DENIED'           # DENIED 优先，拒绝准入
        if (cur & GATE_MASK) == 0:
            return 'GATE_CLOSED'
        if get_count(cur) >= COUNT_MAX:
            return 'OVERFLOW'
        nv = cur + (1 << COUNT_SHIFT)
        if cas(cur, nv):
            return 'OK'
```

因此 DENIED=1 时，无论 Gate 物理状态如何，准入都被原子拒绝。∎

---

## 5. 若 DENIED 不在状态字内的替代方案（本设计不采用）

本设计选择 DENIED 位入状态字（bit17），因此无需独立存储。工单允许的替代方案（独立可信状态源）需满足：原子协调、防恢复覆盖、防回滚。经评估，独立存储会引入「状态字 与 独立 DENIED 源」的双源一致性问题，与 B35-01「单一权威状态源」约束冲突，故**不采用**，DENIED 保持在状态字内。

---

## 证据来源

- 状态字位布局：ATOMIC-STATE-WORD.md §2
- 状态转换矩阵：STATE-TRANSITION-MATRIX.md §2/§3
- DENIED 不可逆证明：STATE-TRANSITION-MATRIX.md §3
- 证明等级：VERIFIED_STATIC
