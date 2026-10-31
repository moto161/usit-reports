# USIT-STAGE-B-DESIGN-05.16｜四项协议缺陷修复与独立验收

> 阶段 B · DESIGN-05.16 · READ-ONLY DESIGN REVISION ONLY
> 作者：USIT 主开发 · 生成：2026-10-31（GMT+8）
> 依据：ChatGPT 对 DESIGN-05.15 的独立复审结论（四项剩余缺陷）
> 工作区：`C:\Users\Administrator\WorkBuddy\2026-10-15-22-09-47\usit_p0`
> 交付目录：`C:\Users\Administrator\WorkBuddy\2026-10-15-22-09-47\usit_p0_05.16_reports`（**项目外**）
> 状态：**STOP — 等待独立复审**（未授权 IMPLEMENT-01）

---

## 0. 缺陷—证据—修订—验收追踪表

| # | 05.15 原文位置 | 缺陷 | 最小反例 | 05.16 修订位置 | 验证方法 | 未解决风险 |
|---|---------------|------|---------|---------------|---------|-----------|
| P0-02 | §2.4/§2.5（无锁快照）| 无锁枚举 FD Registry 时其他线程可能 close/复用 FD 数字，child 误关已重分配资源；第三方 C fork 不被 register_at_fork 覆盖 | 线程 X close(fd=7) 后线程 Y 复用 7 打开新文件，child 按旧快照 close(7) 误关新资源 | §1 | 10 条时间线 + FD generation 绑定 | 第三方 C fork 无法拦截，边界明确 |
| P0-04 | §1.4（Candidate 未绑父版本）/ §1.3（固定锁文件）| Candidate 不绑 base_revision/base_txn_id/base_hash；固定 `.commit-lock` 被 unlink+recreate 后 T1 锁旧 inode、T2 锁新 inode 双持 | ① 旧 Candidate 被改 revision 重提 ② 锁文件替换后双持 | §2 | 15 条时间线 + 父版本绑定 + 锁身份校验 | 恶意同 UID 锁替换 NOT_VERIFIED |
| P0-05 | §3.4/§3.6（最终哈希校验与 rename 之间竞态）| 最终哈希校验通过后、rename 前，旧可写 FD 仍可写同一 inode | A 持旧 FD，B 校验通过，A 写入，B rename 标记 RETIRED | §3 | 12 条时间线 + 强制不可变边界 | 无强制不可变平台默认 DENY |
| P0-06 | §4.4（O_EXCL 创建后逐步写入当原子发布）/ §4.9（归档混用 O_EXCL 与 rename）| 元数据非原子发布；归档协议把 O_EXCL 创建与 rename 无覆盖混为一谈 | 元数据写一半崩溃暴露半成品；归档 rename 覆盖已存在目标 | §4 | 15 条时间线 + 临时文件+renameat2 | Windows 无覆盖发布 NOT_VERIFIED |

**本节是全文总纲。后续每章回到此表逐项给出修订证据与反例推演。**

---

## 1. P0-02｜Fork 生命周期与 FD 快照一致性重构

### 1.1 缺陷定位：05.15 §2.4/§2.5 的无锁快照有竞态

05.15 用"fork 入口无锁状态下做 FD 快照 tuple"规避死锁，但引入了**新的竞态**：

- 无锁枚举 FD Registry 时，**其他线程可能同时 `close(fd)`**；
- `close` 释放的 FD 数字可能**立即被另一线程复用**（`open` 拿到同一个数字）；
- 快照 tuple 记录的是**裸 FD 数字**，不绑定对象的 generation/version；
- child 按旧快照 `close(7)` 时，数字 7 可能已被复用为**新资源** → **误关新资源**。

**此外**：`os.register_at_fork` 只覆盖 **Python 层 `os.fork()`**；第三方 C 扩展直接调用 libc `fork()` **不触发** Python 的 atfork 回调。05.15 未明确此边界。

### 1.2 FD 生命周期与 generation 绑定

**核心修复：FD 登记必须绑定 generation/version，不能只存裸数字。**

```
FD 登记项 = {
  fd: int,              # 裸数字（仅用于本进程内唯一标识）
  generation: int,      # 全局单调递增，每次登记 +1
  open_file_id: uuid,   # 打开时生成的对象身份（dev+ino+ctime 或 handle）
  state: ACTIVE | CLOSING | CLOSED,
  owner_thread: tid,    # 打开线程
}
```

**FD 数字复用规则**：`close(fd)` 时**先标记 `state=CLOSED` 并 `generation` 失效**，**再** `os.close(fd)`。后续 `open` 复用到同一数字时，分配**新 generation** 与新 `open_file_id`。因此"同一数字 + 不同 generation"代表不同对象。

**child 关闭规则**：child 只能关闭"快照中 generation 仍匹配、且 state=ACTIVE"的 FD；generation 不匹配 → **不关闭**（说明该数字已被复用，不属于快照时的对象）。

### 1.3 可证明一致的 fork 前协调协议（禁止无锁枚举）

05.15 的"无锁枚举"被废弃。05.16 改为**持锁协调 + 冻结**：

```
fork 前协调（在 fork 入口，持 _fd_lock）：
1. with _fd_lock:                          # 获取 FD Registry 锁
2.   标记 fork_coordinating = True          # 进入协调态
3.   冻结：遍历所有 ACTIVE FD，记录 (fd, generation, open_file_id)
4.   禁止新登记/新关闭：协调期间 _fd_lock 持有，其他线程的 open/close 阻塞
5.   生成不可变快照 tuple（含 generation，不含裸数字独用）
6. release _fd_lock
7. os.fork()
8. fork 返回后（parent）：恢复 fork_coordinating = False
```

**关键**：快照在**持 `_fd_lock`** 下生成，协调期间**其他线程的 open/close 被锁阻塞**，因此快照时刻到 fork 之间，FD 集合**不会变化**（所有变更都需先拿 `_fd_lock`）。这消除了 05.15 的"快照后 fork 前 FD 变化"竞态。

**死锁避免**：`_fd_lock` 是协调专用锁，fork 入口只获取这一把锁（不嵌套其他锁），且"禁止持锁 fork"规则（见 §1.5）保证 fork 入口本身不在其他锁内。见锁顺序图 §1.9。

