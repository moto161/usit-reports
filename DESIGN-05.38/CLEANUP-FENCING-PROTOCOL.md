# CLEANUP-FENCING-PROTOCOL.md — 清理资源 Fencing 协议（05.38 权威）

证据等级：VERIFIED_STATIC（含 NOT_PROVEN 项，如实标注）
目标：修复 05.37 B37-R01——`ownership_version` 递增不能阻止旧执行者继续执行已开始的资源释放操作。

---

## 1. 05.37 缺陷定位

05.37 的接管协议依赖 `ownership_version += 1` 使旧 owner「失效」。但存在根本缺陷：

- **版本递增只能阻止「未来的」释放动作**（旧执行者在版本检查点会失败）。
- **无法阻止「已开始但未完成的」释放动作**：若旧执行者已经通过了版本检查、正在执行外部释放（如 `close()`/`unlink()`/`flock`），版本递增无法撤销这个已在途的操作。
- 结果：接管者重新释放同一资源时，与旧执行者的在途释放发生**并发释放**。

这是「软件层所有权校验」与「资源端实际释放」之间的根本区别，05.37 未区分。

---

## 2. 三个必须区分的安全概念

| 概念 | 保护对象 | 作用域 | 局限 |
|---|---|---|---|
| 软件层所有权校验 | 是否允许发起释放 | 进程内（owner_id + version 检查） | 无法撤销在途外部动作 |
| 资源端 Fencing | 外部资源是否接受旧执行者的操作 | 跨进程/资源端 | 需要资源支持 Fencing Token |
| 操作幂等性 | 重复释放是否安全 | 资源本身 | 仅对天然幂等的释放（unlink/close）有效 |

**三者不能混为一谈**。05.37 把「软件层校验」当成了「Fencing」，是本缺陷的根因。

---

## 3. 权威存储与保护主体

| 对象 | 保护主体 | 权威存储 | 是否跨进程共享 |
|---|---|---|---|
| `owner_id` | 清理任务级锁（进程内） | 进程内 CleanupManager | 否 |
| `ownership_version` | 清理任务级锁（进程内） | 进程内 CleanupManager | 否 |
| 资源状态（released/uncertain） | 资源级 Fencing Token | 资源端（文件/锁/事务） | 是（取决于资源） |
| 释放动作本身 | 资源端 Fencing | 资源端 | 是 |

**关键**：资源状态和释放动作的权威存储在**资源端**，而非进程内。进程内的 `owner_id`/`version` 只能做「是否发起」的软件层校验，不能做「资源端是否接受」的 Fencing。

---

## 4. 资源释放前的验证与 TOCTOU

释放流程必须区分「软件层校验」和「资源端 Fencing」两个步骤：

```python
def release_resource(rname, executor_id, fencing_token):
    r = self._resources[rname]

    # 步骤1：软件层所有权校验（进程内）
    if not self._check_ownership(executor_id):
        return 'OWNERSHIP_LOST'        # 已被接管，禁止发起

    # 步骤2：资源端 Fencing（跨进程，若资源支持）
    if r.fencing_supported:
        if not r.resource.accept_fencing_token(fencing_token):
            return 'FENCED_OUT'        # 资源端拒绝旧 Token
        # 关键：accept 与 release 之间必须是原子或由资源端保证串行
        r.resource.release(fencing_token)   # 资源端校验 Token 后释放
    else:
        # 资源不支持 Fencing → 无法安全接管，见 §6
        return 'FENCING_UNSUPPORTED'
    r.released = True
```

**TOCTOU 窗口**：`accept_fencing_token` 与 `release` 之间若有窗口，旧执行者可能在此间隙被接管。消除方法：
- 资源支持「携带 Token 的原子释放」（release 本身校验 Token）→ 无 TOCTOU。
- 资源不支持 → 无法消除窗口 → NOT_PROVEN，禁止接管。

---

## 5. 旧执行者暂停→接管→复活的完整时序

