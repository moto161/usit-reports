# USIT-STAGE-B-DESIGN-05.14｜五项 P0 缺陷重新设计与协议闭合

> 阶段 B · DESIGN-05.14 · READ-ONLY DESIGN REVISION ONLY
> 作者：USIT 主开发 · 生成：2026-10-31（GMT+8）
> 依据：DESIGN-05.13 独立审查结论（五项未闭合）
> 工作区：`C:\Users\Administrator\WorkBuddy\2026-10-15-22-09-47\usit_p0`
> 交付目录：`C:\Users\Administrator\WorkBuddy\2026-10-15-22-09-47\usit_p0_05.14_reports`（**项目外**）
> 状态：**STOP — 等待独立复审**（未授权 IMPLEMENT-01）

---

## 0. 05.13 缺陷定位表

先建立缺陷定位表，逐项记录：原章节、原设计、缺陷原因、05.14 修订目标、验收证据。

| # | 原章节 | 原设计 | 缺陷原因 | 05.14 修订目标 |
|---|--------|--------|---------|---------------|
| P0-02 | §1.6 | 以 `_fork_fd_snapshot` 是否为空识别外部 fork | `os.register_at_fork` 回调同样被外部 `os.fork()` 触发，`_fork_fd_snapshot` 非空不能证明"受控 fork" | 外部 fork 不可靠识别 → 改为"回调无条件安全重置 + 外部 fork 后状态自检 DENY" |
| P0-03 | §2.3/§2.6 | 永久锁文件 + 禁 unlink 对抗同用户替换 | 同 UID 恶意进程有相同权限能力，协议约定无法阻止故意绕过；`fstat(fd)` 只证明 FD 指向对象，不能证明路径仍指向同一对象 | 承认本地 FS 无法防御同 UID 恶意进程 → 引入跨信任边界锁服务或明确边界 |
| P0-04 | §3.7 | "目标存在时无覆盖发布失败"当 CAS | 无覆盖原语只能"目标不存在时创建"，不能更新已存在的正式文档；"读 revision→比较→写入"非原子 | 不可变版本文件 + 原子指针发布，或数据库事务 |
| P0-05 | §4.3/§4.6 | 双哈希校验 + rename 前再校验 | 最后一次 sha256 与 rename 之间有窗口；rename 不能阻止已持可写 FD 的进程继续写 | 排他打开/文件锁，或"复制后只操作副本，源永不改" |
| P0-06 | §5.7 | tombstone 不 unlink 永久保留 | tombstone 永久保留的唯一性、事务绑定、不可替换边界、磁盘增长未定义 | tombstone 唯一命名 + 事务绑定 + 归档协议 |

**本节是全文的总纲，后续每章回到此表逐项给出修订证据。**

---

## 1. P0-02｜Fork 生命周期与同步对象安全

### 1.1 缺陷定位（05.13 §1.6）

05.13 §1.6 用 `_fork_fd_snapshot` 是否为空来"识别"外部 fork：

> "外部代码直接 os.fork() → 检测：child 回调 `_after_fork_child` 会因 `_fork_fd_snapshot` 为空（prepare 未执行）而不关闭任何 fd"

**这个检测是无效的**：`os.register_at_fork` 注册的 `before` 回调**同样会被外部 `os.fork()` 触发**——因为 `register_at_fork` 是进程级的，任何 `os.fork()`（无论是否经过本库的 `fork_controlled` 封装）都会先执行 `before`（快照 `_fork_fd_snapshot`）再执行 `after_in_child`。因此外部 fork 后，`_fork_fd_snapshot` 同样非空，检测失效。

### 1.2 正确认识：外部 fork 无法通过快照内容区分

**结论（不再回避）**：在 Python 层面，**无法可靠区分**"经 `fork_controlled` 的受控 fork"与"外部直接 `os.fork()`"——两者都触发 `register_at_fork` 的回调链。

因此 05.14 的策略从"识别外部 fork 并拒绝"改为：

**"无论何种 fork，child 回调都执行无条件安全重置；外部 fork 的代价由'子进程后续操作的自我检测'承担，而非依赖一个不可靠的识别标志。"**

### 1.3 before / parent / child 三回调完整契约

| 回调 | 执行时机 | 职责 | 异常处理 | 资源生命周期 |
|------|---------|------|---------|-------------|
| `before` | fork 前，父进程多线程，在 fork 系统调用前 | 用锁安全地快照 FD 列表为不可变 tuple；快照其他需重置对象的引用 | 异常 → 取消 fork，抛回调用方 | 快照 tuple 只读，fork 后由 child 消费 |
| `after_in_parent` | fork 后，父进程 | 无操作（父进程状态不变）| 异常 → 记录审计，不影响父进程 | 无 |
| `after_in_child` | fork 后，子进程单线程 | 读 before 快照的 tuple → close fd（不 LOCK_UN）→ 重建全部锁对象 | 异常 → 设置 `_fork_reset_failed=True`，子进程后续锁操作 DENY | 重建的锁对象是子进程私有 |

### 1.4 before 回调获取 Registry 锁不会死锁的证明

`before` 回调在 fork 系统调用**之前**、父进程**仍多线程**时执行，此时调用 `LockHandleRegistry.snapshot_fds()`（内部 `with self._lock`）是**安全的**，因为：

1. 父进程此时是正常的、可调度多线程环境；
2. 若 `self._lock` 被其他线程持有，`before` 只是**正常阻塞等待**（就像任何普通临界区一样），等待持有者释放后获取——这不是死锁，因为持有者在正常线程上下文中会释放；
3. 死锁的定义是"循环等待"，而 `before` 只获取 Registry 这一把锁，不存在锁顺序反转。

**唯一风险**：若 `before` 回调在持锁状态下被 fork 系统调用中断——但这不可能，因为 `before` 是 `register_at_fork` 的 before 回调，它在 `os.fork()` 内部**先完整执行完 before、再执行 fork**。before 执行期间不会发生 fork。

### 1.5 所有同步对象在子进程的处理策略（完整清单）

| 对象 | fork 后子进程状态 | 处理策略 | 是否安全 |
|------|------------------|---------|---------|
| `ForkGate._fork_in_progress` | 继承（可能 locked）| 重建为全新 Lock | 安全（不 acquire 继承锁）|
| `ForkGate._counter_lock` | 继承（可能 locked）| 重建 | 安全 |
| `LockHandleRegistry._lock` | 继承（可能 locked）| 重建为全新 Lock | 安全 |
| `LockHandleRegistry._handles` | 继承（有数据）| 清空为全新 dict | 安全 |
| `ProcessMutex._locks` | 继承（可能含 locked）| 清空 | 安全 |
| `LeaseRegistry._leases` | 继承（有数据）| 清空 | 安全 |
| 任意 Condition/Event | 继承（可能 locked/set）| 重建 | 安全（若无法重建 → DENY）|

**核心规则**：child 回调中**只做两件事**——`os.close(fd)`（读 before 快照的 tuple，不碰锁）+ **直接赋值全新对象**（`= threading.Lock()` / `= {}`，不调用任何 `with` / `acquire`）。

### 1.6 10 个 fork 故障场景逐步推演

