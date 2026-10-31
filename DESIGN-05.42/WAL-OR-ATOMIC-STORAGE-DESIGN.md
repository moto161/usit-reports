# WAL-OR-ATOMIC-STORAGE-DESIGN.md — 事务存储方案选型（05.42 权威）

证据等级：VERIFIED_STATIC（单进程）+ NOT_PROVEN（跨进程/跨重启）
目标：修复 B41-R01/R07——从四种候选方案中选定唯一权威方案，解决「两个独立存储崩溃后如何知道事务是否提交」的根本问题。

---

## 1. 05.41 缺陷定位（B41-R01/R07）

05.41 声称「先写 TRT → CAS 状态字 → 提交」构成原子事务，但：

1. **mutex + CAS 不是崩溃原子事务**：进程在「TRT 已写、状态字 CAS 未提交」之间崩溃，两个存储不一致。
2. **恢复算法错误**：用 `bit=0 && generation!=0` 判定 TRT 领先。但正常释放过的槽位本来就是 `bit=0 && generation>0`，会被错误清零，破坏每槽位代际单调性。
3. **提交定义矛盾**：ATOMIC-TRANSACTION-BOUNDARY.md 同时出现「先 CAS 状态字再写 TRT」「先写 TRT 再 CAS 状态字」「CAS 成功是提交点」「锁释放让其他线程可见」四种说法。
4. **释放路径不一致**：slot_free 不递增 generation，release 递增——同一操作两个实现。

**根因**：把「两个独立存储」通过普通 mutex + CAS 强行伪装成原子事务，而没有真正的持久化提交标记。

---

## 2. 候选方案比较

| 方案 | 崩溃原子性 | 恢复能力 | 复杂度 | 结论 |
|---|---|---|---|---|
| A. 单一事务存储（所有字段一个持久化记录） | 强（单记录原子写） | 强 | 低，但 128 位容不下 | 受限于字段量，见 §3 |
| B. WAL 预写日志 + 提交标记 | 强（COMMIT fsync 是提交点） | 强（重放日志） | 中 | **选定** |
| C. 双副本版本化记录 | 中（需比较两副本版本） | 中（副本冲突判定） | 高 | 复杂，暂不选 |
| D. 单一权威记录表 | 强（表内原子） | 强 | 取决于底层 | 需外部事务引擎，未证明 |

**选定方案 B（WAL）**。理由：

- WAL 的 COMMIT 记录落盘（fsync）是**业界公认、可证明的单一提交点**。
- 恢复靠**重放日志**，不靠「猜状态字与 TRT 谁领先」，从根本上消除 B41-R01 的错误判定。
- 状态字与 TRT 保持各自的内存结构，但**崩溃后只信 WAL**，内存状态字/TRT 只是 WAL 重放后的缓存。

---

## 3. 方案 A（单一存储）为何不可行

05.41 需要 16 槽位 × 16 位 generation = 256 位，加上 COUNT/GATE/DENIED/FROZEN/FREEZING/VERSION/TOKEN_ACTIVE 已远超 128 位。单一原子字无法容纳。因此不能靠「把所有字段塞进一个 128 位原子字」来获得崩溃原子性。

---

## 4. WAL 方案权威定义

### 4.1 存储组成

| 存储 | 内容 | 权威性 |
|---|---|---|
| WAL（追加日志） | 事务记录序列 | **唯一权威**（崩溃后据此恢复） |
| 状态字（内存/可重建） | COUNT/GATE/DENIED/FROZEN/FREEZING/VERSION/TOKEN_ACTIVE | WAL 重放后的缓存 |
| TRT（内存/可重建） | 每槽位 {generation, owner_id, resource_id} | WAL 重放后的缓存 |

**关键**：状态字与 TRT 都**可从 WAL 完整重建**，崩溃后不信任它们的现场值，只信 WAL 重放结果。

### 4.2 事务记录格式

