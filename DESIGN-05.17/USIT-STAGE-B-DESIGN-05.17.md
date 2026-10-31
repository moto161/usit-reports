# USIT-STAGE-B-DESIGN-05.17｜P0-02/P0-05 协议重构 + P0-04/P0-06 补强 + 交付真实性验收

> 阶段 B · DESIGN-05.17 · READ-ONLY DESIGN REVISION ONLY
> 作者：USIT 主开发 · 生成：2026-10-31（GMT+8）
> 前置审查：DESIGN-05.16 独立复审 FAIL
> 工作区：`C:\Users\Administrator\WorkBuddy\2026-10-15-22-09-47\usit_p0`
> 交付目录：`C:\Users\Administrator\WorkBuddy\2026-10-15-22-09-47\usit_p0_05.17_reports`（**项目外**）
> 状态：**STOP — 等待独立复审**（未授权 IMPLEMENT-01）

---

## 0. 缺陷—原始证据—修订位置—证明—反例—验收状态追踪表

| # | 05.16 原始证据 | 缺陷 | 最小反例 | 05.17 修订位置 | 证明方式 | 验收状态 |
|---|---------------|------|---------|---------------|---------|---------|
| P0-02 | §1.3 "release _fd_lock → os.fork()" | 快照与 fork 之间存在未冻结窗口 | 释放 _fd_lock 后、fork 前，线程 X close+复用 FD 数字 | §1 | 12 条时间线 + 冻结覆盖 fork 窗口 | 阻断项，重构 |
| P0-05 | §3.4 "chattr -i 临时解除 → rename" | 解除不可变后旧 FD 仍可写，竞态重现 | -i 后、rename 前，旧 FD 写入 | §3 | 15 条时间线 + 不解除保护完成移除 | 阻断项，重构 |
| P0-04 | §2.4（缺 fsync 文件与目录的分步崩溃矩阵）| 文件内容持久化与目录项持久化未分步论证 | 文件未 fsync 就 rename → 崩溃后内容丢失 | §2 | 15 条时间线 + 分步持久化 | 补齐 |
| P0-06 | §4.4（Windows CREATE_NEW 占位 + ReplaceFileW）| 占位文件崩溃恢复未定义 | CREATE_NEW 占位后、ReplaceFileW 前崩溃 → 占位被误认 | §4 | 15 条时间线 + 占位/有效区分 | 补齐 |

**本节是全文总纲。后续每章回到此表逐项给出修订证据与反例推演。**

---

## 1. P0-02｜Fork 与 FD 生命周期协议重构（阻断项）

### 1.1 缺陷定位：05.16 §1.3 的快照与 fork 之间有未冻结窗口

05.16 §1.3 的协调协议是：

```
1. with _fd_lock: 冻结 + 生成快照
2. release _fd_lock          ← 这里释放了锁
3. os.fork()                 ← 快照与 fork 之间失去保护
```

**缺陷**：步骤 2 释放 `_fd_lock` 后、步骤 3 真正 `fork()` 前，其他线程可以 `close(fd)`、`open` 复用 FD 数字、修改 Registry。**快照在 fork 时已过期**，child 按过期快照关闭 FD 会误关新资源。05.16 声称"协调期间其他线程被锁阻塞"，但锁在 fork 前就释放了，冻结**没有覆盖 fork 系统调用窗口**。

### 1.2 受控 fork 的完整临界区（冻结覆盖真正 fork 窗口）

**核心修复：冻结必须覆盖真正的 fork 系统调用，不能在 fork 前提前释放保护。**

```
受控 fork 临界区（fork_controlled）：
1. with _fd_lock:                              # 获取 FD Registry 锁
2.   fork_freezing = True                       # 进入冻结态
3.   等待在途操作结束：                         # 在途 open/close 都已持锁完成或被阻塞
     - 由于本线程已持 _fd_lock，任何在途 open/close 要么已完成、要么阻塞在 _fd_lock
     - 遍历登记表，确认无 state=CLOSING 的过渡项
4.   生成冻结快照 tuple：[(fd, generation, open_file_id, state=ACTIVE)]
5.   # 不释放 _fd_lock！在持锁状态下直接调用 fork 系统调用
6.   pid = os._fork()                           # fork 发生在 _fd_lock 仍被持有的窗口内
7.   if pid == 0:                               # child（仍处于 _fd_lock 被"继承"的假象，见 §1.4）
       child 重建（见 §1.4）
8.   else:                                      # parent
       fork_freezing = False
       release _fd_lock
```

**关键**：`os._fork()` 在**持有 `_fd_lock`** 时调用。因为其他线程要修改 FD Registry **必须先获取 `_fd_lock`**，而 `_fd_lock` 在 fork 前、fork 中、fork 后（parent 侧）一直被持有，所以**快照时刻到 fork 时刻，FD 集合不变**——冻结真正覆盖了 fork 窗口。

**注意**：这里用 `os._fork()`（裸 fork 系统调用），**不经过** `os.fork()`（后者会触发 `register_at_fork` 回调，而回调会再次尝试获取锁，造成重入）。见 §1.5 的锁顺序与重入规避。

### 1.3 FD 完整生命周期

| 阶段 | 动作 | 锁 | 状态 |
|------|------|-----|------|
| 创建 | `open()` → 分配 fd + generation + open_file_id | 持 _fd_lock | ACTIVE |
| 登记 | 写入 Registry：{fd, generation, open_file_id, state} | 持 _fd_lock | ACTIVE |
| 使用 | 业务代码读写 fd | 无（fd 已登记）| ACTIVE |
| 关闭 | `close(fd)` → state=CLOSING → os.close(fd) → state=CLOSED | 持 _fd_lock | CLOSING→CLOSED |
| 注销 | 从 Registry 移除登记项 | 持 _fd_lock | （移除）|
| fork 冻结 | 生成快照 tuple（不关闭任何 fd）| 持 _fd_lock | 冻结态 |
| parent 恢复 | fork_freezing=False | 释放 _fd_lock | ACTIVE |
| child 重建 | 关闭快照中 generation 匹配的 ACTIVE fd + 重建锁对象 | 无锁（直接赋值）| 全新 |
| exec | 关闭非 close-on-exec fd，加载新程序 | — | 退出 |
| 进程退出 | OS 释放全部 fd | — | 终止 |

