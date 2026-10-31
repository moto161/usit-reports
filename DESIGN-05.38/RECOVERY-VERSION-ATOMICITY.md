# RECOVERY-VERSION-ATOMICITY.md — 恢复版本原子性证明（05.38 权威）

证据等级：VERIFIED_STATIC（含 NOT_PROVEN 项，如实标注）
目标：修复 05.37 B37-R03——版本模型不唯一、状态与版本更新可能分离、ABA 未防护。

---

## 1. 唯一权威版本模型（消除多选项）

05.37 同时提供了「状态字加版本字段」和「独立版本计数器」两个未定选项，导致实现歧义。05.38 选定**唯一权威模型**：

**状态与版本合并在同一原子字内**（composite atomic word）。

```
原子字布局（64 位）：
  bit0–31   状态字段（gate/count/frozen/DENIED 及保留位）
  bit32–63  版本号 version（单调递增，32 位无符号）
```

- 状态与版本**同一内存位置**，由**单条原子 CAS**（64 位 CMPXCHG）同时比较和更新。
- 不存在「状态更新成功但版本未更新」的分离窗口。

---

## 2. 原子读取 / 比较 / 更新

```python
def load_state_version():
    """原子读取 64 位字，返回 (state, version)。"""
    return (word & STATE_MASK, (word >> 32) & 0xFFFFFFFF)

def cas_state_version(old_state, old_version, new_state):
    """单条 64 位 CAS：仅当 (state, version) 与期望一致时提交。
    成功则 version+1（原子递增，与状态更新不可分离）。"""
    old_word = (old_state & STATE_MASK) | ((old_version & 0xFFFFFFFF) << 32)
    new_word = (new_state & STATE_MASK) | (((old_version + 1) & 0xFFFFFFFF) << 32)
    return hardware_cas64(old_word, new_word)
```

**关键**：`cas_state_version` 把「状态更新」和「版本递增」绑定在**同一条硬件 CAS** 中，二者不可分离。

---

## 3. 版本号存储布局 / 对齐 / 溢出 / 回绕 / 持久化

| 项 | 定义 |
|---|---|
| 原子宽度 | 64 位（状态 32 + 版本 32） |
| 对齐 | 8 字节自然对齐 |
| 溢出 | 32 位版本，约 42 亿次更新才溢出 |
| 回绕 | 版本回绕到 0 时，ABA 风险见 §4，采用「版本 + 状态 + epoch」三重判定 |
| 持久化 | 版本号随状态字持久化（若跨进程共享需共享内存 + 持久化映射） |
| 跨进程可见 | 共享内存映射时可见；进程内则仅进程内 |

---

## 4. ABA 防护

**ABA 场景**：状态值从 A→B→A，版本从 V→V+1→V+2。若只比较状态值，会把「最终 A(版本 V+2)」误认为「初始 A(版本 V)」。

**防护**：`cas_state_version` 同时比较 `(state, version)`。即使状态值回绕到相同值，版本号已递增，`(state, version)` 对不等，CAS 失败。

```python
# 演示 ABA 防护
old = (state=A, version=V)
# ... 期间 A→B(version V+1)→A(version V+2)
cas_state_version(A, V, new)   # 期望 (A,V)，实际 (A,V+2) → 不匹配 → 失败
```

版本号回绕（V+2^32 回到 V）是理论 ABA 残余风险，需额外「epoch 持久化锚点」防回绕。**本设计在无持久化 epoch 锚点时，版本回绕后的 ABA 防护 NOT_PROVEN**（见 §6）。

---

## 5. 跨进程原子性证明前提

| 前提 | 要求 | 不满足后果 |
|---|---|---|
| 硬件 64 位 CAS | 平台支持 CMPXCHG8B 或等价 | 非原子 → 撕裂 → NOT_PROVEN |
| 共享内存映射 | 状态字在共享内存 | 进程内变量不可跨进程 CAS |
| 内存序 | acquire-release 语义 | 重排 → 竞态 |
| 持久化版本 | 跨重启需持久化 | 崩溃后版本丢失 → 无法防 ABA/回绕 |

**本设计明确**：进程内状态字 + 64 位 CAS → 原子性成立（VERIFIED_STATIC）。跨进程共享或跨重启持久化 → 需额外证明硬件 CAS + 共享内存 + 持久化 epoch，未证明则 **NOT_PROVEN**，跨进程恢复 DENY。

---

## 6. NOT_PROVEN 边界

- 跨进程共享状态字的 64 位 CAS 原子性：取决于平台 HW 支持，未在目标环境证明 → NOT_PROVEN。
- 版本号回绕后的 ABA 防护（无持久化 epoch 锚点）：NOT_PROVEN。
- 跨重启版本持久化一致性：NOT_PROVEN。

在这些 NOT_PROVEN 条件下，跨进程恢复、跨重启恢复必须 DENY，仅在单进程内（进程内状态字 + HW CAS）可证明原子性。

---

## 证据来源

- 恢复 CAS 协议：RECOVERY-CAS-PROTOCOL.md（05.38 修订）
- 崩溃恢复：CRASH-RECOVERY-PROTOCOL.md（05.38 修订）
- 05.37 缺陷：RECOVERY-CAS-PROTOCOL.md（05.37 版）的独立版本 + 陈旧 old_state
- 证明等级：VERIFIED_STATIC（单进程）+ NOT_PROVEN（跨进程/持久化）
