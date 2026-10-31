# VERSION-MONOTONICITY-AND-EXHAUSTION.md — 版本单调性与耗尽（05.41 权威）

证据等级：VERIFIED_STATIC（单进程）+ NOT_PROVEN（跨进程/跨重启）
目标：修复 B40-R03——field_cas 内部必须实际执行版本递增规则，禁止版本回退/跳跃/静默不递增/回绕。

---

## 1. 05.40 缺陷定位

05.40 的版本递增规则散落在注释与伪代码中，`field_cas` 内部没有真正执行版本递增检查，调用方仍可能在 `field_cas` 之外自行 `version+1`。

---

## 2. 唯一权威版本递增规则（field_cas 内部执行）

```python
def commit_txn(txn):
    # 版本单调性检查（在提交路径内，非注释）
    new_version = txn.state.version
    if 'version' in txn.changed:
        old_version = txn.state.word_version   # 进入事务时读的旧版本
        if old_version == 0xFFFFFFFF:
            return 'VERSION_EXHAUSTED_DENY'    # 禁止回绕
        if txn.state.version != old_version + 1:
            return 'VERSION_NON_MONOTONIC_DENY'  # 禁止跳跃/回退
        new_version = old_version + 1

    # 状态字 CAS（比较旧 word，含旧 version）
    if not cas128(txn.state.word, txn.state.new_word_with(new_version)):
        return 'CAS_FAILED'
    return 'OK'
```

**关键规则**：

1. **禁止回退**：`new_version < old_version` → 拒绝。
2. **禁止跳跃**：`new_version > old_version + 1` → 拒绝（必须 +1）。
3. **禁止静默不递增**：凡修改状态字段（COUNT/位图/GATE/DENIED/FROZEN/FREEZING）的操作，必须同时 VERSION+1；否则拒绝。
4. **禁止回绕**：`old_version == 0xFFFFFFFF` → VERSION_EXHAUSTED_DENY。

---

## 3. 版本递增白名单（哪些操作不修改 VERSION）

**严格白名单**（只有以下操作允许不修改 VERSION）：

| 操作 | 是否改 VERSION | 理由 |
|---|---|---|
| 只读操作（load/read 状态） | 否 | 不修改状态 |
| cleanup 的**资源端**释放（非状态字） | 否 | 不涉及状态字字段 |
| takeover 的**资源端** token 递增 | 否 | 资源端独立计数，非状态字 VERSION |

**其余所有修改状态字字段的操作（admit/release/revoke/freeze/unfreeze/recovery/set_denied）必须 VERSION+1**。

---

## 4. generation 与 version 分别实施单调性

| 计数器 | 位宽 | 单调性 | 耗尽检查 | 独立实施 |
|---|---|---|---|---|
| VERSION（状态字 bit32–63） | 32 位 | 严格 +1 | version==0xFFFFFFFF → DENY | 是 |
| TRT[slot].generation | 16 位/槽位 | 严格 +1（仅该槽位） | gen==0xFFFF → SLOT_GEN_EXHAUSTED_DENY | 是 |

两者独立，互不影响。generation 是每槽位独立；version 是全局状态字版本。

---

## 5. CAS 失败重读重验

- CAS 失败后必须 `begin_txn` 重新读完整状态（状态字 + TRT），重新验证全部条件。
- 禁止复用旧状态计算的目标值。
- 重试上限 16，耗尽 → CAS_LIVELOCK_DENY。

---

## 6. 边界测试推演

| 输入 | 预期 |
|---|---|
| version=0 | 首次递增 → 1（正常） |
| version=0xFFFFFFFE | 递增 → 0xFFFFFFFF（最后一次合法递增） |
| version=0xFFFFFFFF | 任何需递增操作 → VERSION_EXHAUSTED_DENY |
| generation=0 | 首次分配 → 1 |
| generation=0xFFFE | 分配 → 0xFFFF（最后一次） |
| generation=0xFFFF | 该槽位再分配 → SLOT_GENERATION_EXHAUSTED_DENY |

---

## 7. 与 05.40 的差异

| 项 | 05.40 | 05.41 |
|---|---|---|
| 版本递增 | 注释描述，未真正执行 | commit_txn 内强制检查 |
| 回退/跳跃检测 | 未覆盖 | VERSION_NON_MONOTONIC_DENY |
| 白名单 | 未定义 | 严格白名单（只读/资源端/cleanup） |
| generation 单调 | 全局（缺陷） | 每槽位独立 |

---

## 证据来源

- 原子边界：ATOMIC-TRANSACTION-BOUNDARY.md
- 状态转换：STATE-INVARIANTS-AND-TRANSITIONS.md
- 状态模型：UNIFIED-STATE-MODEL.md
- 证明等级：VERIFIED_STATIC（单进程）+ NOT_PROVEN（跨进程/跨重启）