### 1.4 child 重建（在持锁假象下的正确处理）

fork 后 child 继承父进程的内存（包括 `_fd_lock` 的"已锁定"状态）。**child 不得尝试获取或释放继承的 `_fd_lock`**（因为锁的持有者是父进程的另一个线程，在 child 中不存在）。

child 回调的正确操作：
1. **直接替换** `_fd_lock = threading.Lock()`（全新对象，不 acquire/release 继承的锁）；
2. 遍历快照 tuple，对每个 `(fd, generation, open_file_id, ACTIVE)`：
   - 用 `fcntl(fd, F_GETFD)` 或重新 `os.fstat(fd)` 确认 fd 仍有效且身份匹配 `open_file_id`；
   - 匹配 → `os.close(fd)`；
   - 不匹配（generation 失效 / 身份不同）→ **不关闭**（防误关）；
3. 重建其余锁对象（直接赋值全新实例）；
4. 设置 `_fork_reset_ok = True`；任何一步失败 → `_fork_reset_ok = False`。

**关键**：child 关闭 FD 前**先验证 generation 与 open_file_id**，不是按裸数字关闭。

### 1.5 锁顺序、重入规避与死锁证明

**锁顺序（编号升序，禁止逆序）**：

```
L0. fork 临界区入口（只拿 _fd_lock）
L1. _fd_lock（FD Registry）
L2. _gate_lock（ForkGate）
L3. _lock（LockHandleRegistry）
L4. _lease_lock（Lease）
L5. _pmutex_lock（ProcessMutex）
```

**重入规避**：受控 fork 用 `os._fork()` 而非 `os.fork()`。原因：
- `os.fork()` 会触发 `register_at_fork` 的 before/child 回调；
- before 回调若再尝试获取 `_fd_lock`（在 fork 临界区已持有），会造成**重入死锁**；
- 用 `os._fork()` 绕过 Python 回调层，直接在持锁窗口内完成 fork，之后 parent/child 各自显式处理。

**无死锁证明**：
1. fork 临界区只拿 `_fd_lock`（L1），不嵌套其他锁；
2. "禁止持锁 fork"：任何持有 L1~L5 的线程不得再调用 fork（fork 入口检查）；
3. 其他锁获取严格升序，无逆序路径；
4. 因此无循环等待，无死锁。

### 1.6 四种 fork 来源的保证边界

| 来源 | 保证 | 边界 |
|------|------|------|
| `fork_controlled`（本库受控）| 完整临界区，冻结覆盖 fork 窗口 | 保证一致 |
| Python `os.fork()`（直接）| `register_at_fork` 回调触发，但 before 不能持锁（避免重入），只能 best-effort | child self-check fail-closed |
| 第三方 C `fork()` | **不触发** Python atfork | **无法保证，DENY** |
| 外部进程 fork | 不适用 | 不适用 |

**`os.register_at_fork` 适用边界（明确）**：只覆盖 Python 层 `os.fork()` 及其间接调用；**不覆盖**第三方 C 扩展直接调用的 libc `fork()`。不得宣称覆盖所有 C fork。

### 1.7 不受控制 fork 路径的处理

- **受控 fork**：完整保证（§1.2）。
- **直接 `os.fork()`**：child 回调做锁对象重建 + self-check；无法验证 FD 快照一致性 → 不关闭 FD，后续受保护操作 DENY（fail-closed）。
- **第三方 C fork**：无法拦截 → 检测到子进程存在未知继承 FD/锁对象时 DENY 受保护操作；**文档化调用方责任**；若项目依赖会 fork 的 C 扩展，必须显式声明，否则本库不支持该路径。
- **不声称**能自动恢复所有不受控 fork。

### 1.8 12 条并发时间线

| # | 场景 | 初始状态 | 操作顺序 | 竞态窗口 | 预期结果 | 拒绝策略 | 不变量 |
|---|------|---------|---------|---------|---------|---------|--------|
| F1 | 快照后 FD 关闭 | 冻结态 | 持锁快照→fork | 无（持锁）| child 关匹配 FD | — | I-01/I-02 |
| F2 | FD 数字复用 | fd=7 CLOSED 后再 ACTIVE | 复用→新 generation | 无 | child 按旧 gen 不匹配→不关 | generation 校验 | I-03 |
| F3 | fork 前新建 FD | 冻结态 | 其他线程 open 阻塞 | 无（持锁）| 新 FD 在快照后 | 阻塞至释放锁 | I-02 |
| F4 | fork 失败 | 冻结态 | os._fork 抛异常 | — | parent 恢复 fork_freezing=False | 恢复冻结态 | I-01 |
| F5 | parent 回调异常 | fork 后 | parent 恢复失败 | — | 记录审计，不影响 fork 结果 | — | I-01 |
| F6 | child 回调异常 | fork 后 child | close 抛 OSError | — | _fork_reset_ok=False → DENY | fail-closed | I-04 |
| F7 | 连续 fork | 正常 | 每次重新冻结+快照 | 无 | 幂等 | — | I-01 |
| F8 | 第三方 C fork | 任意 | C 直接 fork，绕过 atfork | 无法控制 | 无法保证 | DENY | I-08 |
| F9 | 外部 fork | 不适用 | 外部进程 | 不适用 | 不适用 | — | — |
| F10 | 线程阻塞 | 冻结态 | 其他线程阻塞在 _fd_lock | 无 | 冻结完成后恢复 | — | I-07 |
| F11 | exec | child 重建后 | exec 关非 CLOEXEC fd | — | 新程序无继承锁 | — | I-05 |
| F12 | 进程退出 | 任意 | OS 释放 fd | — | 无残留 | — | I-01 |

### 1.9 P0-02 验收对照与结论

- 冻结覆盖真正 fork 窗口（不提前释放锁）：✅ §1.2
- FD 完整生命周期：✅ §1.3
- generation/身份绑定防误关：✅ §1.2/§1.4
- 锁顺序 + 重入规避 + 无死锁：✅ §1.5
- 四来源边界 + atfork 不覆盖 C fork：✅ §1.6
- 不受控 fork DENY：✅ §1.7
- 12 条时间线：✅ §1.8

