# RESOURCE-OWNER-AUTHORIZATION.md — 资源端 Owner 授权（05.43 权威）

工单：USIT-STAGE-B-DESIGN-05.43
工作包：B43-06（修复 B42-R06）
证据等级：VERIFIED_STATIC（单进程）+ NOT_PROVEN（跨进程/跨重启）
目标：明确资源端释放操作的调用方身份可信来源，杜绝普通递增 u64 未经证明就被当作不可伪造授权凭据。

---

## 1. B42-R06 缺陷（05.42 真实问题）

05.42 的 `atomic_release(token, req_id, operation)` **没有 caller_owner 参数**，也没有展示对 `Resource.owner` 的校验。当前主要依赖 `token == current_token` 判断，但：

- `current_token` 是普通递增 u64，**不是不可伪造凭据**——任何知道当前值的调用方都可构造匹配值。
- 缺少 `owner` 校验，无法防止「持有 token 数值但非资源所有者」的越权释放。

---

## 2. 授权模型（05.43 修订）

**资源端释放授权 = 三重校验，缺一不可**：

1. **Token 有效性**：`slot` 活动 + `generation` 匹配（旧代际被拒）。
2. **Owner 授权**：`trusted_caller_owner == token.owner_id == resource.owner`。
3. **资源绑定**：`token.resource_id == resource.resource_id`。

**可信身份来源**：
- `trusted_caller_owner` 由**服务端受信调用上下文**注入（例如已认证会话的 principal、进程内受信调用者），**不是调用方自行传入的参数**。
- 调用方传入的 `owner_id` 是声明值，**不可信**；只有系统注入的 `trusted_caller_owner` 才是可信身份。

---

## 3. 授权伪代码（唯一权威）

```python
def atomic_release(token, req_id, trusted_caller_owner, operation):
    # 授权：三重校验，全部通过才允许执行
    if not is_slot_active(token.slot):
        return 'INVALID_TOKEN'
    if token.generation != trt[token.slot].generation:
        return 'STALE_TOKEN'
    if token.owner_id != trusted_caller_owner:       # 可信身份，非自报值
        return 'OWNER_MISMATCH'
    if token.resource_id != resource.resource_id:
        return 'RESOURCE_MISMATCH'

    # 授权通过，才进入幂等查询与 operation（见 RESOURCE-FAILURE-AND-RECOVERY.md）
    ...
```

**关键**：
1. `trusted_caller_owner` 是函数参数，但由**系统在调用点注入**，调用方无法伪造。
2. 授权失败**不执行 operation、不写任何成功消费记录**。
3. `current_token`（Fencing 单调值）只用于**防旧 Owner 复活**，**不单独承担 Owner 授权**——它不证明调用者是谁。

---

## 4. current_token 的定位（澄清）

| 机制 | 职责 | 是否构成授权凭据 |
|---|---|---|
| `current_token`（Fencing 单调推进） | 防旧 Owner 复活、防重复释放 | **否**（普通递增 u64，可被观察者猜测） |
| `trusted_caller_owner` | 证明调用者身份 | **是**（服务端注入，不可伪造） |
| `generation`（每槽位代际） | 防旧 Token 复活 | 否（配合 owner 校验才有效） |

**结论**：`current_token` 与 `generation` 是**单调推进机制**，本身不证明调用者身份；真正的授权来自 `trusted_caller_owner` 与资源记录的比对。

---

## 5. 攻击反例（对应 B43-OWN-xxx）

| 攻击 | 预期结果 | 设计结果 |
|---|---|---|
| 伪造 token（猜测 generation/current_token 数值） | 拒绝 | `STALE_TOKEN` / `INVALID_TOKEN`（generation 不匹配） |
| 窃取 token（知道正确 generation，但非 owner） | 拒绝 | `OWNER_MISMATCH`（trusted_caller_owner 不匹配） |
| 旧 Owner 复活（接管后带旧 token 释放） | 拒绝 | `STALE_TOKEN`（generation 已推进） |
| 并发接管与释放 | 单一方成功 | Fencing 串行化（见 RESOURCE-TAKEOVER-RELEASE-SERIALIZATION） |
| 网络延迟后重放旧请求 | 幂等返回 | `ALREADY_RELEASED` 或 `STALE_TOKEN` |
| 重复请求 | 幂等返回 | `ALREADY_RELEASED` |

---

## 6. 跨进程授权边界

- 单进程内，`trusted_caller_owner` 由进程内受信上下文注入，**VERIFIED_STATIC**。
- 跨进程/跨主机，如何传递可信身份（例如证书、签名票据、受信通道）**NOT_PROVEN**，当前不声称支持，跨进程资源释放授权 → **DENY**。
- 若未来引入跨进程身份（如 mTLS 证书、签名的 owner 票据），需单独设计身份传递协议，并证明其不可伪造性；在此之前继续 DENY。

---

## 7. 与 05.42 的差异

| 项 | 05.42 | 05.43 |
|---|---|---|
| caller_owner 参数 | 无 | `trusted_caller_owner`（系统注入） |
| Resource.owner 校验 | 缺失 | 三重校验之一 |
| current_token 定位 | 隐含当作授权凭据 | 明确仅防复活，不证明身份 |
| 授权失败副作用 | 未明确 | 不执行 operation、不写消费记录 |

---

## 8. 证据来源与证明边界

- Token 身份：TOKEN-IDENTITY-CANONICAL-MODEL.md
- 释放异常：RESOURCE-FAILURE-AND-RECOVERY.md
- Fencing 串行化：RESOURCE-TAKEOVER-RELEASE-SERIALIZATION（05.42，保留）

**证明等级**：单进程可信上下文三重校验 → VERIFIED_STATIC；跨进程可信身份传递 → NOT_PROVEN，维持 DENY。
