# USIT-STAGE-B-DESIGN-05.15｜四项核心协议重构与独立验收

> 阶段 B · DESIGN-05.15 · READ-ONLY DESIGN REVISION ONLY
> 作者：USIT 主开发 · 生成：2026-10-31（GMT+8）
> 依据：ChatGPT 对 DESIGN-05.14 的独立复审结论（四项核心缺陷）
> 工作区：`C:\Users\Administrator\WorkBuddy\2026-10-15-22-09-47\usit_p0`
> 交付目录：`C:\Users\Administrator\WorkBuddy\2026-10-15-22-09-47\usit_p0_05.15_reports`（**项目外**）
> 状态：**STOP — 等待独立复审**（未授权 IMPLEMENT-01）

---

## 0. 缺陷追踪表（05.14 原文位置 → 05.15 修改位置）

| # | 05.14 原文位置 | 缺陷描述 | 最小反例 | 05.15 修改位置 | 验证方式 | 剩余风险 |
|---|---------------|---------|---------|---------------|---------|---------|
| P0-02 | §1.4（before 无死锁证明）| fork 回调安全性不足：未建立全局锁顺序，未给出"持锁时 fork"的禁止规则，未证明"持 A 锁触发 fork 而死锁" | 线程 A 持 Registry 锁，线程 B 持 ForkGate 锁并触发 fork → before 要拿 Registry 锁 → 循环等待 | §1 | 8 条锁顺序时间线 + 5 条 fork 异常场景 | 多线程 fork 的 OS 级行为 NOT_VERIFIED |
| P0-04 | §3.4/§3.5（`link(current, current.lock.<txn_id>)`）| **假 CAS**：每事务独立锁文件名，T1/T2 可同时 link 成功再分别 rename → 丢失更新 | T1 link `current.lock.A` 成功，T2 link `current.lock.B` 成功，双方都 rename current | §2 | 单一固定竞争对象 + 12 条并发时间线 | Windows 原子原语 NOT_VERIFIED |
| P0-05 | §4.3（sealed/append-only/O_EXCL 当不可变）| sealed 约定 ≠ 不可变：seal 后旧可写 FD 仍可写，rename 不撤销旧 FD | A seal 前持可写 FD，B seal，A seal 后继续写 | §3 | 源对象状态机 + 10 条时间线 | 文件锁协作 NOT_TESTED |
| P0-06 | §5.3（源文件 rename 到 tombstone 路径 + tombstone 内容含元数据）| 对象模型矛盾：把"源文件改名"与"tombstone 内容含事务元数据"混为同一对象 | 源是二进制，无法既当源文件又当元数据记录 | §4 | 分离源对象与 Tombstone 元数据 + 12 条时间线 | 崩溃恢复语义 NOT_TESTED |

**本节是全文总纲，后续每章回到此表逐项给出修订证据与反例推演。**

---

## 1. P0-04（优先）｜废弃假 CAS，重新设计真正原子提交

> 工单要求把 P0-04 的假 CAS 作为首要修复项，因此 §1 先处理它（05.14 编号为 §3，05.15 重排为 §1）。

### 1.1 缺陷定位：05.14 §3.4/§3.5 的 link 方案是假 CAS

05.14 §3.5 步骤 7 用：

```
link(current, current.lock.<txn_id>)   # 原子占有：成功=独占权
```

**这是假的 CAS，必须废弃。** 原因：

`link(old, new)` 的原子性保证是"`new` 已存在时 EEXIST 失败"。但 05.14 给**每个事务起了不同名字的锁文件**（`current.lock.A`、`current.lock.B`、…），所以：

- T1 `link(current, current.lock.A)` → `current.lock.A` 不存在 → **成功**；
- T2 `link(current, current.lock.B)` → `current.lock.B` 不存在 → **成功**；
- 两者都以为"独占"了，随后各自 `rename(current.<txn> → current)`，**互相覆盖，丢失更新**。

**根本错误**：竞争对象必须是**单一、固定**的，而不是"每事务一个独占名字"。每事务独立名字等于没有互斥。

### 1.2 最小反例（工单强制）

| 时刻 | T1 | T2 | current 状态 |
|------|----|----|-------------|
| t0 | 读 current → revision=1 | 读 current → revision=1 | rev=1 |
| t1 | `link(current, current.lock.A)` 成功 | `link(current, current.lock.B)` 成功 | rev=1（两个锁标记共存）|
| t2 | `rename(current.t1 → current)` → rev=2 | — | rev=2（T1 胜）|
| t3 | — | `rename(current.t2 → current)` → rev=2 | **rev=2，但内容被 T2 覆盖** |

**结论**：T1 的提交被 T2 覆盖，revision 未单调递增（T1 rev=2 被 T2 rev=2 覆盖，两个事务都以为成功，实际丢了一个更新）。**这是丢失更新，05.14 方案 FAIL。**

### 1.3 正确方案：单一固定竞争对象 + 真正 CAS

**核心原则**：所有事务必须竞争**同一个**原子互斥对象，而不是各自创建独占标记。

05.15 最终选择：**单一固定锁文件 + OS 级文件锁（flock / LockFileEx）作为提交互斥**，配合**指针内容内的 revision 单调校验**作为第二道防线。

**为什么选文件锁而非 SQLite**：
- 本工单 READ-ONLY，不能引入数据库依赖或实现；文件锁是 OS 内核原语，可直接推演其语义。
- SQLite 事务 CAS 作为**替代契约**在 §1.8 给出（NOT_IMPLEMENTED），供实现阶段选型。

**关键区分（工单 §四.4 要求）**：
1. Candidate 创建：写不可变版本文件（与 05.14 相同，保留）。
2. PREPARED 持久化：先于提交竞争持久化。
3. 提交竞争：**在固定锁文件上 flock(LOCK_EX)** —— 这是唯一的互斥点。
4. revision 检查：**持锁后**再读 current，断言 `new_rev == old_rev + 1`。
5. 指针发布：持锁状态下 rename。
6. COMMIT 确认：写回执后释放锁。

### 1.4 完整提交协议（POSIX）

固定锁文件：`doc/.commit-lock`（**单一固定名，所有事务共用**）。