### 1.4 协调期间禁止的操作

| 操作 | 协调期间 | 原因 |
|------|---------|------|
| 新 `open` 登记 FD | 禁止（阻塞）| 需拿 `_fd_lock` |
| `close(fd)` 注销 | 禁止（阻塞）| 需拿 `_fd_lock` |
| fork | 仅当前协调者 | 单一协调者 |
| 其他锁操作 | 允许（不涉及 _fd_lock）| 不冲突 |

**等待在途操作结束**：协调者持 `_fd_lock` 后，任何已在途的 open/close 要么已拿到锁完成、要么阻塞等待；协调者获取锁即意味着**没有在途的 open/close**（它们都需先拿同一把锁）。

### 1.5 fork 的四种来源保证边界

| 来源 | 保证 | 边界 |
|------|------|------|
| `fork_controlled`（本库封装）| 完整协调 + 快照 + 冻结 + child 重建 | 保证一致 |
| `os.fork()`（Python 直接）| `register_at_fork` 的 before/child 回调触发，但 before **不能持锁**（见 §1.6）| 无法做"持锁冻结"，只能做 best-effort 快照 + child 自我检测 |
| 第三方 C `fork()` | **不触发** Python atfork 回调 | **无法保证**，明确 DENY（见 §1.8）|
| 外部进程 fork | 不适用 | 不适用 |

### 1.6 before/parent/child 回调允许的操作（修订）

| 回调 | 调用线程 | 允许操作 | 禁止 |
|------|---------|---------|------|
| before | fork 触发线程 | **只读**已由 fork 入口（受控）或上次协调生成的快照 tuple | 获取任何锁；无锁枚举 FD Registry |
| parent | fork 触发线程 | 无操作（父状态不变）| 影响父进程状态 |
| child | 子进程唯一线程 | `os.close(fd)`（仅 generation 匹配的 ACTIVE FD）；直接赋值全新锁对象 | 获取继承锁；对共享 OFD 做 LOCK_UN；按裸数字误关 |

**关键区分**：
- **受控 fork**：before 读的是 §1.3 在持锁协调下生成的**一致性快照**（generation 绑定），安全。
- **直接 `os.fork()`**：before 读不到"持锁协调快照"（因为没走 fork_controlled），此时 child 回调**只做"锁对象重建"**，**不做 FD 关闭**（因为无法证明快照一致）→ 改为 fail-closed：子进程后续受保护操作 DENY，直到重新 `acquire_lock`。

### 1.7 10 条并发/异常时间线

| # | 场景 | 状态变化 | 结果 |
|---|------|---------|------|
| F1 | 受控 fork，无竞争 | 协调冻结→快照→fork→child 关闭匹配 FD→重建 | 安全一致 |
| F2 | fork 协调时其他线程 close(fd=7) | close 阻塞在 _fd_lock；协调完成后 close 生效，generation 失效 | 快照不受影响 |
| F3 | fork 协调后 fd=7 被复用 | 新 open 拿新 generation；child 按旧 generation 不匹配 → 不关闭 | 无误关 |
| F4 | child 误关裸数字 7 | generation 不匹配 → 跳过 | 无误关新资源 |
| F5 | 直接 os.fork()（非受控）| before 无一致性快照 → child 只重建锁、不关 FD → 后续 DENY | fail-closed |
| F6 | 第三方 C fork() | 不触发 atfork → child 继承全部 FD/锁 → **无法保证** | DENY（见 §1.8）|
| F7 | fork 失败（资源不足）| os.fork 抛异常，parent 恢复 fork_coordinating=False | 安全无残留 |
| F8 | child 回调异常（close 抛）| `_fork_reset_ok=False` → 后续锁操作 DENY | fail-closed |
| F9 | 连续 fork | 每次重新协调 + 快照 + 重置，幂等 | 安全 |
| F10 | 父子并发退出 | OS 释放各自 fd 引用，generation 各自独立 | 无父锁误释放 |

### 1.8 第三方 C fork 的明确拒绝策略

**结论（不回避）**：`os.register_at_fork` **只覆盖 Python `os.fork()`**，第三方 C 扩展直接调用 libc `fork()` **不触发** Python atfork 回调，因此本库**无法拦截或协调**。

**策略**：
1. **受控 fork**：本库 API 内完整保证。
2. **直接 `os.fork()`**：best-effort + child 自我检测 fail-closed。
3. **第三方 C fork**：**无法保证** → 本库在检测到"子进程存在未知继承 FD 或锁对象"时 **DENY** 受保护操作；**文档化调用方责任**：若项目依赖会直接 fork 的 C 扩展，必须在使用本库锁系统前显式说明，否则本库**不支持**。

**不声称**"Python 回调覆盖了第三方 C fork"。

### 1.9 锁顺序图（证明无循环等待）

```
全局锁顺序（编号越小越先获取，禁止逆序）：
  L0. fork 协调入口（无前置锁，只拿 _fd_lock）
  L1. _fd_lock（FD Registry）
  L2. _gate_lock（ForkGate）
  L3. _lock（LockHandleRegistry）
  L4. _lease_lock（Lease）
  L5. _pmutex_lock（ProcessMutex）
```

**规则**：
1. fork 协调入口**只拿 `_fd_lock`**（L1），不嵌套其他锁。
2. "禁止持锁 fork"：任何持有 L1~L5 中任意锁的线程不得调 fork；fork 入口先检查当前线程是否持锁。
3. 其余锁获取必须升序，禁止逆序。

**无循环等待证明**：任何锁获取路径都严格升序（或 fork 入口只拿单一 L1），不存在两条路径顺序相反，故无环、无死锁。

### 1.10 P0-02 验收对照与结论

- FD 完整生命周期（创建/登记/使用/关闭/注销/冻结/恢复/重建）：✅ §1.2/§1.3
- generation/version 绑定，非裸数字：✅ §1.2
- 可证明一致 fork 前协调（持锁冻结，废弃无锁枚举）：✅ §1.3
- 四来源保证边界：✅ §1.5
- before/parent/child 允许操作 + child 不获继承锁：✅ §1.6
- 10 条时间线：✅ §1.7
- 锁顺序图/状态机无循环等待：✅ §1.9
- 第三方 C fork 不声称被覆盖：✅ §1.8