| # | 场景 | 推演 | 结果 |
|---|------|------|------|
| F1 | 受控 fork，无其他线程持锁 | before 快照 → fork → child 读 tuple + close + 重建 | 安全 |
| F2 | fork 时其他线程持 ProcessMutex | before 快照（不碰 ProcessMutex 锁）→ fork → child 清空 ProcessMutex._locks（赋新 dict，不 acquire）| 安全（不继承 locked）|
| F3 | fork 时其他线程持 Registry._lock | before 的 snapshot_fds 会阻塞等 Registry._lock → 持有者释放 → 快照完成 → fork | 安全（正常阻塞，非死锁）|
| F4 | 外部 os.fork() | register_at_fork 仍触发 before（快照）+ child（重置）→ 子进程锁系统已重置 → 但子进程**无有效 lease**，后续 acquire 需重新走完整流程 | 安全（自我检测兜底，非靠识别标志）|
| F5 | child 回调异常（close 抛 OSError）| 捕获 OSError，继续；其他异常 → `_fork_reset_failed=True` | DENY（后续锁操作拒绝）|
| F6 | fork 失败（资源不足）| os.fork 抛异常，before 已执行但无副作用（快照是临时 tuple）| 安全，无残留 |
| F7 | fork 后 exec | child 回调已 close 锁 fd（若未设 close-on-exec 则已显式 close）；exec 后新程序无继承锁 | 安全 |
| F8 | 连续 fork | 每次 fork 都重新 before 快照 + child 重置，幂等 | 安全 |
| F9 | 子进程先退出 | OS 释放子进程 fd 引用，不影响父进程 OFD 锁 | 安全 |
| F10 | 子进程重新 acquire | child 已重置锁系统，acquire 走完整流程，独立 lease | 安全 |

**分类**：F1~F3、F6~F10 可保证；F4 靠自我检测兜底（不靠识别标志）；F5 需 DENY。**没有"必须靠不可靠标志才能处理"的场景**——这是 05.14 相对 05.13 的关键改进。

### 1.7 外部 fork 的拒绝/隔离策略（可执行，非声称自动恢复）

**策略**：不声称能"识别并自动恢复"外部 fork。而是：

1. `register_at_fork` 的 `after_in_child` 无条件执行安全重置（对任何 fork 都生效）。
2. 外部 fork 的子进程，其锁系统已被重置为空/全新，**不存在继承的有效 lease**。
3. 子进程若尝试使用父进程的锁对象（通过某种继承的引用），会因 registry 已清空、锁对象已重建而 `assert_lock_owned` 失败 → DENY。
4. 若子进程要正常操作，必须重新 `acquire_lock`，这是完整、安全的路径。

**不可恢复的情况**：如果外部代码 fork 后，子进程**绕过本库**直接使用继承的裸 fd 或锁对象（不经过 acquire_lock），本库无法拦截——但这属于"调用方不遵守 API"，本库明确**不支持**，文档化调用方责任。

### 1.8 验收对照（以推演证据为准）

- 10 个 fork 故障场景逐项推演：✅ §1.6
- 外部 fork 不再依赖失效的识别标志：✅ §1.2
- before 无死锁证明：✅ §1.4
- 同步对象完整清单：✅ §1.5
- 可执行拒绝策略（非声称自动恢复）：✅ §1.7

**P0-02 结论：PASS（设计级，识别失效缺陷已修正）**，fork/register_at_fork 行为 NOT_TESTED。

---

## 2. P0-03｜锁对象稳定性与跨进程互斥

### 2.1 缺陷定位（05.13 §2.3/§2.6）

05.13 的锁文件设计依赖两点假设：
1. "锁文件是**永久**的、参与者**约定不 unlink**"；
2. `fstat(fd)` 拿到的 inode 身份能证明"路径仍指向同一对象"。

**两点都不成立**：
- **假设 1 失效**：同 UID 的另一个进程拥有与该进程**完全相同的文件系统权限能力**（同一用户、同一 ACL）。"约定不 unlink" 是协议约定，**对遵守协议的进程有效，对恶意同 UID 进程无效**——恶意进程可以无视约定，直接 `unlink(lockfile)` 再 `open(O_CREAT)` 重建同名文件，得到一个**新 inode**。
- **假设 2 失效**：`fstat(fd)` 只能证明"这个 fd 当前指向某个 inode"，**不能证明路径字符串现在仍指向同一个 inode**。fd 是打开时绑定的快照；路径是每次重新解析的。

### 2.2 inode-A / inode-B 分裂时间线（最小反例）

两进程 A、B，锁文件路径 `/var/run/usit/lock`：

| 时刻 | 进程 A | 进程 B（恶意同 UID）|
|------|--------|---------------------|
| t0 | `open(/var/run/usit/lock)` → 得 fd-A → inode-A | — |
| t1 | `flock(fd-A, LOCK_EX)` → **持锁 inode-A** | — |
| t2 | 进入临界区 | `unlink(/var/run/usit/lock)` |
| t3 | （仍在临界区）| `open(/var/run/usit/lock, O_CREAT)` → fd-B → **inode-B** |
| t4 | — | `flock(fd-B, LOCK_EX)` → **持锁 inode-B** |
| t5 | **A 在临界区** | **B 也在临界区** |

**结果：A、B 同时持有"同一路径"的锁，却锁的是不同 inode，双方同时进入受保护临界区。**

`fstat(fd-A)` 在任意时刻都返回 inode-A 的 stat，但这**无法阻止** t2 的 unlink 与 t3 的重建。协议约定"参与者不 unlink"对 B 无约束力。

### 2.3 本地文件系统能证明什么、不能证明什么

**能证明（对遵守协议的参与者）**：
- 若所有参与者都通过本库 API 打开锁文件，且都不 unlink，则 `fstat(fd)` + `flock(fd)` 保证它们锁的是**同一个 inode**——因为没人删除重建，路径始终解析到同一 inode。
- 在此前提下，flock 的 OFD 语义给出**真实的互斥**：同一 inode 上的 `LOCK_EX` 排他。

**不能证明（对恶意同 UID 进程）**：
- 本地 POSIX/Windows 文件系统**没有机制**能让"同 UID 进程"之间建立无法绕过的强制边界——同 UID 拥有相同权限能力，任何基于文件身份 + 协议约定的方案都能被故意绕过。
- 这不是 USIT 设计的缺陷，而是**本地文件系统锁模型的固有能力边界**：`flock`/`LockFile` 是**协作锁（advisory）**，不是强制锁（mandatory）。

### 2.4 05.14 的诚实结论与两条可选路径

**结论（不再假装能防）**：本地文件系统锁**无法防御同 UID 恶意进程**。要获得"所有参与者锁同一稳定对象"的强保证，只有两条路径：

**路径一（本工单不实现，仅给契约）**：引入**跨信任边界锁服务**——独立锁守护进程 / DB 行锁 / 分布式锁。权威在服务端，恶意客户端无法伪造"持锁"。
- 契约：`acquire(lock_id) → lease_token`，`release(lease_token)`，`assert_held(lock_id)` 由服务端权威判定。
- 安全边界：服务端是唯一锁权威，客户端仅持有令牌，无法自行宣称持锁。
- 状态：**NOT_IMPLEMENTED / NOT_VERIFIED**（本工单 READ-ONLY，不实现）。

**路径二（本工单采用的边界声明）**：明确**安全边界**——"本设计的互斥保证仅对**遵守协议的进程**成立；对恶意同 UID 进程，本设计标记 **NOT_VERIFIED / INDETERMINATE**，并在检测到锁文件身份异常时**拒绝继续**。"

### 2.5 路径二下的锁文件生命周期协议（POSIX/Linux）

**目标**：在"遵守协议"边界内，证明所有参与者锁同一稳定对象。

