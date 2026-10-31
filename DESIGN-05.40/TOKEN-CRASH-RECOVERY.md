# TOKEN-CRASH-RECOVERY.md — Token 崩溃恢复（05.40 权威）

证据等级：VERIFIED_STATIC（单进程）+ NOT_PROVEN（跨进程/跨重启）
目标：修复 B39-R01——定义崩溃后 Token 账本如何判定未完成事务，以及旧 Token 复活与 generation 回退的防护。

---

## 1. 恢复的输入证据

恢复操作只依据以下**内嵌于状态字**的字段（不依赖进程内内存）：

| 证据 | 来源 | 可信等级 |
|---|---|---|
| TOKEN_ACTIVE 位图 | 状态字 bit64–79 | 随状态字持久化则可信，否则 NOT_PROVEN |
| TOKEN_GENERATION | 状态字 bit80–95 | 同上 |
| COUNT | 状态字 bit1–15 | 同上 |
| VERSION | 状态字 bit32–63 | 同上 |
| DENIED | 状态字 bit17 | 单调置位，不可清除 |

**关键**：恢复不依赖 `admit_tokens` 字典（05.39 方案 B 已废弃）。Token 账本即状态字内的位图 + generation。

---

## 2. 崩溃窗口分析（内嵌模型）

内嵌模型下，Token 发放/消费是单条 128 位 CAS，**不存在跨字段的中间态**：

| 崩溃点 | 状态字结果 | 恢复判定 |
|---|---|---|
| admit CAS 前 | 无变化 | 无 Token，无需恢复 |
| admit CAS 后 | 槽位占用 + gen+1 + COUNT+1 原子完成 | Token 有效，正常 |
| release CAS 前 | 槽位占用，COUNT 计入 | 业务是否提交见 §3 |
| release CAS 后 | 槽位清位 + COUNT−1 原子完成 | Token 已消费，正常 |

**与 05.39 的差异**：05.39 的「COUNT 已改但 Token 未入账」崩溃窗口（CAS 后写 `admit_tokens` 字典失败）在 05.40 内嵌模型中**不存在**，因为 Token 账本就在状态字内，无独立字典写入步骤。

---

## 3. LEAKED Token 的恢复判定

崩溃时槽位仍占用（业务未 release）：

| 业务事务状态 | 恢复动作 |
|---|---|
| 已提交（有持久化提交证据） | 槽位清位 + COUNT−1（可回收） |
| 未提交 | 槽位清位 + COUNT−1（回滚） |
| 不确定（无提交/回滚证据） | **ISOLATED**，不自动回收，COUNT 保持，冻结 DENY |

**禁止**：无提交证据时自动重放或自动回收。无法确认 Commit/Receipt 是否完成时，不得自动重放（继承 05.37 约束）。

---

## 4. 恢复 CAS 与旧 Token 复活防护

恢复操作同样通过 `field_cas`（128 位）提交，且必须：

1. 重读状态字，重验全部证据。
2. DENIED 若并发置位，恢复必须让 DENIED 优先，不得覆盖。
3. generation 单调，恢复不得回退 generation。

```python
def recover_token(slot):
    while True:
        w = load_word()
        active = field_get_token_active(w)
        gen = field_get_token_generation(w)
        denied = get_denied(decode(w))
        if denied:
            return 'DENIED'              # DENIED 优先，不回收
        if not (active & (1 << slot)):
            return 'ALREADY_FREE'
        # 判定业务事务终态（需持久化证据，见 §3）
        verdict = business_txn_verdict(slot)   # COMMITTED / ABORTED / UNKNOWN
        if verdict == 'UNKNOWN':
            return 'ISOLATED'            # 不确定，不回收
        # 可回收：清位 + COUNT−1 + VERSION+1
        r = field_cas(w, {
            'token_active': active & ~(1 << slot),
            'count':        get_count(decode(w)) - 1,
            'version':      field_get_version(w) + 1,
        })
        if r == 'OK':
            return 'RECOVERED'
        if r == 'VERSION_EXHAUSTED_DENY':
            return r
        # CAS_FAILED → 重读重算
```

**旧 Token 复活防护**：恢复只按「当前槽位位图 + 当前 generation」判定，不按任何历史 token 值。恢复清位后，旧 token 的 `gen` 与当前 generation 不匹配 → 后续 `release` 被拒。

---

## 5. 崩溃类型与恢复决策矩阵

| 崩溃类型 | generation 状态 | 恢复决策 |
|---|---|---|
| 单进程崩溃（无持久化） | 丢失，重启归 0 | **DENY**（generation 回退，旧 Token 复活风险） |
| 跨进程崩溃（共享内存） | 共享内存可能残留 | NOT_PROVEN → DENY |
| 持久化崩溃（跨重启） | 需持久化 generation | 未证明 → NOT_PROVEN → DENY |
| 进程内线程崩溃 | 状态字仍在（同进程） | 按 §3 判定，可回收或 ISOLATED |

**核心规则**：跨重启时若 generation 无法证明单调持久化，**禁止恢复 Token 账本**，整体 DENY。因为 generation 回退会让旧 Token 的 `gen` 重新等于当前 generation，旧 Token 复活。

---

## 6. 重复恢复 / 恢复者崩溃 / 恢复中断

| 场景 | 处理 |
|---|---|
| 重复恢复同一 slot | 第二次 `active & (1<<slot)==0` → ALREADY_FREE（幂等） |
| 恢复者崩溃 | 恢复 CAS 原子，要么完成要么无变化，无中间态 |
| 恢复中断 | 重读重算，幂等重试，有重试上限 |
| 恢复失败 | 返回 DENIED/ISOLATED，不自动放行 |

---

## 7. NOT_PROVEN 边界

- 跨重启 generation 持久化：NOT_PROVEN → 恢复 DENY。
- 跨进程共享内存 + HW CAS：NOT_PROVEN → DENY。
- 业务事务终态判定依赖持久化提交证据，无证据 → UNKNOWN → ISOLATED。

---

## 证据来源

- Token 身份：TOKEN-IDENTITY-AND-BITMAP-MODEL.md
- 原子事务：TOKEN-ATOMIC-TRANSACTION.md
- 状态字布局：UNIFIED-ATOMIC-STATE-WORD.md
- 历史约束：无提交证据不自动重放（05.37）、无证据 DENY
- 证明等级：VERIFIED_STATIC（单进程）+ NOT_PROVEN（跨进程/跨重启）