```
阶段 A：Candidate 创建（无锁，可并发）
A1. 写 versions/<txn_id>.<new_rev>（O_EXCL，写全文+自校验字段，fsync）
A2. 校验版本文件 SHA256（重读比对）

阶段 B：PREPARED 持久化（先于提交竞争）
B1. 写 prepared/<txn_id>（内容：txn_id、版本文件身份、new_rev、old_rev_expected）
B2. fsync(prepared/<txn_id>)

阶段 C：提交竞争（唯一互斥点）
C1. open(.commit-lock, O_RDWR|O_CREAT, 0600) → fd_lock
C2. flock(fd_lock, LOCK_EX)   # ← 阻塞直到获得排他锁，这是真正互斥

阶段 D：revision 检查（持锁后）
D1. 读 current → <cur_txn>\n<cur_rev>
D2. assert new_rev == cur_rev + 1
    - 不满足 → 放弃：unlink(prepared/<txn_id>)，flock LOCK_UN，重读重试

阶段 E：指针发布（持锁状态）
E1. 写 current.<txn_id>（内容 new_txn\nnew_rev，fsync）
E2. rename(current.<txn_id> → current)   # 持锁下 rename，无竞争
E3. fsync(doc_dir)

阶段 F：COMMIT 确认
F1. 写 COMMIT 回执（内容：txn_id、new_rev、指针新内容，fsync）
F2. unlink(prepared/<txn_id>)
F3. flock(fd_lock, LOCK_UN)   # 释放提交锁
```

### 1.5 为什么这是真 CAS（证明 revision 检查与发布之间无窗口）

**关键**：`flock(fd_lock, LOCK_EX)` 把"读 current → 断言 rev → rename"这一整段**串行化**了。

- 任何事务要发布指针，必须先获得 `fd_lock` 的排他锁。
- 持锁期间，其他事务在 `flock(LOCK_EX)` 上**阻塞**，无法读 current、无法 rename。
- 因此"读 current 得到 rev=N"到"rename 发布 rev=N+1"之间，**没有任何其他事务能插入**。

**这正是真 CAS 的本质**：用 OS 内核的 flock 排他性，把 check-revision 和 publish 变成原子的临界区。而 05.14 的 link 方案错在"每事务独立锁名"，无法形成临界区。

**revision 单调性证明**：因为在临界区内 `assert new_rev == cur_rev + 1`，且发布是原子 rename，所以 current 的 rev 严格递增，不存在回退。

### 1.6 双事务竞争时间线（最小反例的解法）

| 时刻 | T1 | T2 | current |
|------|----|----|---------|
| t0 | 写版本文件，PREPARED | 写版本文件，PREPARED | rev=1 |
| t1 | flock 成功，进临界区 | flock 阻塞 | rev=1 |
| t2 | 读 cur_rev=1，断言 2==1+1 ✓ | （阻塞中）| rev=1 |
| t3 | rename → rev=2，fsync，COMMIT | （阻塞中）| rev=2 |
| t4 | LOCK_UN | flock 成功，进临界区 | rev=2 |
| t5 | — | 读 cur_rev=2，断言 2==2+1 ✗ → **放弃**，unlink prepared，重读重试 | rev=2 |

**结论**：T1 胜（rev=2），T2 因 revision 检查失败**放弃并重试**（下一轮读 rev=2 后提交 rev=3）。**T2 不可能覆盖 T1，revision 严格单调，无丢失更新。**

### 1.7 12 条并发/崩溃时间线

| # | 场景 | 推演 | 结果 |
|---|------|------|------|
| X1 | 单事务正常提交 | A→B→C→D→E→F 全成功 | rev 单调 +1 |
| X2 | 双事务并发，一胜一弃 | T1 进临界区胜，T2 revision 检查失败放弃重试 | 不覆盖，不丢更新 |
| X3 | 双事务同时读 revision=1 | 都在 flock 上串行化，先获锁者胜 | 只有一个 rev=2 |
| X4 | Candidate 写一半崩溃 | versions/ 半成品，SHA256 校验失败删除 | 无副作用 |
| X5 | PREPARED 写一半崩溃 | prepared/ 半成品，恢复校验失败忽略 | 无副作用 |
| X6 | 持锁后、rename 前崩溃 | 进程崩溃 → OS 自动释放 flock → current 未变 → 恢复：prepared 存在但 current rev 未变 → 判定未提交，重试 | 无丢更新 |
| X7 | rename 后、COMMIT 前崩溃 | current 已 rev=N+1，COMMIT 未写 → 恢复：current 为准，版本文件存在 → 确认提交 | current 是唯一真相 |
| X8 | 锁文件被 unlink+recreate | 已 open 的 fd_lock 指向旧 inode，新进程 open 新 inode → 分裂（同 P0-03）| 靠固定锁 + 身份校验检测（见 §2）|
| X9 | 重复 transaction_id | versions/<txn_id> O_EXCL 创建失败 → 拒绝 | 不重复提交 |
| X10 | revision 回退伪造 | 断言 new_rev==cur_rev+1 失败 → 放弃 | 拒绝 |
| X11 | 目录 fsync 失败 | rename 后 fsync(doc_dir) 失败 → 标记 NOT_DURABLE，不回滚（rename 已原子）| 明确非持久 |
| X12 | 失败事务残留 prepared | 放弃时 unlink(prepared/<txn_id>)；若崩溃未 unlink → 恢复按 current rev 判定未提交，清理 prepared | 无脏数据 |

### 1.8 SQLite 替代契约（NOT_IMPLEMENTED，供选型）

若实现阶段选数据库：

```
BEGIN IMMEDIATE;                              -- 立即获取写锁，等价 flock LOCK_EX
SELECT revision FROM current WHERE id=1;      -- 读当前 rev
UPDATE current SET txn_id=:new, revision=:new_rev
       WHERE id=1 AND revision=:old_rev;      -- 条件更新 = 真 CAS
-- affected_rows == 1 → 提交成功
-- affected_rows == 0 → 他人已提交，ROLLBACK 重试
COMMIT;
```

- `BEGIN IMMEDIATE` 等价于 `flock(LOCK_EX)`（获取写锁，阻塞其他写者）。
- `UPDATE ... WHERE revision=:old_rev` 的 **affected_rows** 是真正的 CAS 判定：只有旧 revision 仍匹配时才更新。
- 崩溃恢复：SQLite 的 WAL/journal 保证事务原子性，未 COMMIT 的事务回滚。
- 持久化边界：`COMMIT` 后 fsync 数据库文件（SQLite 默认 synchronous=FULL）。

**选择理由记录**：本工单 READ-ONLY 采用文件锁方案（§1.4）作为可推演的完整协议；SQLite 方案作为实现阶段的等价替代，两者语义等价（flock 临界区 ≡ BEGIN IMMEDIATE 写锁）。

