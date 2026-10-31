# FREEZE-STATE-MACHINE.md — 冻结状态机（05.37 权威定义）

证据等级：VERIFIED_STATIC
本文件为冻结状态机的**唯一权威定义**。其他文档引用本文件，不得重复定义冲突规则。

---

## 1. 05.36 的三套不一致描述（问题定位）

05.36 存在三套冻结描述，互相矛盾：

| 文档 | 描述 | 问题 |
|---|---|---|
| ATOMIC-STATE-WORD.md | 一次 CAS 同时关闭 Gate 并设置 FROZEN | 与「等待计数归零」矛盾 |
| STATE-TRANSITION-MATRIX.md | 先关 Gate → 等计数归零 → 再设 FROZEN | 与「一次 CAS」矛盾 |
| DENIED-ATOMIC-PROTECTION.md | 两阶段冻结，但 DENIED 通配 `*` | 与矩阵 T4 的 DENIED=0 要求冲突 |

05.37 统一为下面这一套协议，消除全部不一致。

---

## 2. 状态定义（唯一）

状态字四字段：`gate`(bit0)、`admit_count`(bit1–15)、`frozen`(bit16)、`DENIED`(bit17)。

| 复合状态 | gate | frozen | DENIED | 语义 |
|---|---|---|---|---|
| OPEN | 1 | 0 | 0 | 正常运行，可准入 |
| FREEZING | 0 | 0 | 0 | Gate 已关，等待 count 归零 |
| FROZEN | 0 | 1 | 0 | 冻结完成，禁止 Fork 前写入 |
| DENIED(open-gate) | 1 | 0 | 1 | DENIED，Gate 尚未物理关（瞬态） |
| DENIED(closed) | 0 | 0 | 1 | DENIED，Gate 已关 |
| DENIED(frozen) | 0 | 1 | 1 | DENIED 且 FROZEN |

**FREEZING 不再是状态字中的独立位**，而是 `(gate=0, frozen=0)` 的组合语义。FROZEN 是 `frozen=1` 位。这是 05.36 已确立、05.37 继续继承并明确的规则。

**非法组合**：`gate=1 且 frozen=1`（OPEN 与 FROZEN 矛盾）→ 立即 ISOLATED。

---

## 3. 冻结必须分为三阶段

冻结不是一步，而是三个有明确线性化点的阶段：

### 阶段 1：关闭 Gate（OPEN → FREEZING）

- 线性化点：`cas(gate=1→0)` 成功。
- 进入条件：当前 gate==1 且 DENIED==0。
- 退出条件：gate 已变为 0。
- 效果：此后任何新 `admit()` 被拒绝（gate==0）。

### 阶段 2：等待准入归零（FREEZING 内部）

- 线性化点：无单点；`count` 在每次 `release()` 的 CAS 中递减，归零的判定在 `count==0` 读取时成立。
- 进入条件：gate==0。
- 退出条件：`count==0`（且 DENIED==0）。
- 效果：冻结者等待所有已准入业务退出，通过条件变量观察 `count`。

### 阶段 3：设置 FROZEN（FREEZING → FROZEN）

- 线性化点：`cas_frozen()` 成功（见 §4）。
- 进入条件：gate==0 且 count==0 且 DENIED==0。
- 退出条件：frozen 位变 1。
- 效果：FROZEN 完成，Fork 快照的前置满足。

---

## 4. cas_frozen() 原子检查（修正）

```python
def cas_frozen():
    while True:
        cur = load()
        if (cur & GATE_MASK) != 0:      # gate 仍 OPEN → 未完成阶段1
            return 'GATE_OPEN'
        if (cur & DENIED_MASK) != 0:    # DENIED 已置位 → 冻结绝不允许
            return 'DENIED'
        if (cur & COUNT_MASK) != 0:     # 准入未归零 → 未完成阶段2
            return 'COUNT_NOT_ZERO'
        nv = cur | FROZEN_MASK          # 只设 frozen 位
        if cas(cur, nv):
            return 'OK'
```

**关键**：`gate==0`、`DENIED==0`、`count==0` 三项在**同一个 CAS 的 expected 状态 `cur`** 上原子检查。任一不满足，冻结不成功且不修改状态字。这修复了 STATE-TRANSITION-MATRIX.md T4 中 DENIED 通配 `*` 与伪代码 DENIED=0 的冲突——**权威规则是 DENIED 必须为 0**。

---

## 5. DENIED 与 FROZEN 同时出现的语义

**FROZEN 与 DENIED 可以同时出现**（`frozen=1 且 DENIED=1`，即 DENIED(frozen) 状态），但语义上 DENIED **优先**：

- DENIED=1 时，任何 `unfreeze()`、`reopen()`、Fork 都被拒绝，无论 frozen 为何值。
- DENIED(frozen) 表示「系统曾在冻结后进入拒绝态」，但拒绝态是终态，冻结态已无实际意义。
- 优先级规则：**DENIED > FROZEN > FREEZING > OPEN**。任何操作先检查 DENIED。

