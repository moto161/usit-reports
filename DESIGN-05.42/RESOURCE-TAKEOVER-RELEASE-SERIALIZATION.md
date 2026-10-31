# RESOURCE-TAKEOVER-RELEASE-SERIALIZATION.md — 资源接管与释放串行化（05.42 权威）

证据等级：VERIFIED_STATIC（含 NOT_PROVEN 项）
目标：接管与释放共用 fence_lock，给出完整并发交错分析。

---

## 1. 唯一串行化边界

`fence_lock` 是接管与释放的**唯一**串行化边界：

- `takeover(new_owner)`：获取 fence_lock，`current_token += 1`，设置 owner。
- `atomic_release(token, req_id, op)`：获取 fence_lock，幂等检查 → fencing 检查 → 释放 → 记录。

---

## 2. 全部并发交错（完整枚举）

设 `R = atomic_release(old_token)`（旧 Owner），`T = takeover(new_owner)`（接管者）。

| 交错 | 执行顺序 | 结果 |
|---|---|---|
| 1 | R 获锁 → 释放完成 → 释放锁 → T 获锁 | R 合法释放，T 接管已释放资源 |
| 2 | T 获锁 → token 递增 → 释放锁 → R 获锁 | R 见 token != current → FENCED_OUT |
| 3 | R 获锁 → 校验 → 释放 → 释放锁；T 等待 | 同交错 1 |
| 4 | T 获锁 → 递增 → 释放锁；R 等待 → FENCED_OUT | 同交错 2 |

**结论**：无论何种交错，fence_lock 保证 R 与 T 串行。R 要么在 T 前完成（合法），要么在 T 后见 token 不匹配（FENCED_OUT）。无「校验通过但被 T 打断后仍释放」窗口。

---

## 3. 线性化点

线性化点 = 在 fence_lock 内完成「幂等检查 + fencing 检查 + 释放 + 记录」的时刻。获取锁本身不是操作完成。

---

## 4. 旧 Owner 各场景

| 场景 | 结果 |
|---|---|
| 已进入调用、暂停后恢复 | 恢复后继续，比较 token 时已递增 → FENCED_OUT |
| 网络延迟 | 迟到旧 token → FENCED_OUT |
| 进程复活 | 携带旧 token → FENCED_OUT |
| 重复接管 | 两次 takeover，token 再递增，后接管者胜 |

---

## 5. 与 05.41 的差异

| 项 | 05.41 | 05.42 |
|---|---|---|
| 串行化 | fence_lock（已统一） | fence_lock（保持） |
| 幂等 | 靠 released 标志（缺陷） | consumed_records 独立记录 |
| 线性化点 | 获取锁 | 锁内完成状态变更 |

---

## 证据来源

- Fencing 协议：RESOURCE-FENCING-PROTOCOL.md
- 幂等与恢复：RESOURCE-IDEMPOTENCY-AND-RECOVERY.md
- 证明等级：VERIFIED_STATIC（有 fence_lock）+ NOT_PROVEN（无原子原语）