### 1.9 Windows 提交协议

Windows 无 flock，用 `LockFileEx` + 固定锁文件：

```
C1. CreateFileW(doc\.commit-lock, GENERIC_READ|GENERIC_WRITE, 0, OPEN_ALWAYS, ...)
C2. LockFileEx(hLock, LOCKFILE_EXCLUSIVE_LOCK, 0, 1, 0, &ov)  # 独占锁字节范围
D1. 读 current → cur_rev
D2. assert new_rev == cur_rev + 1
E1. 写 current.<txn_id>（FlushFileBuffers）
E2. ReplaceFileW(current, current.<txn_id>, NULL, REPLACEFILE_WRITE_THROUGH, ...)
E3. FlushFileBuffers(doc_dir_handle)
F1. 写 COMMIT 回执
F2. UnlockFileEx(hLock)
F3. CloseHandle(hLock)
```

**注意**：Windows 的 `LockFileEx` 锁定的是**文件字节范围**，不是整个文件对象；05.15 约定所有事务锁定 `.commit-lock` 的**同一字节范围（offset 0, len 1）**，从而形成同一互斥点。**该语义、ReplaceFileW 的原子性 NOT_VERIFIED**（未实测）。

### 1.10 P0-04 验收对照与结论

- 废弃 link 假 CAS：✅ §1.1/§1.2
- 单一固定竞争对象：✅ §1.3（`.commit-lock` 固定名，flock 排他）
- 区分六阶段：✅ §1.4（Candidate/PREPARED/竞争/检查/发布/确认）
- 证明无插入窗口：✅ §1.5（flock 串行化 check+publish）
- 12 条时间线：✅ §1.7
- 5 条反例：✅ §1.2 + X3/X6/X8/X10/X12
- POSIX/Windows 分开：✅ §1.4/§1.9

**P0-04 结论：PASS（设计级，假 CAS 已废弃，flock 临界区给出真 CAS）**。Windows 原子原语 NOT_VERIFIED。

---

## 2. P0-02｜Fork 生命周期与死锁证明

### 2.1 缺陷定位：05.14 §1.4 的"before 无死锁证明"不充分

05.14 §1.4 只论证了"before 回调获取 Registry 锁时是正常阻塞，非死锁"，但**没有建立全局锁顺序**，也没有回答工单点名的关键反例：

> **线程 A 持 Registry 锁，线程 B 持 ForkGate 锁并触发 fork。**

这个场景下，before 回调（在线程 B 触发 fork 的上下文中）需要获取 Registry 锁，而 Registry 锁被线程 A 持有；线程 A 若同时等待 ForkGate 锁（被线程 B 持有），则形成循环等待 = 死锁。05.14 未处理此反例，FAIL。

### 2.2 全部同步对象清单（含内部锁）

| 对象 | 内部锁 | 创建位置 | 持有线程 | fork 前状态 |
|------|--------|---------|---------|-----------|
| ForkGate | `_gate_lock` | 模块初始化 | 任意调用 fork 的线程 | 可能 locked |
| LockHandleRegistry | `_lock` | 模块初始化 | 任意 | 可能 locked |
| FD Registry | `_fd_lock` | 模块初始化 | 任意 | 可能 locked |
| ProcessMutex | `_pmutex_lock` | 首次 acquire | 任意 | 可能 locked |
| Lease 管理 | `_lease_lock` | 首次 acquire | 任意 | 可能 locked |
| Condition/Event | 内建 | 使用处 | 任意 | 可能 set/waited |

### 2.3 全局锁获取顺序（消除循环等待）

**规则：所有锁的获取必须遵循固定全局顺序，禁止逆序获取。**

全局锁顺序（从小到大，编号即顺序）：

```
1. _gate_lock（ForkGate）
2. _lock（LockHandleRegistry）
3. _fd_lock（FD Registry）
4. _lease_lock（Lease 管理）
5. _pmutex_lock（ProcessMutex）
```

**铁律**：
1. 任何代码路径**禁止**在持有编号 N 的锁时再去获取编号 < N 的锁（即只能按升序获取，或一次性获取）。
2. **禁止在持有任何锁时调用 fork**（见 §2.4）。
3. 锁的释放按逆序（与获取顺序相反），或"一次性获取 + 一次性释放"。

**为什么消除循环等待**：循环等待的必要条件是"存在两条锁获取路径，其顺序相反"。强制全局升序后，不存在逆序获取路径，故不存在环，死锁不可构造。

### 2.4 最小死锁反例及其消除

**反例**：线程 A 持 `_lock`（编号 2），线程 B 持 `_gate_lock`（编号 1）并触发 fork：

- 按 05.14 旧设计，before 回调要拿 `_lock`（Registry 锁）→ 被 A 持有 → B 阻塞；
- A 继续执行，需要拿 `_gate_lock`（编号 1）→ 被 B 持有 → A 阻塞；
- 循环等待，死锁。

**消除方案（05.15）**：
1. **禁止持锁 fork**：任何线程在持有 `_gate_lock` 或 `_lock` 或 `_fd_lock` 时，**不得调用 `os.fork()`**（含 `fork_controlled`）。fork 入口先检查"当前线程是否持锁"，持锁则抛异常拒绝 fork。
2. **before 回调不获取任何锁**：before 回调改为**无锁快照**——它不 `acquire` Registry 锁，而是读取一个**已由调用方在进入 fork 前、无锁状态下准备好的不可变快照 tuple**。fork 入口（无锁）先做快照，再调 `os.fork()`。
3. 因此 before 回调在 fork 系统调用前**不进入任何临界区**，不可能与任何持锁线程形成等待关系。

**修订后时间线**：

| 时刻 | 线程 B（触发 fork）| 线程 A |
|------|------------------|--------|
| t0 | 无锁状态下做 FD 快照 tuple（不持锁）| 持 `_lock` 做 registry 操作 |
| t1 | 调 `os.fork()`（before 回调无锁，直接读快照）| 继续持 `_lock` |
| t2 | fork 完成，child 回调 close fd + 重建锁 | 释放 `_lock` |
| t3 | parent 回调无操作 | 完成 |

**无死锁**：before 不拿锁，A/B 之间不存在锁等待关系。

### 2.5 before / parent / child 回调契约（修订）