| 步骤 | 操作 | 使用句柄还是路径 | 说明 |
|------|------|----------------|------|
| 1 | 打开可信锁目录 | `open(lock_dir, O_DIRECTORY)` → dirfd | 句柄 | 目录一旦打开，后续操作绑定 dirfd |
| 2 | 目录身份校验 | `fstat(dirfd)` 记录 dev+ino | 句柄 | 后续每次操作前比对 |
| 3 | 打开锁文件 | `openat(dirfd, "lock", O_RDWR\|O_CREAT\|O_NOFOLLOW, 0600)` → fd | 句柄（相对 dirfd）| O_NOFOLLOW 拒符号链接 |
| 4 | 锁文件身份记录 | `fstat(fd)` 记录 dev+ino+ctime | 句柄 | 这就是"稳定对象身份" |
| 5 | 加锁 | `flock(fd, LOCK_EX)` | 句柄 | 锁关联 OFD |
| 6 | 进入临界区前复查 | `fstat(fd)` 再取身份，与步骤 4 比对 | 句柄 | **仅复核 fd 指向对象未变** |
| 7 | 受保护操作 | 全部用已打开的 fd/dirfd，**不再用路径字符串** | 句柄 | 消除"验证后按路径重开" |
| 8 | 释放 | `flock(fd, LOCK_UN)` | 句柄 | — |

**关键改进**：受保护操作**全程只使用步骤 1~4 打开并校验过的句柄（dirfd/fd）**，**绝不**在步骤 6 校验之后"按路径字符串重新打开或操作"——这一步消除了"验证对象 A、操作对象 B"的经典 TOCTOU。

**dirfd 的相对操作**：步骤 3 用 `openat(dirfd, ...)`，即使 `lock_dir` 路径被替换，`openat` 仍作用于**打开时绑定的原目录 inode**，不受路径重定向影响。

**O_NOFOLLOW**：拒绝锁文件是符号链接的情况（适用 Linux；macOS 的 O_NOFOLLOW 在 openat 上语义受限，见 §2.7）。

### 2.6 路径二下的锁文件生命周期协议（Windows）

Windows 用**文件句柄 + 文件 ID** 绑定对象：

| 步骤 | 操作 | 说明 |
|------|------|------|
| 1 | 打开锁文件 | `CreateFileW(path, GENERIC_READ\|GENERIC_WRITE, 0 /*无共享*/, ..., OPEN_ALWAYS, FILE_FLAG_OPEN_REPARSE_POINT, ...)` | `0` 共享模式 = 排他，阻止其他进程再打开 |
| 2 | 取文件身份 | `GetFileInformationByHandle` → `dwVolumeSerialNumber` + `nFileIndexHigh/Low`（等价 inode）| 句柄绑定 |
| 3 | 加锁 | `LockFileEx(hFile, LOCKFILE_EXCLUSIVE_LOCK, ...)` | 句柄级排他锁 |
| 4 | 复查身份 | 再 `GetFileInformationByHandle` 比对 | 仅复核句柄未变 |
| 5 | 受保护操作 | 全部用 hFile，不用路径 | — |
| 6 | 释放 | `UnlockFileEx` | — |

**共享模式 0** 是关键：`CreateFileW` 的 `dwShareMode=0` 使其他进程**无法再打开该文件**（含 unlink），从 OS 层阻止"打开后替换"。但**同样只对遵守协议的进程有效**——恶意进程可先 `DeleteFileW`（若其未被打开则成功）再重建。

**重解析点处理**：`FILE_FLAG_OPEN_REPARSE_POINT` 阻止符号链接/挂载点被透明解析，打开的是重解析点本身而非目标；配合 `GetFinalPathNameByHandle` 可验证。

### 2.7 平台语义差异与未验证项（明确区分，不混用）

| 平台 | flock | OFD 锁 | openat | O_NOFOLLOW | 目录 fsync |
|------|-------|--------|--------|-----------|-----------|
| Linux | 支持，锁关联 OFD | 支持（fcntl F_OFD_SETLK）| 支持 | 支持 | 支持 |
| macOS | 支持 | 支持 | 支持 | openat 上 O_NOFOLLOW **受限**（需另用 O_SYMLINK 或 fstatat 校验）| 目录 fsync 语义弱 |
| BSD | 支持 | 支持（版本相关）| 支持 | 版本相关 | 版本相关 |
| Windows | 无 flock，用 LockFileEx | 无（句柄锁等价）| 无（用句柄）| 用重解析点标志 | 无目录 fsync，用 FlushFileBuffers |

**明确声明**：macOS/BSD 的 `O_NOFOLLOW` 在 `openat` 上的行为、以及目录 fsync 的持久化语义，**未在本工单实测**，标记 **NOT_VERIFIED**。Windows 的共享模式 0 阻止打开、`GetFileInformationByHandle` 的 nFileIndex 稳定性，**未实测**，标记 **NOT_VERIFIED**。

### 2.8 锁文件替换/删除/重建故障矩阵

| 场景 | 遵守协议进程 | 恶意同 UID 进程 | 本设计处理 |
|------|-------------|----------------|-----------|
| 锁文件被 unlink | 不会（协议禁）| 可能 | 检测：步骤 6 复查 fstat 身份，若 fd 的 inode 已被 unlink（`fstat` 的 `st_nlink==0`）→ **拒绝继续** |
| 锁文件被重建（新 inode）| 不会 | 可能 | 已打开的 fd 仍指向旧 inode；新进程打开新 inode → 双方不同对象 → **身份校验失败，拒绝** |
| 符号链接替换 | 不会 | 可能 | O_NOFOLLOW / 重解析点标志拒绝 |
| 目录被替换 | 不会 | 可能 | dirfd 绑定原目录 inode，openat 不受影响 |
| 锁文件权限被改 | 不会 | 可能 | 步骤 3 用 0600 创建；后续 `fstat` 校验 uid/权限，异常拒绝 |
| 进程崩溃 | — | — | OS 释放 fd 与 flock，锁自动释放，无残留 |
| 锁文件被硬链接 | 不会 | 可能 | 硬链接指向同一 inode，身份（dev+ino）相同 → **不算分裂**，但需审计硬链接数 |

**核心**：本设计在"遵守协议"边界内**能证明** A/B 不会同时进入临界区（见 §2.9 时间线）；对恶意进程，检测到身份异常即**拒绝继续**，而非假装仍在互斥保护下。

### 2.9 A/B 两进程遵守协议的时间线（证明互斥成立）

两进程 A、B 都通过本库 API 获取锁：

| 时刻 | A | B | 共享状态 |
|------|---|----|---------|
| t0 | openat → fd-A → fstat 记 inode-I | — | inode-I 存在 |
| t1 | flock(fd-A, LOCK_EX) 成功 | — | inode-I 被 A 排他 |
| t2 | 进临界区 | openat → fd-B → fstat 记 **inode-I** | 同一 inode |
| t3 | 操作 | flock(fd-B, LOCK_EX) → **阻塞** | flock 对同一 inode 排他 |
| t4 | 完成，LOCK_UN | （阻塞解除）flock 成功 | B 获得排他 |
| t5 | 退出 | 进临界区 | B 独占 |

**证明**：因为 A、B 都遵守协议（不 unlink），路径 `lock` 始终解析到 inode-I，`fstat` 记录的身份一致，`flock` 在同一 inode 上给出**真实排他**，故 t2~t4 期间 B 无法进入临界区。**不存在"A 锁 inode-A、B 锁 inode-B 却同时持锁"的路径**——这是遵守协议边界内的严格结论。

**对恶意进程的诚实边界**：若 B 是恶意同 UID 进程，§2.2 已证明它可通过 unlink/recreate 制造 inode 分裂。本设计**不能证明**能防住它，故对恶意进程标记 **FAIL（本地 FS 无法防御）/ NOT_VERIFIED**，并靠 §2.8 的身份异常检测**拒绝继续**，而不是伪造互斥。

### 2.10 验收对照与结论

