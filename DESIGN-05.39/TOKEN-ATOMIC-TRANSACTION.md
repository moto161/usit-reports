# TOKEN-ATOMIC-TRANSACTION.md — 准入令牌与 COUNT 原子事务（05.39 权威）

证据等级：VERIFIED_STATIC
目标：修复 B38-R02——Token 发放与 COUNT+1、Token 消费与 COUNT−1 不在同一原子提交，存在崩溃窗口与并发重复消费。

---

## 1. 05.38 缺陷定位

05.38 的 `admit()` / `release()` 将 Token 账本（`admit_tokens` 字典）与状态字（COUNT）分离，导致两个竞态：

### 问题 A：admit 崩溃窗口
```
admit():
  CAS 成功，COUNT+1          ← 状态字已改
  token = new_token()
  admit_tokens[token] = 1    ← 若此处进程崩溃，Token 未入账
  → COUNT 永久多计，冻结永远无法完成
```

### 问题 B：release 并发重复消费
```
release(token):
  token in admit_tokens      ← 线程 A、B 同时通过检查
  CAS COUNT-1                ← A 成功；B 重读 COUNT 后也成功
  del admit_tokens[token]    ← A、B 都删除
  → 同一 Token 消费两次，COUNT 少计两次
```

根因：Token 账本与 COUNT **不在同一原子状态**中，`token in admit_tokens` 检查不构成原子消费。

---

## 2. 唯一权威设计决策

**结论：Token 必须内嵌到状态字，或 Token 账本与状态字共享同一原子提交边界。**

本设计选择**方案 A：Token 作为状态字字段的一部分**（扩展状态字，加入 token 序列号），使 Token 发放/消费与 COUNT 更新**天然处于同一原子字内**，由单条 CAS 保证原子性。

若目标平台状态字宽度不足以容纳 token 字段，则**退化到方案 B（独立账本 + WAL 事务协议）**，但必须完整实现 §6 的事务协议，且**跨进程安全降级为 NOT_PROVEN → DENY**。本设计默认方案 A，方案 B 仅作为宽度不足时的替代，且不声称跨进程安全。

---

## 3. 方案 A：Token 内嵌状态字（权威）

### 状态字扩展布局（128 位）

```
bit0–31   状态字段（gate/count/frozen/DENIED）
bit32–63  version（版本号，单调）
bit64–127 token 账本（64 位：token_seq + active_token 位图）
```

- `token_seq`（bit64–95，32 位）：单调递增的 Token 序列号。
- `active_mask`（bit96–127，32 位）：最多 32 个活动 Token 的位图（或改用「单活动 Token 计数」简化模型）。

**简化模型（本设计采用）**：由于 admit/release 是成对出现，采用「活跃准入计数 == 活动 Token 数」的约束。Token 直接由 `(token_seq)` 标识，`admit()` 的 CAS 原子地完成「COUNT+1 且 token_seq+1 且记录 token」，`release(token)` 的 CAS 原子地完成「验证 token 有效 且 COUNT−1 且标记 token 已消费」。

### admit（原子：COUNT+1 + 发放 token）

```python
def admit():
    while True:
        w = load_word()               # 128 位原子读
        state, version, token_seq, active = decode(w)
        if not (not get_denied(state) and get_gate(state) and get_count(state) < COUNT_MAX):
            return 'DENIED' if get_denied(state) else ('GATE_CLOSED' if not get_gate(state) else 'OVERFLOW')
        new_token = token_seq + 1     # 新 Token ID
        new_state = field_set_count(state, get_count(state) + 1)
        new_active = mark_active(active, new_token)   # 位图标记活动
        new_w = encode(new_state, version + 1, new_token, new_active)
        if cas128(w, new_w):
            return ('OK', new_token)  # ← 线性化点：COUNT+1 与 Token 发放原子完成
        # CAS 失败重读重算
```

**关键**：`COUNT+1`、`token_seq+1`、`active` 位图更新，全部在**同一条 128 位 CAS** 中提交。不存在「COUNT 已改但 Token 未入账」的中间态。

### release（原子：验证 token + COUNT−1 + 消费 token）

```python
def release(token):
    while True:
        w = load_word()
        state, version, token_seq, active = decode(w)
        if not is_active(active, token):
            return 'INVALID_TOKEN'     # token 无效或已消费 → 拒绝
        if get_count(state) == 0:
            return 'UNDERFLOW'
        new_state = field_set_count(state, get_count(state) - 1)
        new_active = clear_active(active, token)   # 原子消费：清除位图
        new_w = encode(new_state, version + 1, token_seq, new_active)
        if cas128(w, new_w):
            return 'OK'                # ← 线性化点：验证+COUNT−1+消费原子完成
        # CAS 失败重读重算
```