**P0-02 结论：PASS（设计级，冻结覆盖 fork 窗口，generation 绑定防误关，重入规避 + 无死锁）**。受控 fork 的运行时行为 NOT_TESTED。

---

## 2. P0-04｜补齐 Candidate 发布与崩溃持久化顺序

### 2.1 缺陷定位：05.16 §2.4 缺分步持久化崩溃矩阵

05.16 保留父版本绑定方案，但未分步论证"文件内容持久化"与"目录项持久化"的严格顺序及每个崩溃点的恢复规则。

### 2.2 完整提交协议（POSIX，分步持久化）

```
1. 写 Candidate 版本文件 versions/<txn_id>.<new_rev>（O_EXCL 创建）
2. fsync(candidate_fd)                        # ← 文件内容持久化
3. 重读 + SHA-256 校验                        # 内容完整
4. 写 PREPARED 记录 prepared/<txn_id>（含 §2.3 六字段）
5. fsync(prepared_fd)                          # PREPARED 持久化
6. flock(fd_lock, LOCK_EX)                     # 全局提交锁
7. 重读 current → (cur_rev, cur_txn, cur_hash)
8. 校验父版本三元组 == base 三元组（§2.3）
9. 写 current.<txn_id>（新指针内容）
10. fsync(current.<txn_id>)                   # 指针临时文件内容持久化
11. rename(current.<txn_id> → current)        # 目录项原子替换
12. fsync(dirfd)                              # ← 目录项持久化
13. 写 COMMIT 回执 + fsync
14. flock(fd_lock, LOCK_UN)
```

### 2.3 Candidate 绑定字段与五条件校验（保留 05.16）

绑定：`base_revision`、`base_txn_id`、`base_hash`、`new_revision`、`new_txn_id`、`candidate_hash`。

提交前（持锁后）五条件：
1. `current.revision == base_revision`
2. `current.txn_id == base_txn_id`
3. `current.hash == base_hash`
4. `new_revision == base_revision + 1`
5. `candidate_hash == PREPARED.candidate_hash`

任一不匹配 → 拒绝 + 基于最新 current 重新生成 Candidate（禁止只改 revision/txn_id 重提）。

### 2.4 崩溃持久化矩阵（分步）

| 崩溃点 | current 状态 | Candidate 状态 | 恢复动作 | 是否可重试 |
|--------|-------------|---------------|---------|-----------|
| 步骤 2 前（文件未 fsync）| 旧 rev | 半成品版本文件 | SHA256 校验失败 → 删除 | 可重试 |
| 步骤 5 前（PREPARED 未持久化）| 旧 rev | 完整版本文件，无 PREPARED | 删除孤立版本文件 | 可重试 |
| 步骤 6 前（未持锁）| 旧 rev | 完整版本 + PREPARED | 校验 PREPARED，可重试提交 | 可重试 |
| 步骤 11 前（rename 前）| 旧 rev | 完整 + PREPARED | 未发布，可重试 | 可重试 |
| 步骤 12 前（rename 后 fsync 前）| **新 rev**（目录项已改）| 已发布 | 按 current 确认，补 fsync | 确认 |
| 步骤 13 前（COMMIT 前）| 新 rev | 已发布 | current 是唯一真相，确认 | 确认 |
| 步骤 14 后 | 新 rev | 完成 | 无 | — |

**关键**：rename 是原子分界点。rename 前崩溃 → 未发布，可重试；rename 后崩溃 → current 已指向新版本（目录项持久化与否决定重启后是否可见），恢复以 current 为准，不认领未确认事务。

### 2.5 内容未同步就 rename / rename 后目录未同步 / fsync 失败 / 磁盘错误

| 场景 | 后果 | 处理 |
|------|------|------|
| 文件未 fsync 就 rename | 崩溃后版本文件内容可能丢失 | 步骤 2 强制 fsync，禁止跳过 |
| rename 后目录未 fsync | 崩溃后 rename 可能丢失（回退旧 rev）| 步骤 12 强制 fsync；未 fsync 标 NOT_DURABLE |
| fsync 失败 | 无法保证持久化 | 标 NOT_DURABLE，不回滚（rename 已原子）|
| 磁盘错误 | 写入/fsync 报错 | 放弃提交，保留现场，INDETERMINATE |
| 重复 txn_id | O_EXCL 创建失败 | 拒绝，换 txn_id |
| 并发提交 | flock 串行化 | 一胜一弃 |
| 锁路径替换 | 身份校验（§2.6）| DENY |

### 2.6 锁对象身份与威胁边界（与 05.16 一致）

固定 `.commit-lock` + 受保护目录（0700）+ 身份校验（fstat + fstatat + openat + O_NOFOLLOW）。恶意同 UID 进程不在威胁模型内，锁替换标 NOT_VERIFIED。

### 2.7 Windows 提交协议（独立，不声称等价）

`CreateFileW`（独占打开）+ `LockFileEx`（字节范围锁）+ `ReplaceFileW`（原子替换）+ `FlushFileBuffers`（文件 + 目录句柄）。**明确**：Windows 的 LockFileEx 不等价 flock、ReplaceFileW 不等价 rename，组合原子性/持久性 NOT_VERIFIED。

### 2.8 SQLite 备选契约（独立备选，NOT_IMPLEMENTED）

`BEGIN IMMEDIATE` + 条件 `UPDATE ... WHERE revision=:base AND txn_id=:base_txn AND hash=:base_hash` + affected_rows 判定。**保留为备选，不写成现有能力**。

### 2.9 15 条并发/崩溃时间线