| 回调 | 调用线程 | 可获取锁 | 禁止操作 |
|------|---------|---------|---------|
| before | fork 触发线程（父）| **无**（用无锁快照）| acquire 任何锁；持锁状态下 fork |
| parent | fork 触发线程（父）| 无 | 任何影响父进程状态的操作 |
| child | 子进程唯一线程 | 无（直接赋值全新对象，不 acquire）| acquire 继承锁；对共享 OFD 做 LOCK_UN |

**child 回调只做两件事**：
1. `os.close(fd)` —— 关闭 before 快照 tuple 中记录的 FD（不 LOCK_UN）。
2. 直接赋值全新对象 —— `= threading.Lock()` / `= {}` / `= None`，不调用 `with`/`acquire`。

### 2.6 child 中必须重建/关闭/作废的对象

| 对象 | 处理 | 原因 |
|------|------|------|
| ForkGate._gate_lock | 重建全新 Lock | 继承的锁可能 locked（其他线程持有）|
| LockHandleRegistry._lock | 重建 | 同上 |
| LockHandleRegistry._handles | 清空为全新 dict | 继承 FD 表应作废 |
| FD Registry._fd_lock + 表 | 重建 + 清空 | 同上 |
| ProcessMutex._pmutex_lock + 表 | 重建 + 清空 | 同上 |
| Lease._lease_lock + 表 | 重建 + 清空 | 子进程不得复用父 Lease |
| 快照 tuple 中的 FD | `os.close(fd)` | 释放继承 FD（仅 close，不 LOCK_UN）|

### 2.7 外部 os.fork() 的处理（不依赖快照/来源推断）

**不靠空快照、不靠调用来源推断**。策略：

1. `os.register_at_fork` 的 `after_in_child` **无条件执行安全重置**（对任何 fork 生效，无论受控还是外部）。
2. 子进程后续操作**自我检测**：任何锁操作前，`assert _fork_reset_ok`（child 回调成功设置的标志）；若未设置（回调异常）→ DENY。
3. 子进程若尝试使用父进程锁对象，因 registry 已清空、锁对象已重建，`assert_lock_owned` 失败 → DENY。
4. 外部 fork 的子进程要正常操作，必须**重新 `acquire_lock`** 走完整流程。

**不支持的场景（明确文档化）**：外部代码 fork 后子进程**绕过本库 API**直接使用继承的裸 fd/锁对象——本库无法拦截，属"调用方不遵守 API"，**不支持**。

### 2.8 8 条锁顺序/并发时间线

| # | 场景 | 锁状态推演 | 结果 |
|---|------|-----------|------|
| T1 | 单线程 fork | 无锁，快照→fork→child 重置 | 无死锁 |
| T2 | A 持 _lock，B 无锁 fork | before 无锁快照，不与 A 竞争 | 无死锁 |
| T3 | A 持 _gate_lock，B 触发 fork | 违反"持锁禁 fork"→ fork 入口检查抛异常拒绝 | 拒绝，无死锁 |
| T4 | A 持 _lock，B 持 _gate_lock | 全局锁顺序禁止逆序获取 → 此状态不可达 | 不可构造 |
| T5 | 双线程各自持不同锁并 fork | 均违反"持锁禁 fork"→ 拒绝 | 无死锁 |
| T6 | child 回调异常 | `_fork_reset_ok` 未设 → 后续锁操作 DENY | fail-closed |
| T7 | 连续 fork | 每次重新快照 + 重置，幂等 | 安全 |
| T8 | 父子并发退出 | OS 释放各自 fd 引用，互不影响 OFD 锁 | 无父锁误释放 |

### 2.9 5 条 fork 崩溃/异常场景

| # | 场景 | 推演 | 结果 |
|---|------|------|------|
| K1 | fork 失败（资源不足）| os.fork 抛异常，before 无锁快照无副作用 | 安全，无残留 |
| K2 | child 回调 close 抛 OSError | 捕获继续；其他异常 → `_fork_reset_ok=False` → DENY | fail-closed |
| K3 | fork 后 exec | child 已 close 锁 fd，exec 后新程序无继承锁 | 安全 |
| K4 | 子进程先退出 | OS 释放子进程 fd，父进程 OFD 锁不受影响 | 安全 |
| K5 | 父进程持锁时崩溃 | OS 释放 flock，锁自动释放，无残留死锁 | 安全 |

### 2.10 fail-closed 设计

**若无法证明回调安全，采用 fail-closed**：child 回调中任何一步失败（快照消费失败、close 失败、重建失败）→ 设置 `_fork_reset_ok=False` → 子进程的**所有锁操作拒绝（DENY）**，且不尝试"恢复"父进程锁。**不把"理论上不会发生"当作结论**。

### 2.11 P0-02 验收对照与结论

- 同步对象完整清单：✅ §2.2
- 三回调执行顺序/线程/锁：✅ §2.5
- 全局锁顺序 + 禁持锁 fork：✅ §2.3/§2.4
- 最小死锁反例及消除：✅ §2.4
- 外部 fork 不靠快照/来源推断：✅ §2.7
- child 重建/关闭/作废清单：✅ §2.6
- 8 条锁顺序 + 5 条异常：✅ §2.8/§2.9
- fail-closed：✅ §2.10

**P0-02 结论：PASS（设计级，全局锁顺序 + 禁持锁 fork + 无锁快照消除死锁）**。多线程 fork 的 OS 级行为 NOT_VERIFIED。

---

## 3. P0-05｜Sealed 源对象模型重构

### 3.1 缺陷定位：05.14 §4.3 的 sealed/append-only/O_EXCL ≠ 不可变

05.14 把"sealed / append-only / O_EXCL 约定"当作不可变，但：
- **seal 后旧可写 FD 仍可写**：`rename` 只改目录项，不撤销已 open 的写 FD。
- **O_EXCL 只保证创建时独占**，不保证创建后内容不变。
- **append-only 是写入者约定**，恶意/外部进程可无视。

**必须重构为可执行、可验证的对象生命周期协议。**

### 3.2 源对象状态机

状态：`WRITABLE → SEALING → SEALED → QUARANTINED → RETIRED`，异常态 `INDETERMINATE`。

| 状态 | 含义 | 进入条件 | 可执行操作 |
|------|------|---------|-----------|
| WRITABLE | 源对象可写 | 初始 | 写入者写入 |
| SEALING | 正在封存 | 发起 seal | 禁止新写入（协作）|
| SEALED | 已封存，内容不再变 | seal 完成（见 §3.3）| 只读、可 quarantine |
| QUARANTINED | 已隔离副本 | 副本发布完成 | 只读 |
| RETIRED | 已退役（改名/删除）| Source Retire 完成 | 归档 |
| INDETERMINATE | 无法证明状态 | 任何证据不足 | 人工介入 |

