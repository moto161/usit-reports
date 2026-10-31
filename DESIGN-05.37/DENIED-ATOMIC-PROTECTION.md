# DENIED-ATOMIC-PROTECTION.md — DENIED 与 FROZEN 原子转换（05.37）

证据等级：VERIFIED_STATIC
权威引用：冻结状态机见 FREEZE-STATE-MACHINE.md；状态字位布局见 ATOMIC-STATE-WORD.md（继承 05.36）。

---

## 1. 本文件职责

本文件只负责 **DENIED 位与冻结/解冻/回滚/恢复/父子初始化之间的原子竞争保护**，不重复定义状态机（避免 05.36 的多套描述问题）。

---

## 2. DENIED 编码与存储

- DENIED 位：状态字 bit17，`DENIED_MASK = 0x20000`。
- 唯一权威状态源：状态字本身（单一状态源，继承 B35-01）。
- 合法组合见 FREEZE-STATE-MACHINE.md §2。

---

## 3. DENIED 设置的线性化点

`set_denied()` 的线性化点 = CAS `nv |= DENIED_MASK` 成功的那一刻。

```python
def set_denied():
    while True:
        cur = load()
        if (cur & DENIED_MASK) != 0:
            return 'ALREADY_DENIED'   # 幂等
        nv = cur | DENIED_MASK
        if cas(cur, nv):
            return 'OK'               # ← 线性化点
```

- 幂等：重复设置无副作用。
- 多线程并发设置：`|` 运算幂等，最终 DENIED=1 不丢失。

---

## 4. DENIED 不可逆性（完整证明）

**定理**：DENIED 位一旦被 `set_denied()` 成功置 1，任何普通路径都无法清除。

证明（枚举所有可能写 DENIED 位的操作）：

1. `set_denied()`：只写 `1`，不写 `0`。
2. `admit()`：field-aware CAS 只改 count 字段，`DENIED_MASK` 不被触碰；且前置检查 `DENIED==0`。
3. `release()`：只改 count 字段，不触碰 DENIED。
4. `cas_frozen()`：只设 frozen 位（`| FROZEN_MASK`），不触碰 DENIED。
5. `cas_unfreeze()`：用 `& ~FROZEN_MASK` 清 frozen，不触碰 DENIED；前置检查 DENIED==0。
6. `reopen()`：前置检查 DENIED==0，DENIED=1 时拒绝。
7. `rollback()`：前置检查 DENIED==0，拒绝。
8. 父进程恢复 / 子进程初始化：重建状态字时显式保留 DENIED 位（见 RECOVERY-CAS-PROTOCOL.md）。
9. 清理操作：与 DENIED 正交，不触碰 DENIED 位。

全系统**不存在** `DENIED_MASK` 被写回 `0` 的代码路径。因此 DENIED 一旦置位即不可逆。∎

---

## 5. 并发时间线

完整 8 场景见 FREEZE-STATE-MACHINE.md §6。本文件补充两条与本文件职责直接相关的关键时序：

### T-A：DENIED 设置后，FROZEN 转换的 CAS 竞争

```
时间轴：
  T1: set_denied CAS 成功（DENIED=1）   ← 线性化点
  T2: cas_frozen 读 cur，发现 DENIED=1 → 返回 'DENIED'，不设 frozen
结论：DENIED 先于 frozen，frozen 永不设置。
```

### T-B：DENIED 设置后，unfreeze 的 CAS 竞争

```
时间轴：
  T1: set_denied CAS 成功（DENIED=1）
  T2: cas_unfreeze 读 cur，DENIED=1 → 返回 'DENIED'，不清 frozen
结论：DENIED 优先，unfreeze 拒绝。
```

---

## 6. 优先级规则（权威）

**DENIED > FROZEN > FREEZING > OPEN**。

任何状态转换操作，第一步检查 `DENIED_MASK`：
- DENIED=1 → 拒绝一切 unfreeze/reopen/rollback/admit/fork，保持终态。
- DENIED=0 → 按各自前置条件继续。

此优先级规则在 FREEZE-STATE-MACHINE.md §5 与 STATE-TRANSITION-MATRIX.md 中一致引用。

---

## 7. 与 05.36 的差异

| 项 | 05.36 | 05.37 |
|---|---|---|
| DENIED 通配 `*`（矩阵 T4） | 允许 | **修正为 DENIED 必须 =0** |
| 状态机权威 | 三套不一致 | 统一为 FREEZE-STATE-MACHINE.md |
| cas_unfreeze 版本变化 | 未明确 | 明确：CAS 失败重读，不覆盖并发更新 |

---

## 证据来源

- 状态字位布局：ATOMIC-STATE-WORD.md（05.36，继承）
- 状态机：FREEZE-STATE-MACHINE.md
- 证明等级：VERIFIED_STATIC
