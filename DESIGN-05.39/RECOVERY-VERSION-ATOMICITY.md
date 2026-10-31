# RECOVERY-VERSION-ATOMICITY.md — 恢复版本原子性（05.39 修订）

证据等级：VERIFIED_STATIC（含 NOT_PROVEN 项）
版本耗尽见 VERSION-EXHAUSTION-PROTOCOL.md（权威）。

---

## 1. 唯一权威版本模型

状态（bit0–31）+ 版本（bit32–63）合并为 **64 位原子字**，单条 CAS 比较更新。**无 epoch**（除非有持久化 epoch 锚点并进入 CAS，见 VERSION-EXHAUSTION-PROTOCOL.md §5）。

---

## 2. 原子读取/比较/更新（含版本耗尽检查）

```python
def cas_state_version(old_state, old_version, new_state):
    # 版本耗尽检查（提交路径内，禁止回绕）
    if old_version == 0xFFFFFFFF:
        return 'VERSION_EXHAUSTED_DENY'
    old_word = (old_state & STATE_MASK) | ((old_version & 0xFFFFFFFF) << 32)
    new_word = (new_state & STATE_MASK) | (((old_version + 1) & 0xFFFFFFFF) << 32)
    return hardware_cas64(old_word, new_word)
```

**关键修正（B38-R03）**：版本耗尽检查在提交路径内，删除静默回绕。`old_version+1` 在 `old_version < 0xFFFFFFFF` 时不会溢出（已前置检查）。

---

## 3. ABA 防护

`cas_state_version` 同时比较 `(state, version)`。状态值回绕但版本递增时，`(state, version)` 对不等，CAS 失败。

版本回绕到 0 的 ABA 残余风险：因版本耗尽直接 DENY（不回绕），**ABA 通过「禁止回绕」而非「回绕后检测」来防护**。无持久化 epoch 时，版本不会回绕，故 ABA 风险被消除。

---

## 4. 硬件前提

| 前提 | 要求 | 不满足 |
|---|---|---|
| 64 位 HW CAS | CMPXCHG8B 或等价 | 撕裂 → NOT_PROVEN |
| 8 字节对齐 | 自然对齐 | 非原子 |
| acquire-release 内存序 | 至少 | 重排 → 竞态 |
| 共享内存（跨进程） | mmap | 进程内变量不可跨进程 |
| 持久化（跨重启） | 版本持久化 | 崩溃丢版本 |

---

## 5. 三个安全边界

| 边界 | 原子性 | 状态 |
|---|---|---|
| 单进程 | 进程内 64 位 CAS | VERIFIED_STATIC |
| 跨进程 | 需共享内存 + HW CAS | NOT_PROVEN |
| 跨重启 | 需持久化版本 | NOT_PROVEN |

---

## 6. 与 05.38 的差异

| 项 | 05.38 | 05.39 |
|---|---|---|
| 版本回绕 | `(old+1) & 0xFFFFFFFF` 静默回绕 | 版本耗尽 DENY，禁止回绕 |
| epoch | 声称三重判定但未进 CAS | 移除，或进入 CAS（需证明） |
| ABA 防护 | 依赖版本比较 | 依赖「禁止回绕」消除残余风险 |

---

## 证据来源

- 版本耗尽：VERSION-EXHAUSTION-PROTOCOL.md
- 05.38 缺陷：RECOVERY-VERSION-ATOMICITY.md（05.38 版）§2
- 证明等级：VERIFIED_STATIC（单进程）+ NOT_PROVEN（跨进程/持久化）