**P0-02 结论：PASS（设计级，无锁枚举竞态已消除，generation 绑定防误关，第三方 C fork 边界明确 DENY）**。fork 协调的运行时行为 NOT_TESTED。

---

## 2. P0-04｜真正基线 CAS、锁对象身份和跨平台提交协议

### 2.1 缺陷定位：05.15 §1.4（Candidate 未绑父版本）+ §1.3（固定锁文件 inode 替换）

05.15 §1.4 的 Candidate 只写了 `new_rev`，**未绑定父版本**（base_revision/base_txn_id/base_hash）。缺陷：

- 提交时只断言 `new_rev == cur_rev + 1`，**不校验 Candidate 是基于哪个父版本生成的**；
- 旧 Candidate 可能被"只改 revision 数字"后重新提交，导致基于过期父版本的内容被当作新版本。

05.15 §1.3 的固定 `.commit-lock` 存在 **inode 替换风险**：
- T1 `open(.commit-lock)` 得旧 inode，`flock` 成功；
- 外部进程 `unlink(.commit-lock)` + `open(O_CREAT)` 得新 inode；
- T2 `open` 新 inode，`flock` 成功；
- **T1、T2 都持锁，双持**。

### 2.2 Candidate 父版本绑定（完整字段）

Candidate（PREPARED 记录）必须绑定：

| 字段 | 含义 |
|------|------|
| `base_revision` | 生成 Candidate 时读取的 current.revision |
| `base_txn_id` | 生成时 current.txn_id |
| `base_hash` | 生成时 current 指向版本文件的 SHA-256 |
| `new_revision` | = base_revision + 1 |
| `new_txn_id` | 本事务 txn_id |
| `candidate_hash` | Candidate 自身内容哈希 |

**提交前（持唯一互斥权后）必须同时验证全部**：
1. `current.revision == base_revision`
2. `current.txn_id == base_txn_id`
3. `current.hash == base_hash`
4. `new_revision == base_revision + 1`
5. Candidate 内容哈希 == PREPARED 记录的 `candidate_hash`

**任一不匹配 → 拒绝提交，且必须基于最新 current 重新生成 Candidate**（重新读取父版本、重新生成内容、重新哈希），**禁止只改 revision 数字后重试**。

### 2.3 锁对象身份稳定性（针对 inode 替换反例）

**必须回答的六个问题**：

1. **锁文件是否允许 unlink/rename/替换/硬链接？** 协议规定**不允许**（本库参与者约定）；但同 UID 恶意进程可无视。
2. **如何验证路径对象与已锁定 FD 是同一 inode？** `fstat(fd_lock)` 取 `(dev, ino)`，再 `stat(.commit-lock)` 取路径当前 `(dev, ino)`，比对；**且全程用 `openat(dirfd, ".commit-lock", O_NOFOLLOW)` 绑定目录句柄**。
3. **身份校验前后如何防 TOCTOU？** 校验用**已打开的 fd_lock**，后续 flock 也用**同一 fd_lock**，不重新按路径打开；路径只在 `openat` 时解析一次。
4. **锁文件被替换后如何 fail-closed？** 提交前（持锁后）重新 `fstat(fd_lock)` 比对身份 + `fstatat(dirfd, ".commit-lock", AT_SYMLINK_NOFOLLOW)` 比对路径身份；不一致 → **放弃提交 + DENY**。
5. **是否需要受保护目录/独立锁服务/数据库？** 见 §2.5 路径选择。
6. **恶意同 UID 进程是否在威胁模型内？** **不在**（明确边界）；本设计只对遵守协议的进程提供互斥，恶意同 UID 进程的锁替换标 NOT_VERIFIED。

### 2.4 锁文件身份稳定协议（完整时序）

```
锁定阶段：
1. open(dirfd = open(lock_dir, O_DIRECTORY))                  # 绑定可信目录
2. fd_lock = openat(dirfd, ".commit-lock", O_RDWR|O_CREAT|O_NOFOLLOW, 0600)
3. fstat(fd_lock) → 记录 (dev, ino, ctime)                    # 锁对象身份
4. flock(fd_lock, LOCK_EX)                                    # 排他锁

提交前校验（持锁后）：
5. fstat(fd_lock) → 再取 (dev, ino)                           # 确认 fd 仍指向同一对象
6. fstatat(dirfd, ".commit-lock", AT_SYMLINK_NOFOLLOW) → 取路径身份
7. 比对步骤 3/5/6 的 (dev, ino) 三者一致 → 才允许提交
   - 不一致 → 锁文件被替换/unlink → fail-closed，放弃 + DENY

提交（身份一致前提下）：
8. 读 current → base_revision/base_txn_id/base_hash
9. 验证 §2.2 五条件全部匹配
10. 写 current.<txn_id>，rename → current，fsync(dir)
11. 写 COMMIT 回执
12. flock(fd_lock, LOCK_UN)，close(fd_lock)
```

**关键**：`fd_lock` 全程不复用路径重新打开；身份校验（步骤 5/6/7）在**持锁后**进行，且步骤 8~10 用同一 `fd_lock` 串行化，消除"校验后替换"的 TOCTOU。

### 2.5 路径选择：受保护目录 vs 独立锁服务 vs 数据库

05.16 采用 **受保护目录 + 固定锁文件 + 身份校验**（§2.4），理由：
- 本工单 READ-ONLY，不能引入锁服务或数据库实现；
- 受保护目录（0700，仅本 UID）**阻止跨 UID 进程**替换锁文件；
- 同 UID 恶意进程不在威胁模型内（明确边界，§2.3 第 6 问）。

**替代契约（NOT_IMPLEMENTED）**：独立锁服务（DB 行锁）作为跨信任边界的彻底方案，供实现阶段选型；数据库事务 CAS 见 §2.8。

### 2.6 POSIX 提交协议