DENIED 的不可逆性由 STATE-TRANSITION-MATRIX.md §3 保证（无写回 0 路径），本文件 §6 补充原子竞争证明。

---

## 6. 并发竞争时间线（8 场景全覆盖）

每个场景证明：DENIED 一旦成功提交，普通操作不能清除 DENIED、恢复 OPEN 或允许 Fork。

### C1：Gate 关闭 与 DENIED 设置竞争
- A 执行阶段1 `cas(gate 1→0)`：读 cur(gate=1,DENIED=0)。
- B `set_denied()`：CAS 置 DENIED=1 成功。
- A 的 CAS 因 cur 被 B 改变而失败，重读发现 DENIED=1，阶段1 终止（冻结不继续）。
- 结果：DENIED 胜出，Gate 语义上由 DENIED 处理关闭。∎

### C2：最后一个 admit 退出 与 冻结完成竞争
- 最后一个业务 `release()`：CAS count 1→0 成功，唤醒冻结者。
- 冻结者 `cas_frozen()`：读 cur(gate=0,count=0,DENIED=0)，CAS 设 frozen=1。
- 若此时 DENIED 被并发设置，`cas_frozen` 的 CAS 失败，重读 DENIED=1，返回 'DENIED'。
- 结果：FROZEN 不设置，DENIED 胜出。∎

### C3：FROZEN 设置 与 DENIED 设置竞争
- 同 C2 后半段：`cas_frozen` 与 `set_denied` 竞争 CAS。
- 谁先成功取决于线性化点；若 DENIED 先成功，frozen 的 CAS 失败。若 frozen 先成功，DENIED 仍可叠加（DENIED 优先）。
- 结果：DENIED 最终不可清除。∎

### C4：unfreeze 与 DENIED 设置竞争
- A `unfreeze()`：读 cur(frozen=1,DENIED=0)。
- B `set_denied()`：CAS 置 DENIED=1。
- A CAS 失败，重读 DENIED=1，unfreeze 拒绝（前置 DENIED==0）。
- 结果：DENIED 不可被 unfreeze 清除。∎

### C5：rollback 与 DENIED 设置竞争
- A `rollback()`：目标 frozen 1→0，读 cur(DENIED=0)。
- B `set_denied()`：置 DENIED=1。
- A CAS 失败，重读 DENIED=1，rollback 拒绝。
- 结果：DENIED 不可被 rollback 清除。∎

### C6：两个冻结请求同时发生
- A、B 都执行阶段1 `cas(gate 1→0)`。
- 线性化点：只有一个 CAS 能成功（gate 从 1→0 只能发生一次）。成功者继续阶段2/3，失败者重读 gate=0 后退出（或转等待）。
- 结果：无双重冻结，冻结所有权唯一。∎

### C7：冻结超时 与 DENIED 设置竞争
- 冻结者阶段2 等待超时，准备回退（unfreeze）。
- 若超时回退前 DENIED 被设置，回退的 unfreeze 前置 DENIED==0 失败。
- 结果：DENIED 优先，超时不得清除 DENIED。∎

### C8：父进程恢复 与 DENIED 设置竞争
- 父进程恢复重建状态字，读 DENIED=0 准备恢复 OPEN。
- DENIED 并发设置。
- 恢复逻辑在最终 CAS 提交前重新校验 DENIED，发现 DENIED=1 则放弃恢复 OPEN。
- 结果：恢复不得覆盖 DENIED。∎

---

## 7. cas_unfreeze() 修正

```python
def cas_unfreeze():
    while True:
        cur = load()
        if (cur & DENIED_MASK) != 0:      # DENIED 优先，禁止解冻
            return 'DENIED'
        if (cur & FROZEN_MASK) == 0:      # 本就未冻结
            return 'NOT_FROZEN'
        nv = cur & ~FROZEN_MASK           # 清 frozen 位，其他字段原样
        if cas(cur, nv):
            return 'OK'
```

- 用 `& ~FROZEN_MASK` 清除 frozen 位，**不触碰 gate/count/DENIED**。
- CAS 失败后重读 `cur`，不覆盖其他线程的并发更新（版本变化被 CAS 检测）。

---

## 8. 非法状态处理

- `gate=1 且 frozen=1`：ISOLATED，拒绝一切准入与 Fork。
- `count` 超出 [0, 32767]：ISOLATED。
- 字段值非 0/1：ISOLATED。
- DENIED=1 且尝试 unfreeze/reopen/rollback：拒绝（不修改状态）。

---

## 证据来源

- 状态字位布局：ATOMIC-STATE-WORD.md §2（05.36，继承）
- 状态转换矩阵：STATE-TRANSITION-MATRIX.md（05.36，本文件修正其 T4 冲突）
- 证明等级：VERIFIED_STATIC
