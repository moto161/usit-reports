# ATOMIC-TRANSACTION-BOUNDARY.md — 原子事务边界（05.41 权威）

证据等级：VERIFIED_STATIC（单进程）+ NOT_PROVEN（跨进程/跨重启）
目标：证明 COUNT（状态字）与 Token 记录（TRT）在**同一原子事务**内提交，消除两存储分离导致的崩溃窗口与并发重复消费。

---

## 1. 问题定义

05.41 的数据模型有两个存储：

- **状态字**（128 位）：COUNT、TOKEN_ACTIVE 位图、GATE/DENIED/FROZEN/FREEZING/VERSION。
- **TRT**（Token 记录表）：每槽位 `{generation, owner_id, resource_id}`。

`admit`/`release` 需要同时修改两者。若两者不在同一原子边界，会出现：

- **崩溃窗口**：COUNT+1 成功，TRT 未写入 → 账本不一致。
- **并发重复消费**：两个线程都通过「TRT 校验」，各自 COUNT−1 → COUNT 少计。

**必须**：COUNT 与 TRT 的修改在同一原子事务内提交。

---

## 2. 唯一权威方案：事务锁 + 单一提交点

采用**单写者事务锁 `txn_lock`** 作为唯一串行化边界，所有修改 COUNT 或 TRT 的操作在锁内完成，提交点 = 释放锁前的最后一次可见性操作。

### 事务结构

```python
class Txn:
    def __init__(self):
        self.state = None      # 128 位状态字快照（进入事务时读）
        self.trt = None        # TRT 快照（进入事务时读）
        self.changed = set()   # 修改的字段集合

def begin_txn():
    txn_lock.acquire()          # ① 获取唯一串行化锁
    t = Txn()
    t.state = load_word()       # ② 读状态字快照
    t.trt = read_trt()          # ③ 读 TRT 快照
    return t

def commit_txn(t):
    # ④ 写状态字（128 位 CAS，比较 t.state 快照，防并发）
    if not cas128(t.state.word, t.state.new_word):
        txn_lock.release()
        return 'CAS_FAILED'      # 冲突，调用方重读重算
    write_trt(t.trt)             # ⑤ 写 TRT（在锁内，无并发写者）
    txn_lock.release()           # ⑥ 释放锁 = 提交点（线性化点）
    return 'OK'
```

**关键**：
- 状态字写（④）与 TRT 写（⑤）**都在 txn_lock 内**，无其他线程能在二者之间插入。
- 提交点（线性化点）= 释放锁（⑥）。锁释放前，④⑤ 均已完成；锁释放后，其他事务才能读取新状态。

---

## 3. 崩溃窗口分析

| 崩溃点 | 结果 | 恢复判定 |
|---|---|---|
| ④ 之前（状态字未写） | 无变化 | 无事务，无需恢复 |
| ④ 之后、⑤ 之前（状态字已写，TRT 未写） | **不一致** | 见 §4 恢复 |
| ⑤ 之后、⑥ 之前 | 状态字 + TRT 都已写，但未释放锁 | 进程崩溃，锁释放，事务已完成 |

**关键崩溃窗口**：④ 与 ⑤ 之间。状态字（COUNT/位图）已改，TRT 未改 → 账本不一致。

### 解决：提交顺序与恢复

将提交顺序固定为「**先写 TRT，再写状态字**」，且状态字写作为**提交的原子触发点**：

```python
def commit_txn(t):
    write_trt(t.trt)             # ① 先写 TRT（锁内）
    if not cas128(t.state.word, t.state.new_word):
        # CAS 失败 → 回滚 TRT 到 t.trt 快照（锁内，无并发）
        rollback_trt(t.trt)
        txn_lock.release()
        return 'CAS_FAILED'
    txn_lock.release()           # ② 释放锁 = 提交点
    return 'OK'
```

**新崩溃窗口分析**：

| 崩溃点 | 结果 | 恢复 |
|---|---|---|
| ① 之前 | 无变化 | 无事务 |
| ① 之后、② CAS 之前 | TRT 已写，状态字未写 | **TRT 领先状态字**，但 COUNT 未 +1 → 恢复时检测「TRT 有记录但位图未置位」→ 回滚 TRT（见 TOKEN-CRASH-RECOVERY.md） |
| ② CAS 成功、锁释放前 | TRT + 状态字一致 | 事务完成 |

**不变式保证**：状态字 CAS 成功是**唯一提交点**。CAS 成功 ⟺ COUNT+位图+TRT 全部一致。CAS 失败 ⟺ 回滚 TRT，回到一致态。因此**对外可见的永远是「COUNT 与 TRT 一致」**。

---

## 4. 事务级不变式

- **提交点 = 状态字 CAS 成功**（不是锁释放，而是 CAS 成功那一刻，锁释放只是让其他线程可见）。
- 任何时刻，**对外可见**的状态必须满足 `COUNT == popcount(TOKEN_ACTIVE)` 且 `TOKEN_ACTIVE 每 bit 对应 TRT 中一条 generation 匹配的记录`。
- 违反不变式的中间态**不得对外可见**（在锁内被回滚或完成）。

---

## 5. 并发重复消费防护

```python
def release(txn, token):
    # 在 txn_lock 内
    if not (txn.state.token_active & (1 << token.slot)):
        return 'INVALID_TOKEN'
    if txn.trt[token.slot].generation != token.gen:
        return 'INVALID_TOKEN'
    txn.trt[token.slot].generation += 1        # 消费：代际+1，使旧 token 失效
    txn.state.token_active &= ~(1 << token.slot)
    txn.state.count -= 1
    txn.state.version += 1
    r = commit_txn(txn)
    return r
```

两个线程并发 release(token)：
- 线程 A 先获取 txn_lock，完成「清位 + COUNT−1 + TRT.generation+1」，CAS 成功，释放锁。
- 线程 B 获取 txn_lock，读到的 `token_active` bit 已=0 → INVALID_TOKEN。
- **同一 Token 只消费一次**。∎

---

## 6. 事务锁的适用边界

| 边界 | txn_lock 机制 | 状态 |
|---|---|---|
| 单进程 | 进程内互斥锁（mutex） | VERIFIED_STATIC |
| 跨进程 | 需跨进程锁（共享内存 futex / 文件锁） | NOT_PROVEN → DENY |
| 跨重启 | 需持久化锁 + 恢复协议 | NOT_PROVEN → DENY |

**禁止**：用进程内 mutex 声称跨进程原子性；退回无原子保证的独立字典（05.39 方案 B 已被本模型取代）。

---

## 7. 与 05.40 的差异

| 项 | 05.40 | 05.41 |
|---|---|---|
| Token 存储 | 内嵌状态字（单 128 位 CAS 即原子） | 独立 TRT + 状态字（需 txn_lock） |
| 原子边界 | 单 CAS | txn_lock + CAS 提交点 |
| 崩溃窗口 | 无（内嵌） | TRT 领先状态字的窗口，靠 CAS 回滚 |
| 每槽位代际 | 不支持（全局） | 支持（TRT[slot].generation） |

---

## 证据来源

- 状态模型：UNIFIED-STATE-MODEL.md
- 每槽位代际：TOKEN-IDENTITY-AND-PER-SLOT-GENERATION.md
- 崩溃恢复：TOKEN-CRASH-RECOVERY.md
- 证明等级：VERIFIED_STATIC（单进程）+ NOT_PROVEN（跨进程/跨重启）