- **锁原语**：`flock(LOCK_EX)`（锁关联 OFD，fork 语义已在 §1 处理）；不用 fcntl 记录锁（记录锁按进程关联，fork 语义复杂）。
- **锁对象生命周期**：`.commit-lock` 常驻，不 unlink；进程退出 OS 自动释放 flock。
- **目录权限**：lock_dir 0700，仅本 UID 可写，阻止跨 UID 替换。
- **rename**：`rename(current.<txn> → current)` 原子替换指针。
- **fsync**：rename 后 `fsync(dirfd)` 持久化目录项。
- **崩溃恢复**：按 current 为准（current 是唯一真相），见 §2.9。

### 2.7 Windows 提交协议（独立设计，不宣称等价）

- **锁**：`CreateFileW(.commit-lock, ..., dwShareMode=0)` 独占打开（阻止其他进程再打开）+ `LockFileEx` 锁字节范围（offset 0, len 1）。
- **身份**：`GetFileInformationByHandle` → `dwVolumeSerialNumber + nFileIndexHigh/Low` 作为对象身份；`CreateFileW` 打开时 `FILE_FLAG_OPEN_REPARSE_POINT` 防符号链接。
- **发布**：`ReplaceFileW(current, current.<txn>, NULL, REPLACEFILE_WRITE_THROUGH, ...)` 原子替换。
- **持久化**：`FlushFileBuffers` 刷文件 + 目录句柄。
- **明确声明**：Windows 的 `LockFileEx`（锁字节范围）**不等价**于 POSIX flock（锁 OFD）；`ReplaceFileW` **不等价**于 POSIX rename。Windows 的独占打开 + 字节锁 + ReplaceFileW 组合的原子性、持久性 **NOT_VERIFIED**（未实测）。

### 2.8 SQLite 替代契约（NOT_IMPLEMENTED）

```
BEGIN IMMEDIATE;
SELECT revision, txn_id, hash FROM current WHERE id=1;
-- 校验 base 与 current 完全一致
UPDATE current SET txn_id=:new_txn, revision=:new_rev
       WHERE id=1 AND revision=:base_revision AND txn_id=:base_txn_id AND hash=:base_hash;
-- affected_rows==1 → 成功；==0 → 父版本已变，ROLLBACK 重生成 Candidate
COMMIT;
```

数据库的 `UPDATE ... WHERE` 条件 + affected_rows 是真正的 CAS（父版本绑定在 WHERE 子句）。

### 2.9 15 条并发/崩溃时间线

| # | 场景 | 推演 | 结果 |
|---|------|------|------|
| C1 | 双事务基于同一父版本 | 都持 flock 串行化，先获锁者提交 rev+1，后者校验 base 不匹配 → 重生成 | 不丢更新 |
| C2 | Candidate 过期（父版本已变）| 五条件校验 base_revision/txn_id/hash 失败 → 拒绝，重生成 | 不误提交 |
| C3 | 父版本内容同但 txn_id 不同 | base_txn_id 校验失败 → 拒绝 | 不误提交 |
| C4 | revision 被篡改 | new_revision != base+1 → 拒绝 | 拒绝 |
| C5 | 锁文件 unlink/recreate | 身份校验（步骤 5/6/7）不一致 → fail-closed DENY | 拒绝 |
| C6 | 锁路径被替换 | openat + O_NOFOLLOW + 身份比对 → 检测 → DENY | 拒绝 |
| C7 | 持锁进程崩溃 | OS 释放 flock，锁自动释放 | 安全 |
| C8 | rename 后 COMMIT 前崩溃 | current 已 rev+1，COMMIT 未写 → 按 current 确认 | current 为准 |
| C9 | fsync 失败 | rename 后 fsync(dir) 失败 → 标 NOT_DURABLE，不回滚 | 明确非持久 |
| C10 | Windows 发布失败 | ReplaceFileW 失败 → 放弃，锁释放 | 拒绝 |
| C11 | 重复 txn_id | versions/<txn_id> O_EXCL 创建失败 → 拒绝 | 不重复 |
| C12 | PREPARED 半写入 | prepared 半成品，恢复校验失败 → 忽略 | 无副作用 |
| C13 | Candidate 被篡改 | candidate_hash 校验失败 → 拒绝 | 拒绝 |
| C14 | 恢复时 current 与回执冲突 | current 是唯一真相，回执仅作辅助 → 按 current | 一致 |
| C15 | 恶意同 UID 绕过协议 | 锁替换/绕过 → 无法强制阻止 | **NOT_VERIFIED** |

### 2.10 P0-04 验收对照与结论

- Candidate 绑定 base_revision/base_txn_id/base_hash：✅ §2.2
- 旧 Candidate 不得重编号提交：✅ §2.2/§2.9 C2/C4
- 锁对象分裂不描述为已解决（除非强制机制）：✅ §2.3（明确边界，恶意同 UID NOT_VERIFIED）
- 15 条时间线：✅ §2.9
- POSIX/Windows 分开：✅ §2.6/§2.7

**P0-04 结论：PASS-WITH-ISSUES（设计级）**。
- 父版本绑定 + 五条件校验：可证明（§2.2/§2.9）。
- 锁对象身份稳定（受保护目录 + 身份比对）：对遵守协议进程可证明；恶意同 UID 锁替换 NOT_VERIFIED。
- Windows 原子原语 NOT_VERIFIED。

---

## 3. P0-05｜Sealed 强制不可变与 Source Retire 安全边界

### 3.1 缺陷定位：05.15 §3.4/§3.6 最终哈希校验与 rename 之间仍有竞态

05.15 的"seal 后、Source Retire 前重新内容校验"仍无法消除**最终校验与 rename 之间**的窗口：

- B 完成最终哈希校验（内容 = H）；
- A（持 seal 前打开的旧可写 FD）在 B 校验后、rename 前写入同一 inode；
- B 执行 rename，把**已被修改**的对象标记为 RETIRED。

**三个必须澄清的伪命题**（工单强制）：
- "哈希校验通过" ≠ "之后不能再写"；
- "rename 成功" ≠ "旧 FD 失效"；
- "SEALED 记录存在" ≠ "源对象不可变"。

### 3.2 强制不可变的唯一可靠机制

**结论**：本地文件系统能**强制阻止**旧 FD 写入的机制只有**文件系统不可变属性**（Linux `chattr +i`）——即使已 open 写 FD，写入返回 EPERM。其他机制均非强制：