- inode-A/inode-B 分裂场景正面处理：✅ §2.2
- "永久锁文件 + 禁 unlink"不是充分证明：✅ §2.1
- `fstat(fd)` 只证明 FD 指向对象、不证明路径一致：✅ §2.1
- 保证所有参与者锁同一稳定对象的方案：✅ §2.5/§2.6（句柄绑定 + 全程不用路径重开）
- POSIX/Windows 分开：✅ §2.5/§2.6/§2.7
- 无法防御恶意进程的边界声明：✅ §2.3/§2.4/§2.9
- A/B 时间线证明遵守协议互斥成立：✅ §2.9

**P0-03 结论：PASS-WITH-ISSUES（设计级）**。
- 遵守协议的参与者互斥：**可证明成立**（§2.9）。
- 恶意同 UID 进程：**本地 FS 无法防御**，标记 NOT_VERIFIED，靠身份异常检测拒绝继续。
- 引入跨信任边界锁服务（路径一）为彻底解决方向，本工单 NOT_IMPLEMENTED。
- macOS/BSD/Windows 具体 API 语义 NOT_VERIFIED。

---

## 3. P0-04｜正式文档版本化发布与真正 CAS

### 3.1 缺陷定位（05.13 §3.7）

05.13 §3.7 用"目标存在时无覆盖发布失败"来更新正式文档，并把它当作 CAS：

> "无覆盖发布：目标不存在时 rename 成功，目标已存在时 rename 失败 → 调用方重试"

**两个致命缺陷**：
1. **无覆盖原语只能创建新对象，不能更新已存在对象**。`rename` 的 `RENAME_NOREPLACE`（Linux）/ `link` + `unlink` 方案（POSIX）只能保证"目标不存在时原子创建"，**一旦正式文档已存在，无覆盖原语必然失败**——它根本无法完成"更新已有文档"这个基本操作。
2. **"读 revision → 比较 → 写入"不是原子 CAS**。读与写之间有时间窗，两个事务可能读到相同 revision，各自写入，导致**丢失更新**。

### 3.2 05.14 的解决方案：不可变版本文件 + 原子指针

**核心思想**：正式文档**永不原地修改**。每个提交生成一个**不可变版本文件**，用一个**原子指针**指向当前版本。

```
doc/
├── current                    ← 指针：内容为 "<txn_id>\n<revision>"
├── versions/
│   ├── <txn_id_a>.<rev_1>    ← 不可变版本文件（内容=完整文档）
│   ├── <txn_id_b>.<rev_2>    ← 不可变，永不覆盖
│   └── ...
└── prepared/<txn_id>          ← PREPARED 记录
```

- **不可变版本文件**：`versions/<txn_id>.<revision>` 用 `O_EXCL` 创建，内容含 `transaction_id` + `revision` + 文档正文 + SHA256 自校验。一旦写完永不修改、永不覆盖。
- **原子指针**：`current` 是**单槽**，指向"当前生效版本"。更新指针 = 用原子 rename 替换 `current`。

### 3.3 提交时序（完整）

| 阶段 | 操作 | 持久化 | 说明 |
|------|------|--------|------|
| 1 | 写不可变版本文件 `versions/<txn_id>.<rev>` | `write` + `fsync` | O_EXCL 创建，写全文+自校验字段 |
| 2 | 校验版本文件 | `fstat` + 重读 + SHA256 比对 | 确认版本文件完整 |
| 3 | 写 PREPARED 记录 `prepared/<txn_id>` | `write` + `fsync` | 记录 txn_id、版本文件身份、revision、目标指针 |
| 4 | **原子替换指针** `current` | `rename(tmp_current → current)` + `fsync(doc_dir)` | 这是唯一的竞争点 |
| 5 | 写 COMMIT 回执 | `write` + `fsync` | 记录指针已指向新版本 |

### 3.4 真正的 CAS：指针的单调 revision 校验

**关键**：更新 `current` 指针必须是**真正的原子 CAS**，而非"读-比较-写"。

**方案**：`current` 指针文件内容为 `<txn_id>\n<revision>`。提交方在替换指针前：
1. 读 `current` 得到 `<old_txn>\n<old_rev>`；
2. 断言 `new_rev == old_rev + 1`（revision **单调递增**）；
3. 写临时指针 `current.<txn_id>`（内容 `<new_txn>\n<new_rev>`）→ fsync；
4. `rename(current.<txn_id> → current)` 原子替换。

**为什么这是 CAS**：`rename` 是**原子**的——它要么把 `current` 替换成新内容，要么不替换，不存在"读到一半"的中间态。但 rename **不能条件替换**（不能"仅当 current 仍是 X 时替换"）。

**因此还需要一个真正的 CAS 原语**。05.14 采用以下**二者之一**（并明确"应用锁 ≠ 系统级 CAS"）：

**CAS 方案 A（POSIX）**：用 `link()` + `unlink()` 实现**有条件的指针更新**：
- `link(current, current.lock.<txn_id>)` —— 若 `current` 存在则成功，创建硬链接作为"占有标记"；`link` 对已存在的目标**原子失败**（EEXIST），这是**内核保证的原子条件操作**。
- 若 `link` 失败 → 说明另一事务已抢先，**放弃本次提交**，重读 current 重试。
- 若 `link` 成功 → 本事务独占指针更新权 → `rename(tmp_current → current)`（此时只有本事务持有占有标记，不会竞争）→ `unlink(current.lock.<txn_id>)`。

**CAS 方案 B（数据库事务，本工单不实现）**：用支持 ACID 的存储（SQLite/DB）的 `UPDATE ... WHERE revision = :old_rev`，DB 的 row 锁 + 事务隔离给出真正的 CAS。**契约**：`UPDATE current SET txn_id=:new_txn, revision=:new_rev WHERE revision=:old_rev`，affected_rows==1 才算成功。

**本工单采用方案 A**（纯文件系统、无需外部依赖），方案 B 作为 NOT_IMPLEMENTED 的替代契约。

### 3.5 POSIX 提交协议（方案 A）

```
1. 写 versions/<txn_id>.<new_rev>（O_EXCL，写全文，fsync）
2. 校验版本文件 SHA256
3. 写 prepared/<txn_id>（fsync）
4. 读 current → old_txn, old_rev
5. assert new_rev == old_rev + 1（不满足 → ABORT，重读）
6. 写 current.<txn_id>（临时指针，内容 new_txn\nnew_rev，fsync）
7. link(current, current.lock.<txn_id>)   # 原子占有：成功=独占权
   - EEXIST → 他人抢先 → 放弃，重读 current 重试
8. rename(current.<txn_id> → current)     # 原子替换（此时独占）
9. fsync(doc_dir)
10. unlink(current.lock.<txn_id>)         # 释放占有标记
11. 写 COMMIT 回执（fsync）
```

### 3.6 Windows 提交协议

Windows 的 `MoveFileExW` 不支持"条件替换"，但提供 `ReplaceFileW`（原子替换，可要求备份）：

```
1. 写 versions\<txn_id>.<new_rev>（CREATE_NEW，写全文，FlushFileBuffers）
2. 校验版本文件
3. 写 prepared\<txn_id>（FlushFileBuffers）
4. 读 current → old_txn, old_rev
5. assert new_rev == old_rev + 1
6. 写 current.<txn_id>（FlushFileBuffers）
7. 用 CreateFileW(current, dwShareMode=0 /*独占打开*/) 尝试独占 current
   - 若失败（他人已打开）→ 放弃重试
   - 若成功 → 本事务独占指针
8. ReplaceFileW(current, current.<txn_id>, ...)  # 原子替换
9. CloseHandle（释放独占）
10. 写 COMMIT 回执
```

