# RECOVERY-CAS-PROTOCOL.md — 带版本校验的恢复 CAS 协议（05.37）

证据等级：VERIFIED_STATIC
本文件为恢复操作原子提交的**唯一权威定义**。CRASH-RECOVERY-PROTOCOL.md 引用本协议。

---

## 1. 05.36 缺陷定位

05.36 恢复伪代码使用普通 `commit(state)` 无条件覆盖状态，存在两个问题：
1. 不校验状态版本，恢复期间其他线程/进程的更新会被覆盖。
2. 恢复期间其他线程能否修改状态未明确。

05.37 修复：恢复必须使用**带版本校验的 CAS**，禁止无条件 commit。

---

## 2. 状态版本机制

状态字增加版本号字段（扩展 bit18–31 或独立版本计数器），或使用「读版本 → 计算新值 → CAS」的标准乐观并发控制：

```python
def load_with_version():
    """原子读取状态字 + 版本号。"""
    return (state, version)

def cas_with_version(old_state, old_version, new_state):
    """仅当当前状态和版本都与期望一致时才提交。"""
    # 原子比较 (state, version) == (old_state, old_version)
    # 成功则写 new_state 并 version+1
```

**规则**：任何恢复提交必须 `cas_with_version`，禁止无条件 `commit(state)`。

---

## 3. 恢复 CAS 伪代码

```python
def recover(evidence):
    # 1. 读初始状态 + 版本
    old_state, old_version = load_with_version()

    # 2. 恢复期间，其他线程/进程能否修改状态？
    #    本协议采用乐观并发：不持锁，允许并发修改，
    #    但任何修改都会使版本号递增，导致恢复的 CAS 失败。

    # 3. 计算 allowed（见 CRASH-RECOVERY-PROTOCOL.md §5）
    if not compute_allowed(evidence):
        return 'DENY'

    # 4. 构建新状态（保留 DENIED 位）
    new_state = build_new_state(evidence)
    new_state = (new_state & ~DENIED_MASK) | (old_state & DENIED_MASK)
    #  ↑ 保留旧状态的 DENIED 位，恢复不得清除 DENIED

    # 5. 带版本校验的 CAS
    if cas_with_version(old_state, old_version, new_state):
        return 'REBUILT'
    else:
        # 6. CAS 失败：状态已被并发修改，重读最新状态重新验证
        return recover(reload_evidence())   # 重新采集证据，不重试旧值
```

**关键**：
- CAS 失败后**重新读取状态、重新验证全部证据**，不得直接重试旧状态值（避免 ABA 与陈旧值）。
- DENIED 位显式保留（`old_state & DENIED_MASK`），恢复不得覆盖 DENIED。

---

## 4. 恢复期间其他线程/进程能否修改状态

采用乐观并发控制，**允许**并发修改，但：
- 任何修改使版本号递增。
- 恢复的 CAS 因版本不匹配而失败。
- 失败后重新加载证据，以最新状态重新判定。

这避免了「恢复期间阻塞所有写入」的代价，同时保证恢复不覆盖并发更新。

---

## 5. DENIED 证据的 TOCTOU 防护

`denied_evidence` 的读取、验证、提交存在 TOCTOU 风险（读证据后、提交前 DENIED 被并发设置）。防护：

1. **证据读取**：`denied_evidence` 从持久化审计日志 + 状态字 DENIED 位双重读取。
2. **提交前复核**：CAS 提交前，`build_new_state` 再次读取当前状态字的 DENIED 位，`new_state` 的 DENIED 位 = `old_state & DENIED_MASK | 证据中的 DENIED`，即**取并集**，绝不取 0。
3. **CAS 原子性**：DENIED 的并发设置会改变版本号，使恢复 CAS 失败，重读后 DENIED 位被保留。

```python
def build_new_state(evidence):
    s = evidence.base_state_reconstructed()
    # DENIED 位 = 当前状态 DENIED 位 OR 证据 DENIED（取并集，绝不清除）
    s = s | (evidence.current_state & DENIED_MASK)
    s = s | (evidence.persistent_denied_flag & DENIED_MASK)
    return s
```

因此 DENIED 优先且不可被恢复覆盖。∎

---

## 6. 证据不足 / 冲突 / 版本变化 / 来源不可信

以下任一情况 → DENY（不进入 CAS）：

- 证据缺失（无法判定事务/写入者状态）。
- 证据冲突（同一事务矛盾记录）。
- 版本变化（恢复期间状态被修改，且重载后仍不一致）。
- 来源不可信（审计根未证明、租约不可信）。

---

## 7. 幂等性

- 恢复操作本身无副作用（CAS 未提交则状态不变）。
- 重复恢复：CAS 版本校验保证幂等——若已由其他恢复者重建，版本变化使重复恢复失败，重读后确认已完成。
- 恢复者崩溃：无副作用，下次重做。

---

## 证据来源

- 恢复证据清单/决策矩阵：CRASH-RECOVERY-PROTOCOL.md
- 证明等级：VERIFIED_STATIC
