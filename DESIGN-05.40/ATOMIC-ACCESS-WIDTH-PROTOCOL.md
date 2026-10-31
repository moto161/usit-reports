# ATOMIC-ACCESS-WIDTH-PROTOCOL.md — 原子访问宽度协议（05.40 权威）

证据等级：VERIFIED_STATIC（单进程）+ NOT_PROVEN（跨进程/跨重启）
目标：修复 B39-R02——盘点全部原子状态字段与读写入口，统一原子访问宽度，禁止 64/128 位 CAS 重叠修改。

---

## 1. 全部原子状态字段盘点（权威，全文档唯一）

| 字段 | 位段 | 读写入口 | 允许的修改方式 |
|---|---|---|---|
| COUNT | bit1–15 | admit / release / freeze / recovery | 仅 field_cas 的 field_set_count |
| GATE | bit16 | admit / freeze / recovery | 仅 field_cas 的 field_set_gate |
| DENIED | bit17 | DENIED / recovery | 仅 field_cas 的 field_set_denied（只置 1） |
| FROZEN | bit18 | freeze / unfreeze | 仅 field_cas 的 field_set_frozen |
| FREEZING | bit19 | freeze | 仅 field_cas 的 field_set_freezing |
| VERSION | bit32–63 | 全部入口 | 仅 field_cas 的 field_set_version（+1，耗尽 DENY） |
| TOKEN_ACTIVE | bit64–79 | admit / release / token_create / token_consume | 仅 field_cas 的 slot 置/清位 |
| TOKEN_GENERATION | bit80–95 | token_create（槽位复用） | 仅 field_cas 的 generation+1（耗尽 DENY） |
| 保留位 | bit20–31 / bit96–127 | 无 | 恒 0，非 0 判非法 |

**唯一写入者**：所有字段只有一个权威写入入口——`field_cas`。任何直接读改写、非原子字段修改、无条件整字覆盖均禁止。

---

## 2. 唯一原子访问宽度

**权威：128 位（16 字节）单条 CAS。**

- 64 位 CAS 路径**废除**（05.39 的 RECOVERY-VERSION-ATOMICITY.md 使用的 `hardware_cas64` 被删除）。
- 不得存在对同一状态字的任何其它宽度的 CAS、普通 store、或部分字段的非原子读写。

**硬件前提**：

| 前提 | 要求 | 不满足时的处理 |
|---|---|---|
| 128 位原子 CAS | x86-64 CMPXCHG16B（或等价） | 整体协议 DENY，不退化为 64 位 |
| 16 字节对齐 | 状态字地址 16 字节对齐 | 未对齐 → NOT_PROVEN → DENY |
| 内存序 | acquire（读）/ release（写），或 seq_cst | 弱序 → 竞态 → DENY |
| 共享内存（跨进程） | mmap + 16 字节对齐 | 跨进程 NOT_PROVEN → DENY |
| 持久化（跨重启） | 128 位状态字持久化 | 跨重启 NOT_PROVEN → DENY |

---

## 3. CAS 失败重读重算协议（强制）

每次 `field_cas` 返回 `CAS_FAILED` 后，调用者**必须**：

1. 重新 `load_word()` 读取最新完整 128 位状态字。
2. 重新验证**全部**前置条件（DENIED、GATE、FROZEN、COUNT、VERSION、TOKEN 槽位、generation）。
3. 用新读取的字重新计算目标字段值。
4. 禁止复用旧状态计算出的 `new_fields`。

```python
def admit():
    while True:
        w = load_word()
        if not admit_predicate(w):          # 谓词：DENIED=0 且 GATE=OPEN 且 COUNT<MAX 且 版本未耗尽
            return admit_fail_reason(w)
        new_fields = admit_transform(w)      # 由当前 w 计算 COUNT+1、槽位置位、VERSION+1
        r = field_cas(w, new_fields)         # 原子比较 w 整字，一致才提交
        if r == 'OK':
            return ('OK', token_from(w))     # 线性化点
        if r == 'VERSION_EXHAUSTED_DENY':
            return r
        if r == 'INVALID_STATE':
            return r
        # CAS_FAILED → 重读重算，重试
```

---

## 4. 重试上限与 LIVELOCK 终止

所有 `field_cas` 调用方统一重试上限：

| 参数 | 值 |
|---|---|
| 单操作最大重试次数 | 16（任意一次操作内） |
| 重试耗尽动作 | 返回 `CAS_LIVELOCK_DENY`，终止当前操作，安全降级 |
| 超时 | 无全局超时；依赖重试上限 |
| 状态异常 | 返回 `INVALID_STATE`，终止 |

**禁止**外层 `while True` 无限重试。内层 field_cas 循环与外层业务循环必须都有明确终止条件（继承 05.38 的重试闭合要求）。

---

## 5. 内存序与可见性

- **读**：`load_word()` 使用 acquire 语义，确保读到最新已提交状态。
- **写**：`field_cas` 成功提交使用 release 语义，确保字段修改对后续读可见。
- 跨线程：同一进程内共享内存，128 位 CAS 保证原子性（VERIFIED_STATIC）。
- 跨进程：需 mmap 共享 + HW CAS，未证明 → NOT_PROVEN。

---

## 6. 三个安全边界（统一）

| 边界 | 原子访问保证 | 状态 |
|---|---|---|
| 单进程 | 进程内 128 位 CMPXCHG16B | VERIFIED_STATIC |
| 跨进程 | 共享内存 + 16 字节对齐 + HW CAS | NOT_PROVEN → DENY |
| 跨重启 | 128 位持久化 + 恢复 | NOT_PROVEN → DENY |

---

## 7. 禁止清单（05.39 遗留必须删除）

1. `hardware_cas64()` 对状态字 bit0–63 的写入。
2. `commit(state)` 无条件整字覆盖。
3. `new_word = (old_word + 1) & 0xFFFFFFFF` 整字算术（版本与 token_seq 均禁止）。
4. `token in admit_tokens` 字典检查作为消费证明（05.39 方案 B 已废弃，Token 内嵌状态字）。
5. 任何 `atomic=True` / `transaction()` 命名而不给实际原子边界的伪代码。

---

## 证据来源

- 状态字布局：UNIFIED-ATOMIC-STATE-WORD.md
- 字段操作矩阵：STATE-TRANSITION-AND-CAS-MATRIX.md
- 05.39 缺陷：TOKEN-ATOMIC-TRANSACTION.md（128 位）与 RECOVERY-VERSION-ATOMICITY.md（64 位）重叠修改
- 证明等级：VERIFIED_STATIC（单进程）+ NOT_PROVEN（跨进程/跨重启）