**说明**：Windows 的 `CreateFileW(dwShareMode=0)` 独占打开 + `ReplaceFileW` 是"应用层独占 + 原子替换"的组合，等价于 POSIX 的 link 占有标记，但**独占打开只对遵守协议的进程有效**（同 §2 的边界）。`ReplaceFileW` 的原子性、`dwShareMode=0` 阻止打开的语义，**未实测，NOT_VERIFIED**。

### 3.7 10 个并发/崩溃场景推演

| # | 场景 | 推演 | 结果 |
|---|------|------|------|
| C1 | 单事务正常提交 | 步骤 1~11 全部成功，current 指向新 rev | 成功，不丢失更新 |
| C2 | 两事务并发，revision 竞争 | T1、T2 都读到 old_rev；T1 link 成功占有，T2 link EEXIST 放弃重读 | 只有一个提交成功，不覆盖 |
| C3 | 写版本文件中断（崩溃）| versions/ 有半成品 → 恢复时校验 SHA256 失败 → 删除半成品 | 无副作用 |
| C4 | PREPARED 写入中崩溃 | prepared/<txn_id> 半成品 → 恢复校验失败 → 忽略 | 无副作用 |
| C5 | link 成功后、rename 前崩溃 | current.lock.<txn_id> 残留 + current 未变 → 恢复时发现 lock 标记且 prepared 存在 → 回滚：删除 lock 标记，保留 prepared 供重试 | 无丢失，可重试 |
| C6 | rename 成功后、COMMIT 前崩溃 | current 已指向新 rev，COMMIT 未写 → 恢复时 current 指向新版本 + 版本文件存在 → **确认提交**（因为 current 是唯一真相）| 成功（current 为准）|
| C7 | 目标被其他事务抢先创建版本文件 | O_EXCL 创建 versions/<txn_id>.<rev> 失败 → 换新 txn_id 重试 | 不覆盖 |
| C8 | 重复 transaction_id | O_EXCL 保证 versions/<txn_id> 唯一 → 重复 txn_id 创建失败 → 拒绝 | 不重复提交 |
| C9 | revision 回退（伪造旧 rev）| assert new_rev == old_rev+1 失败 → ABORT | 拒绝 |
| C10 | 部分持久化（目录 fsync 失败）| rename 后 fsync(doc_dir) 失败 → 标记 NOT_DURABLE，但不回滚 current（因为 rename 已原子完成）→ 恢复时按 current 为准 | 明确非持久，不假装已持久 |

**证明不变量**：
- **不丢失更新**：C2 证明 link 占有标记保证同一时刻只有一个事务能替换指针。
- **不覆盖他人提交**：C2/C5 证明失败的提交不残留、成功的提交以 current 为唯一真相。
- **不错误确认 COMMITTED**：C6 证明"确认"以 current 指向 + 版本文件存在为据，而非回执；C4/C3 证明半成品不被误认为提交。

### 3.8 验收对照与结论

- 区分 Candidate 创建/版本提交/指针更新/事务确认：✅ §3.3
- 不覆盖他人结果地更新已有文档：✅ §3.2/§3.5（不可变版本 + 指针替换）
- 真正可执行 CAS 原语：✅ §3.4（POSIX link 占有标记 / DB 契约）
- 10 个并发/崩溃场景：✅ §3.7
- 不把应用锁等同 OS 级 CAS：✅ §3.4（明确区分）
- POSIX/Windows 分开：✅ §3.5/§3.6

**P0-04 结论：PASS-WITH-ISSUES（设计级）**。
- 核心 CAS（POSIX link 占有标记）语义可证明；`link` 的 EEXIST 原子性是 POSIX 明确定义的行为。
- Windows 的 `ReplaceFileW` + `dwShareMode=0` 组合语义 NOT_VERIFIED。
- 目录 fsync 持久化语义（C10）在 macOS/Windows 上 NOT_VERIFIED。
- 数据库 CAS（方案 B）NOT_IMPLEMENTED。

---

## 4. P0-05｜源对象并发写入与 Source Retire

### 4.1 缺陷定位（05.13 §4.3/§4.6）

05.13 用"复制前 sha256 → 复制 → rename 前再 sha256 → 一致则 rename"来保证源对象一致性：

> "最后一次 `sha256(src)` 与 `rename(src → src.retired)` 之间"

**致命缺陷**：
1. **最后一次 sha256 与 rename 之间有窗口**：两次哈希校验之间，其他进程可能写入源对象，导致"复制的是 H1，最后 rename 的却是 H2"。
2. **rename 不能阻止已持可写 FD 的进程继续写**：POSIX 的 `rename` 只改变目录项，**不撤销**任何已打开 FD 的写权限——某个进程在 rename 前 `open(src, O_WRONLY)` 拿到 fd，rename 后仍能用该 fd 继续写入**同一个 inode**（现在叫 `src.retired`），内容仍在变。

### 4.2 三层结论：可证明一致 / 只能检测 / 无法证明

05.14 把"源对象一致性"分为三个等级，明确各自可用的机制与边界：

| 等级 | 机制 | 能否阻止外部写入 | 结论 |
|------|------|----------------|------|
| **A 可证明一致** | 源对象不可变（append-only / O_EXCL 独占 / 文件锁），所有写入者遵守 | 能（协作）| 可证明 snapshot 内容 == 源内容 |
| **B 只能检测变化** | 复制前/后哈希比对 | 不能阻止，只能发现 | 检测到变化即 DENY，未检测到 ≠ 证明 |
| **C 无法证明** | 无任何机制 | 不能 | 必须 DENY Source Retire，保留源 + INDETERMINATE |

**核心原则**：**不能把"复制的是 H1、处理的是 H2"的路径当作安全**。如果无法达到等级 A，只能退到 B（检测到变化就 DENY），绝不允许"检测没发现变化就继续"冒充 A。

### 4.3 等级 A 的可选机制（本工单契约，不实现运行验证）

**机制 1：排他打开 + 文件锁**
- 源对象用 `open(src, O_RDWR)` + `flock(fd, LOCK_EX)` 独占；所有本库写入者遵守"写前先 LOCK_EX"。
- quarantine 复制前 LOCK_EX 持有，复制期间外部写入者阻塞，复制完 LOCK_UN。
- **边界**：只对遵守协议的写入者有效（同 §2 的恶意进程边界）。

**机制 2：源对象不可变（推荐）**
- **根本方案**：源对象是 **append-only 不可变对象**——一旦写入完成即 sealed（如写临时文件 + 原子 rename 到源路径），**永不原地修改**。
- quarantine 只需**读副本**（`open(O_RDONLY)` + 哈希），因为源不可变，读取期间内容不会变。
- 更新源 = 写新源对象（新 inode）+ 原子指针（复用 §3 的 CAS 方案），旧源对象永不改。
- **这是最彻底的方案**：从设计上消除"源内容在复制后变化"的可能。

**机制 3：复制后只操作副本，源永不改（本工单采用）**
- quarantine 的操作对象是**副本**（qtmp/qfinal），**源对象从不被 quarantine 移动或删除**。
- Source Retire（源的改名/删除）是**独立事务**，且**前提是源已被 sealed 或内容已被验证一致**。
- 若无法证明源 sealed 且内容一致，Source Retire 必须 DENY。

### 4.4 05.14 采用的完整协议

**Quarantine 阶段（只读源，不改源）**：

| 步骤 | 操作 | 说明 |
|------|------|------|
| 1 | 写 QUARANTINE_INTENT（fsync）| 绑定 txn_id + 源身份 + 源 SHA256 期望值 |
| 2 | 打开源 `open(src, O_RDONLY\|O_NOFOLLOW)` → fd | 只读 |
| 3 | `fstat(fd)` 记录源身份（dev+ino+ctime+size+mtime）| 身份快照 |
| 4 | 流式复制到 qtmp，同时计算 SHA256 | 边读边算哈希 |
| 5 | `fstat(fd)` 再取身份，与步骤 3 比对 | 检测复制期间是否被替换（unlink/recreate）|
| 6 | SHA256(qtmp) 与步骤 4 计算的源哈希比对 | 检测内容一致性 |
| 7 | rename(qtmp → qfinal) + fsync | 发布隔离副本 |
| 8 | 写 COMPLETE 回执（fsync）| 隔离完成 |

