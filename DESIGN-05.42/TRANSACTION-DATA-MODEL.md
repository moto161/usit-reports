# TRANSACTION-DATA-MODEL.md — 事务数据模型（05.42 权威）

证据等级：VERIFIED_STATIC（单进程）+ NOT_PROVEN（跨进程/跨重启）
目标：定义 WAL 方案下的权威数据模型，明确状态字、TRT、WAL 三者的关系与单一权威性。

---

## 1. 三层存储与权威性

| 层 | 内容 | 权威性 | 崩溃后 |
|---|---|---|---|
| WAL（追加日志） | 事务记录序列（唯一权威） | **最高** | 据此重放恢复 |
| 状态字（128 位） | COUNT/GATE/DENIED/FROZEN/FREEZING/VERSION/TOKEN_ACTIVE | 缓存 | 丢弃，由 WAL 重建 |
| TRT（Token 记录表） | 每槽位 {generation, owner_id, resource_id} | 缓存 | 丢弃，由 WAL 重建 |

**核心原则**：状态字与 TRT 是 WAL 重放后的**派生缓存**，不是权威。崩溃后不信任它们的现场值，只信任 WAL。

---

## 2. 状态字布局（128 位，缓存）

```
bit0–15    COUNT（COUNT_MASK=0xFFFE, SHIFT=1, MAX=32767）
bit16      GATE（1=OPEN）
bit17      DENIED（单调置位）
bit18      FROZEN
bit19      FREEZING
bit20–31   保留位（恒 0）
bit32–63   VERSION（单调递增）
bit64–79   TOKEN_ACTIVE（16 槽位位图）
bit80–127  保留位（恒 0）
```

---

## 3. TRT 结构（缓存）

```
TRT[slot] = { generation: u16, owner_id: u64, resource_id: u64 }
```

每槽位独立 generation，仅该槽位复用分配时 +1。

---

## 4. WAL 记录（唯一权威，追加）

```
WAL 记录 = {
  txn_id:      u64      # 全局单调，唯一
  txn_version: u64      # 事务内版本，重放幂等
  op:          enum     # ADMIT/RELEASE/REVOKE/FREEZE/UNFREEZE/SET_DENIED/RECOVERY
  slot:        u8       # Token 操作槽位（非 Token 操作为 0xFF）
  old_generation: u16   # 槽位旧代际
  new_generation: u16   # 槽位新代际
  owner_id:    u64
  resource_id: u64
  fields: {             # 状态字字段变化
    count_delta:    i32   # COUNT 变化（+1/-1/0）
    active_bit:     i8    # 槽位位图变化（+1 置位/-1 清位/0 不变）
    version_delta:  i32   # VERSION 变化（必须 +1 或 0）
    gate:           i8    # GATE 变化（-1 置位/0 不变）
    denied:         i8    # DENIED（0/1）
    frozen:         i8
    freezing:       i8
  }
  status:      enum     # PREPARE / COMMIT / APPLY
  checksum:    u64      # 记录校验
}
```

---

## 5. Token 完整身份（B42-04 前置）

Token 身份 = `(slot, generation, owner_id, resource_id, state_version)`。

- `slot`：状态字 TOKEN_ACTIVE 位图 bit 索引。
- `generation`：TRT[slot].generation。
- `owner_id` / `resource_id`：TRT[slot] 绑定。
- `state_version`：发放时 VERSION（**仅审计，不参与身份验证**，见 TOKEN-IDENTITY-VALIDATION.md §3）。

**每槽位独立代际**：`TRT[slot].generation` 仅该槽位复用时 +1，他槽位不影响。

---

## 6. 单一权威写入路径

所有状态修改必须走 WAL 事务（见 TRANSACTION-COMMIT-PROTOCOL.md），禁止绕过 WAL 直接改状态字或 TRT。

- 状态字 / TRT 的**内存更新**发生在 COMMIT 后的 APPLY 阶段。
- 任何非事务路径直接改状态字/TRT → 视为非法（INVALID_STATE）。

---

## 7. NOT_PROVEN 边界

- 跨进程 WAL 共享 + 追加原子性：NOT_PROVEN → DENY。
- 跨重启 WAL 持久化介质 + fsync 原子性：NOT_PROVEN → DENY。

---

## 证据来源

- 方案选型：WAL-OR-ATOMIC-STORAGE-DESIGN.md
- 提交协议：TRANSACTION-COMMIT-PROTOCOL.md
- 证明等级：VERIFIED_STATIC（单进程）+ NOT_PROVEN（跨进程/跨重启）