**关键**：`is_active(active, token)` 的验证、`COUNT−1`、`clear_active` 三者在同一 CAS 内。两个线程并发 `release(token)`：
- 线程 A CAS 成功，`active` 位被清除，COUNT−1。
- 线程 B 重读 `w`，`is_active` 返回 False → `INVALID_TOKEN`。
- **同一 Token 只能消费一次**。∎

---

## 4. Token 生命周期（内嵌模型）

| 阶段 | 状态 | 触发 |
|---|---|---|
| 未发放 | 不在 active 位图 | 初始 |
| 发放 | active 位图置位 | admit CAS 成功 |
| 活动 | active 位图仍置位 | 业务进行中 |
| 消费 | active 位图清除 | release CAS 成功 |
| 失效 | 位图未置位 | 消费后 / 崩溃恢复判定 |

**唯一性**：`token_seq` 单调递增，Token ID 永不重复（无回绕，见 §5）。
**一次性**：`active` 位图一旦清除，`release(token)` 的 `is_active` 恒 False，不可再消费。

---

## 5. Token 回绕与耗尽

`token_seq` 为 32 位，达到 `0xFFFFFFFF` 时：
- **禁止回绕**（与版本耗尽协议一致，见 VERSION-EXHAUSTION-PROTOCOL.md）。
- `admit()` 在 `token_seq == 0xFFFFFFFF` 时返回 `TOKEN_EXHAUSTED_DENY`，拒绝新准入。
- 无持久化 epoch 时，token_seq 回绕防护 NOT_PROVEN，耗尽后 DENY。

---

## 6. 方案 B：独立账本 + WAL 事务协议（仅宽度不足时）

若平台无法提供 128 位 CAS，退化为独立 Token 账本。此时**必须**实现完整事务协议，且**跨进程安全 NOT_PROVEN → DENY**：

### 事务协议（WAL：Write-Ahead Log）

```
admit:
  1. 写 WAL 准备记录 {op=admit, token=T, status=PREPARED} 并 fsync
  2. CAS COUNT+1
  3. 写 WAL 提交记录 {op=admit, token=T, status=COMMITTED} 并 fsync
  4. 更新 admit_tokens[T] = 1

release(token):
  1. 读 admit_tokens[T]，若不存在 → INVALID_TOKEN
  2. 写 WAL 准备记录 {op=release, token=T, status=PREPARED} 并 fsync
  3. CAS COUNT-1
  4. 写 WAL 提交记录 {op=release, token=T, status=COMMITTED} 并 fsync
  5. del admit_tokens[T]
```

### 幂等键与恢复判定

- 幂等键 = token（同一 token 的 admit/release 各只有一条有效记录）。
- 恢复时重放 WAL：PREPARED 未 COMMITTED 的记录 → 回滚或隔离（不确定则隔离）。
- 崩溃窗口：步骤 2/3 之间崩溃 → COUNT 与账本不一致 → 恢复按 WAL 判定，不确定则 ISOLATED，不自动放行。

### 强制规则（方案 B）

- 不得仅用 `token in admit_tokens` 证明一次性消费。
- 跨进程方案 B 的原子性 NOT_PROVEN → 跨进程 Token 功能 DENY。
- 单进程方案 B 可用进程内锁 + WAL 保证，但复杂度高，仅作退化路径。

---

## 7. 线性化点汇总

| 操作 | 线性化点 |
|---|---|
| admit（方案 A） | 128 位 CAS 成功（COUNT+1 + token 发放原子） |
| release（方案 A） | 128 位 CAS 成功（验证 + COUNT−1 + 消费原子） |
| admit（方案 B） | WAL 提交记录 fsync 完成 |
| release（方案 B） | WAL 提交记录 fsync 完成 |

---

## 8. 崩溃与恢复（内嵌模型）

内嵌模型下，Token 账本在状态字内，崩溃后随状态字一起持久化/恢复，无「COUNT 与 Token 分离」问题。恢复规则见 TOKEN-CRASH-RECOVERY.md。

---

## 证据来源

- 05.38 缺陷：ADMISSION-FIELD-OPERATIONS.md §5（05.38 版）的 admit_tokens 字典
- 状态字字段：ATOMIC-CONDITION-CAS-PROTOCOL.md（05.38，继承谓词框架）
- 证明等级：VERIFIED_STATIC（方案 A 单进程）+ NOT_PROVEN（方案 B 跨进程）