**关键**：步骤 5 检测"身份是否被替换"，步骤 6 检测"内容是否被修改"。**但这两个检测只能发现变化，不能阻止变化**（等级 B）。要达等级 A，必须依赖 §4.3 的机制 2（源不可变）。

**Source Retire 阶段（独立事务，仅当源 sealed/一致）**：

| 步骤 | 操作 | 说明 |
|------|------|------|
| 1 | 写 SOURCE_RETIRE_INTENT（fsync）| 独立意图，绑定 txn_id + 源身份 + 源哈希 |
| 2 | 验证源仍 sealed 或内容仍 == 已验证哈希 | 若变化 → DENY + INDETERMINATE |
| 3 | rename(src → src.retired) | 改名，不删 |
| 4 | 写 RETIRE 回执（fsync）| 退役完成 |

**quarantine 完成 ≠ Source Retire 授权**：两个独立事务，前者成功只说明"副本已隔离"，后者还需**再次验证源内容未变**。

### 4.5 源对象并发写入场景推演（覆盖工单要求）

| # | 场景 | 推演 | 结果 |
|---|------|------|------|
| W1 | 校验前写入 | 步骤 1 之后、步骤 4 之前外部写入 → 步骤 6 哈希比对发现不一致 → DENY | 检测到，拒绝 |
| W2 | 校验期间写入 | 步骤 4 复制到一半外部写入 → 步骤 6 哈希不一致 → DENY | 检测到，拒绝 |
| W3 | 校验后写入（哈希通过后、rename 前）| 若源非 sealed，步骤 5/6 无法覆盖 rename 之后的写入 → **源不可变（机制2）才能消除**；否则 DENY | 取决于源是否 sealed |
| W4 | rename 前写入 | 同 W3，若源可变则 rename 的可能是新内容 → 源不可变才安全 | DENY（源可变时）|
| W5 | rename 后旧 FD 写入 | 某进程 rename 前 open(O_WRONLY) 拿到 fd，rename 后继续写同一 inode → **内容仍在变，且 rename 后的路径已指向新名字** | **等级 B 无法防**，必须源不可变或排他 |
| W6 | 进程崩溃（复制中）| qtmp 半成品 → 恢复校验失败 → 删除 qtmp | 无副作用 |
| W7 | 源被替换（unlink + recreate）| 步骤 5 身份（ino）比对发现 inode 变化 → DENY | 检测到，拒绝 |
| W8 | 源被硬链接 | 硬链接指向同一 inode，身份相同，但另一路径可写 → 需审计 | 需额外处理 |
| W9 | 无法证明一致性 | 无 sealed、无排他、哈希又未检测到（可能恰好一致）| **不能证明** → DENY Source Retire + INDETERMINATE | DENY |
| W10 | 源不可变（sealed）| 源是 append-only 不可变 → 复制期间内容不会变 → 可证明一致 | 等级 A，安全 |

**核心结论**：W5 是**最硬的场景**——rename 无法阻止已持可写 FD 的进程继续写。**唯一能彻底解决的是机制 2（源不可变）+ 机制 1（排他/文件锁协作）**；纯哈希比对（等级 B）对 W5 无能为力，必须明确标注。

### 4.6 验收对照与结论

- 最后一次 sha256 与 rename 之间窗口：✅ §4.1 定位，§4.4 用源不可变/独立事务消除
- rename 不能阻止已持可写 FD 进程继续写：✅ §4.1/§4.5 W5
- 排他打开/文件锁/复制后只操作副本：✅ §4.3/§4.4
- 校验前/期间/后、rename 前/后、旧 FD、崩溃场景：✅ §4.5 W1~W10
- quarantine 与 Source Retire 独立事务：✅ §4.4

**P0-05 结论：PASS-WITH-ISSUES（设计级）**。
- 源不可变（机制 2）+ 排他（机制 1）可证明一致（等级 A）。
- 纯哈希比对（等级 B）只能检测、不能阻止，对 W5（旧 FD 写入）无能为力——已明确标注。
- 无法证明一致时 DENY Source Retire + INDETERMINATE（等级 C）。
- 实际运行时行为（文件锁协作、sealed 语义）NOT_TESTED。

---

## 5. P0-06｜Tombstone 事务归属与删除恢复

### 5.1 缺陷定位（05.13 §5.7）

05.13 §5.7 提出"tombstone 永久保留，不 unlink"，但**没有定义**：
- tombstone 的**唯一性**（如何保证不会撞名）；
- tombstone 的**事务绑定**（如何证明它属于本事务）；
- tombstone 的**不可替换边界**（如何防止外部替换/篡改）；
- tombstone 的**磁盘增长与归档**（永久保留会无限增长）。

**且核心误区**：05.13 把 `DELETE_INTENT` / `UNLINK_INTENT` 当作删除动作已完成的证据——**意图 ≠ 事实**。

### 5.2 三态归因的核心原则

**删除完成的证据链**必须满足"有持久化的、可恢复的、属于本事务的证据"，否则归因为 INDETERMINATE：

| 状态 | 含义 | 需要的持久化证据 | 允许的后续动作 |
|------|------|----------------|--------------|
| `ALREADY_ABSENT` | 清理前对象已不存在 | 清理开始时 `fstat`/`stat` 确认对象不存在（审计记录）| 直接结束，不写 REMOVED |
| `REMOVED_BY_THIS_CLEANUP` | 有证据证明本事务删除 | ① DELETE_INTENT 已持久化 ② tombstone 存在且身份/哈希匹配 ③ rename 已执行 ④ REMOVED 记录已持久化 | 写终态审计，可归档 tombstone |
| `INDETERMINATE` | 无法证明删除归属 | 证据链不完整（tombstone 丢失/被替换/REMOVED 未持久化）| **保留现场，不自动覆盖/删除，人工介入** |

**铁律**：
1. `DELETE_INTENT` / `UNLINK_INTENT` 只证明**意图**，不证明**动作完成**。
2. "对象不存在"不能自动归因为"本事务删除成功"。
3. 无法证明归属 → **INDETERMINATE**，绝不补写 `REMOVED` 伪造确定性。

### 5.3 Tombstone 的唯一命名与不可替换

**唯一命名**：`tombstone/<cleanup_id>.<src_name>`
- `cleanup_id` 是全局唯一 ID（`uuid4` 或单调序列），由删除事务生成。
- 用 `O_EXCL` 创建，**保证同名 tombstone 不会被第二个事务抢占**（EEXIST 即失败）。

**事务绑定**：tombstone 的**内容**写入元数据，而非仅靠文件名：

```
cleanup_id: <uuid>
transaction_id: <txn_id>
src_identity: <dev>:<ino>:<ctime>
src_sha256: <sha256>
src_path: <original path>
created_at: <timestamp>
```

**不可替换边界**：tombstone 一旦创建，**内容不可修改**（写入后 `O_RDONLY` + 记录 SHA256）。后续任何 `fstat` 身份变化 / SHA256 不匹配 → **INDETERMINATE**，因为 tombstone 被外部替换/篡改，不能再作为归因证据。

### 5.4 删除状态机与持久化边界

状态转换表（每步标持久化动作）：