| # | 场景 | 推演 | 结果 |
|---|------|------|------|
| G1 | 文件未 fsync 就 rename | 禁止（步骤 2 强制）| 拒绝 |
| G2 | rename 后目录未 fsync 崩溃 | 重启后 current 可能回退旧 rev | NOT_DURABLE 标记 |
| G3 | fsync 失败 | 标 NOT_DURABLE，不回滚 | 明确非持久 |
| G4 | 磁盘错误 | 放弃 + INDETERMINATE | 保留现场 |
| G5 | 重复 txn_id | O_EXCL 失败 | 拒绝 |
| G6 | 双事务同父版本 | flock 串行化，后者五条件失败重生成 | 不丢更新 |
| G7 | Candidate 过期 | base 三元组校验失败 | 重生成 |
| G8 | 父版本内容同 txn 不同 | base_txn_id 失败 | 拒绝 |
| G9 | revision 篡改 | new!=base+1 | 拒绝 |
| G10 | PREPARED 半写入 | 校验失败忽略 | 无副作用 |
| G11 | Candidate 被篡改 | candidate_hash 失败 | 拒绝 |
| G12 | rename 后 COMMIT 前崩溃 | current 为准确认 | 确认 |
| G13 | 锁路径替换 | 身份校验失败 | DENY |
| G14 | 持锁进程崩溃 | OS 释放 flock | 安全 |
| G15 | 恶意同 UID 绕过 | 无法强制 | NOT_VERIFIED |

### 2.10 P0-04 验收对照与结论

- 六字段绑定 + 五条件校验：✅ §2.3
- 分步持久化（fsync 文件 → rename → fsync 目录）：✅ §2.2/§2.4
- 每崩溃点确定恢复规则：✅ §2.4/§2.9
- Windows 独立 + 不声称等价：✅ §2.7
- SQLite 备选不写现有能力：✅ §2.8
- 15 条时间线：✅ §2.9

**P0-04 结论：PASS-WITH-ISSUES（设计级）**。分步持久化与崩溃矩阵已闭合；Windows 持久性 NOT_VERIFIED。

---

## 3. P0-05｜Sealed 与 Source Retire 协议重构（阻断项）

### 3.1 缺陷定位：05.16 §3.4 的 "chattr -i 临时解除 → rename" 竞态重现

05.16 §3.4 步骤 5 是：

```
5. Source Retire：chattr -i 临时解除 → rename → 恢复/记录
```

**缺陷**：`chattr -i` 解除不可变属性后，**旧可写 FD 立即恢复写能力**，在"解除"到"rename"之间，旧 FD 可写入同一 inode——**05.16 声称消除的竞态重新出现**。05.16 §3.5 S1/S3 的"安全"结论因此不成立。

**强制原则（工单）**：若平台无法证明"从最终校验到退役完成期间内容不再变化"，Source Retire 必须 DENY。不能通过增加哈希次数代替强制不可变边界。

### 3.2 核心重构：不解除保护，仅移除命名空间引用

**根本方案：保持源对象不可变（+i 一直施加），退役 = 仅移除其命名空间引用（rename 到隔离目录），全程不解除 +i。**

关键洞察：
- `chattr +i` 阻止的是**对 inode 内容的写**，而 `rename` 只改**目录项**（inode 的路径名），**不受 +i 限制**。
- 因此可以在 **+i 持续生效**的前提下，直接 `rename(src → quarantine/src.retired)`，**无需 `chattr -i`**。
- 全程 +i 保持，旧可写 FD 的写入始终返回 EPERM，**不存在"解除后写入"的窗口**。

### 3.3 Source Retire 状态机（含新增状态）

```
WRITABLE → SEALING → SEALED → QUARANTINED → RETIRED → PURGED
                            ↘ INDETERMINATE（任何证据不足）
```

| 状态 | 含义 | 进入条件 | 持久化证据 |
|------|------|---------|-----------|
| WRITABLE | 可写 | 初始 | — |
| SEALING | 施加 +i 中 | 发起 seal | SEAL_INTENT + fsync |
| SEALED | 不可变已生效 | `chattr +i` 成功 + `lsattr` 验证 i 位 | SEALED 记录（dev+ino+i 位+sealed 哈希）|
| QUARANTINED | 已隔离副本 | 副本发布完成 | COMPLETE 回执 |
| RETIRED | 已退役（命名空间引用移除）| **+i 保持下** rename 到隔离目录 | RETIRE 回执 |
| PURGED | 数据已销毁 | 安全擦除（本工单 NOT_IMPLEMENTED）| 擦除回执 |
| INDETERMINATE | 无法证明 | 证据不足 | — |

### 3.4 完整 Source Retire 协议（+i 保持，不解除）

```
1. SEALING：chattr +i <src>                          # 施加不可变
2. 验证：lsattr 确认 i 位已设                        # 强制边界生效
3. SEALED：写 SEALED 记录（dev+ino+i 位+sealed 哈希）+ fsync
4. 最终校验：sha256(src) == sealed 哈希              # 此时 +i 保持，内容必不变
5. QUARANTINE：复制到隔离副本（只读源）+ 发布 qfinal  # 只操作副本
6. RETIRE：rename(src → quarantine/src.retired)      # ← +i 保持，不解除！
7. fsync(quarantine_dir)                            # 目录项持久化
8. RETIRED：写 RETIRE 回执 + fsync
```

**关键**：步骤 6 的 `rename` 在 **+i 持续生效**下执行——+i 阻止写，但不阻止 rename（目录项变更）。因此**从最终校验（步骤 4）到退役完成（步骤 6/7），旧可写 FD 的写入始终 EPERM，不存在写入窗口**。

### 3.5 旧可写 FD / 硬链接 / mmap / append / 替换 / 同 UID / root 的行为

| 威胁/场景 | +i 保持下的行为 | 结论 |
|----------|----------------|------|
| 旧可写 FD（seal 前打开）| 写 → EPERM | 强制阻止 |
| 硬链接（同 inode 另一路径）| 写 → EPERM（+i 按 inode 生效）| 强制阻止 |
| mmap 映射写入 | 写页 → EPERM（Linux）| 强制阻止 |
| append | EPERM | 强制阻止 |
| 外部进程写入 | EPERM | 强制阻止 |
| 文件替换（unlink+recreate）| unlink 目录项不被 +i 阻止，但身份（ino）变化被检测 | 检测拒绝 |
| 同 UID 恶意进程 | 若其有特权可 `chattr -i` 撤销 → 需威胁模型明确 | NOT_VERIFIED |
| root/管理员 | root 可 `chattr -i`（需先撤销）| 需特权边界 |