```
WAL 记录 = {
  txn_id:     u64（单调递增，全局唯一）
  txn_version: u64（事务内版本，用于重放幂等）
  op:         enum（ADMIT / RELEASE / REVOKE / FREEZE / UNFREEZE / SET_DENIED / RECOVERY）
  slot:       u8（Token 相关操作）
  generation: u16（该槽位新旧代际）
  owner_id:   u64
  resource_id: u64
  fields:     {count_delta, active_bit_delta, version_delta, ...}
  status:     enum（PREPARE / COMMIT / APPLY）
  checksum:   u64（记录校验，防损坏）
}
```

### 4.3 提交点

**提交点 = COMMIT 记录 fsync 成功**。这是唯一的权威提交点。APPLY 记录表示内存状态已更新（可选，崩溃后由 COMMIT 重放）。

---

## 5. PREPARE / COMMIT / APPLY / RECOVERY 状态机

```
事务开始
  └─ 写 PREPARE 记录（含全部字段变化）→ fsync
       ├─ 失败 → 事务 ABORT（丢弃 PREPARE）
       └─ 成功 → 写 COMMIT 记录 → fsync
            ├─ 失败 → 事务状态 UNKNOWN（见恢复）
            └─ 成功 → 提交点 ✓ → APPLY（更新状态字+TRT 内存）→ 写 APPLY 记录（可选）
                    └─ 返回 OK

崩溃恢复：
  扫描 WAL，按 txn_id 升序重放：
    - PREPARE 无 COMMIT → 未提交，丢弃（不 APPLY）
    - COMMIT 无 APPLY → 已提交未应用，重放 APPLY
    - COMMIT + APPLY → 已应用，跳过（幂等）
    - 记录损坏（checksum 失败）→ 隔离，DENY
```

---

## 6. 崩溃窗口分析（每个持久化步骤）

| 崩溃点 | WAL 状态 | 恢复判定 |
|---|---|---|
| PREPARE 写前 | 无记录 | 无事务 |
| PREPARE 写中 | 半条记录（checksum 失败） | 隔离，DENY（不可信） |
| PREPARE 写后、fsync 前 | PREPARE 未持久化 | 视为未提交（fsync 前丢失） |
| PREPARE fsync 后、COMMIT 写前 | PREPARE 持久化，无 COMMIT | 未提交，丢弃 |
| COMMIT 写中 | 半条 COMMIT（checksum 失败） | 隔离，DENY |
| COMMIT 写后、fsync 前 | COMMIT 未持久化 | 视为未提交（可能丢失） |
| COMMIT fsync 后 | 已提交 | 重放 APPLY |
| APPLY 写中/后 | 已提交，APPLY 可选 | 重放 APPLY（幂等） |

**核心**：任何崩溃后，恢复只依据「COMMIT 是否 fsync 成功」。fsync 成功 = 已提交；未 fsync = 未提交。**不存在**「正常历史 generation 被清零」的问题，因为 generation 的每次变化都有 WAL 记录，重放按记录恢复，不靠猜位图。

---

## 7. NOT_PROVEN 边界

- 跨进程 WAL 共享 + fsync 原子性：NOT_PROVEN → DENY。
- 跨重启 WAL 持久化 + 重放正确性：NOT_PROVEN → DENY（单进程内存 WAL 可证明，跨重启需持久化介质原子性证明）。
- WAL 记录 checksum 无法检测的静默损坏：NOT_PROVEN → 依赖底层存储完整性。

---

## 8. 与 05.41 的差异

| 项 | 05.41 | 05.42 |
|---|---|---|
| 原子机制 | mutex + CAS（非真正原子） | WAL + COMMIT fsync 提交点 |
| 恢复依据 | 猜状态字/TRT 谁领先 | 重放 WAL 记录 |
| 历史 generation | 可能被错误清零 | 由 WAL 记录恢复，永不清零 |
| 提交定义 | 四种矛盾说法 | 唯一：COMMIT fsync |

---

## 证据来源

- 05.41 缺陷：ATOMIC-TRANSACTION-BOUNDARY.md（05.41 版）提交顺序矛盾
- 05.41 缺陷：TOKEN-CRASH-RECOVERY.md（05.41 版）bit=0 判 TRT 领先错误
- 证明等级：VERIFIED_STATIC（单进程 WAL）+ NOT_PROVEN（跨进程/跨重启）