| 状态 | 触发动作 | 持久化动作 | 崩溃后可观察证据 |
|------|---------|-----------|----------------|
| `DELETE_INTENT` | 记录删除意图 | 写 intent 记录 + fsync | intent 记录存在 |
| `TOMBSTONED` | `rename(src → tombstone/<cleanup_id>.<src>)` | rename + fsync(dir) | tombstone 存在 + 源路径消失 |
| `UNLINK_INTENT` | 记录"将删除 tombstone"意图 | 写记录 + fsync | unlink-intent 存在 |
| `REMOVED` | 记录删除已确认 | 写 REMOVED + fsync | REMOVED 记录存在 |
| `CLEANED` | 终态确认，可归档 | 写 CLEANED + fsync | CLEANED 记录存在 |

**关键顺序**（消除 05.13 的矛盾）：**`REMOVED` 必须持久化之后，才允许归档/删除 tombstone**。05.13 的矛盾是"§6.3 要求 tombstone 存在才能判 REMOVED，但 §6.4 却在 REMOVED 前就 unlink tombstone"——05.14 修正为：**tombstone 在 REMOVED 持久化之前绝不删除**。

### 5.5 删除归因的证据模型

`REMOVED_BY_THIS_CLEANUP` 的判定需要**同时满足全部**：
1. `DELETE_INTENT` 记录存在，且 `transaction_id` == 本事务；
2. `tombstone/<cleanup_id>.<src>` 存在；
3. tombstone 的 `fstat` 身份（dev+ino）与创建时记录一致（未被替换）；
4. tombstone 的 SHA256 与创建时记录一致（未被篡改）；
5. 源路径已不存在（`stat(src)` 返回 ENOENT）—— rename 已执行；
6. `REMOVED` 记录已持久化，且绑定同一 `transaction_id` + `cleanup_id`。

**任一不满足 → INDETERMINATE**。特别是：
- tombstone 被外部删除 → 条件 2 失败 → INDETERMINATE（即使源已消失）；
- tombstone 被替换（新 inode）→ 条件 3 失败 → INDETERMINATE；
- REMOVED 未持久化（rename 后崩溃）→ 条件 6 失败 → INDETERMINATE。

### 5.6 Tombstone 永久保留的磁盘增长与归档协议

**问题**：永久保留 tombstone 会无限增长磁盘。

**归档协议**：
1. **归档前提**：仅当 tombstone 已 `CLEANED`（终态确认，删除归因已成立）才可归档。
2. **归档动作**：`rename(tombstone/<cleanup_id>.<src> → archive/<cleanup_id>.<src>)`，**同样走 tombstone 协议**（O_EXCL、内容含元数据、不可修改）。
3. **归档本身也是事务**：写 ARCHIVE_INTENT → rename → 写 ARCHIVE 回执，崩溃可恢复。
4. **归档中断**：rename 成功但回执未写 → 恢复时按 archive/ 是否存在判定；archive 与 tombstone 内容哈希一致才确认。
5. **永不 unlink**：归档后仍保留（archive 是冷存储），除非外部运维手动清理（需记录）。

### 5.7 删除崩溃恢复场景推演（覆盖工单要求）

| # | 场景 | 推演 | 结果 |
|---|------|------|------|
| D1 | rename 前崩溃 | 只有 DELETE_INTENT 持久化，源对象还在 | 恢复：源还在 → 未删除 → 重新走流程 |
| D2 | rename 后崩溃（REMOVED 未写）| tombstone 存在，源消失，REMOVED 未持久化 | 恢复：条件 6 失败 → **INDETERMINATE**（不能证明是本事务删的）|
| D3 | REMOVED 写入中崩溃 | REMOVED 半成品 → 校验失败 → 忽略 | 恢复：等价 D2，INDETERMINATE |
| D4 | 外部替换 tombstone | tombstone 身份（ino）变化 → 条件 3 失败 | INDETERMINATE |
| D5 | 外部删除 tombstone | tombstone 消失 → 条件 2 失败 | INDETERMINATE |
| D6 | 重复恢复 | 恢复幂等：已 INDETERMINATE 的保持 INDETERMINATE，不重复判定 | 幂等 |
| D7 | 归档中断 | rename 到 archive 成功但回执未写 → 按 archive 存在 + 哈希匹配确认 | 可确认（哈希一致）|
| D8 | 源被同名重建 | rename 后外部 recreate 同名源 → 源路径又存在 → 条件 5 失败 → INDETERMINATE | INDETERMINATE |
| D9 | 清理前源已不存在 | 清理开始时 stat(src) ENOENT → ALREADY_ABSENT，不写 REMOVED | 正确归因 ALREADY_ABSENT |
| D10 | 并发删除 | 两事务同时删同一源 → tombstone O_EXCL 保证只有一个创建成功，另一个 EEXIST → 放弃 | 不重复删除 |

**关键**：D2/D3 是 05.13 的核心矛盾点——rename 成功但 REMOVED 未持久化时，05.13 因 tombstone 仍存在而误判，或因 tombstone 已 unlink 而无法判。05.14 明确：**REMOVED 未持久化 → INDETERMINATE，绝不补写 REMOVED 伪造确定性**。

### 5.8 验收对照与结论

- tombstone 唯一命名：✅ §5.3（cleanup_id + O_EXCL）
- tombstone 事务绑定：✅ §5.3（内容写 txn_id + cleanup_id + 身份 + 哈希）
- 不可替换边界：✅ §5.3（内容不可改 + 身份/哈希校验）
- 磁盘增长与归档协议：✅ §5.6
- DELETE_INTENT/UNLINK_INTENT 不是删除完成证据：✅ §5.2/§5.5
- 三态区分 + INDETERMINATE：✅ §5.2
- 7+ 个崩溃/外部干扰场景：✅ §5.7

**P0-06 结论：PASS-WITH-ISSUES（设计级）**。
- tombstone 状态机与归因证据链已闭合，消除 05.13 的"REMOVED 未持久化却 unlink tombstone"矛盾。
- rename 成功但 REMOVED 未持久化 → INDETERMINATE（诚实标注，不伪造确定性）。
- tombstone 永久保留的磁盘增长靠归档协议控制；归档的运行时行为 NOT_TESTED。
- 文件系统 rename/fsync 的崩溃恢复语义（D2/D3/D7）NOT_TESTED，依赖 POSIX 定义的行为。

---

## 6. 强制设计不变量（I-01 ~ I-20）

每条给出：证明章节、失效场景、拒绝策略。