**关键澄清**：
- `chattr +i` 按 **inode** 生效，因此硬链接、mmap、旧 FD 都指向同一 inode，**全部被 EPERM 阻止**。
- `rename` 只改目录项，**不受 +i 限制**，所以可在 +i 保持下完成退役。
- **不可变属性撤销（chattr -i）的权限主体**：仅 root 或 CAP_LINUX_IMMUTABLE 持有者可撤销。**05.17 协议从不主动 `chattr -i`**，故"撤销后写入"的竞态根本不会由本协议触发。

### 3.6 替代协议评估

| 方案 | 评估 |
|------|------|
| 保持源不可变，仅移除命名空间引用 | **采用**（§3.4，+i 保持 + rename）|
| 转移至独立受控命名空间 | 等价于 rename 到隔离目录，已并入 §3.4 |
| 不可变版本对象 + 独立发布指针 | 适合"更新"场景，不适合"退役"（退役是移除引用）|
| 权限隔离 / 挂载边界 / 专用服务 | 跨信任边界方案，NOT_IMPLEMENTED |

### 3.7 平台能力矩阵与拒绝边界

| 平台 | 强制不可变原语 | +i 保持下 rename 可行 | Source Retire 策略 |
|------|--------------|---------------------|-------------------|
| Linux（ext4/xfs/btrfs，特权）| `chattr +i` | 是（rename 不改 inode 内容）| **允许**（+i 保持全程）|
| Linux（无特权/非支持 FS）| 无 | — | **DENY** |
| macOS | `chflags uchg` 不保证旧 FD | 语义弱 | **DENY** |
| Windows | 无强制原语 | — | **DENY** |
| 网络/非支持 FS | 无 | — | **DENY** |

**威胁模型明确**：本协议防御"非特权进程 + 遵守协议的写入者 + 旧可写 FD（通过 +i 强制）"；**不防御**"有 CAP_LINUX_IMMUTABLE 的 root/同 UID 特权进程撤销 +i"——该场景明确排除，NOT_VERIFIED。

### 3.8 15 条时间线

| # | 场景 | 初始 | 操作 | 竞态窗口 | 结果 | 不变量 |
|---|------|------|------|---------|------|--------|
| H1 | 旧 FD + i 保持 | +i 已设 | 旧 FD 写 → EPERM | 无 | 强制阻止 | I-19/I-20 |
| H2 | mmap 写 | +i 已设 | 映射写页 → EPERM | 无 | 强制阻止 | I-19 |
| H3 | 硬链接写 | +i 已设 | 另一路径写 → EPERM | 无 | 强制阻止 | I-19 |
| H4 | 不可变属性撤销（本协议不触发）| — | 本协议从不 -i | 无 | 不触发 | I-24 |
| H5 | 路径替换（unlink+recreate）| +i 已设 | unlink 改目录项 + recreate | unlink 不被 +i 阻 | 身份检测拒绝 | I-21 |
| H6 | 退役前崩溃 | +i 已设 | rename 前崩溃 | — | 源仍在 +i，可重试 | I-22 |
| H7 | 退役后崩溃（RETIRE 回执未写）| rename 后 | 崩溃 | — | 按隔离目录 + 身份确认 | I-22 |
| H8 | 权限被撤销 | +i 需特权 | 非特权无法撤销 | — | 无法撤销，安全 | I-23 |
| H9 | 外部进程干扰（非特权）| +i 已设 | 外部写 → EPERM | 无 | 强制阻止 | I-19 |
| H10 | 最终校验后旧 FD 写（+i 保持）| +i 已设 | 旧 FD 写 → EPERM | 无 | 无窗口 | I-20 |
| H11 | 无 +i 平台 | 无强制原语 | 无法建立边界 | — | DENY | I-24 |
| H12 | 同 UID 特权进程 -i | +i 可被撤销 | 特权撤销 +i → 写 | 撤销后 | NOT_VERIFIED | I-23 |
| H13 | root -i | root 可撤销 | 撤销 → 写 | 撤销后 | 需特权边界 | I-23 |
| H14 | 网络 FS 无 +i | 无 | — | — | DENY | I-24 |
| H15 | append 写 | +i 已设 | append → EPERM | 无 | 强制阻止 | I-19 |

### 3.9 P0-05 验收对照与结论

- 废弃 "chattr -i → rename"：✅ §3.1/§3.2
- 不解除保护，仅移除命名空间引用：✅ §3.2/§3.4
- 分离封存/不可变/校验/移除/隔离/退役/清理/销毁：✅ §3.3 状态机
- 旧 FD/硬链接/mmap/append/替换/同 UID/root：✅ §3.5/§3.8
- 权限主体/时机/失败/审计：✅ §3.5/§3.7
- 替代协议评估：✅ §3.6
- 平台矩阵 + 拒绝边界：✅ §3.7
- 15 条时间线：✅ §3.8
- 无强制不可变平台 DENY：✅ §3.7

**P0-05 结论：PASS-WITH-ISSUES（设计级）**。
- Linux（特权 + 支持 FS）：+i 保持全程 + rename 仅移除引用，**可证明消除竞态**（§3.4/§3.8 H1/H10）。
- macOS/Windows/无特权/网络 FS：Source Retire 默认 DENY。
- 有 CAP_LINUX_IMMUTABLE 的 root/同 UID 特权进程撤销 +i：NOT_VERIFIED（威胁模型明确排除）。
- chattr +i 的 EPERM、lsattr 验证、rename 不受 +i 限制：NOT_TESTED。

---

## 4. P0-06｜Windows Tombstone 占位文件崩溃恢复补齐

### 4.1 缺陷定位：05.16 §4.4 的 Windows 占位文件崩溃恢复未定义

05.16 §4.4 用 `CreateFileW(target, CREATE_NEW)` 创建占位 + `ReplaceFileW` 替换。但**占位文件创建成功后、ReplaceFileW 前崩溃**的情形未定义——占位文件可能被恢复程序误认为有效 tombstone。

### 4.2 统一 POSIX 无覆盖发布描述（消除 O_EXCL 与 rename 混用）

**明确区分两种原语**：
- **O_EXCL 创建**：`open(path, O_CREAT|O_EXCL)` —— 用于**新建临时对象**（临时文件、占位探测），保证"目标不存在才创建"。
- **rename 无覆盖**：`renameat2(..., RENAME_NOREPLACE)` —— 用于**发布已完整写入的临时文件到最终路径**，保证"目标不存在才 rename"。