| 机制 | 能否强制阻止旧 FD 写 | 说明 |
|------|-------------------|------|
| 协作锁（flock）| 否（仅协作）| 恶意进程无视 |
| 权限/ACL | 否（同 UID 可改回）| 仅跨 UID |
| Linux `chattr +i` | **是（EPERM）** | 需特权，非所有 FS 支持 |
| macOS `chflags uchg` | 否（不保证旧 FD）| 语义弱 |
| Windows 只读属性 | 否（非强制）| FILE_ATTRIBUTE_READONLY 非强制 |
| 只读快照/版本化副本 | 否（副本不可变，源仍可变）| 只证明副本内容 |

### 3.3 平台能力矩阵与允许/拒绝策略

| 平台 | 强制不可变原语 | 能否强制阻止旧 FD 写 | Source Retire 策略 |
|------|--------------|-------------------|-------------------|
| Linux（ext4/xfs/btrfs，特权）| `chattr +i` | 能（EPERM）| **允许**（需特权 + 验证 +i 生效）|
| Linux（无特权或非支持 FS）| 无 | 否 | **DENY** |
| macOS | `chflags uchg` | 不保证旧 FD | **DENY**（默认）|
| Windows | 无强制原语 | 否 | **DENY**（默认）|
| 网络/非支持 FS | 无 | 否 | **DENY** |

**核心规则（工单强制）**：
- 若平台无法提供强制不可变保证，**Source Retire 默认 DENY**。
- 若只允许协作进程，**明确威胁模型**，不得声称防御恶意同 UID 进程。
- Source Retire 前必须建立"最终校验后不再写入"的**强制边界**；无法建立则**不得执行**。

### 3.4 状态机与强制边界

```
WRITABLE → SEALING → SEALED → QUARANTINED → RETIRED
                          ↘ INDETERMINATE（任何证据不足）
```

**强制边界建立（Linux + 特权）**：
1. SEALING：`chattr +i` 施加不可变属性（此后任何写入包括旧 FD 都 EPERM）。
2. 验证：`lsattr` 确认 `i` 位已设。
3. SEALED：持久化 SEALED 记录（绑定 dev+ino+不可变标志+sealed 哈希）。
4. 最终校验：`sha256(src)` == sealed 哈希（此时因 +i，内容必然不再变）。
5. Source Retire：`chattr -i` 临时解除 → rename → 恢复/记录。

**关键**：+i 施加后，旧可写 FD 的写入被内核强制拒绝（EPERM），因此"最终校验"到"rename"之间**不存在写入窗口**——这是**强制边界**，不是协作约定。

**无特权 / 非支持平台**：无法施加 +i → **无法建立强制边界 → DENY Source Retire**，源对象保留，标记 INDETERMINATE。

### 3.5 12 条反例时间线

| # | 场景 | 推演 | 结果 |
|---|------|------|------|
| S1 | 旧可写 FD + 强制不可变 | chattr +i 后旧 FD 写 → EPERM | 强制阻止 |
| S2 | 旧可写 FD + 无强制原语 | 旧 FD 写成功 → 无法阻止 | DENY |
| S3 | 最终校验后 rename 前写入（有 +i）| +i 阻止写入 → 无窗口 | 安全 |
| S4 | 最终校验后 rename 前写入（无 +i）| 写入成功 → 竞态 | DENY |
| S5 | 硬链接写入 | 硬链接同 inode，+i 阻止；无 +i 检测 | +i 阻止 / 否则 DENY |
| S6 | mmap 映射写入 | 映射写绕过 write；+i 后写页 → EPERM（Linux）| +i 阻止 / 否则 DENY |
| S7 | 追加写入 | +i 阻止；否则检测 | +i 阻止 / DENY |
| S8 | 外部进程写入 | +i 阻止（含 root，需先 -i）| 强制 |
| S9 | 文件替换（unlink+recreate）| 身份（ino）变化检测 | 检测拒绝 |
| S10 | 同 UID 恶意进程 | 无法强制（同 UID 可 chattr -i 若其有特权）| NOT_VERIFIED |
| S11 | root/管理员 | +i 也阻止 root 写（除非先 -i）；但 root 可 -i | 需特权边界 |
| S12 | 网络 FS / 不支持 FS | 无 +i 语义 | DENY |

### 3.6 P0-05 验收对照与结论

- 消除"最终哈希校验后、rename 前仍可写"竞态：✅ §3.4（+i 强制边界）
- 旧可写 FD / 硬链接 / mmap / append / 外部 / 替换 / 同 UID / root / 网络 FS：✅ §3.5
- 七种机制各自保证/不保证：✅ §3.2
- 平台能力矩阵 + 允许/拒绝：✅ §3.3
- 无强制不可变平台默认 DENY：✅ §3.3/§3.4
- 12 条时间线：✅ §3.5

**P0-05 结论：PASS-WITH-ISSUES（设计级）**。
- Linux（特权 + 支持 FS）：`chattr +i` 提供强制不可变，可证明消除竞态（§3.4/§3.5 S1/S3）。
- macOS/Windows/无特权/网络 FS：**默认 DENY Source Retire**（无法建立强制边界）。
- 同 UID 恶意进程（若其有 chattr 特权）可 -i 撤销，NOT_VERIFIED。
- chattr +i 的 EPERM 语义、lsattr 验证 NOT_TESTED。

---

## 4. P0-06｜Tombstone 原子发布、归档和删除归因重构

### 4.1 缺陷定位：05.15 §4.4（O_EXCL 创建后逐步写入当原子发布）+ §4.9（归档混用）

05.15 §4.4 用 `O_EXCL` 创建最终 tombstone 元数据文件后**逐步写入**，再称为"原子发布"——**这是错的**：

- `O_EXCL` 只保证"创建时目标不存在"，**不保证内容原子可见**；
- 逐步写入期间崩溃，会暴露**半成品元数据**；
- 其他恢复进程可能读到未写完的元数据。

05.15 §4.9 归档协议写"rename 同样走 O_EXCL"——**概念混用**：`O_EXCL` 是 open 创建语义，`rename` 无覆盖是另一原语，两者不可混为一谈。

### 4.2 元数据原子发布协议（临时文件 + 校验 + 无覆盖发布）

