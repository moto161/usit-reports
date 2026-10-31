# VERSION-MONOTONICITY-AND-EXHAUSTION.md — 版本单调性与耗尽（05.42 权威）

证据等级：VERIFIED_STATIC（单进程）+ NOT_PROVEN（跨进程/跨重启）
目标：修复 B41-R06——提交入口根据实际新旧状态自动检测字段变化，不依赖调用方维护 txn.changed。

---

## 1. 05.41 缺陷定位

05.41 的 `commit_txn()` 用 `'version' in txn.changed` 判断是否检查版本。但：

- 未证明所有状态字段修改都会自动加入 `txn.changed`。
- 若调用方修改 COUNT/TOKEN_ACTIVE 却遗漏该集合，提交入口绕过版本检查。
- 依赖调用方正确维护辅助集合，是脆弱设计。

**根因**：版本递增检查依赖外部辅助集合，而非提交入口自动检测。

---

## 2. 唯一权威版本递增规则（提交入口自动检测）

在 WAL 事务的 APPLY 阶段，提交入口**自动比较新旧状态字**，检测字段变化，强制版本规则：

```python
def apply_and_check_version(old_state, new_state, rec):
    changed = detect_changes(old_state, new_state)   # 自动检测，非调用方维护
    # 规则：任何状态字段变化，VERSION 必须 +1
    if changed.any_field_except_version:
        if new_state.version != old_state.version + 1:
            return 'VERSION_NON_MONOTONIC_DENY'   # 回退/跳跃/静默不递增
        if old_state.version == 0xFFFFFFFF:
            return 'VERSION_EXHAUSTED_DENY'       # 回绕
    else:
        # 无字段变化（纯只读/审计），VERSION 允许不变（白名单）
        if new_state.version != old_state.version:
            return 'VERSION_UNEXPECTED_CHANGE'
    return 'OK'
```

**关键**：
- `detect_changes` 由提交入口**自动执行**，比较 old_state 与 new_state 的逐字段差异。
- 不依赖调用方维护 `txn.changed`。
- 任何字段变化 → VERSION 必须 +1（禁止回退/跳跃/静默不递增/回绕）。

---

## 3. 版本递增白名单（不修改 VERSION 的操作）

| 操作 | 是否改 VERSION | 理由 |
|---|---|---|
| 只读（load/read） | 否 | 无字段变化 |
| cleanup 资源端释放（非状态字） | 否 | 不涉及状态字字段 |
| takeover 资源端 token（非状态字） | 否 | 资源端独立计数 |
| 审计字段（state_version 记录） | 否 | 仅记录，不改状态字 |

**其余所有修改状态字字段的操作（admit/release/revoke/freeze/unfreeze/set_denied）必须 VERSION+1**。

---

## 4. 事务重试 / 回滚 / 恢复时 VERSION 是否变化

| 场景 | VERSION 变化 |
|---|---|
| 事务重试（CAS 冲突） | 不变（未提交，重读重算） |
| 事务回滚（ABORT） | 不变（未提交，无字段变化） |
| 恢复（重放 WAL） | 按 WAL 记录的 version_delta 重放（= +1） |

**关键**：只有 COMMIT 成功的事务才导致 VERSION 变化。重试/回滚不产生 VERSION 变化。

---

## 5. VERSION 与 generation 分别耗尽

| 计数器 | 位宽 | 耗尽 | 独立 |
|---|---|---|---|
| VERSION | 32 位 | 0xFFFFFFFF → VERSION_EXHAUSTED_DENY | 是 |
| TRT[slot].generation | 16 位/槽位 | 0xFFFF → SLOT_GENERATION_EXHAUSTED_DENY | 是 |

两者独立耗尽、独立拒绝、禁止回绕。

---

## 6. 边界测试

| 输入 | 预期 |
|---|---|
| version=0 | 递增 → 1 |
| version=0xFFFFFFFE | 递增 → 0xFFFFFFFF（最后一次） |
| version=0xFFFFFFFF | 递增 → VERSION_EXHAUSTED_DENY |
| generation=0 | 分配 → 1 |
| generation=0xFFFE | 分配 → 0xFFFF |
| generation=0xFFFF | 分配 → SLOT_GENERATION_EXHAUSTED_DENY |

---

## 证据来源

- 提交协议：TRANSACTION-COMMIT-PROTOCOL.md
- 状态转换：STATE-INVARIANTS-AND-TRANSITIONS.md
- 证明等级：VERIFIED_STATIC（单进程）+ NOT_PROVEN（跨进程/跨重启）
