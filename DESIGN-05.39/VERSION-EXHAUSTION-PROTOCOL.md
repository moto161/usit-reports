# VERSION-EXHAUSTION-PROTOCOL.md — 版本耗尽协议（05.39 权威）

证据等级：VERIFIED_STATIC
目标：修复 B38-R03——版本回绕伪代码 `(old_version + 1) & 0xFFFFFFFF` 在 version==0xFFFFFFFF 时静默回绕，与「回绕必须 DENY」的文字声明矛盾。

---

## 1. 05.38 缺陷定位

05.38 的 `RECOVERY-VERSION-ATOMICITY.md` 伪代码：

```python
new_word = (
    (new_state & STATE_MASK)
    | (((old_version + 1) & 0xFFFFFFFF) << 32)   # ← 版本回绕
)
```

当 `old_version == 0xFFFFFFFF` 时，`(old_version + 1) & 0xFFFFFFFF == 0`，版本静默回绕到 0。但文档文字声明「回绕必须 DENY」。**伪代码与文字矛盾**。

---

## 2. 唯一权威规则：禁止回绕

版本耗尽（`version == 0xFFFFFFFF`）时，**必须在 CAS 提交前返回终止状态**，禁止回绕：

```python
def cas_state_version(old_state, old_version, new_state):
    # 版本耗尽检查（CAS 提交前，强制）
    if old_version == 0xFFFFFFFF:
        return 'VERSION_EXHAUSTED_DENY'   # 禁止回绕，终止
    old_word = (old_state & STATE_MASK) | ((old_version & 0xFFFFFFFF) << 32)
    new_word = (new_state & STATE_MASK) | (((old_version + 1) & 0xFFFFFFFF) << 32)
    return hardware_cas64(old_word, new_word)
```

**关键**：版本耗尽检查在**提交路径**中，而不是仅在文字说明。删除 05.38 的静默回绕逻辑。

---

## 3. 版本耗尽状态转移表

| 阶段 | 版本 | 状态 | 动作 |
|---|---|---|---|
| 耗尽前 | < 0xFFFFFFFF | 正常 | CAS 正常递增 |
| 耗尽时 | == 0xFFFFFFFF | 可提交最后一次 | 若再需递增 → DENY |
| 耗尽后 | （不可回绕） | VERSION_EXHAUSTED | 拒绝一切需递增的操作 |
| 恢复时 | 耗尽态 | DENY | 禁止快照恢复绕过耗尽 |

---

## 4. 禁止绕过耗尽

以下路径均**不得**绕过版本耗尽：

- 静默清零（`& 0`）→ 禁止。
- 回绕（`& 0xFFFFFFFF` 环绕）→ 禁止。
- 重新初始化版本 → 禁止。
- 快照恢复旧版本 → 禁止（版本单调，不回退）。

---

## 5. Epoch 模型的统一

05.38 声称「版本+状态+epoch 三重判定」，但实际 CAS 只比较 `state+version`，epoch 未进入 CAS。05.39 统一：

**权威模型 = 状态 + 版本（64 位原子字），无 epoch。**

- epoch 仅在「有持久化 epoch 锚点」时才引入，且必须进入 CAS 比较值。
- 无持久化 epoch 时，不声称三重判定。
- 版本耗尽在无 epoch 时直接 DENY，不通过 epoch 续期。

```python
# 有持久化 epoch 时（需证明），版本耗尽可续期：
if old_version == 0xFFFFFFFF and epoch_supported():
    return epoch_renew()   # 仅当 epoch 证明成立才续期
return 'VERSION_EXHAUSTED_DENY'
```

---

## 6. 三个安全边界

| 边界 | 版本语义 | 耗尽处理 | 状态 |
|---|---|---|---|
| 单进程 | 进程内 64 位 | DENY | VERIFIED_STATIC |
| 跨进程 | 共享内存 HW CAS | DENY（HW CAS 未证明） | NOT_PROVEN |
| 跨重启 | 需持久化版本 | DENY（持久化未证明） | NOT_PROVEN |

---

## 7. 与 Token 耗尽的一致性

Token `token_seq`（32 位）与版本 version（32 位）采用**同一耗尽协议**：达到上限即 DENY，禁止回绕。二者统一由本协议约束。

---

## 证据来源

- 版本原子性：RECOVERY-VERSION-ATOMICITY.md（05.39 修订）
- 05.38 缺陷：RECOVERY-VERSION-ATOMICITY.md（05.38 版）§2 的回绕逻辑
- 证明等级：VERIFIED_STATIC