| # | 不变量 | 证明章节 | 失效场景 | 拒绝策略 |
|---|--------|---------|---------|---------|
| I-01 | 没有有效锁，不得执行受保护操作 | §2.5 步骤 5~6 | fd 无 flock 成功返回 | 拒绝进入临界区 |
| I-02 | 所有参与进程必须锁定同一个稳定锁对象 | §2.5/§2.9 | inode 分裂 | 身份校验失败即拒绝 |
| I-03 | fork 后子进程不得使用失效的继承同步对象 | §1.5/§1.6 | child 重建失败 | DENY 后续锁操作 |
| I-04 | 子进程清理不得解除父进程共享 OFD 锁 | §1.5（只 close 不 LOCK_UN）| 子进程误 LOCK_UN | 禁止 LOCK_UN 清理，仅 close |
| I-05 | 正式 Candidate 不得暴露半成品 | §3.2（不可变版本文件 O_EXCL）| 版本文件写一半 | 恢复时 SHA256 校验失败删除 |
| I-06 | PREPARED 必须先于正式发布持久化 | §3.3 步骤 3→4 | PREPARED 未写就 rename | 拒绝：无 PREPARED 不发布 |
| I-07 | 正式目标不得被其他事务覆盖 | §3.4（link 占有标记）| 两事务抢指针 | link EEXIST 即放弃 |
| I-08 | 恢复程序不得认领无法证明归属的文件 | §3.7 C6/§5.5 | 身份/哈希不匹配 | INDETERMINATE |
| I-09 | quarantine 副本必须与源对象内容及事务建立可验证关系 | §4.4 | 源内容变化 | 哈希比对失败 DENY |
| I-10 | 源对象处理前必须证明其内容仍符合预期 | §4.4 Source Retire 步骤 2 | 源被修改 | DENY + INDETERMINATE |
| I-11 | 删除归因必须具有真实、可恢复的证据 | §5.5 | 证据链不完整 | INDETERMINATE |
| I-12 | 对象不存在不等于本次事务删除成功 | §5.2/§5.7 D9 | 源消失但无 tombstone | ALREADY_ABSENT 或 INDETERMINATE |
| I-13 | 无法证明的状态必须 DENY 或 INDETERMINATE | §2.4/§4.2/§5.2 | 任何不确定 | 不自动猜测 |
| I-14 | 设计级通过不得冒充运行时验证通过 | 全文 NOT_TESTED 标注 | — | 保留 NOT_TESTED/NOT_VERIFIED |
| I-15 | 平台保证必须限定到有明确依据的系统/版本/API | §2.7/§3.6/§5.6 | 语义差异 | 标记 NOT_VERIFIED |
| I-16 | 外部 fork 不可靠识别，靠 self-check 兜底 | §1.2/§1.7 | 绕过 API | 文档化不支持 |
| I-17 | 同 UID 恶意进程无法防御，只能检测+拒绝 | §2.3/§2.4/§2.8 | unlink/recreate | 身份异常即拒绝 |
| I-18 | 丢失更新不可发生（指针单槽 + 占有标记）| §3.7 C2 | 无占有标记就写指针 | 禁止 |
| I-19 | quarantine 完成 ≠ Source Retire 授权 | §4.4 | 跳过源再校验 | 强制独立事务 + 再校验 |
| I-20 | tombstone 在 REMOVED 持久化前不得删除 | §5.4/§5.5 | 提前 unlink tombstone | 禁止，违反即 INDETERMINATE |

**不变量数量：20 条，全部在对应章节给出推演证据。无"未证明"的 PASS。**

---

## 7. 端到端交叉审查

### 7.1 八条端到端链路静态推演

| 链路 | 前置 | 关键持久化边界 | 崩溃点 | 恢复动作 | 不变量 | 结论 |
|------|------|--------------|--------|---------|--------|------|
| L1 锁→Lease→fork→退出 | 有效锁 | flock 成功 | fork 中/后 | child 重置锁系统 | I-03/04 | PASS（设计级）|
| L2 锁身份→Candidate→PREPARED→发布→revision | 有效锁 | PREPARED 先于 rename | rename 前后 | 按 current 为准 | I-05/06/07 | PASS |
| L3 Journal意图→qtmp→qfinal→源处理→审计 | 意图持久化 | qtmp→qfinal rename | 复制中/发布中 | 哈希+身份校验 | I-09/10 | PASS-WITH-ISSUES |
| L4 DELETE_INTENT→TOMBSTONED→UNLINK_INTENT→unlink→REMOVED→CLEANED | 意图持久化 | rename→REMOVED | rename 后 | INDETERMINATE 若 REMOVED 未写 | I-11/12 | PASS-WITH-ISSUES |
| L5 任意崩溃→重启→恢复→归因→继续/拒绝 | 全链路 | 各步 fsync | 任意 | 按证据链归因 | I-08/13 | PASS-WITH-ISSUES |
| L6 两事务竞争同一对象 | 并发 | link 占有 / O_EXCL | 竞争点 | 失败方放弃重试 | I-07/18 | PASS |
| L7 路径替换/目标存在/摘要不符/审计不可用 | 异常 | 身份/哈希校验 | 异常点 | DENY/INDETERMINATE | I-08/13 | PASS |
| L8 同 UID 外部进程绕过协议改文件 | 恶意进程 | 无（无法阻止）| 任意 | 身份异常检测 + 拒绝 | I-17 | **NOT_VERIFIED（本地 FS 无法防御）** |

**交叉审查结论**：L1~L7 在遵守协议边界内无已知矛盾；L8（恶意同 UID 进程）是本设计的**已知能力边界**，如实标注 NOT_VERIFIED，不伪造。

### 7.2 最小反例清单（对每个 FAIL/边界给反例）

| 反例 | 对应问题 | 时间线 |
|------|---------|--------|
| inode 分裂 | P0-03 | §2.2（A 持 inode-A，B unlink+recreate 得 inode-B，双持）|
| 旧 FD 写入 | P0-05 | §4.5 W5（rename 后旧 fd 继续写同一 inode）|
| REMOVED 未持久化 | P0-06 | §5.7 D2（rename 后崩溃，tombstone 在但 REMOVED 无）|
| 读-比较-写丢更新 | P0-04 | §3.1（两事务读同 revision 各自写）|

---

## 8. 阻断状态（本轮保持不变）

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

## 9. 最终结论

**DESIGN-05.14 = PASS-WITH-ISSUES（设计级）**。

五项 P0 修订结论：

| # | 结论 | 核心依据 | 未验证项 |
|---|------|---------|---------|
| P0-02 | PASS（设计级）| §1.6 十场景推演，外部 fork 改 self-check 兜底 | register_at_fork 行为 NOT_TESTED |
| P0-03 | PASS-WITH-ISSUES | §2.9 遵守协议互斥可证明；恶意进程边界明确 | 恶意同 UID 进程 NOT_VERIFIED |
| P0-04 | PASS-WITH-ISSUES | §3.7 十场景，link 占有标记 CAS 可证明 | Windows ReplaceFileW NOT_VERIFIED |
| P0-05 | PASS-WITH-ISSUES | §4.5 十场景，源不可变达等级 A | 文件锁协作 NOT_TESTED |
| P0-06 | PASS-WITH-ISSUES | §5.7 十场景，归因证据链闭合 | rename/fsync 崩溃语义 NOT_TESTED |

**诚实边界声明（不伪造确定性）**：
1. 本地 POSIX/Windows 文件系统**无法防御同 UID 恶意进程**（P0-03 的 inode 分裂、P0-05 的旧 FD 写入），这是文件系统锁模型的固有能力边界，不是本设计可"修复"的缺陷。
2. 所有 NOT_TESTED/NOT_VERIFIED 项均保留，**未通过降低标准或删除失败分支来取得 PASS**。
3. 设计级通过**不等于**运行时验证通过，**不等于**解除任何阻断。

**IMPLEMENT-01 继续 STOP**，等待 ChatGPT 独立复审。

---

## 10. 交付物索引

| 文件 | 说明 |
|------|------|
| `USIT-STAGE-B-DESIGN-05.14.md` | 本主设计文档 |
| `USIT-STAGE-B-DESIGN-05.14-ISSUES.md` | 五项问题清单及剩余风险 |
| `USIT-STAGE-B-DESIGN-05.14-DIFF.md` | 相对 05.13 逐项差异 |
| `USIT-STAGE-B-DESIGN-05.14-CROSS-REVIEW.md` | 端到端交叉审查记录 |
| `USIT-STAGE-B-DESIGN-05.14-EXECUTION-REPORT.md` | 实际执行报告 |
| `USIT-STAGE-B-DESIGN-05.14-HASH-MANIFEST.md` | 交付文件 SHA256 清单 |
| `USIT-STAGE-B-DESIGN-05.14-HASH-RECEIPT.txt` | 哈希验证回执 |
| `USIT-STAGE-B-DESIGN-05.14-交付物.zip` | 完整打包 |

---