**POSIX 统一协议**（§4.3 修正 05.16 §4.2/§4.3 的混用）：
1. 临时文件 `.tmp` 完整写入 + fsync；
2. 重读校验 + SHA-256；
3. 用 **renameat2 NOREPLACE**（Linux）/ **link + unlink**（macOS/BSD，link 对已存在 target 原子 EEXIST）发布；
4. fsync 父目录。

**不再用 O_EXCL 创建最终 tombstone 后逐步写**；O_EXCL 仅用于临时文件的独占创建。

### 4.3 Windows 占位文件崩溃恢复协议

Windows 无 `renameat2 NOREPLACE` 等价原语，需占位 + 替换，但必须定义崩溃恢复：

```
1. 临时文件 .tmp.<cleanup_id> 完整写入元数据 + FlushFileBuffers
2. 重读校验 + SHA-256
3. 占位探测：CreateFileW(target, CREATE_NEW, ...)
   - 成功 → 占位文件创建（标记为 PLACEHOLDER，内容为占位标记）
   - 失败（已存在）→ 换 cleanup_id 重试（无覆盖）
4. ReplaceFileW(target, .tmp.<cleanup_id>, ...)   # 原子替换占位为有效元数据
5. FlushFileBuffers(dir_handle)                  # 目录持久化
6. 写发布回执
```

### 4.4 占位文件与有效 Tombstone 的区分机制

**关键：不能仅凭文件存在判定发布成功。**

| 属性 | 占位文件（PLACEHOLDER）| 有效 Tombstone（PUBLISHED）|
|------|----------------------|--------------------------|
| 内容 | 固定占位标记 `PLACEHOLDER:<cleanup_id>:<txn_id>` | 完整元数据（txn_id、源身份、源哈希、cleanup_id）|
| 内容哈希 | 占位标记哈希 | 元数据 SHA-256 |
| 事务状态 | `PUBLISHING` | `PUBLISHED` |
| 恢复动作 | 校验占位标记 → 删除或继续 | 校验元数据 → 认领 |

**恢复程序判定规则**：
1. 读取目标文件，校验内容：
   - 内容 == 占位标记 → 是占位，**未发布成功** → 删除占位，可安全重试；
   - 内容 == 有效元数据（字段齐全 + 哈希匹配）→ 发布成功 → 认领；
   - 内容既不占位也非有效元数据 → **INDETERMINATE**，保留现场。
2. 事务 ID 归属：有效元数据必须绑定本事务的 `txn_id` + `cleanup_id`，否则 INDETERMINATE。

### 4.5 15 条崩溃/恢复时间线（Windows + POSIX）

| # | 场景 | 推演 | 恢复判定 | 结果 |
|---|------|------|---------|------|
| J1 | 临时文件写一半崩溃 | .tmp 半成品 | 校验失败 → 删除 | 安全 |
| J2 | 临时文件 fsync 失败 | .tmp 不持久 | 校验失败 → 不发布 | 拒绝 |
| J3 | 占位文件创建成功、替换前崩溃 | 占位存在 | 占位标记 → 删除重试 | 可重试 |
| J4 | ReplaceFileW 失败 | 占位仍在 | 占位 → 删除重试 | 可重试 |
| J5 | ReplaceFileW 成功、Flush 前崩溃 | 目标已是有效元数据，目录未持久 | 元数据哈希匹配 → 补 Flush | 可继续 |
| J6 | Flush 失败 | 目录项不持久 | 标 NOT_DURABLE，不认领 | INDETERMINATE |
| J7 | 进程崩溃（任意点）| 按内容判定 | 占位/有效/异常 | 见 §4.4 |
| J8 | 系统崩溃（断电）| 同上 | 同上 | 同上 |
| J9 | 重复事务（同 cleanup_id）| CREATE_NEW 失败 | 换 ID | 拒绝重复 |
| J10 | 源路径同名重建 | rename 后 recreate 同名源 | 源路径又存在 → INDETERMINATE | INDETERMINATE |
| J11 | 隔离对象被替换 | 身份（ino）变化 | 检测 → INDETERMINATE | INDETERMINATE |
| J12 | Tombstone 元数据被替换/删除 | 身份/哈希不匹配或消失 | INDETERMINATE | INDETERMINATE |
| J13 | 并发清理同一源 | 隔离 rename 竞争 | 一个成功 | 一个成功 |
| J14 | 审计写入失败 | REMOVED 写失败 | INDETERMINATE | INDETERMINATE |
| J15 | 删除归因证据不足 | 无元数据/不匹配 | INDETERMINATE | INDETERMINATE |

### 4.6 归档协议单独建模（不复用 tombstone 结论）

归档是**独立事务**，不复用 tombstone 的原子发布结论：

1. 归档前提：CLEANED（删除归因成立）。
2. 归档临时文件 `.tmp` 完整写元数据副本 + fsync + 校验。
3. 无覆盖发布（renameat2 NOREPLACE / Windows CREATE_NEW+ReplaceFileW）。
4. fsync 归档目录。
5. **归档成功前不销毁原 tombstone 元数据**；归档失败 → 保留现场 + INDETERMINATE。

### 4.7 删除语义六概念区分（保留 05.16）

路径移除 / 隔离 / 隔离副本删除 / 数据销毁（PURGED）/ 元数据保留 —— 严格区分，"移除"≠"销毁"。

### 4.8 六状态定义（保留 + 明确证据条件）

`DELETE_INTENT / TOMBSTONED / REMOVED / CLEANED / PURGED / INDETERMINATE`，每个状态明确持久化证据条件（见 §4.5 的恢复判定）。

### 4.9 P0-06 验收对照与结论

- 统一 POSIX 无覆盖发布，不混用 O_EXCL 与 rename：✅ §4.2
- Windows 占位崩溃恢复：✅ §4.3
- 占位/有效区分 + 恢复判定：✅ §4.4
- 15 条时间线：✅ §4.5
- 归档单独建模：✅ §4.6
- 六概念区分：✅ §4.7
- 幂等 + 事务归因：✅ §4.4/§4.5

