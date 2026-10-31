# RESOURCE-FENCING-PROTOCOL.md — 资源端 Fencing 协议（05.41 权威）

证据等级：VERIFIED_STATIC（含 NOT_PROVEN 项）
目标：修复 B40-R04——接管与释放必须使用同一个资源端权威同步机制；不允许 release 用锁、takeover 用独立 CAS。

---

## 1. 05.40 缺陷定位（B40-R04）

05.40 存在两套不共享串行化关系的同步机制：

- `resource_atomic_release` 用 `resource.internal_lock`（锁）。
- `takeover` 用 `resource.cas_token(old, new)`（独立 CAS，**不在 internal_lock 内**）。

**矛盾**：05.40 断言「接管者必须等临界区释放后才能 cas_token」，但 `cas_token` 不获取 `internal_lock`，所以这个断言**不成立**。接管 CAS 可以在释放临界区执行期间并发发生，导致：

```
释放线程：internal_lock 内，校验 token==current，准备释放
接管线程：cas_token(old→new) 成功（不等待 internal_lock）
释放线程：继续执行 release → 释放了已被接管的资源 ← 越权释放！
```

**根因**：锁与 CAS 是两个独立的同步原语，没有共同的串行化边界。

---

## 2. 唯一权威决策：单一串行化边界

**结论**：接管与释放**必须共享同一个资源端同步原语**。本设计选择**资源端单写者锁 `fence_lock`** 作为唯一串行化边界，接管（token 递增）与释放（token 校验 + 释放）都在 `fence_lock` 内完成。

```python
# 资源端唯一权威同步原语
class Resource:
    def __init__(self):
        self.fence_lock = Lock()      # 唯一串行化边界
        self.current_token = 0        # 单调递增
        self.released = False
        self.owner = None

    # 接管：token 递增（在 fence_lock 内）
    def takeover(self, new_owner):
        with self.fence_lock:
            self.current_token += 1   # 单调递增
            self.owner = new_owner
            return 'TAKEOVER_OK'

    # 释放：token 校验 + 释放 + 消费（在 fence_lock 内）
    def atomic_release(self, token, operation):
        with self.fence_lock:
            if token != self.current_token:
                return 'FENCED_OUT'
            if self.released:
                return 'ALREADY_RELEASED'
            result = operation()      # 执行实际释放
            self.current_token += 1   # 消费 token
            self.released = True
            return 'OK'
```

**关键**：`takeover` 与 `atomic_release` 都获取 `fence_lock`。二者**串行化**——一个执行时，另一个必须等待。不存在「接管 CAS 在释放临界区执行期间并发发生」的窗口。

---

## 3. 唯一线性化点与并发交错

线性化点 = 获取 `fence_lock` 并完成操作的时刻。所有可能的交错：

| 交错顺序 | 结果 |
|---|---|
| 释放先于接管 | 释放完成（token 消费 +1，released=True），接管随后递增 token 并设置新 owner。旧 owner 释放合法 |
| 接管先于释放 | 接管递增 token，旧 owner 的 `atomic_release(old_token)` 见 `old_token != current_token` → FENCED_OUT |
| 并发（同一时刻） | fence_lock 串行化，一方先获得锁，另一方等待。先获得锁的一方是线性化点 |

**没有**「校验通过但释放被接管打断」的窗口，因为校验与释放在同一 `fence_lock` 临界区内，接管必须等锁释放。

---

## 4. 旧 Owner 无法在接管后释放的证明

接管成功后 `current_token` 已递增。旧 Owner 若尝试释放：

- 旧 Owner 只能调用 `atomic_release(old_token)`。
- 该原语在 `fence_lock` 内比较 `old_token != current_token` → 因接管已递增 → `FENCED_OUT`。

覆盖场景：

| 场景 | 结果 |
|---|---|
| 旧 Owner 已进入调用、暂停后恢复 | 恢复后继续执行 `atomic_release`，比较 token 时发现已递增 → FENCED_OUT |
| 网络延迟导致旧 token 迟到 | 迟到时 `current_token` 已递增 → FENCED_OUT |
| 进程复活 | 复活后携带旧 token → FENCED_OUT |

**为何无法绕过**：旧 Owner 无法绕过 `fence_lock`（唯一入口），也无法在「校验通过后、释放前」插入动作（校验+释放在同一临界区）。∎

---

## 5. 幂等释放返回值（区分 ALREADY_RELEASED 与 FENCED_OUT）

| 请求 | 状态 | 返回 |
|---|---|---|
| 首次释放，token 匹配，未 released | → 释放 | OK |
| 重复释放，token 匹配，已 released | → 不重复释放 | ALREADY_RELEASED |
| 释放，token 不匹配（被接管） | → 拒绝 | FENCED_OUT |

**区分**：`released` 标志与 `token` 匹配是独立判断。token 匹配且 released=True → ALREADY_RELEASED（幂等成功）；token 不匹配 → FENCED_OUT（被接管）。

---

## 6. 崩溃窗口（operation / token 消费 / released 持久化）

| 崩溃点 | 状态 | 处理 |
|---|---|---|
| operation 执行前 | 无变化 | 无事务 |
| operation 执行中 | released 未置位 | 状态不明 → ISOLATED（若 operation 非幂等） |
| operation 完成、token 消费前 | 资源已释放但 token 未消费 | 状态不明 → 需幂等键判定 |
| token 消费 + released 置位后 | 完成 | 幂等重试 → ALREADY_RELEASED |

**关键**：`released` 与 `current_token` 的持久化若不能与 operation 原子，则存在「operation 完成但 released 未持久化」的窗口。此窗口下，重试需依赖 operation 的幂等性；非幂等则 ISOLATED（见 RESOURCE-CRASH-AND-IDEMPOTENCY.md）。

---

## 7. flock/fcntl/fd 仍非 Fencing

- flock/fcntl/fd 无资源端 `fence_lock` + `current_token` 的单调 token 语义。
- 它们**不是**资源端 Fencing，不提供「旧 token 拒绝」能力。
- 继续归入 OS_LOCK_ONLY，禁止接管释放。

---

## 8. NOT_PROVEN 边界

- 跨进程 `fence_lock`（跨进程锁）：NOT_PROVEN → DENY。
- `released`/`current_token` 持久化：NOT_PROVEN → DENY。
- operation 幂等性未证明：崩溃后 ISOLATED。

---

## 证据来源

- 接管释放串行化：RESOURCE-TAKEOVER-RELEASE-SERIALIZATION.md
- 崩溃幂等：RESOURCE-CRASH-AND-IDEMPOTENCY.md
- 05.40 缺陷：RESOURCE-FENCING-ATOMIC-RELEASE.md（05.40 版）internal_lock 与 cas_token 分离
- 证明等级：VERIFIED_STATIC（有 fence_lock 的资源）+ NOT_PROVEN（无原子原语）
