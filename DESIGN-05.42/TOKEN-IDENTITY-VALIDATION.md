# TOKEN-IDENTITY-VALIDATION.md — Token 身份完整校验（05.42 权威）

证据等级：VERIFIED_STATIC（单进程）+ NOT_PROVEN（跨进程/跨重启）
目标：修复 B41-R05——明确 Token 完整身份字段哪些参与授权校验、哪些仅审计，消除「身份字段未校验」的漏洞。

---

## 1. Token 完整身份字段与校验归属

| 字段 | 存储 | 参与授权校验？ | 校验时机 |
|---|---|---|---|
| slot | 状态字 TOKEN_ACTIVE 位图 | **是** | 消费/撤销时：bit 必须置位 |
| generation | TRT[slot].generation | **是** | 消费/撤销时：gen 必须匹配 |
| owner_id | TRT[slot].owner_id | **是** | 消费/撤销时：调用方 owner 必须匹配 |
| resource_id | TRT[slot].resource_id | **是** | 消费/撤销时：资源必须匹配 |
| state_version | 发放时 VERSION | **否（仅审计）** | 仅记录，不参与授权判定 |

**关键修正（B41-R05）**：05.41 只校验了 slot + generation，未校验 owner_id / resource_id。05.42 明确：owner_id 与 resource_id **必须参与授权校验**，防止「不同 owner/resource 携带相同 slot+generation 越权」。

state_version **从身份授权判定中拆分**，仅作审计（它不提供授权信息，混入会导致「审计字段冒充身份字段」）。

---

## 2. 完整校验路径（消费/撤销）

```python
def validate_token(txn, token, caller_owner, caller_resource):
    # token = (slot, generation)，caller 提供 owner/resource
    if not (txn.state.token_active & (1 << token.slot)):
        return 'INVALID_TOKEN'                       # slot 未活动
    rec = txn.trt[token.slot]
    if rec.generation != token.generation:
        return 'INVALID_TOKEN'                       # 代际不匹配（旧 Token）
    if rec.owner_id != caller_owner:
        return 'OWNER_MISMATCH'                      # owner 不匹配 → 越权拒绝
    if rec.resource_id != caller_resource:
        return 'RESOURCE_MISMATCH'                   # resource 不匹配 → 越权拒绝
    return 'OK'
```

**关键**：owner_id 与 resource_id 的校验**真实存在于执行路径**，不是注释。两个不同 owner/resource 携带相同 (slot, generation) 时，会因 owner/resource 不匹配而被拒绝。

---

## 3. 防伪造与防重放

| 攻击 | 防护 |
|---|---|
| 伪造 slot | slot 必须对应置位的位图 bit，且 TRT 记录匹配 |
| 伪造 generation | generation 必须等于 TRT[slot].generation |
| 伪造 owner | owner 必须等于 TRT[slot].owner_id |
| 伪造 resource | resource 必须等于 TRT[slot].resource_id |
| 重放旧 Token | generation 已递增（复用/消费），旧 gen 不匹配 |
| 越权（他 owner 的 Token） | owner_id 校验拒绝 |

---

## 4. 身份字段的来源与可信度

| 字段 | 来源 | 可信度 |
|---|---|---|
| slot | 状态字位图（WAL 重建） | 可信（随 WAL 重放） |
| generation | TRT（WAL 重建） | 可信（随 WAL 重放） |
| owner_id | TRT（WAL 重建） | 可信 |
| resource_id | TRT（WAL 重建） | 可信 |
| state_version | 发放时 VERSION（审计） | 审计用，不参与授权 |

---

## 5. 与 05.41 的差异

| 项 | 05.41 | 05.42 |
|---|---|---|
| owner_id 校验 | 未展示 | 真实校验 OWNER_MISMATCH |
| resource_id 校验 | 未展示 | 真实校验 RESOURCE_MISMATCH |
| state_version 归属 | 身份字段 | 审计字段（拆分） |

---

## 证据来源

- 数据模型：TRANSACTION-DATA-MODEL.md
- 分配/释放：TOKEN-ALLOCATION-RELEASE-REUSE.md
- 证明等级：VERIFIED_STATIC（单进程）+ NOT_PROVEN（跨进程/跨重启）