### 3.3 WRITABLE → SEALED 的唯一合法转换及持久化顺序

**唯一合法转换**：

```
1. 写 SEAL_INTENT 记录（fsync）        # 声明将封存，绑定 src 身份 + 期望哈希
2. 阻止新写入（协作）：
   a. flock(src_fd, LOCK_EX)          # 排他锁，阻塞遵守协议的写入者
   b. 或 chmod 去写权限（仅同 UID 协作有效）
3. 等待在途写入排空：
   a. 遍历已登记的可写 FD，确认全部已 close（协作登记）
   b. 无法确认在途 FD 时 → 无法证明 sealed → 降级
4. 最终内容校验：sha256(src) 记录为 sealed 哈希
5. 写 SEALED 记录（fsync）            # 封存完成，绑定 sealed 哈希
```

**关键**：SEALED 的持久化证据 = SEAL_INTENT + SEALED 记录 + sealed 哈希。**仅靠"sealed 约定"或 O_EXCL 不足以证明，必须有持久化的 SEALED 记录。**

### 3.4 旧可写 FD 在 seal 后继续写（强制反例）

**反例**：进程 A 在 seal 前 `open(src, O_WRONLY)` 拿到 fd-A；进程 B 执行 seal；进程 A 在 seal 后通过 fd-A 继续写。

**推演**：
- `rename(src → src.sealed)` **不撤销** fd-A 的写权限——A 仍能用 fd-A 写**同一 inode**（现在路径叫 src.sealed）。
- `flock` 是**协作锁**：只有 A 也遵守"写前 LOCK_EX"约定才会被阻塞；若 A 是恶意/外部进程，flock 无强制力。
- **结论**：seal 后旧 FD 写入**无法被可靠阻止**，只能靠"写入者协作"（都遵守 flock 协议）或"内容检测"（哈希比对发现变化）。

**必须证明**：要么写入被阻止（协作），要么系统能可靠检测并拒绝退役。05.15 采用后者：**seal 后、Source Retire 前，重新做内容校验**；若发现内容变化 → DENY + INDETERMINATE。

### 3.5 机制区分（各自能保证什么、不能保证什么）

| 机制 | 能保证 | 不能保证 | 信任边界 |
|------|--------|---------|---------|
| 应用协作锁（flock）| 遵守协议的写入者被阻塞 | 恶意/外部进程不受约束 | 本库参与者 |
| 文件系统权限（chmod）| 非授权用户（跨 UID）不能写 | 同 UID 进程可 chmod 回来 | 跨 UID |
| 文件系统不可变属性（chattr +i）| 所有进程（含 root）不能写，需特权 | 需特权，且非所有 FS 支持；同 UID 若 root 可撤销 | OS 强制（特权）|
| 只读快照/版本化对象 | 副本内容不可变（新 inode）| 源对象本身仍可变 | 副本 |

**核心结论**：
- **应用协作锁 + 权限**：只能防"遵守协议的进程"，不能防恶意同 UID 进程。
- **chattr +i 不可变属性**：是唯一能**强制**阻止旧 FD 写入的机制（Linux 上即使已 open 写 FD，`+i` 后写入返回 EPERM），但需特权、平台相关。
- **只读快照/版本化**：通过"复制后只操作副本"规避源可变问题（但副本复制期间源仍可变，需 §3.6 的一致性校验）。

### 3.6 Quarantine 只操作副本 + Source Retire 独立事务

**Quarantine 阶段（只读源，不改源）**：
1. 写 QUARANTINE_INTENT（绑定 src 身份 + sealed 哈希期望值，fsync）
2. 打开源 `open(src, O_RDONLY|O_NOFOLLOW)` → fd
3. `fstat(fd)` 记录源身份（dev+ino+ctime+size+mtime）
4. 流式复制到 qtmp，同时计算 sha256
5. `fstat(fd)` 再取身份，与步骤 3 比对（检测替换）
6. sha256(qtmp) 与 sealed 哈希比对（检测内容变化）
7. rename(qtmp → qfinal) + fsync
8. 写 COMPLETE 回执

**Source Retire 阶段（独立事务，重新验证源）**：
1. 写 SOURCE_RETIRE_INTENT（独立意图，fsync）
2. **重新验证源状态**：源必须处于 SEALED，且内容 == sealed 哈希
3. 重新 `fstat(src)` 确认身份未变（未被替换）
4. 若 sealed 哈希匹配且身份一致 → rename(src → src.retired)
5. 写 RETIRE 回执（fsync）

**quarantine 完成 ≠ Source Retire 授权**：两个独立事务，Retire 前**必须重新验证** sealed 状态与内容一致。

### 3.7 平台差异（未验证标 NOT_VERIFIED）

| 机制 | Linux | macOS | Windows |
|------|-------|-------|---------|
| flock 排他 | 支持 | 支持 | 无，用 LockFileEx（NOT_VERIFIED）|
| chattr +i 不可变 | 支持（需特权）| 无（用 chflags uchg，语义不同）| 无（用 FILE_ATTRIBUTE_READONLY，非强制）|
| 阻止已 open 写 FD | chattr +i 可（EPERM）| chflags 不保证 | 不支持（NOT_VERIFIED）|
| O_NOFOLLOW | 支持 | openat 上受限 | 用重解析点标志 |

**明确声明**：macOS `chflags uchg`、Windows 文件属性、Windows LockFileEx 的阻止语义，均 **NOT_VERIFIED**。

### 3.8 10 条并发/异常时间线

| # | 场景 | 推演 | 结果 |
|---|------|------|------|
| Y1 | 正常 seal → quarantine → retire | 全流程，sealed 哈希一致 | 成功 |
| Y2 | seal 前可写 FD，seal 后继续写 | 协作：flock 阻塞；恶意：无法阻止 → 检测哈希变化 | 检测到则 DENY |
| Y3 | seal 后、retire 前外部写入 | retire 前重新验证 sealed 哈希 → 不匹配 → DENY + INDETERMINATE | 拒绝 |
| Y4 | 硬链接写入 | 硬链接指向同一 inode，另一路径写 → 哈希变化检测 | 检测到则 DENY |
| Y5 | 映射写入（mmap）| 已映射页写入不经过 write 系统调用 → 哈希在 fsync 后才反映 | 需 fsync 后重验，NOT_VERIFIED |
| Y6 | 追加写入 | 同 Y2，flock 协作阻止，恶意检测 | 检测则 DENY |
| Y7 | 外部进程替换源 | 身份（ino）变化 → fstat 比对失败 | 拒绝 |
| Y8 | 同 UID 恶意进程 | 无法强制阻止 → 只能检测 + 拒绝 | DENY + INDETERMINATE |
| Y9 | chattr +i 后旧 FD 写 | Linux 返回 EPERM，写入被强制阻止 | 强制阻止（需特权）|
| Y10 | 无法证明 sealed/无旧句柄/内容一致 | 证据不足 → DENY Source Retire + INDETERMINATE | 拒绝 |