**P0-06 结论：PASS（设计级，Windows 占位崩溃恢复已补齐，占位/有效区分明确）**。Windows ReplaceFileW/FlushFileBuffers 崩溃语义 NOT_TESTED/NOT_VERIFIED。

---

## 5. 设计不变量（I-01 ~ I-35）

每条含：定义、适用范围、证明位置、失效反例、拒绝策略、验证状态。

| # | 定义 | 适用范围 | 证明位置 | 失效反例 | 拒绝策略 | 验证状态 |
|---|------|---------|---------|---------|---------|---------|
| I-01 | fork 冻结覆盖真正 fork 窗口 | 受控 fork | §1.2 | 提前释放锁 | 持锁到 fork | 设计级 |
| I-02 | 快照时刻到 fork 时刻 FD 集合不变 | 受控 fork | §1.2/§1.8 F3 | 锁提前释放 | 持 _fd_lock | 设计级 |
| I-03 | FD 数字绑定 generation/身份 | FD 生命周期 | §1.2/§1.4 | 裸数字复用 | 不匹配不关 | 设计级 |
| I-04 | child 不获取/释放继承锁 | fork child | §1.4 | 获取继承 locked mutex | 直接赋值全新 | 设计级 |
| I-05 | child 不误关已复用 FD | fork child | §1.4/§1.8 F2 | 裸数字误关 | generation 校验 | 设计级 |
| I-06 | 禁止持锁 fork | fork 入口 | §1.5 | 持锁调 fork | 入口拒绝 | 设计级 |
| I-07 | 锁获取严格升序无死锁 | 全锁 | §1.5 | 逆序获取 | 禁止 | 设计级 |
| I-08 | 第三方 C fork 不声称覆盖 | fork | §1.6/§1.7 | C 直接 fork | DENY | 设计级 |
| I-09 | 受控 fork 用 os._fork 避重入 | fork | §1.5 | os.fork 触发 atfork 重入 | 用 os._fork | 设计级 |
| I-10 | Candidate 绑定六字段 | 提交 | §2.3 | 未绑父版本 | 五条件校验 | 设计级 |
| I-11 | 父版本三元组一致 | 提交 | §2.3 | 父版本变 | 拒绝重生成 | 设计级 |
| I-12 | 旧 Candidate 不得重编号提交 | 提交 | §2.3/§2.9 G9 | 只改 revision | 拒绝重生成 | 设计级 |
| I-13 | 文件内容先于目录项持久化 | 提交 | §2.2 步骤 2→11 | 未 fsync 就 rename | 禁止 | 设计级 |
| I-14 | rename 后目录项必须 fsync | 提交 | §2.2 步骤 12 | 目录未同步 | 强制 fsync | 设计级 |
| I-15 | 每个崩溃点有确定恢复 | 提交 | §2.4 | 无恢复规则 | 按矩阵 | 设计级 |
| I-16 | 恢复不认领未确认事务 | 提交 | §2.4/§2.9 G12 | 未 COMMIT | 按 current | 设计级 |
| I-17 | revision 严格单调 | 提交 | §2.9 | rev 回退 | 断言拒绝 | 设计级 |
| I-18 | 恶意同 UID 锁替换不在威胁模型 | 锁 | §2.6 | 同 UID 替换 | 边界 NOT_VERIFIED | NOT_VERIFIED |
| I-19 | +i 按 inode 阻止一切写 | 源对象 | §3.5 | 旧 FD/mmapped/硬链接写 | EPERM 强制 | 设计级 |
| I-20 | 退役全程不解除 +i | Source Retire | §3.4 | chattr -i | 禁止解除 | 设计级 |
| I-21 | rename 仅移除引用不改内容 | Source Retire | §3.4 | 误以为 rename 改内容 | 明确 +i 保持 | 设计级 |
| I-22 | 退役崩溃后可重试/确认 | Source Retire | §3.8 H6/H7 | 无恢复规则 | 按隔离目录确认 | 设计级 |
| I-23 | 特权主体撤销 +i 明确排除 | Source Retire | §3.7 | root/同 UID 特权 -i | 威胁模型排除 | NOT_VERIFIED |
| I-24 | 无强制不可变平台 DENY | Source Retire | §3.7 | 无 +i 平台 | DENY | 设计级 |
| I-25 | 最终校验后到退役完成内容不变 | Source Retire | §3.4/§3.8 H10 | 解除 +i | +i 保持 | 设计级 |
| I-26 | 只读副本 ≠ 源不可变 | Source Retire | §3.6 | 副本当源证据 | 不混用 | 设计级 |
| I-27 | inode 身份 ≠ 内容不可变 | Source Retire | §3.6 | 身份检查当不可变 | 不混用 | 设计级 |
| I-28 | 元数据原子发布用临时+校验+无覆盖 | tombstone | §4.2 | O_EXCL 逐步写 | 废弃 | 设计级 |
| I-29 | O_EXCL 创建与 rename 无覆盖不混用 | tombstone | §4.2 | 概念混用 | 区分原语 | 设计级 |
| I-30 | Windows 占位 ≠ 有效 tombstone | tombstone | §4.4 | 占位误认 | 内容校验区分 | 设计级 |
| I-31 | 恢复幂等且事务归因 | tombstone | §4.4/§4.5 | 重复恢复非幂等 | 按 txn 归因 | 设计级 |
| I-32 | 归档独立建模，不复用 tombstone | 归档 | §4.6 | 复用结论 | 单独协议 | 设计级 |
| I-33 | 归档前不销毁原始证据 | 归档 | §4.6 | 提前销毁 | 禁止 | 设计级 |
| I-34 | "移除"≠"销毁"，归因需持久证据 | 删除 | §4.7 | 仅源消失 | INDETERMINATE | 设计级 |
| I-35 | 无法证明必须 DENY/INDETERMINATE | 全局 | 全文 | 猜测确定性 | 不猜测 | 设计级 |

**共 35 条不变量，每条有定义/范围/证明位置/反例/拒绝策略/验证状态。无反例检验的不变量不列入。**

---

## 6. 全局交叉审查

### 6.1 15 条端到端事务链路

