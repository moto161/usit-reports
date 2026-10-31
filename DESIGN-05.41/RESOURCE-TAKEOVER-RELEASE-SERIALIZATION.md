# RESOURCE-TAKEOVER-RELEASE-SERIALIZATION.md — 资源接管与释放串行化（05.41 权威）

证据等级：VERIFIED_STATIC（含 NOT_PROVEN 项）
目标：修复 B40-R04——接管与释放共享同一 `fence_lock`，给出完整并发交错分析。

---

## 1. 唯一串行化边界

`fence_lock`（资源端单写者锁）是接管与释放的**唯一**串行化边界。二者都必须获取该锁：

- `takeover(new_owner)`：获取 `fence_lock`，`current_token += 1`，设置 owner。
- `atomic_release(token, op)`：获取 `fence_lock`，校验 token、执行释放、消费 token、置 released。

---

## 2. 全部并发交错（完整枚举）

设两个操作：`R = atomic_release(old_token)`（旧 Owner），`T = takeover(new_owner)`（接管者）。

| 交错 | 执行顺序 | 结果 |
|---|---|---|
| 1 | R 获锁 → 完成释放 → 释放锁 → T 获锁 | R 合法释放（token 匹配），T 接管已释放资源 |
| 2 | T 获锁 → 递增 token → 释放锁 → R 获锁 | R 见 `old_token != current_token` → FENCED_OUT |
| 3 | R 获锁 → 校验通过 → （临界区内）释放 → 释放锁；T 等待 | 同交错 1，R 合法 |
| 4 | T 获锁 → 递增 → 释放锁；R 等待 → 获锁 → FENCED_OUT | 同交错 2 |

**结论**：无论何种交错，`fence_lock` 保证 R 与 T 串行执行。R 要么在 T 之前完成（合法释放），要么在 T 之后见 token 不匹配（FENCED_OUT）。**不存在**「R 校验通过但被 T 打断后仍释放」的窗口。

---

## 3. 线性化点

- 线性化点 = 获取 `fence_lock` 并完成操作的时刻。
- 所有操作的线性化点通过 `fence_lock` 的互斥关系构成全序。

---

## 4. 旧 Owner 各场景分析

| 场景 | 过程 | 结果 |
|---|---|---|
| 已进入调用、暂停后恢复 | 旧 Owner 在 `atomic_release` 内暂停（持有或等待 fence_lock），恢复后继续。若 T 已递增 token，则比较失败 | FENCED_OUT |
| 网络延迟 | 旧 token 请求迟到，到达时 `current_token` 已递增 | FENCED_OUT |
| 进程复活 | 复活后携带旧 token，`atomic_release(old_token)` | FENCED_OUT |
| 重复接管 | 两次 takeover，第二次 token 再递增，第一次接管者失效 | 后接管者胜 |

---

## 5. 与 05.40 的差异

| 项 | 05.40 | 05.41 |
|---|---|---|
| 释放同步 | internal_lock | fence_lock |
| 接管同步 | cas_token（独立 CAS） | fence_lock（与释放同一锁） |
| 串行化关系 | 无（锁与 CAS 分离） | 单一 fence_lock |
| 旧 Owner 越权释放 | 存在窗口（cas_token 不等锁） | 消除（同锁串行化） |

---

## 6. NOT_PROVEN 边界

- 跨进程 `fence_lock`：NOT_PROVEN → DENY。
- `current_token`/`released`/`owner` 持久化：NOT_PROVEN → DENY。

---

## 证据来源

- Fencing 协议：RESOURCE-FENCING-PROTOCOL.md
- 崩溃幂等：RESOURCE-CRASH-AND-IDEMPOTENCY.md
- 05.40 缺陷：RESOURCE-FENCING-ATOMIC-RELEASE.md（05.40 版）锁与 CAS 分离
- 证明等级：VERIFIED_STATIC（有 fence_lock）+ NOT_PROVEN（无原子原语）