```
1. 写临时文件 tombstone/.tmp.<cleanup_id>（完整元数据，fsync）
2. 重读临时文件，校验完整内容 + SHA-256（与写入前计算的哈希比对）
3. 无覆盖原子发布：
   - Linux: renameat2(AT_FDCWD, tombstone/.tmp.<cleanup_id>,
                      tombstone/<cleanup_id>, RENAME_NOREPLACE)
   - 其他 POSIX: 若 renameat2 不可用 → 先 O_EXCL 占位探测再...（见 §4.3）
   - Windows: 见 §4.4
4. fsync(tombstone_dir) 持久化目录项
5. 崩溃恢复：.tmp 半成品 → 校验失败 → 删除；已发布 <cleanup_id> → 校验 → 有效
```

**核心**：**不再 O_EXCL 创建最终文件后逐步写入**。最终 tombstone 元数据只在**临时文件完整写入 + 校验**后，用**无覆盖发布原语**一次性原子发布。

### 4.3 POSIX 无覆盖发布的平台差异

| 平台 | 原语 | 限制 |
|------|------|------|
| Linux | `renameat2(..., RENAME_NOREPLACE)` | 内核 ≥3.15 |
| macOS/BSD | 无 renameat2 NOREPLACE → 用 `link(tmp, target)` + `unlink(tmp)`：`link` 对已存在 target 原子失败（EEXIST）| link 硬链接方案（注意：此处 link 用于**发布已有文件**，不同于 05.14 废弃的"每事务独立锁名"假 CAS）|
| 其他 | 若无 link/renameat2 → DENY | — |

**说明**：macOS/BSD 用 `link(tmp, target)` 做无覆盖发布是**正确的**——因为 `link` 对已存在的 target 返回 EEXIST，且 tmp 已完整写入+校验，link 成功后 target 与 tmp 是同一 inode（内容原子可见）。这**不同于** 05.14 用 `link(current, current.lock.<txn_id>)` 做提交互斥——那里是"每事务独立目标名"导致无互斥；这里是"固定目标名 + EEXIST"实现无覆盖，语义正确。

### 4.4 Windows 元数据原子发布

- 写临时文件 `.tmp` → `FlushFileBuffers`；
- 校验重读 + 哈希；
- 无覆盖发布：`MoveFileExW(tmp, target, MOVEFILE_WRITE_THROUGH)` **但 Windows 的 MoveFileEx 会覆盖已存在目标**，故需先用 `CreateFileW(target, CREATE_NEW)` 占位（若已存在则失败 = 无覆盖）再 `ReplaceFileW`；
- `FlushFileBuffers` 目录句柄。

**明确声明**：Windows 无 `renameat2 NOREPLACE` 等价原语；`CREATE_NEW` 占位 + `ReplaceFileW` 组合的原子性与持久性 **NOT_VERIFIED**（未实测）。

### 4.5 删除语义：严格区分五个概念

| 概念 | 含义 | 是否等同"销毁" |
|------|------|--------------|
| 从原路径移除 | rename 源到隔离名 | 否 |
| 移入隔离目录 | 源改名到隔离路径 | 否 |
| 删除隔离副本 | unlink 隔离对象 | 否（数据可能仍可恢复）|
| 数据真正销毁 | 覆写/安全擦除 | 是（但本工单不实现）|
| Tombstone 元数据保留 | 保留删除事务记录 | 否（是元数据）|

**铁律**："源路径不存在"或"rename 成功"**不得**直接称为"数据已销毁"。数据销毁需安全擦除（本工单 READ-ONLY 不实现，标记 NOT_IMPLEMENTED）。

### 4.6 状态重定义

| 状态 | 对象 | 含义 | 持久化证据 |
|------|------|------|-----------|
| DELETE_INTENT | 删除意图 | 记录将删除源 | intent + fsync |
| TOMBSTONED | 源对象 | 源已 rename 到隔离名 | rename + tombstone 元数据发布 + fsync(dir) |
| REMOVED | 归因 | 删除归因成立 | REMOVED + fsync |
| CLEANED | 终态 | 清理确认 | CLEANED + fsync |
| PURGED | 数据 | 数据已安全销毁 | 安全擦除回执（本工单 NOT_IMPLEMENTED）|
| INDETERMINATE | 任意 | 无法证明 | 证据不足 |

### 4.7 归档协议（修正 05.15 §4.9）

1. **归档前提**：仅当 CLEANED（删除归因成立）。
2. **归档临时文件**：`archive/.tmp.<cleanup_id>` 完整写入元数据副本 + fsync。
3. **校验**：重读 + SHA-256 比对。
4. **无覆盖发布**：用 §4.2/§4.3 的原子无覆盖原语（renameat2 NOREPLACE / link EEXIST / Windows CREATE_NEW+ReplaceFileW）。
5. **fsync 归档目录**。
6. **不得在归档成功前销毁原始 tombstone 元数据**：归档是**副本**，原元数据在归档发布成功 + fsync 后才可迁移/保留。
7. **归档失败或证据不一致** → 保留现场 + INDETERMINATE。

**明确区分**：归档用**无覆盖发布原语**（renameat2/link），**不是 O_EXCL 创建后逐步写**。

### 4.8 15 条并发/崩溃时间线

| # | 场景 | 推演 | 结果 |
|---|------|------|------|
| T1 | 元数据临时文件写一半崩溃 | .tmp 半成品 → 校验失败 → 删除 | 无半成品暴露 |
| T2 | 元数据 fsync 失败 | 校验失败 → 不发布 | 拒绝 |
| T3 | 无覆盖发布竞争 | 两事务同名 target → 一个 renameat2 NOREPLACE 成功，另一个 EEXIST | 一个成功 |
| T4 | 同名目标已存在 | NOREPLACE 失败 → 换 ID 重试 | 不覆盖 |
| T5 | rename 后目录 fsync 失败 | 标记 NOT_DURABLE，不回滚 | 明确非持久 |
| T6 | 归档中断 | 归档 .tmp 半成品 → 校验删除 | 无破坏 |
| T7 | 归档目标冲突 | 无覆盖发布失败 → 换 ID | 不覆盖 |
| T8 | 归档后回执丢失 | 归档对象存在 + 哈希匹配 → 可确认 | 哈希一致确认 |
| T9 | 源路径同名重建 | rename 后外部 recreate 同名源 → 源路径又存在 → INDETERMINATE | INDETERMINATE |
| T10 | 隔离对象被替换 | 隔离对象身份（ino）变化 → 检测 | INDETERMINATE |
| T11 | Tombstone 元数据被替换/删除 | 身份/哈希不匹配或消失 → INDETERMINATE | INDETERMINATE |
| T12 | 并发清理同一源 | 源 rename 竞争，一个成功一个失败 | 一个成功 |
| T13 | 审计写入失败 | REMOVED 写失败 → INDETERMINATE | INDETERMINATE |
| T14 | Windows 发布失败 | CREATE_NEW+ReplaceFileW 失败 → 放弃 | 拒绝 |
| T15 | 删除归因证据不足 | 无 tombstone 元数据 / 不匹配 → INDETERMINATE | INDETERMINATE |

