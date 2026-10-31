# TOKEN-IDENTITY-CANONICAL-MODEL.md — Token 身份规范模型（05.43 权威）

工单：USIT-STAGE-B-DESIGN-05.43
工作包：B43-07（修复 B42-R07）
证据等级：VERIFIED_STATIC（单进程）+ NOT_PROVEN（跨进程/跨重启）
目标：消除 05.42 中「state_version 既写入身份元组又说仅审计不参与身份验证」的前后矛盾，建立唯一的 Token 身份定义。

---

## 1. B42-R07 缺陷（05.42 真实矛盾）

05.42 存在两处冲突：

- `TRANSACTION-DATA-MODEL.md §5`：Token 身份元组 = `(slot, generation, owner_id, resource_id, state_version)`。
- `TOKEN-IDENTITY-VALIDATION.md`：`state_version` 仅用于审计，不参与身份验证。

**矛盾本质**：同一字段 `state_version` 既被列入身份元组，又被声明为「不参与身份验证」，语义自相矛盾。身份字段的定义必须唯一。

---

## 2. 唯一权威 Token 身份定义

**Token 身份（参与鉴权的字段，唯一权威）**：

```
TokenIdentity = (slot, generation, owner_id, resource_id)
```

**审计字段（不参与鉴权）**：

```
state_version   # 仅记录 Token 发放时的状态版本，供审计追溯，不参与任何身份比较
```

**规则**：
1. 身份元组**只包含 4 个字段**：`slot`、`generation`、`owner_id`、`resource_id`。
2. `state_version` 明确从身份元组中**移除**，归类为审计字段。
3. 任何消费/撤销/释放操作的身份校验，**只比较这 4 个字段**，不比较 `state_version`。

---

## 3. 各身份字段的来源、提供方、校验方

| 字段 | 含义 | 权威来源 | 校验位置 | 防伪造/重放 |
|---|---|---|---|---|
| `slot` | 槽位号（0–15） | 分配时由系统分配 | 活动位图 + TRT | 槽位须处于 ACTIVE |
| `generation` | 每槽位代际 | TRT，单调递增 | TRT 记录比对 | 旧代际因不匹配被拒 |
| `owner_id` | 所有者身份 | **可信身份上下文**（非调用方自报） | 资源端授权 | 与资源 owner 比对 |
| `resource_id` | 绑定资源 | 分配时绑定 | TRT 记录比对 | 与资源记录比对 |
| `state_version`（审计） | 发放时状态版本 | 状态字 VERSION | 不参与鉴权 | 仅审计 |

---

## 4. 可信身份上下文 vs 调用方声明值

**关键修正（对应 B43-06 与 B43-07）**：

- `owner_id` **不能由调用方自行传入当作已认证身份**。调用方传入的 `owner_id` 是**声明值**，不是**可信身份**。
- 可信身份来自**服务端建立的身份上下文**（例如已认证会话绑定的 principal、进程内受信调用者），在 `run_txn` 入口由系统注入，标记为 `caller_owner`（可信）。
- 校验时用**可信的 `caller_owner`** 与资源端记录的 `resource.owner` / TRT 中的 `owner_id` 比对，**不用调用方自报值**。

```python
# 正确：owner 来自可信上下文，而非调用方参数
def release(token, req_id, trusted_caller_owner):   # trusted_caller_owner 由系统注入
    ...
    if token.owner_id != trusted_caller_owner:
        return 'OWNER_MISMATCH'      # 拒绝，不执行 operation
```

---

## 5. 统一校验伪代码（唯一权威）

```python
def validate_token(token, trusted_caller_owner, resource):
    # 1. 槽位活动性
    if not is_slot_active(token.slot):
        return 'INVALID_TOKEN'
    # 2. 代际匹配
    if token.generation != trt[token.slot].generation:
        return 'STALE_TOKEN'               # 旧代际被拒
    # 3. owner 匹配（用可信身份，不用自报值）
    if token.owner_id != trusted_caller_owner:
        return 'OWNER_MISMATCH'
    # 4. resource 匹配
    if token.resource_id != resource.resource_id:
        return 'RESOURCE_MISMATCH'
    # state_version 不参与上述任何比较
    return 'VALID'
```

**关键**：`state_version` 不出现在任何比较分支中。四个身份字段逐一校验，任一不匹配即拒绝。

---

## 6. 与资源端授权的关系

- Token 身份中的 `owner_id` 与资源端 `Resource.owner` 的比对是**资源端授权的核心**（见 RESOURCE-OWNER-AUTHORIZATION.md）。
- 普通递增 u64（generation）本身**不构成不可伪造凭据**；Token 的授权能力来自「可信 caller_owner + 服务端持有 token 记录」这一组合，而非 token 数值的不可伪造性。
- 跨进程场景下，若调用方无法提供可信身份上下文，Token 授权 **NOT_PROVEN → DENY**。

---

## 7. 与 05.42 的差异

| 项 | 05.42 | 05.43 |
|---|---|---|
| 身份元组 | 含 state_version（5 字段） | 4 字段，state_version 移除 |
| state_version 定位 | 身份字段 + 审计字段（矛盾） | 纯审计字段 |
| owner_id 来源 | 调用方自报 | 可信身份上下文注入 |
| 校验字段 | 未完整展示 | slot/generation/owner/resource 四字段逐一校验 |

---

## 8. 证据来源与证明边界

- 资源端授权：RESOURCE-OWNER-AUTHORIZATION.md
- 数据模型：TRANSACTION-DATA-MODEL（05.42，身份字段以本文为准）
- 崩溃矩阵：CRASH-AND-CONCURRENCY-MATRIX.md

**证明等级**：单进程可信上下文 + 四字段校验 → VERIFIED_STATIC；跨进程可信身份传递 → NOT_PROVEN，维持 DENY。