### 3.9 P0-05 验收对照与结论

- 源对象六状态：✅ §3.2
- WRITABLE→SEALED 唯一转换 + 持久化：✅ §3.3
- 旧可写 FD seal 后写入：✅ §3.4（协作阻止 / 检测拒绝）
- 硬链接/多路径/映射/追加/外部/同 UID：✅ §3.8
- 机制区分（协作锁/权限/不可变/快照）：✅ §3.5
- 平台差异：✅ §3.7
- quarantine 只操作副本 + retire 独立：✅ §3.6
- 无法证明 → DENY + INDETERMINATE：✅ §3.4/§3.8 Y10
- 10 条时间线：✅ §3.8

**P0-05 结论：PASS-WITH-ISSUES（设计级）**。
- 协作进程：sealed + flock + 重新验证可证明一致。
- 恶意同 UID / 旧可写 FD 强写：**无法强制阻止**（除非 chattr +i 需特权），只能检测 + 拒绝，标记 NOT_VERIFIED。
- mmap 映射写入的检测需 fsync 后重验，NOT_VERIFIED。
- 运行时行为（flock 协作、chattr +i 阻止）NOT_TESTED。

---

## 4. P0-06｜Tombstone 对象模型、无覆盖发布与删除归因

### 4.1 缺陷定位：05.14 §5.3 对象模型矛盾

05.14 §5.3 把"源文件 rename 到 tombstone 路径"与"tombstone 文件内容含事务元数据"**混为同一对象**。但：

- 源对象可能是**任意二进制**，不能既当源文件又当元数据记录；
- 若 rename 源文件到 tombstone 路径，则 tombstone 路径下的文件是**源内容本身**，不是元数据；
- 若 tombstone 内容要写 `transaction_id` 等元数据，则**必须另建独立记录**，不能覆盖源内容。

**必须分离两个对象**。

### 4.2 对象模型（分离四个对象）

| 对象 | 含义 | 内容 | 生命周期 |
|------|------|------|---------|
| **源对象** | 待处理的原始文件 | 任意二进制 | WRITABLE→…→RETIRED（§3）|
| **隔离对象** | 源的隔离副本 | 源的字节拷贝（qfinal）| quarantine 阶段 |
| **Tombstone 元数据** | 描述本次清理事务的独立不可变记录 | 元数据（txn_id、cleanup_id、源身份、源哈希）| 独立于源，永久保留 |
| **Journal 记录** | 事务日志 | 各阶段状态 | 与事务绑定 |

**关键**：Tombstone 元数据是**独立文件**，**不承载源内容**。源内容要么保留在源路径、要么移到隔离/归档路径，但**绝不混入 tombstone 元数据文件**。

### 4.3 唯一 transaction_id / cleanup_id 生成与冲突处理

- `transaction_id`：全局唯一，`uuid4`（128-bit 随机，碰撞概率可忽略）。
- `cleanup_id`：每次清理事务独立生成，`uuid4`，与 `transaction_id` 一一对应。
- **冲突处理**：所有对象用 `O_EXCL` 创建；若同名已存在（uuid 碰撞，概率 ~2^-122），`O_EXCL` 返回 EEXIST → **重新生成 ID 重试**，不覆盖。

### 4.4 真正的原子无覆盖发布（区分 O_EXCL 创建 vs rename 无覆盖）

工单 §六.3 要求区分两者：

- **O_EXCL 创建**：`open(path, O_CREAT|O_EXCL)` 保证"目标不存在时才创建成功"，用于**新建对象**（版本文件、tombstone 元数据）。
- **rename 无覆盖**：`renameat2(RENAME_NOREPLACE)`（Linux）保证"目标不存在时才 rename 成功"，用于**发布对象且不覆盖已存在目标**。

**05.15 约定**：
- Tombstone 元数据、Journal 记录用 **O_EXCL 创建**（新建，不覆盖）。
- 隔离对象 qtmp→qfinal 的发布用 **renameat2(RENAME_NOREPLACE)**（目标存在则失败，不覆盖）。
- **不得**把"O_EXCL 创建"描述成"rename 无覆盖"，两者语义不同：前者是 open 时独占创建，后者是 rename 时拒绝覆盖。

### 4.5 状态定义与持久化顺序（消除 05.14 矛盾）

| 状态 | 准确含义 | 持久化动作 | 崩溃后可观察证据 |
|------|---------|-----------|----------------|
| DELETE_INTENT | 记录"将删除源对象"意图 | 写 intent + fsync | intent 存在 |
| TOMBSTONED | 源对象已改名为隔离名（不是 tombstone 元数据）| rename + fsync(dir) | 源路径消失 + 隔离对象存在 |
| REMOVED | 删除已确认（归因成立）| 写 REMOVED + fsync | REMOVED 记录存在 |
| CLEANED | 终态确认 | 写 CLEANED + fsync | CLEANED 记录存在 |

**关键修正（区别于 05.14）**：
- **TOMBSTONED 指的是"源对象被 rename 走"**，而 **Tombstone 元数据是独立文件**，两者不是一回事。
- **REMOVED 持久化之前，不得删除 Tombstone 元数据或归档**。

### 4.6 rename 成功但 REMOVED 未持久化的恢复规则

**场景**：rename 源对象到隔离名成功，但 REMOVED 记录尚未持久化时崩溃。

**恢复规则**：
1. 恢复时发现：DELETE_INTENT 存在、隔离对象存在（源已 rename 走）、但 REMOVED 不存在。
2. **不能**仅凭"源对象已消失"判定删除成功 —— 因为无法证明是本事务 rename 的（外部进程也可能 rename）。
3. **必须检查 Tombstone 元数据**：若 tombstone 元数据存在且身份/哈希匹配、绑定本 txn_id → 可补写 REMOVED（归因成立）。
4. 若 tombstone 元数据缺失/被替换/不匹配 → **INDETERMINATE**，保留现场，不补写 REMOVED。