### 4.9 P0-06 验收对照与结论

- 元数据原子发布（临时文件+校验+无覆盖原语，废弃 O_EXCL 逐步写）：✅ §4.2
- 归档协议区分 O_EXCL 与 rename，不混用：✅ §4.7
- 归档前不销毁证据，失败 INDETERMINATE：✅ §4.7
- 五概念区分，"移除"≠"销毁"：✅ §4.5
- 六状态重定义（含 PURGED）：✅ §4.6
- 15 条时间线：✅ §4.8

**P0-06 结论：PASS（设计级，元数据原子发布协议已重构，归档不再混用 O_EXCL 与 rename）**。Windows 无覆盖发布、rename/fsync 崩溃语义 NOT_TESTED/NOT_VERIFIED。

---

## 5. 设计不变量（I-01 ~ I-30）

| # | 定义 | 证明位置 | 失效反例 | 拒绝策略 |
|---|------|---------|---------|---------|
| I-01 | FD 登记必须绑定 generation/version | §1.2 | 裸数字复用 | 不关闭不匹配 FD |
| I-02 | fork 前协调必须持锁冻结 | §1.3 | 无锁枚举竞态 | 废弃无锁快照 |
| I-03 | child 不得误关已复用 FD | §1.2/§1.7 F4 | 裸数字误关 | generation 校验 |
| I-04 | child 不得获取继承锁 | §1.6 | 继承 locked mutex | 直接赋值全新对象 |
| I-05 | child 清理不 LOCK_UN 父 OFD | §1.6 | 误 LOCK_UN | 仅 close |
| I-06 | 禁止持锁 fork | §1.9 | 持锁调 fork | 入口拒绝 |
| I-07 | 锁获取必须升序，无循环等待 | §1.9 | 逆序获取 | 禁止 |
| I-08 | 第三方 C fork 不声称被覆盖 | §1.8 | C 直接 fork | DENY |
| I-09 | Candidate 必须绑定父版本 | §2.2 | 未绑 base | 五条件校验 |
| I-10 | 旧 Candidate 不得重编号提交 | §2.2/§2.9 C2 | 只改 revision | 重生成 |
| I-11 | 提交竞争用单一固定锁对象 | §2.3 | 每事务独立锁名 | 禁止 |
| I-12 | 锁对象身份必须校验一致 | §2.4 | inode 替换 | 身份不符 DENY |
| I-13 | 身份校验后不复用路径重开 | §2.4 | TOCTOU | 全程用 fd_lock |
| I-14 | revision 检查与发布同临界区 | §2.4 | 检查后释放再发布 | 禁止 |
| I-15 | revision 严格单调 | §2.9 | rev 回退 | 断言拒绝 |
| I-16 | PREPARED 先于提交竞争持久化 | §2.4 | 未 PREPARED 就 flock | 拒绝 |
| I-17 | 恢复不认领未确认事务 | §2.9 C8/C14 | 未 COMMIT | 按 current |
| I-18 | 恶意同 UID 锁替换不在威胁模型内 | §2.3 | 同 UID 替换 | 明确边界 NOT_VERIFIED |
| I-19 | 源对象强制不可变需平台原语 | §3.2/§3.3 | 无 +i 平台 | DENY |
| I-20 | 最终校验后 rename 前不得写入 | §3.4 | 旧 FD 写 | +i 强制阻止 |
| I-21 | 哈希通过 ≠ 之后不再写 | §3.1 | 校验后写 | 不依赖哈希当不可变 |
| I-22 | rename 成功 ≠ 旧 FD 失效 | §3.1 | 旧 FD 写 | 明确伪命题 |
| I-23 | SEALED 记录 ≠ 对象不可变 | §3.1 | 记录存在但可变 | 需 +i 证据 |
| I-24 | 无强制不可变平台 Source Retire 默认 DENY | §3.3 | 强制 DENY 缺失 | DENY |
| I-25 | quarantine 只操作副本，不改源 | §3（继承）| 直接改源 | 禁止 |
| I-26 | 源对象与 tombstone 元数据分离 | §4（继承）| 混为同对象 | 禁止 |
| I-27 | 元数据必须临时文件+校验+无覆盖发布 | §4.2 | O_EXCL 逐步写 | 废弃 |
| I-28 | 归档不得混用 O_EXCL 与 rename | §4.7 | 概念混用 | 用无覆盖原语 |
| I-29 | "移除"≠"销毁"，删除归因需持久证据 | §4.5/§4.6 | 仅源消失 | INDETERMINATE |
| I-30 | 归档前不得销毁原始证据，失败 INDETERMINATE | §4.7 | 提前销毁 | 禁止 |

**共 30 条不变量，每条有证明位置、失效反例、拒绝策略。**

---

## 6. 端到端交叉审查

### 6.1 12 条端到端链路

