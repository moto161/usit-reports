# UNIFIED-STATE-MODEL.md — 统一状态模型（05.41 权威）

证据等级：VERIFIED_STATIC（单进程）+ NOT_PROVEN（跨进程/跨重启）
目标：修复 B40-R01——05.40 的全局 `TOKEN_GENERATION` 导致任意槽位新分配使其他活动 Token 失效；重新评估 128 位状态字是否够用，并选择可证明的数据结构。

---

## 1. 05.40 缺陷定位（B40-R01）

05.40 使用**单个全局** `TOKEN_GENERATION`（16 位）作为所有槽位的代际：

- Token 身份 = `(slot, gen_at_issue)`，`gen_at_issue` 是发放时的全局 generation。
- 校验规则：`cur_gen == token.gen_at_issue` 才有效。

**致命缺陷**：全局 generation 在**任意槽位**分配时 +1。因此：

```
初始：槽位 0 活动 Token(slot0, g=5)，槽位 1 活动 Token(slot1, g=5)
线程 A：slot_alloc 分配槽位 2 → 全局 generation 变为 6
线程 B：release(slot0, g=5) → cur_gen=6 != 5 → INVALID_TOKEN  ← 槽位0的Token被误杀！
```

**违反 B41-01 要求**：「任意槽位新分配，不得导致其他仍然 ACTIVE 的 Token 失效」。05.40 文档 §4 自己也承认了这一点（「无法为 16 个槽位各存一个 16 位 generation」），但没有真正解决。

**根因**：`(slot, 全局 generation)` 二元组无法同时满足「每槽位独立代际」与「128 位状态字容纳」。

---

## 2. 唯一权威决策：每槽位独立代际 + 独立 Token 记录表

**结论**：128 位状态字无法容纳 16 槽位 × 16 位独立 generation（需 16×16=256 位，加上状态/版本字段更远超 128 位）。因此**放弃「Token 内嵌 128 位状态字」模型**，改为：

1. **状态字（128 位）** 只保留准入控制字段：COUNT / GATE / DENIED / FROZEN / FREEZING / VERSION / TOKEN_ACTIVE（16 槽位位图）。
2. **独立 Token 记录表（Token Record Table, TRT）** 存储每个槽位的完整身份：每槽位一条记录，含 `generation`（16 位）、`owner_id`、`resource_id`。
3. **COUNT 与 Token 记录的原子一致性** 通过**同一把事务锁 + 单一提交点**保证（见 §4 与 ATOMIC-TRANSACTION-BOUNDARY.md），不得退回无原子保证的独立字典。

### 数据布局

```
状态字（128 位，单一准入状态源）：
  bit0–15    COUNT（COUNT_MASK=0xFFFE, SHIFT=1, MAX=32767）
  bit16      GATE
  bit17      DENIED（单调置位）
  bit18      FROZEN
  bit19      FREEZING
  bit20–31   保留位（恒 0）
  bit32–63   VERSION（单调递增，禁止回绕）
  bit64–79   TOKEN_ACTIVE（16 槽位位图，每 bit = 槽位活动状态）
  bit80–127  保留位（恒 0）  ← 05.40 这里放 TOKEN_GENERATION，现废除

Token 记录表 TRT（独立存储，每槽位一条记录）：
  TRT[slot] = { generation: u16, owner_id: u64, resource_id: u64 }
```

> **关键变更**：`TOKEN_GENERATION`（全局 16 位）从状态字中**删除**，改由 TRT 的每槽位独立 `generation` 字段承担。状态字 bit80–95 与 bit96–127 一并成为保留位。

---

## 3. Token 完整身份模型

Token 身份 = `(slot, generation, owner_id, resource_id, state_version)`：

| 字段 | 存储位置 | 说明 |
|---|---|---|
| slot | 状态字 TOKEN_ACTIVE 位图 bit 索引 | 0–15，活动状态索引 |
| generation | TRT[slot].generation | 每槽位独立 16 位，槽位复用时 +1 |
| owner_id | TRT[slot].owner_id | 持有者，审计/绑定 |
| resource_id | TRT[slot].resource_id | 资源绑定 |
| state_version | 发放时的 VERSION | 恢复判定 |

**每槽位独立代际**：`TRT[slot].generation` 只在**该槽位**被复用分配时 +1，不影响其他槽位。因此：

```
初始：TRT[0].generation=5（Token0 活动），TRT[1].generation=5（Token1 活动）
slot_alloc 分配槽位 2 → 只改 TRT[2].generation=1，TRT[0]/TRT[1] 不变
release(slot0, gen=5) → TRT[0].generation 仍=5 → 校验通过 ✓
```

**旧 Token 拒绝**：槽位 0 被释放后复用，`TRT[0].generation` 从 5 变 6。旧 Token(slot0, gen=5) 再释放时 `TRT[0].generation=6 != 5` → 拒绝。

---

## 4. COUNT 与 Token 记录的原子一致性

**不变式**：`COUNT == popcount(TOKEN_ACTIVE)`（活动 Token 数 = 准入计数）。

由于 COUNT（状态字）与 Token 记录（TRT）现在是**两个存储**，必须证明二者在同一原子事务内提交。采用**单一事务锁 + 单一提交点**（详见 ATOMIC-TRANSACTION-BOUNDARY.md）：

- 所有修改 COUNT 或 TRT 的操作，先获取**事务锁 `txn_lock`**。
- 在锁内完成「状态字 CAS（改 COUNT + TOKEN_ACTIVE）+ TRT 记录更新」。
- 提交点 = 释放 txn_lock 前的最后一次持久化/可见性操作。

**禁止**：任何在锁外修改 COUNT 或 TRT 的路径；任何「COUNT 改了但 TRT 没改」或「TRT 改了但 COUNT 没改」的中间态对外可见。

---

## 5. 三个安全边界

| 边界 | 前提 | 状态 |
|---|---|---|
| 单进程 | 进程内 txn_lock + 128 位 CAS | VERIFIED_STATIC |
| 跨进程 | 共享内存 + 跨进程 txn_lock + HW CAS | NOT_PROVEN → DENY |
| 跨重启 | 状态字 + TRT 持久化 | NOT_PROVEN → DENY |

---

## 6. 与 05.40 的差异

| 项 | 05.40 | 05.41 |
|---|---|---|
| Token 代际 | 全局 TOKEN_GENERATION（16 位） | 每槽位独立 TRT[slot].generation |
| Token 存储 | 内嵌 128 位状态字 | 独立 TRT + 状态字位图 |
| 多 Token 并存 | 全局代际导致互相失效（缺陷） | 每槽位独立，互不影响 |
| 原子边界 | 单 128 位 CAS | txn_lock + 单一提交点（见 ATOMIC-TRANSACTION-BOUNDARY.md） |

---

## 证据来源

- 05.40 缺陷：TOKEN-IDENTITY-AND-BITMAP-MODEL.md（05.40 版）§4 全局 generation 无法区分
- 历史约束：COUNT_MASK=0xFFFE（05.36）、单一准入状态源（B35-01）
- 证明等级：VERIFIED_STATIC（单进程）+ NOT_PROVEN（跨进程/跨重启）