```
T0: 旧执行者 A 取得所有权(owner=A, version=1)
T1: A 通过软件层校验，准备释放资源 R（尚未执行外部释放）
T2: A 被暂停（崩溃/阻塞）
T3: 心跳超时 → 接管者 B 触发接管调查
T4: B 验证 R 状态（未释放），取得接管所有权(owner=B, version=2)
T5: B 通过资源端 Fencing Token 释放 R（若 R 支持 Fencing）
T6: A 复活，尝试继续释放 R
    → 软件层校验失败（version=2≠1）→ A 无法发起
    → 即使 A 在 T2 前已通过校验、正在释放，资源端 Fencing Token 拒绝 A 的旧 Token
```

**结论**：只有「软件层校验」+「资源端 Fencing」双保险，才能保证 A 复活后无法并发释放 R。仅有 version 递增（05.37 的做法）**不能**阻止 T2 前已发起的在途释放。

---

## 6. Fencing 不支持的资源 → NOT_PROVEN

如果外部资源不支持 Fencing Token 或等价机制（如普通文件 `unlink` 无 Token 概念、某些锁实现）：

- **必须标记该资源无法安全接管**。
- 禁止仅凭心跳超时接管并并发释放。
- 接管动作降级为「只读调查」，不执行释放；旧执行者复活后自行完成释放，或资源进入 ISOLATED。

```python
if not r.fencing_supported:
    # 无法安全接管 → 不释放，进入隔离
    r.uncertain = True
    return 'FENCING_UNSUPPORTED'   # 状态不明，ISOLATED
```

**心跳超时只能触发「接管调查」，不能单独作为「旧执行者已停止」的证明。** 这是硬性规则。

---

## 7. 完整状态转换（含负向场景）

| 场景 | 软件层校验 | 资源端 Fencing | 结果 |
|---|---|---|---|
| 正常释放 | 通过 | 通过 | released=True |
| 旧执行者复活后发起 | 失败（version 不匹配） | — | OWNERSHIP_LOST |
| 旧执行者在途释放 + 接管者释放 | 旧已通过 | 资源端拒绝旧 Token | FENCED_OUT（旧动作无效） |
| 释放结果不确定 | — | 资源状态未知 | uncertain=True → ISOLATED |
| 执行者崩溃 | — | 资源状态需调查 | 心跳超时 → 只读调查 |
| 接管者崩溃 | — | 资源状态需再调查 | 新接管者重新调查 |
| 重复释放 | 幂等 | 幂等（若支持） | no-op |
| 部分释放 | — | 部分状态不明 | uncertain → ISOLATED |
| 资源端拒绝旧 Token | — | FENCED_OUT | 旧动作无效，安全 |

---

## 8. 资源级状态 vs 任务级状态的原子性边界

- **任务级状态**（CLEANUP_NOT_STARTED/CLAIMED/IN_PROGRESS/PARTIAL/COMPLETED/FAILED/ISOLATED）：由进程内清理锁保护，不跨进程。
- **资源级状态**（released/uncertain/fencing_token）：由资源端保护，跨进程。

两者的原子性边界：**任务级状态只描述「清理流程走到哪一步」，不描述「资源是否真的释放了」**。资源是否释放由资源端 Fencing 决定。05.37 把任务级 `_cleanup_done` 当成了资源释放的证明，是混淆。

修正后：`_cleanup_done = True` 仅在「每项资源都通过资源端 Fencing 确认 released 或 verified-safe」时置位。

---

## 9. 跨进程安全声明

- 进程内锁**只能**保护进程内的 owner/version 状态。
- 跨进程资源释放安全**必须**依赖资源端 Fencing Token。
- 不满足资源端 Fencing 时，不得声称跨进程释放安全 → NOT_PROVEN → 禁止接管释放。

---

## 10. NOT_PROVEN 边界（如实标注）

- 对**不支持 Fencing Token** 的资源（普通文件 unlink、部分锁、无 Token 语义的句柄），跨进程安全接管释放**无法证明** → NOT_PROVEN → 禁止自动接管释放，资源进入 ISOLATED。
- 这是本协议的核心诚实边界：宁可标记 NOT_PROVEN 并 DENY，也不伪造「版本递增即安全」的假证明。

---

## 证据来源

- 05.37 缺陷：IDEMPOTENT-CLEANUP-PROTOCOL.md（05.37 版）§5 的 version 递增接管
- 资源所有权矩阵：CLEANUP-OWNERSHIP-MATRIX.md（05.38 修订）
- 证明等级：VERIFIED_STATIC + NOT_PROVEN（Fencing 不支持资源）