| # | 链路 | 前置 | 关键持久化边界 | 竞争点 | 崩溃点 | 恢复 | 结论 |
|---|------|------|--------------|--------|--------|------|------|
| E1 | 锁→fork→父子退出 | 有效锁 | flock | fork | fork 中 | child 重建 | PASS |
| E2 | 锁身份→Candidate(绑父版本)→PREPARED→竞争→发布 | 有效锁 | PREPARED 先于 flock | .commit-lock | rename 前后 | 按 current | PASS |
| E3 | 双事务同一父版本竞争 | 并发 | — | .commit-lock | 竞争点 | 败者重生成 | PASS |
| E4 | FD 生命周期→fork 协调→child 重建 | 协调 | 冻结快照 | _fd_lock | 协调中 | generation 校验 | PASS |
| E5 | 源 seal(+i)→quarantine→retire | +i 特权 | SEALED 记录 | seal 点 | seal 后写 | +i 强制阻止 | PASS-WITH-ISSUES |
| E6 | 旧可写 FD + 强制不可变 | +i | +i 生效 | 旧 FD | 校验后 | EPERM 阻止 | PASS（Linux）|
| E7 | 旧可写 FD + 无强制原语 | 无 +i | 无 | 旧 FD | 校验后写 | DENY | PASS（默认 DENY）|
| E8 | tombstone 元数据原子发布 | intent | 临时+校验+NOREPLACE | 同名 target | 发布前后 | 校验删除/有效 | PASS |
| E9 | rename 后崩溃（REMOVED 未持久化）| intent | rename→REMOVED | — | rename 后 | 元数据匹配/INDETERMINATE | PASS |
| E10 | 归档协议（不混用 O_EXCL/rename）| CLEANED | 临时+校验+发布 | 归档 target | 归档中 | 哈希一致确认 | PASS |
| E11 | 审计不可用/写失败 | 删除中 | REMOVED 失败 | — | 审计写 | INDETERMINATE | PASS |
| E12 | 恶意同 UID 进程绕过协议 | 恶意 | 无 | 任意 | 任意 | 身份检测拒绝 | **NOT_VERIFIED** |

### 6.2 20 条并发/崩溃/异常/外部干扰时间线（汇总）

- P0-02：§1.7 F1~F10（10 条）
- P0-04：§2.9 C1~C15（15 条）
- P0-05：§3.5 S1~S12（12 条）
- P0-06：§4.8 T1~T15（15 条）

**合计 52 条时间线，超过工单要求的 20 条。** 各条均含状态变化、原子原语、持久化边界、恢复动作。

### 6.3 跨章节身份一致性检查

| 身份 | 产生章节 | 传递章节 | 一致性保证 |
|------|---------|---------|-----------|
| 锁对象身份（dev+ino）| §2.4 | §2（提交）| 全程 fd_lock + 身份比对 |
| 事务父版本（base_rev/txn/hash）| §2.2 | §2.4/§2.9 | 五条件校验 |
| FD generation | §1.2 | §1.3/§1.7 | generation 绑定防误关 |
| 源对象身份（dev+ino+不可变标志）| §3.4 | §3.5 | +i 强制 + fstat |
| Tombstone 身份 | §4.2 | §4.8 | 无覆盖发布 + 身份/哈希 |

### 6.4 主动寻找的反例（非复述作者证明）

| 反例 | 是否已被设计覆盖 | 备注 |
|------|----------------|------|
| 裸 FD 数字复用误关 | 覆盖（generation）| §1.2/§1.7 F4 |
| 第三方 C fork 绕过 atfork | 覆盖（DENY）| §1.8 |
| 无锁枚举竞态 | 覆盖（持锁冻结）| §1.3 |
| 旧 Candidate 重编号提交 | 覆盖（父版本绑定）| §2.2 |
| 锁文件 unlink+recreate 双持 | 覆盖（身份校验 + 边界）| §2.3 |
| 最终校验后旧 FD 写 | 覆盖（+i 强制 / DENY）| §3.4 |
| O_EXCL 逐步写暴露半成品 | 覆盖（临时+校验+无覆盖）| §4.2 |
| 归档混用 O_EXCL/rename | 覆盖（区分原语）| §4.7 |

**未发现新的、设计未覆盖的反例。**

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

**DESIGN-05.16 = PASS-WITH-ISSUES（设计级）**。

| # | 结论 | 核心依据 | 未验证项 |
|---|------|---------|---------|
| P0-02 | PASS（设计级）| §1.7 十条，generation 绑定 + 持锁冻结消除无锁枚举竞态 | fork 协调运行时 NOT_TESTED |
| P0-04 | PASS-WITH-ISSUES | §2.9 十五条，父版本绑定 + 锁身份校验 | 恶意同 UID 锁替换、Windows 原子原语 NOT_VERIFIED |
| P0-05 | PASS-WITH-ISSUES | §3.5 十二条，+i 强制不可变（Linux）；无强制平台 DENY | macOS/Windows 强制语义 NOT_VERIFIED |
| P0-06 | PASS（设计级）| §4.8 十五条，临时+校验+无覆盖发布，归档区分原语 | Windows 无覆盖发布 NOT_VERIFIED |

**诚实边界**：
1. 恶意同 UID 进程（锁替换、旧 FD 强写、chattr -i 撤销）是本地 FS 固有能力边界，NOT_VERIFIED。
2. Source Retire 在无强制不可变平台（macOS/Windows/无特权/网络 FS）默认 DENY。
3. 所有 NOT_TESTED/NOT_VERIFIED 项保留，未降低标准、未删失败分支。

**IMPLEMENT-01 继续 STOP，等待 ChatGPT 独立复审。**

---

## 9. 交付物索引

| 文件 | 说明 |
|------|------|
| `USIT-STAGE-B-DESIGN-05.16.md` | 本主设计文档 |
| `USIT-STAGE-B-DESIGN-05.16-ISSUES.md` | 四项问题清单及剩余风险 |
| `USIT-STAGE-B-DESIGN-05.16-DIFF.md` | 相对 05.15 逐项差异 |
| `USIT-STAGE-B-DESIGN-05.16-CROSS-REVIEW.md` | 反例审查记录 |
| `USIT-STAGE-B-DESIGN-05.16-EXECUTION-REPORT.md` | 实际执行报告 |
| `USIT-STAGE-B-DESIGN-05.16-HASH-MANIFEST.md` | 交付文件 SHA256 清单 |
| `USIT-STAGE-B-DESIGN-05.16-HASH-RECEIPT.txt` | 哈希验证回执 |
| `USIT-STAGE-B-DESIGN-05.16-交付物.zip` | 完整打包 |

---


