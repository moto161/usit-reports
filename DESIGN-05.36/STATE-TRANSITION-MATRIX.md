# STATE-TRANSITION-MATRIX.md — 状态转换矩阵（05.36）

证据等级：VERIFIED_STATIC
权威状态源：单一原子状态字 `_admit_state`（见 ATOMIC-STATE-WORD.md）
本文件为状态转换的权威定义，其他文档引用本矩阵，不得重复定义冲突规则。

---

## 1. 状态组合定义

状态字由四个字段构成：`gate`(bit0)、`admit_count`(bit1–15)、`frozen`(bit16)、`DENIED`(bit17)。

合法复合状态以四元组 `(gate, frozen, DENIED)` 表示（count 为独立数值字段，不参与合法组合判定，但受 0..32767 约束）：

| 编号 | gate | frozen | DENIED | 语义 | 合法性 |
|---|---|---|---|---|---|
| S0 | OPEN | 0 | 0 | 正常运行，可准入 | 合法 |
| S1 | CLOSED | 0 | 0 | 已关 Gate，未冻结 | 合法（冻结进行中） |
| S2 | CLOSED | 1 | 0 | 冻结完成（FROZEN） | 合法 |
| S3 | OPEN | 0 | 1 | DENIED 但 Gate 尚未物理关闭 | 合法（DENIED 优先，禁止新准入） |
| S4 | CLOSED | 0 | 1 | DENIED 且 Gate 已关 | 合法 |
| S5 | CLOSED | 1 | 1 | DENIED 且 FROZEN | 合法 |
| S6 | OPEN | 1 | 0 | **非法**（OPEN 与 FROZEN 矛盾） | 非法 → ISOLATED |
| S7 | OPEN | 1 | 1 | **非法** | 非法 → ISOLATED |

**FREEZING 说明**：05.35 曾保留 FREEZING 与 FROZEN 两个概念。05.36 统一为：`frozen` 位即冻结完成标志；「冻结进行中」由 `gate==CLOSED 且 frozen==0`（S1）表达，**不单独设 FREEZING 位**，消除冗余。此决定记录于 CROSS-DOC-CONSISTENCY.md。

---

## 2. 状态转换表

| 转换 | 源状态 | 目标状态 | 触发操作 | 线性化点（同一 CAS 内前置检查） | 拒绝条件 |
|---|---|---|---|---|---|
| T1 准入 | (OPEN,0,0) | (OPEN,0,0) count+1 | admit() | gate==OPEN 且 count<COUNT_MAX | gate!=OPEN 或 count==MAX → 拒绝 |
| T2 退出 | 任意 count>0 | 同字段 count-1 | release() | count>0 | count==0 → UNDERFLOW |
| T3 关 Gate | (OPEN,0,*) | (CLOSED,0,*) | freeze() 阶段1 | gate==OPEN | gate!=OPEN → 拒绝 |
| T4 冻结完成 | (CLOSED,0,*) | (CLOSED,1,*) | freeze() 阶段2 | gate==CLOSED 且 count==0 | count!=0 → 等待排空 |
| T5 解冻 | (CLOSED,1,0) | (CLOSED,0,0) | unfreeze() | frozen==1 且 **DENIED==0** | DENIED==1 → 永久拒绝 |
| T6 设置 DENIED | 任意 | 同字段 DENIED=1 | set_denied() | 无（可叠加） | 无 |
| T7 重新打开 | (CLOSED,0,0) | (OPEN,0,0) | reopen() | DENIED==0 且 frozen==0 | DENIED==1 或 frozen==1 → 拒绝 |

---

## 3. DENIED 不可逆性证明

**定理**：一旦 `DENIED` 位被成功置 1（线性化点 = `set_denied()` 的 CAS 成功），任何普通路径（unfreeze、reopen、rollback、恢复、父子进程初始化）都无法清除它。

证明：
1. DENIED 位只能由 `set_denied()` 写入 `1`，全系统**不存在**任何将 DENIED 位写回 `0` 的操作（`field_cas` 的 `denied` 参数只有 `True` 一种取值，无「清除 DENIED」路径）。
2. `unfreeze()`（T5）前置检查 `DENIED==0`，DENIED=1 时直接拒绝，不触碰 frozen。
3. `reopen()`（T7）前置检查 `DENIED==0`，拒绝。
4. 恢复路径（`CRASH-RECOVERY-PROTOCOL.md`）在重建状态字时，**显式保留 DENIED 位**，禁止任何重建逻辑写入 DENIED=0。
5. 子进程继承（Fork）复制状态字副本，副本同样携带 DENIED=1，且子进程初始化不包含清除 DENIED 的操作。

因此不存在从 `DENIED=1` 回到 `DENIED=0` 的状态转换边。∎

（唯一可能的清除路径是 Break-glass 紧急恢复，其在 `DENIED-ATOMIC-PROTECTION.md` 中定义，且**当前审计信任根 NOT_PROVEN，功能禁用**，不构成普通路径。）

---

## 4. 非法状态与非法转换处理

- 检测到 S6/S7（OPEN 与 FROZEN 同时为 1）→ 立即进入 ISOLATED，禁止一切准入与 Fork。
- 检测到 count 超出 [0, 32767] 或字段值非 0/1 → ISOLATED。
- 非法转换（如 frozen=0→1 但 gate 仍 OPEN）→ 拒绝，不写状态字。

ISOLATED 是不可逆的本地安全终态，其不可逆性由「ISOLATED 后拒绝一切状态转换操作」保证（见 IDEMPOTENT-CLEANUP-PROTOCOL.md 的清理状态机）。

---

## 5. 与 05.35 的差异

| 项 | 05.35 | 05.36 |
|---|---|---|
| COUNT_MASK | 0x7FFE（错误） | **0xFFFE（修正）** |
| FREEZING 独立状态 | 有（冗余） | 移除，用 (CLOSED, frozen=0) 表达 |
| unfreeze 检查 DENIED | `cas_unfreeze` 检查不完整 | 完整：同 CAS 内检查 DENIED==0 |
| 掩码遗漏 bit15 | 存在（16384 误判为 0） | 消除 |

---

## 证据来源

- 状态字位布局：ATOMIC-STATE-WORD.md §2
- DENIED 不可逆证明：本文件 §3
- 证明等级：VERIFIED_STATIC
