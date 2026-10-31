# RECOVERY-CAS-PROTOCOL.md — 恢复 CAS 协议（05.38 修订）

证据等级：VERIFIED_STATIC
版本原子性见 RECOVERY-VERSION-ATOMICITY.md（权威）；本文件定义恢复的 CAS 流程与证据新鲜度。

---

## 1. 05.37 缺陷定位

05.37 的恢复 CAS 存在：
1. 使用陈旧 `old_state` 和 `evidence.current_state`（两个不一致的「当前状态」来源）。
2. `build_new_state()` 未说明最新 DENIED 证据如何进入提交条件。
3. `recover()` 递归无限调用，无重试上限。
4. 独立版本字段，状态与版本更新可能分离。

05.38 修复见下。

---

## 2. 唯一版本模型

采用 RECOVERY-VERSION-ATOMICITY.md 的「状态 + 版本合并 64 位原子字」模型。恢复的 CAS 使用 `cas_state_version`（单条 64 位 CAS，状态与版本不可分离）。

---

## 3. 恢复 CAS 流程（含证据新鲜度 + 重试上限）

```python
MAX_RECOVERY_RETRY = 8
RECOVERY_BACKOFF_BASE = 0.1   # 秒

def recover(evidence):
    retries = 0
    while retries < MAX_RECOVERY_RETRY:
        # 1. 原子读最新状态 + 版本（唯一权威，不复用旧值）
        state, version = load_state_version()

        # 2. 重新采集/校验证据（基于最新 state/version，不复用旧证据）
        fresh_evidence = reload_and_validate_evidence(evidence, state, version)
        if fresh_evidence.invalid:
            return 'DENY'               # 证据缺失/冲突/过期 → DENY

        # 3. 计算 allowed（用 fresh_evidence）
        if not compute_allowed(fresh_evidence):
            return 'DENY'

        # 4. 构建新状态，DENIED 单调语义（取并集，绝不清除）
        new_state = build_new_state(fresh_evidence)
        new_state = (new_state & ~DENIED_MASK) | (state & DENIED_MASK)
        new_state = new_state | (fresh_evidence.persistent_denied_flag & DENIED_MASK)

        # 5. 带版本校验的 CAS（单条 64 位）
        if cas_state_version(state, version, new_state):
            return 'REBUILT'

        # 6. CAS 失败 → 重读最新状态，重新验证完整证据（不复用旧证据）
        retries += 1
        if retries >= MAX_RECOVERY_RETRY:
            return 'CONFLICT'           # 终止，不无限递归
        time.sleep(RECOVERY_BACKOFF_BASE * retries)   # 退避

    return 'CONFLICT'
```

**关键修复**：
- 每次循环**重新 `load_state_version()`**，不复用陈旧 `old_state`。
- `fresh_evidence` 基于最新 state/version 重新采集，不复用旧证据。
- DENIED 采用单调语义：`new_state` 的 DENIED 位 = `state & DENIED_MASK | 证据 DENIED`，取并集，**绝不清除**。
- **禁止递归无限调用 `recover()`**：用 `while` + `MAX_RECOVERY_RETRY` + 退避替代递归。

---

## 4. 最新 DENIED 证据如何进入提交条件

`build_new_state(fresh_evidence)` 使用**三类来源**，且都必须基于最新原子读：

1. `state & DENIED_MASK`：最新状态字的 DENIED 位（原子读）。
2. `fresh_evidence.persistent_denied_flag`：持久化审计日志中的 DENIED 标志（带版本/可信等级）。
3. 二者取**并集**，任何一方 DENIED=1，最终 new_state 的 DENIED=1。

提交时 `cas_state_version(state, version, new_state)` 保证：若并发 DENIED 置位改变了 `state`/`version`，CAS 失败，重读后 DENIED 被保留。因此**提交时不丢失并发 DENIED**。

---

## 5. build_new_state 的证据来源要求

`build_new_state()` 只能使用有明确来源、版本、可信等级的证据：

| 证据 | 来源 | 版本 | 可信等级 |
|---|---|---|---|
| 最新状态字 | load_state_version() | 原子版本号 | 直接（内存） |
| DENIED 持久化标志 | 审计日志 | 审计链版本 | 高（若审计根可信） |
| 事务状态 | 磁盘 Receipt/Commit 日志 | 日志序列号 | 高（落盘） |
| 旧写入者清单 | 进程表 + 租约 | 租约版本 | 中（可能 UNKNOWN） |

**禁止**把未经原子验证的快照称为「最新状态」。`evidence.current_state` 必须来自 `load_state_version()` 的原子读，而非恢复开始时的陈旧快照。

---

## 6. 恢复者崩溃 / 并发恢复者 / 重复恢复 / 版本溢出 / 断电 / 快照回滚

| 场景 | 处理 |
|---|---|
| 恢复者崩溃 | 无副作用（CAS 未提交则状态不变），下次重做 |
| 并发恢复者 | 版本 CAS 串行化，只有一个 CAS 成功，另一个重读后确认完成 |
| 重复恢复 | 幂等：CAS 失败重读，已恢复则确认完成 |
| 版本溢出 | 32 位版本溢出，无持久化 epoch 锚点 → NOT_PROVEN → DENY |
| 系统断电 | 内存版本丢失，跨重启需持久化版本，未证明 → DENY |
| 快照回滚 | 版本回绕，ABA 风险，无持久化 epoch → NOT_PROVEN → DENY |

---

## 7. DENY 条件（继承）

- 证据缺失 / 冲突 / 过期 → DENY。
- 审计根不可信 → DENY。
- 锁外写入者 UNKNOWN → DENY。
- 旧写入者未排除 → DENY。
- 事务 UNKNOWN → DENY。
- 版本原子性 / 持久化未证明 → NOT_PROVEN → DENY。

**不得恢复 OPEN、不得清除 DENIED、不得自动重放 UNKNOWN 事务。**

---

## 证据来源

- 版本原子性：RECOVERY-VERSION-ATOMICITY.md
- 证据清单/决策矩阵：CRASH-RECOVERY-PROTOCOL.md（05.38 修订）
- 05.37 缺陷：陈旧 old_state + 递归 recover
- 证明等级：VERIFIED_STATIC + NOT_PROVEN（跨进程/持久化原子性）
