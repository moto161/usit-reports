# TOKEN-CRASH-RECOVERY.md — Token 崩溃恢复（05.39）

证据等级：VERIFIED_STATIC（含 NOT_PROVEN 项）
原子事务见 TOKEN-ATOMIC-TRANSACTION.md。

---

## 1. 崩溃窗口分析（内嵌模型）

内嵌模型下，Token 账本在状态字内，`admit`/`release` 都是单条 CAS，**不存在「COUNT 已改但 Token 未入账」的中间态**。因此崩溃窗口被消除：

| 崩溃点 | 状态字结果 | 后果 |
|---|---|---|
| admit CAS 前 | 无变化 | 正常 |
| admit CAS 后（原子完成） | COUNT+1 且 token 已 active | 一致，无中间态 |
| release CAS 前 | 无变化 | 正常 |
| release CAS 后（原子完成） | COUNT−1 且 token 已消费 | 一致 |

**结论**：内嵌模型（方案 A）彻底消除 05.38 的问题 A（admit 崩溃窗口）和问题 B（release 并发重复消费）。

---

## 2. 方案 B（独立账本）的崩溃恢复

若退化到方案 B（独立 Token 账本），崩溃恢复必须基于 WAL：

### 恢复判定表

| WAL 记录 | COUNT 状态 | 判定 | 动作 |
|---|---|---|---|
| PREPARED 未 COMMITTED | 已改 | 不确定 | 回滚或隔离 |
| COMMITTED | 已改 | 已提交 | 确认完成 |
| 无记录 | 未改 | 无操作 | 跳过 |

### 幂等键

幂等键 = token。同一 token 的 admit/release 各只有一条有效 WAL 记录，恢复时按幂等键去重，防止重复重放。

### 不确定态

- PREPARED 未 COMMITTED → 不确定 → ISOLATED，不自动重放。
- 无法确认业务是否提交 → 不自动回收 token，COUNT 保持，冻结 DENY。

---

## 3. 恢复者崩溃 / 重复恢复

- 恢复操作无副作用（CAS/WAL 未提交则无变化）。
- 重复恢复幂等：按幂等键（token）去重，已处理的不再重放。
- 恢复者崩溃：下次恢复重做，无中间态。

---

## 4. Token 账本与状态字版本一致性

内嵌模型：Token 账本与状态字同一原子字，版本号一并递增，天然一致。

方案 B：独立账本需与状态字版本对齐，否则出现「账本版本 ≠ 状态字版本」。方案 B 必须：
- 账本每次变更与状态字 CAS 绑定同一版本。
- 版本不一致 → ISOLATED，不自动修复。

---

## 5. NOT_PROVEN 边界

- 跨进程 Token 功能（内嵌 128 位 CAS 跨进程原子性）：NOT_PROVEN → DENY。
- 方案 B 独立账本跨进程：NOT_PROVEN → DENY。
- 跨重启 Token 持久化：NOT_PROVEN（无持久化证明）→ DENY。

---

## 证据来源

- 原子事务：TOKEN-ATOMIC-TRANSACTION.md
- 所有权/生命周期：TOKEN-OWNERSHIP-AND-LIFECYCLE.md
- 证明等级：VERIFIED_STATIC（单进程内嵌）+ NOT_PROVEN（跨进程/持久化）