**铁律**：不得仅凭对象存在或不存在补写 REMOVED。

### 4.7 删除归因三态

| 状态 | 含义 | 需要的持久化证据 | 后续动作 |
|------|------|----------------|---------|
| ALREADY_ABSENT | 清理前对象已不存在 | 清理开始 stat(src) ENOENT（审计记录）| 直接结束，不写 REMOVED |
| REMOVED_BY_THIS_CLEANUP | 有证据证明本事务删除 | ① DELETE_INTENT ② tombstone 元数据身份/哈希匹配 ③ 隔离对象存在 ④ REMOVED 已持久化 | 写终态审计，可归档 |
| INDETERMINATE | 无法证明删除归属 | 证据链不完整 | 保留现场，人工介入 |

### 4.8 12 条并发/崩溃/外部干扰时间线

| # | 场景 | 推演 | 最终归因 |
|---|------|------|---------|
| Z1 | 单事务正常删除 | intent→rename→REMOVED→CLEANED | REMOVED_BY_THIS_CLEANUP |
| Z2 | 两事务同时删除同一源 | 隔离对象 rename 竞争，一个成功一个失败（目标已存在）| 胜者 REMOVED，败者放弃 |
| Z3 | 两事务同时发布同名 tombstone 元数据 | O_EXCL 创建，一个成功一个 EEXIST | 一个成功，另一个换 ID |
| Z4 | rename 后崩溃（REMOVED 未持久化）| tombstone 元数据匹配 → 补写 REMOVED | REMOVED_BY_THIS_CLEANUP（有元数据）|
| Z5 | rename 后崩溃（tombstone 元数据缺失）| 无元数据 → 无法归因 | INDETERMINATE |
| Z6 | 外部替换 tombstone 元数据 | 身份（ino）变化 → 不匹配 | INDETERMINATE |
| Z7 | 外部删除 tombstone 元数据 | 元数据消失 | INDETERMINATE |
| Z8 | 同名源重建 | rename 后外部 recreate 同名源 → 源路径又存在 | INDETERMINATE |
| Z9 | 审计记录写入失败 | REMOVED 写失败 → 无法确认 | INDETERMINATE |
| Z10 | 目录持久化失败 | fsync(dir) 失败 → 标记 NOT_DURABLE | INDETERMINATE（或按元数据重验）|
| Z11 | 重复恢复 | 幂等：已 INDETERMINATE 的保持 | INDETERMINATE |
| Z12 | 归档中断 | 归档 rename 成功但回执未写 → 按归档对象哈希匹配确认 | 可确认（哈希一致）|

### 4.9 归档策略（不破坏原始事务证据）

1. **归档前提**：仅当删除归因已成立（CLEANED）才可归档。
2. **归档动作**：`rename(tombstone/<cleanup_id> → archive/<cleanup_id>)`，同样走 O_EXCL + 元数据校验。
3. **归档不得破坏证据**：归档的是**元数据副本**，原 tombstone 元数据保留或迁移，迁移时记录源→归档的映射。
4. **归档文件存在 ≠ 删除事务完成证明**：删除完成的证明是"REMOVED 记录 + tombstone 元数据身份/哈希匹配"，归档只是存储位置变化，不改变归因结论。
5. **永不 unlink 元数据**：归档后仍保留（冷存储），除非运维手动清理（需记录）。

### 4.10 P0-06 验收对照与结论

- 分离源对象/隔离对象/tombstone 元数据/journal：✅ §4.2
- 唯一 txn_id/cleanup_id 生成与冲突：✅ §4.3
- 真原子无覆盖发布（区分 O_EXCL 与 renameat2 RENAME_NOREPLACE）：✅ §4.4
- 状态定义 + 持久化顺序：✅ §4.5
- rename 成功 REMOVED 未持久化恢复：✅ §4.6
- 三态归因：✅ §4.7
- 12 条时间线（含并发删除、同名 tombstone、外部替换/删除、审计失败）：✅ §4.8
- 归档策略：✅ §4.9

**P0-06 结论：PASS（设计级，对象模型矛盾已消除，源对象与 tombstone 元数据分离）**。崩溃恢复语义（rename/fsync）NOT_TESTED。

---

## 5. 设计不变量（I-01 ~ I-25）

| # | 不变量 | 证明章节 | 失效场景 | 拒绝策略 |
|---|--------|---------|---------|---------|
| I-01 | 无有效锁不得执行受保护操作 | §2 | fd 无 flock | 拒绝 |
| I-02 | 所有参与者锁同一稳定锁对象 | §2（P0-03 继承）| inode 分裂 | 身份校验拒绝 |
| I-03 | fork 前禁止持锁（禁持锁 fork）| §2.3/§2.4 | 持锁调 fork | 入口抛异常拒绝 |
| I-04 | before 回调不获取任何锁 | §2.4/§2.5 | before 拿锁 | 违反即拒绝 |
| I-05 | 全局锁获取必须升序 | §2.3 | 逆序获取 | 禁止（静态约束）|
| I-06 | 子进程不复用失效同步对象 | §2.6 | child 重建失败 | DENY（fail-closed）|
| I-07 | 子进程清理不 LOCK_UN 父进程共享 OFD | §2.5 | 误 LOCK_UN | 禁止，仅 close |
| I-08 | 提交竞争用单一固定对象 | §1.3 | 每事务独立锁名 | 禁止（假 CAS）|
| I-09 | revision 检查与发布在同一临界区 | §1.5 | 检查后释放锁再发布 | 禁止 |
| I-10 | revision 严格单调递增 | §1.5/§1.6 | rev 回退 | 断言失败拒绝 |
| I-11 | PREPARED 先于提交竞争持久化 | §1.4 阶段 B | 未 PREPARED 就 flock | 拒绝 |
| I-12 | 正式 Candidate 不暴露半成品 | §1.4 阶段 A | 版本文件写一半 | SHA256 校验删除 |
| I-13 | 不丢失更新 | §1.5/§1.6 | 双事务 | flock 串行化 |
| I-14 | 恢复不认领未确认事务 | §1.7 X6/X7 | 未 COMMIT | 按 current 为准 |
| I-15 | 源对象 sealed 必须有持久化证据 | §3.3 | 仅约定无记录 | DENY |
| I-16 | seal 后旧可写 FD 写入必须被阻止或检测 | §3.4 | 强写 | 检测则 DENY/INDETERMINATE |
| I-17 | quarantine 只操作副本，不改源 | §3.6 | 直接改源 | 禁止 |
| I-18 | Source Retire 是独立事务，需重新验证源 | §3.6 | 跳过再验证 | 禁止 |
| I-19 | 无法证明 sealed/无旧句柄/内容一致 → DENY | §3.9 | 证据不足 | DENY + INDETERMINATE |
| I-20 | 源对象与 tombstone 元数据必须分离 | §4.2 | 混为同一对象 | 禁止 |
| I-21 | tombstone 元数据不承载源内容 | §4.2 | 覆盖源内容 | 禁止 |
| I-22 | 原子无覆盖发布用 renameat2 RENAME_NOREPLACE | §4.4 | 用普通 rename 覆盖 | 禁止 |
| I-23 | 删除归因需持久化证据，意图≠事实 | §4.5/§4.7 | 仅 intent | INDETERMINATE |
| I-24 | REMOVED 未持久化不得删/归档 tombstone 元数据 | §4.5/§4.6 | 提前删 | 禁止 |
| I-25 | 对象不存在 ≠ 本事务删除成功 | §4.6/§4.7 | 仅源消失 | INDETERMINATE |