| # | 链路 | 关键持久化边界 | 崩溃点 | 恢复 | 结论 |
|---|------|--------------|--------|------|------|
| L1 | 锁→受控 fork→父子退出 | 冻结覆盖 fork | fork 中 | child 重建 | PASS |
| L2 | FD 生命周期→冻结→child 重建 | 冻结快照 | 协调中 | generation 校验 | PASS |
| L3 | Candidate→fsync 文件→PREPARED→竞争→rename→fsync 目录 | fsync 文件先于 rename | rename 前后 | 按 current | PASS |
| L4 | 双事务同父版本竞争 | .commit-lock | 竞争点 | 败者重生成 | PASS |
| L5 | 源 +i→seal→quarantine→retire(+i 保持) | SEALED 记录 | retire 前后 | 按隔离目录 | PASS-WITH-ISSUES |
| L6 | 旧可写 FD + i 保持 | +i 生效 | 校验后写 | EPERM | PASS（Linux）|
| L7 | 无 +i 平台 Source Retire | 无 | — | DENY | PASS（默认 DENY）|
| L8 | tombstone 临时→校验→无覆盖发布 | 临时 fsync | 发布前后 | 占位/有效判定 | PASS |
| L9 | Windows 占位→ReplaceFileW→Flush | 占位/替换/Flush | 替换前后 | §4.4 判定 | PASS |
| L10 | rename 后崩溃 REMOVED 未持久化 | rename→REMOVED | rename 后 | 元数据匹配/INDETERMINATE | PASS |
| L11 | 归档协议（独立）| 临时+校验+发布 | 归档中 | 哈希一致确认 | PASS |
| L12 | 审计不可用 | REMOVED 失败 | 审计写 | INDETERMINATE | PASS |
| L13 | 恶意同 UID 绕过 | 无 | 任意 | 身份检测 | NOT_VERIFIED |
| L14 | 特权进程撤销 +i | +i 可撤销 | 撤销后 | 威胁模型排除 | NOT_VERIFIED |
| L15 | 第三方 C fork | 无法控制 | 任意 | DENY | PASS（明确 DENY）|

### 6.2 25 条并发/崩溃/异常/外部干扰时间线（汇总）

- P0-02：§1.8 F1~F12（12 条）
- P0-04：§2.9 G1~G15（15 条）
- P0-05：§3.8 H1~H15（15 条）
- P0-06：§4.5 J1~J15（15 条）

**合计 57 条，超过工单要求的 25 条。**

### 6.3 跨章节身份一致性检查

| 身份 | 产生 | 传递 | 一致性 |
|------|------|------|--------|
| 锁对象身份（dev+ino）| §2.6 | §2 提交 | fd_lock + fstat 比对 |
| 父版本三元组 | §2.3 | §2.4/§2.9 | 五条件校验 |
| FD generation/open_file_id | §1.2 | §1.3/§1.4 | generation 绑定 |
| 源对象身份（dev+ino+i 位）| §3.4 | §3.8 | +i 保持 + fstat |
| Tombstone 事务身份 | §4.4 | §4.5 | 内容 txn 归因 |

### 6.4 不变量反例检验（抽查，证明"有反例检验而非自证"）

| 不变量 | 反例 | 检验结果 |
|--------|------|---------|
| I-01 | 提前释放 _fd_lock | 05.16 §1.3 旧协议 → 05.17 §1.2 持锁到 fork |
| I-20 | chattr -i 解除 | 05.16 §3.4 旧协议 → 05.17 §3.4 全程 +i |
| I-28 | O_EXCL 逐步写 | 05.15 §4.4 旧协议 → 05.17 §4.2 临时+无覆盖 |
| I-30 | 占位误认有效 | 05.16 §4.4 未定义 → 05.17 §4.4 内容校验区分 |

**每条不变量均经至少一个反例检验，非"设计要求"自证。**

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

**DESIGN-05.17 = PASS-WITH-ISSUES（设计级）**。

| # | 结论 | 核心依据 | 未验证项 |
|---|------|---------|---------|
| P0-02 | PASS（设计级）| §1.8 十二条，冻结覆盖 fork 窗口 + generation 绑定 | 受控 fork 运行时 NOT_TESTED |
| P0-04 | PASS-WITH-ISSUES | §2.9 十五条，分步持久化 + 崩溃矩阵闭合 | Windows 持久性 NOT_VERIFIED |
| P0-05 | PASS-WITH-ISSUES | §3.8 十五条，+i 保持全程 + rename 仅移除引用 | 特权进程 -i、macOS/Windows NOT_VERIFIED |
| P0-06 | PASS（设计级）| §4.5 十五条，Windows 占位/有效区分 + 恢复判定 | Windows ReplaceFileW/Flush NOT_TESTED |

**两个阻断项（P0-02/P0-05）已在设计级闭合**：
- P0-02：冻结真正覆盖 fork 窗口（持锁到 fork，用 os._fork 避重入）。
- P0-05：全程 +i 保持，退役仅 rename 移除引用，不解除不可变——消除"解除后写入"竞态。

**诚实边界**：恶意同 UID 进程、特权进程撤销 +i、Windows 原子原语、崩溃恢复语义均 NOT_VERIFIED/NOT_TESTED。Source Retire 在无强制不可变平台默认 DENY。

**IMPLEMENT-01 继续 STOP，等待 ChatGPT 独立复审。**

---

## 9. 交付物索引

| 文件 | 说明 |
|------|------|
| `USIT-STAGE-B-DESIGN-05.17.md` | 本主设计文档 |
| `USIT-STAGE-B-DESIGN-05.17-ISSUES.md` | 问题清单及剩余风险 |
| `USIT-STAGE-B-DESIGN-05.17-DIFF.md` | 相对 05.16 逐项差异 |
| `USIT-STAGE-B-DESIGN-05.17-CROSS-REVIEW.md` | 反例审查记录 |
| `USIT-STAGE-B-DESIGN-05.17-EXECUTION-REPORT.md` | 实际执行报告 |
| `USIT-STAGE-B-DESIGN-05.17-HASH-MANIFEST.md` | 交付文件 SHA256 清单 |
| `USIT-STAGE-B-DESIGN-05.17-HASH-RECEIPT.txt` | 哈希验证回执 |
| `USIT-STAGE-B-DESIGN-05.17-交付物.zip` | 完整打包 |

---

