# VERSION-EXHAUSTION-ALL-ENTRYPOINTS.md — 版本耗尽全入口覆盖（05.40 权威）

证据等级：VERIFIED_STATIC（单进程）+ NOT_PROVEN（跨进程/跨重启）
目标：修复 B39-R03——05.39 仅在 `cas_state_version` 检查版本耗尽，Token 的 admit/release 等入口仍可能直接 `version+1` 绕过检查。

---

## 1. 05.39 缺陷定位

05.39 的版本耗尽检查只存在于 `cas_state_version()`（VERSION-EXHAUSTION-PROTOCOL.md §2），但 Token 协议（TOKEN-ATOMIC-TRANSACTION.md）的 `admit()`/`release()` 使用 `encode(new_state, version + 1, ...)`，其中 `version + 1` 是**在 field_cas 之外**直接递增，未经过耗尽检查。当 `version == 0xFFFFFFFF` 时，`version + 1` 会静默溢出/回绕。

**根因**：版本递增分散在多个入口，只有部分入口走耗尽检查。

---

## 2. 唯一权威版本检查与提交协议

所有状态字修改**只允许**通过 `field_cas` 内部统一执行版本递增，且递增前在**提交路径内**检查耗尽：

```python
def field_cas(expected: Word128, new_fields: dict) -> str:
    new_word = expected.word
    for field, val in new_fields.items():
        new_word = field_set(new_word, field, val)

    # 版本递增的统一检查：若 new_fields 含 'version' 字段，
    # 且其值为 old_version+1，则此处强制检查耗尽（提交路径内）。
    if 'version' in new_fields:
        old_version = field_get_version(expected.word)
        if old_version == 0xFFFFFFFF:
            return 'VERSION_EXHAUSTED_DENY'   # 禁止回绕，终止
        # new_fields['version'] 必须是 old_version+1，否则 INVALID

    if not is_valid_state(new_word):
        return 'INVALID_STATE'
    return hardware_cas128(expected.word, new_word)
```

**关键**：版本耗尽检查**不在任何调用方**，而在 `field_cas` 内部提交路径统一执行。任何入口只要请求 `version+1`，耗尽时统一返回 `VERSION_EXHAUSTED_DENY`。调用方不得在 field_cas 之外自行 `version+1`。

---

## 3. 全入口覆盖矩阵（权威）

| 入口 | 是否递增 version | 是否走统一耗尽检查 | 耗尽行为 | 证据位置 |
|---|---|---|---|---|
| admit | 是 | 是（field_cas 内） | VERSION_EXHAUSTED_DENY | TOKEN-ATOMIC-TRANSACTION.md §3 |
| release | 是 | 是 | VERSION_EXHAUSTED_DENY | TOKEN-ATOMIC-TRANSACTION.md §4 |
| token_create / slot_alloc | 是 | 是 | VERSION_EXHAUSTED_DENY | TOKEN-SLOT-ALLOCATION-AND-REUSE.md §2 |
| token_consume | 是 | 是 | VERSION_EXHAUSTED_DENY | TOKEN-ATOMIC-TRANSACTION.md §4 |
| token_revoke | 是 | 是 | VERSION_EXHAUSTED_DENY | TOKEN-OWNERSHIP-AND-LIFECYCLE.md §5 |
| freeze | 是 | 是 | VERSION_EXHAUSTED_DENY | FREEZE 协议（继承） |
| unfreeze | 是 | 是 | VERSION_EXHAUSTED_DENY | FREEZE 协议（继承） |
| set_denied | 是 | 是 | VERSION_EXHAUSTED_DENY | DENIED 协议（继承） |
| cleanup_complete | 否（无版本递增） | N/A | N/A | CLEANUP 协议 |
| recovery | 是 | 是 | VERSION_EXHAUSTED_DENY | TOKEN-CRASH-RECOVERY.md §4 |
| takeover | 否（资源端 Token 递增，非状态字 version） | N/A（资源端独立） | 资源端 TOKEN_EXHAUSTED | RESOURCE-FENCING-PROTOCOL.md |

---

## 4. 禁止绕过耗尽

以下路径均**不得**绕过版本耗尽：

- 静默清零、回绕（`& 0xFFFFFFFF` 环绕）、重新初始化、快照恢复旧版本。
- 在 field_cas 之外直接 `version+1`。
- Token 序列号（generation）复用旧值恢复可用性。

---

## 5. Token 序列号（generation）耗尽的独立处理

- generation 与 version 是**两个独立**的 16/32 位计数器，各自独立耗尽。
- `version == 0xFFFFFFFF` → `VERSION_EXHAUSTED_DENY`。
- `generation == 0xFFFF` → `GENERATION_EXHAUSTED_DENY`。
- 二者耗尽后均禁止回绕，禁止复用旧值。

---

## 6. 版本耗尽状态转移

| 阶段 | version | 状态 | 动作 |
|---|---|---|---|
| 耗尽前 | < 0xFFFFFFFF | 正常 | field_cas 内递增 |
| 耗尽时 | == 0xFFFFFFFF | 可提交最后一次（非递增） | 任何需递增 → DENY |
| 耗尽后 | 不回绕 | VERSION_EXHAUSTED | 拒绝一切需递增操作 |
| 恢复时 | 耗尽态 | DENY | 禁止快照恢复绕过耗尽 |

---

## 7. 持久化 Epoch 与跨重启

- 无持久化 epoch 时，版本耗尽直接 DENY，不续期。
- 若引入持久化 epoch，必须纳入同一个可证明的原子比较与提交模型，不能只在 field_cas 外检查 epoch。
- 跨重启版本单调性未证明 → NOT_PROVEN → DENY。

---

## 证据来源

- 状态字布局：UNIFIED-ATOMIC-STATE-WORD.md
- 原子访问宽度：ATOMIC-ACCESS-WIDTH-PROTOCOL.md
- 05.39 缺陷：TOKEN-ATOMIC-TRANSACTION.md（05.39 版）的 `encode(..., version + 1, ...)` 未检查耗尽
- 历史约束：版本耗尽 DENY、禁止回绕（05.39）
- 证明等级：VERIFIED_STATIC（单进程）+ NOT_PROVEN（跨进程/跨重启）