**共 25 条不变量，每条有证明章节与拒绝策略。**

---

## 6. 端到端交叉审查（10 条链路）

| # | 链路 | 前置 | 状态转换 | 持久化边界 | 竞争点 | 崩溃点 | 恢复动作 | 结论 |
|---|------|------|---------|-----------|--------|--------|---------|------|
| E1 | 锁→fork→父子退出 | 有效锁 | fork | flock | fork | fork 中 | child 重置 | PASS |
| E2 | 锁身份→Candidate→PREPARED→竞争→发布 | 有效锁 | 版本文件→prepared→flock→rename | PREPARED 先于 flock | .commit-lock | rename 前后 | 按 current | PASS |
| E3 | 双事务竞争提交 | 并发 | 竞争→胜/弃 | — | .commit-lock | 竞争点 | 败者重试 | PASS |
| E4 | 源 seal→quarantine→retire | sealed | WRITABLE→SEALED→QUARANTINED→RETIRED | SEALED 记录 | seal 点 | seal 后写入 | 重新验证 | PASS-WITH-ISSUES |
| E5 | 旧可写 FD 强写 | seal 前可写 FD | seal 后写 | — | 旧 FD | seal 后 | 检测哈希变化 | PASS-WITH-ISSUES（恶意 NOT_VERIFIED）|
| E6 | 删除 intent→rename→REMOVED→CLEANED | intent | DELETE_INTENT→TOMBSTONED→REMOVED→CLEANED | rename→REMOVED | tombstone O_EXCL | rename 后 | 元数据匹配补写 | PASS |
| E7 | rename 后崩溃（REMOVED 未持久化）| intent | rename 后 | REMOVED 未写 | — | rename 后 | 元数据匹配/不匹配 | INDETERMINATE（无元数据时）|
| E8 | 外部替换/删除 tombstone 元数据 | 已 tombstoned | 元数据变化 | — | — | 任意 | INDETERMINATE | PASS |
| E9 | 审计不可用/写失败 | 删除中 | REMOVED 写失败 | REMOVED 失败 | — | 审计写 | INDETERMINATE | PASS |
| E10 | 恶意同 UID 进程绕过协议 | 恶意进程 | 任意 | 无（无法阻止）| 任意 | 任意 | 身份异常检测拒绝 | **NOT_VERIFIED** |

**结论**：E1~E9 在遵守协议边界内无已知矛盾；E10 是已知能力边界，NOT_VERIFIED。

---

## 7. 阻断状态（保持不变）

| 项目 | 状态 |
|------|------|
| IMPLEMENT-01 | **STOP** |
| P0-BLOCK-1 | 未解除 |
| P0-BLOCK-2 | 未解除 |
| P1-BLOCK-3 | 未解除 |
| F1 | 不解除 |
| P0.8 | 不解除 |
| F2 | 不解除 |
| 冻结源码哈希 | 不变（见执行报告）|
| A/B 工作区 | 不合并 |

---

## 8. 最终结论

**DESIGN-05.15 = PASS-WITH-ISSUES（设计级）**。

四项 P0 修订结论：

| # | 结论 | 核心依据 | 未验证项 |
|---|------|---------|---------|
| P0-02 | PASS（设计级）| §2.8 八条锁顺序 + §2.9 五条异常，全局锁顺序消除死锁 | 多线程 fork OS 行为 NOT_VERIFIED |
| P0-04 | PASS（设计级）| §1.7 十二条时间线，flock 临界区给出真 CAS，废弃 link 假 CAS | Windows 原子原语 NOT_VERIFIED |
| P0-05 | PASS-WITH-ISSUES | §3.8 十条时间线，sealed 状态机 + 重新验证 | 恶意同 UID/旧 FD 强写 NOT_VERIFIED |
| P0-06 | PASS（设计级）| §4.8 十二条时间线，源对象与 tombstone 元数据分离 | rename/fsync 崩溃语义 NOT_TESTED |

**诚实边界（不伪造确定性）**：
1. 恶意同 UID 进程（旧 FD 强写、inode 分裂）是文件系统锁模型的固有能力边界，NOT_VERIFIED。
2. 所有 NOT_TESTED/NOT_VERIFIED 项保留，未降低标准、未删除失败分支。
3. 设计级通过 ≠ 运行时验证通过 ≠ 解除阻断。

**IMPLEMENT-01 继续 STOP，等待 ChatGPT 独立复审。**

---

## 9. 交付物索引

| 文件 | 说明 |
|------|------|
| `USIT-STAGE-B-DESIGN-05.15.md` | 本主设计文档 |
| `USIT-STAGE-B-DESIGN-05.15-ISSUES.md` | 四项问题清单及剩余风险 |
| `USIT-STAGE-B-DESIGN-05.15-DIFF.md` | 相对 05.14 逐项差异 |
| `USIT-STAGE-B-DESIGN-05.15-CROSS-REVIEW.md` | 反例审查记录 |
| `USIT-STAGE-B-DESIGN-05.15-EXECUTION-REPORT.md` | 实际执行报告 |
| `USIT-STAGE-B-DESIGN-05.15-HASH-MANIFEST.md` | 交付文件 SHA256 清单 |
| `USIT-STAGE-B-DESIGN-05.15-HASH-RECEIPT.txt` | 哈希验证回执 |
| `USIT-STAGE-B-DESIGN-05.15-交付物.zip` | 完整打包 |

---


