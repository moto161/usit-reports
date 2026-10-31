# USIT-STAGE-B-DESIGN-05.13｜五项 P0 协议缺陷定向修复

> 阶段 B · DESIGN-05.13 · READ-ONLY DESIGN REVISION ONLY
> 作者：USIT 主开发 · 生成：2026-10-31（GMT+8）
> 依据：DESIGN-05.12 独立审查结论（FAIL）
> 修订范围：P0-02、P0-03、P0-04、P0-05、P0-06（P0-01 已通过，不重复修订）
> 工作区：`C:\Users\Administrator\WorkBuddy\2026-10-15-22-09-47\usit_p0`
> 交付目录：`C:\Users\Administrator\WorkBuddy\2026-10-15-22-09-47\usit_p0_05.13_reports`（**项目外**）
> 状态：**STOP — 等待独立复审**（未授权 IMPLEMENT-01）

---

## 0. 修订方法论与缺陷定位

本轮**不修改已通过的 P0-01**，仅修复五项缺陷。逐项先定位 05.12 原文，再修复。禁止只根据工单摘要重写。

| 编号 | 05.12 原文定位 | 缺陷性质 | 05.13 修复方向 |
|------|---------------|---------|---------------|
| P0-02 | §2.6 `_after_fork_child_reset` 在 child 回调调用 `snapshot_fds()`（内部持锁）| **确定缺陷**：子进程回调获取可能已锁定的继承锁 | 回调内禁碰锁，改为读 prepare 阶段预快照的不可变 tuple |
| P0-03 | §3.5 目录可信前提"仅授权进程可写"假设目录权限能防同用户替换 | **错误假设**：目录权限无法阻止同用户其他进程替换锁文件 | 永久锁文件 + 禁止 unlink/replace + 身份变化全体 DENY |
| P0-04 | §4.6/§4.7 PREPARED 状态机与恢复矩阵"证据在崩溃点之前尚未生成" | **顺序矛盾**：恢复要求证据但证据未先持久化 | PREPARED 先持久化再发布，恢复矩阵重排 |
| P0-05 | §5.3~5.5 源对象"身份不变"被当成"内容不变" | **错误等同**：inode 身份≠内容 | 双哈希内容校验 + SOURCE_RETIRE_INTENT 独立阶段 |
| P0-06 | §6.5~6.7 UNLINK_INTENT 只证意图，被当成 unlink 事实 | **归因过度**：意图冒充事实 | 结论降级，无归因证据即 INDETERMINATE，不补写 REMOVED |

> 文档设计通过 ≠ 代码实现通过。所有未实测平台行为 NOT_VERIFIED，未执行内容 NOT_TESTED。

---

## 1. P0-02｜Fork 后全部 Python 锁对象及继承资源重建

### 1.1 缺陷定位（05.12 §2.6）

05.12 的 `_after_fork_child_reset` 在 child 回调中调用 `LockHandleRegistry.snapshot_fds()`：

```python
# 05.12 §2.6（有缺陷）
for fd in LockHandleRegistry.snapshot_fds():   # ★ snapshot_fds 内部 with self._lock
    os.close(fd)
```

`snapshot_fds()` 的实现是 `with self._lock: return list(...)`。在 fork 的 child 回调里，这把 `self._lock` 是**继承自父进程的 `threading.Lock`**，它可能正处于**其他线程持有的 locked 状态**。子进程只有一个线程（fork 后），该锁的持有者线程已不存在 → 子进程调用 `snapshot_fds()` 会**永久阻塞死锁**。

这是确定缺陷：**child 回调中调用任何需要获取继承锁的方法都是不安全的**。

### 1.2 Python 同步对象完整清单（fork 前后生命周期）

| 对象 | 创建位置 | 持有线程 | fork 前状态 | 父进程处理 | 子进程处理 | 重建方法 |
|------|---------|---------|------------|-----------|-----------|---------|
| `ForkGate._fork_in_progress` | 模块级 | 任意线程（短暂）| 可能 locked | 无 | **重建**（§1.5）| 全新 `threading.Lock()` |
| `ForkGate._counter_lock` | 模块级 | 任意线程 | 可能 locked | 无 | 重建 | 全新 Lock |
| `LockHandleRegistry._lock` | 单例内 | 任意线程 | 可能 locked | 无 | **重建** | 全新 Lock |
| `LockHandleRegistry._handles` | 单例内 | — | 有数据 | 无 | 清空 | 全新 dict |
| `ProcessMutex._locks` | 模块级 | 任意线程 | 可能含 locked | 无 | 清空 | 全新 dict |
| `LeaseRegistry._leases` | 单例内 | 任意线程 | 有数据 | 无 | 清空 | 全新 dict |
| 其他 Condition/Event | 若有 | 任意线程 | 可能 locked/set | 无 | 重建 | 全新对象 |

### 1.3 核心原则：child 回调禁碰任何锁

**不变量 I-03 的强化**：`after_in_child` 回调中**禁止**：
1. 调用任何 `with ..._lock` / `lock.acquire()` 的方法（如 `snapshot_fds`、`deregister`、`register`、`lookup`）。
2. 调用任何可能阻塞或依赖线程状态的同步原语。

**唯一安全做法**：fork 前（prepare 阶段，父进程仍多线程、可安全持锁）把继承 FD 快照成**不可变 tuple** 存入模块级变量；child 回调只读这个 tuple，不碰任何锁。

### 1.4 prepare / parent / child 三回调职责表（重写）

| 回调 | 时机 | 允许操作 | 禁止操作 |
|------|------|---------|---------|
| `before`（prepare）| fork 前，父进程多线程 | **用锁安全地快照 FD 列表为不可变 tuple**；快照所有需重置的对象引用 | 不得阻塞、不得做 IO 重活 |
| `after_in_parent` | fork 后，父进程 | 无（状态不变）| 不得改动子进程资源 |
| `after_in_child` | fork 后，子进程单线程 | **只读 prepare 快照的 tuple**；`os.close(fd)`；重建锁对象（直接赋值全新 Lock，不 acquire）| **不得获取任何继承锁** |

```python
# prepare 阶段（父进程，可安全持锁）
_fork_fd_snapshot: tuple[int, ...] = ()   # 模块级，prepare 填充

def _before_fork():
    global _fork_fd_snapshot
    _fork_fd_snapshot = tuple(LockHandleRegistry.snapshot_fds())  # 这里持锁是安全的（父进程多线程正常）

# child 阶段（子进程，禁碰锁）
def _after_fork_child():
    # 1. 作废 Lease：直接清空 dict（赋值新对象，不调用带锁方法）
    LeaseRegistry._leases = {}            # 全新 dict，不碰继承锁
    # 2. 关闭继承 FD：只读 prepare 快照的不可变 tuple，不碰锁
    for fd in _fork_fd_snapshot:          # ★ 不调用 snapshot_fds()
        try:
            os.close(fd)                  # 只移除子进程对 OFD 的引用，不 LOCK_UN
        except OSError:
            pass
    _fork_fd_snapshot = ()
    # 3. 重建锁对象（直接赋全新 Lock，不 acquire 继承锁）
    LockHandleRegistry._handles = {}
    LockHandleRegistry._lock = threading.Lock()    # ★ 全新
    ProcessMutex._locks = {}
    ForkGate._fork_in_progress = threading.Lock()  # ★ ForkGate 也重建（不是只重建 ProcessMutex）
    ForkGate._counter_lock = threading.Lock()
    # 4. 重置随机源 / acquisition 计数器
    _reset_acquisition_counter()

os.register_at_fork(before=_before_fork, after_in_parent=None, after_in_child=_after_fork_child)
```

### 1.5 ForkGate 子进程重建（纠正"只重建 ProcessMutex"）

05.12 只重建了 ProcessMutex，**未重建 ForkGate**。若 ForkGate._fork_in_progress 在 fork 时正好被某线程持有，子进程继承的是一把锁死的 ForkGate，后续 `fork_controlled`/`acquire_lock` 会永久阻塞。**必须重建 ForkGate 及其内部所有锁**（§1.4 步骤 3）。

### 1.6 外部 fork 检测与拒绝策略

| 场景 | 策略 |
|------|------|
| 经 `fork_controlled` 的 fork | 安全（prepare 已快照，child 已清理）|
| 外部代码直接 `os.fork()` | **检测**：child 回调 `_after_fork_child` 会因 `_fork_fd_snapshot` 为空（prepare 未执行）而**不关闭任何 fd**，且锁对象仍是继承的 → 子进程后续调用 `acquire_lock` 时 `assert_lock_owned` 会因 registry 状态异常而 DENY |
| 连续 fork / 多次 fork | 每次 prepare/child 都重新快照/重建，幂等 |
| fork 后 exec | 继承 fd 若未设 close-on-exec 则需显式 close；child 回调已 close 锁 fd |
| 子进程异常退出 | OS 释放子进程资源，不影响父进程 |
| 回调异常 | child 回调异常 → 子进程进入 DENY（标记 `_fork_reset_failed`），拒绝后续锁操作 |

**无法证明某同步对象能安全恢复 → 子进程 DENY**：若 `_after_fork_child` 任一步失败，设置 `_fork_reset_failed = True`，之后所有 `acquire_lock`/`assert_lock_owned` 直接 DENY，不继续使用继承的锁系统。

### 1.7 父子进程状态机（重写）

```
[父进程多线程，持锁 L]
   │ fork_controlled → before 快照 FD 为 tuple
   ▼
[父进程继续，L 不变]  +  [子进程单线程]
   │                        │ after_in_child：读 tuple → close fd（不 LOCK_UN）
   │                        │ → 重建全部锁对象（全新 Lock）
   │                        ▼
   │                   [子进程：无有效 lease、全新锁系统]
   │                        │ 重新 acquire_lock
   │                        ▼
   │                   [子进程独立 lease]
```

### 1.8 验收对照

| 验收项 | 结论 |
|--------|------|
| 锁对象完整清单 | ✅ §1.2 |
| fork 前后生命周期矩阵 | ✅ §1.2 |
| 三回调职责表 | ✅ §1.4 |
| child 回调不调用带锁方法 | ✅ §1.3/§1.4（读 tuple 不碰锁）|
| ForkGate 子进程重建 | ✅ §1.5 |
| 外部 fork 检测/拒绝 | ✅ §1.6 |
| 无法证明安全 → DENY | ✅ §1.6 |
| 不声称 register_at_fork 就完成全部管理 | ✅（明确 prepare 快照 + 重建闭环）|

**P0-02 结论：PASS（设计级，child 死锁缺陷已消除）**，register_at_fork/fork 行为 NOT_TESTED。

---

## 2. P0-03｜锁文件稳定身份与跨进程锁一致性

### 2.1 缺陷定位（05.12 §3.5）

05.12 §3.5 的目录可信前提是"目录权限限制为仅授权进程可写（0o700，属主为授权用户）"。但**目录权限无法阻止同一用户下的其他进程替换锁文件**——同 UID 的多个进程都能写这个目录，进程 A 持有 inode-A 的锁后，进程 B 可以 unlink 锁文件再重建（得到 inode-B）并成功加锁，导致 A、B 各持一把锁却都认为自己持有同一把互斥锁。

这是**错误假设**：把"目录权限防跨用户"错误外推到"防同用户多进程替换"。

### 2.2 根本原则：锁对象稳定性不能依赖目录权限

**不变量 I-02 的强化**：所有参与进程必须锁定**同一个稳定锁对象**。稳定性必须由**协议**保证，而非依赖目录权限、文件属性或"通常没人这么做"的假设。

### 2.3 永久锁文件 + 禁止 unlink/replace 协议

引入**永久锁文件**（stable lock file）机制：

| 规则 | 内容 |
|------|------|
| 锁文件生命周期 | **永久**：创建后**永不删除、永不替换、永不 rename** |
| 创建 | 首次 `open(O_CREAT|O_EXCL)` 创建；已存在则仅 open 不加锁 |
| unlink/replace | **协议禁止**，任何参与者不得执行 |
| 身份 | 首次创建后，其 inode（dev+ino）/ volume+index 即**固定**，写入锁文件内容作为稳定标识 |
| 陈旧锁 | 锁文件存在**不等于**锁被持有（进程退出后 OS 释放锁，但文件保留）——严格区分"对象生命周期"与"锁持有状态" |

### 2.4 多进程锁对象一致性证明

**一致性机制（两重保证）：**

1. **协议约束**：所有参与者（本库封装的进程）遵守"永久锁文件、禁 unlink/replace"。因此正常参与者都 open 同一个 inode，加同一把锁。
2. **身份校验 + 失败即全体拒绝**：每个参与者在加锁前后都 `fstat` 锁文件身份，与**首次记录的基准身份**比对；若发现身份变化（说明有人 unlink 重建了），**该进程立即 DENY 并拒绝继续**，而不是"用新 inode 继续加锁"。

**关键：身份变化 = 协议被破坏 = 全体停止**，而非"重新对准新对象"。

### 2.5 同用户多进程竞争场景分析

| 场景 | 分析 | 处理 |
|------|------|------|
| 进程 A 持锁，进程 B 正常 open 同 inode | B 加锁失败（flock/OFD 互斥）→ 等待/超时 | 正常互斥 |
| 进程 B（恶意/绕过协议）unlink 后重建 | B 得到 inode-B，可加锁 | **A 的下次 fstat 发现身份变化 → A DENY 停止**；B 虽拿到"新锁"，但已破坏协议，其行为不可信 |
| 进程 B 硬链接锁文件 | 硬链接同 inode，不改变锁对象 | 无影响 |
| 进程 B 符号链接替换 | open 用 O_NOFOLLOW 拒绝 | 拒绝 |
| 进程 B rename 锁文件 | 身份不变（rename 不改 inode），但路径变了 | 后续 open 需定位新路径；若找不到 → DENY |

### 2.6 锁文件身份变化后的全体停止机制

```
检测到锁文件身份变化（fstat dev+ino != 基准）
  → 该进程立即：
     1. 标记 lease INVALIDATED
     2. 拒绝所有后续提交操作
     3. 记录审计（身份变化事件）
     4. DENY，不尝试重新对准新对象
```

**不变量**：任何参与进程一旦发现锁对象身份变化，**停止并使用 DENY**，绝不"锁新 inode 继续"。这消除了"A 锁 inode-A、B 锁 inode-B 却都认为持同一锁"的路径——因为 A 发现变化后会拒绝，不会假装还持着锁。

### 2.7 POSIX 操作序列（重写）

```
1. open(锁目录, O_DIRECTORY|O_RDONLY) → dirfd
2. fstat(dirfd) 记录目录身份
3. openat(dirfd, lockfile, O_CREAT|O_RDWR|O_NOFOLLOW|O_CLOEXEC, 0o600) → fd
   （首次：O_CREAT|O_EXCL 创建；后续：仅 open）
4. fstat(fd) → 记录 file_identity（dev+ino）
5. 与"基准身份"比对（若首次，则此即基准；若非首次，必须相等，否则 DENY）
6. flock/OFD 加锁（非阻塞 + 超时）
7. fstat(fd) 复核身份未变
8. 构造 Lease，登记 registry
9. 之后所有操作经 fd，加锁期间定期/每次提交前 fstat 复核身份
```

### 2.8 Windows 操作序列（单独定义）

| 步骤 | Windows 机制 |
|------|-------------|
| 打开目录 | `CreateFileW(dir, FILE_LIST_DIRECTORY, ...)` |
| 打开锁文件 | `CreateFileW(path, GENERIC_READ|GENERIC_WRITE, 0, NULL, OPEN_ALWAYS, FILE_FLAG_OPEN_REPARSE_POINT, NULL)` |
| 身份 | `GetFileInformationByHandle` → volume serial + file index（等价 dev+ino）|
| 基准比对 | 与首次记录的 volume+index 比对，不等 → DENY |
| 独占打开 | `dwShareMode=0`（打开阶段排他）|
| 区间锁 | `LockFileEx(LOCKFILE_EXCLUSIVE_LOCK)` |
| 重解析点 | 默认拒绝，除非显式校验 |

**Windows 锁文件同样"永久、禁 delete/replace"**，身份变化（volume+index 变化）→ 全体 DENY。

### 2.9 锁文件替换/删除/重建/硬链接/符号链接故障矩阵

| 故障 | 检测 | 处理 |
|------|------|------|
| 锁文件被 unlink | 下次 fstat 报错（fd 仍指向已 unlink 的 inode）| DENY |
| unlink 后重建 | fstat(fd) 身份 vs 路径身份不一致 | DENY（§2.6）|
| rename | 身份不变但路径变 | 重新定位，找不到 DENY |
| 硬链接 | 同 inode，身份不变 | 无影响 |
| 符号链接 | O_NOFOLLOW 拒绝 | 拒绝 |
| 目录被替换 | fstatat 报错 | DENY |
| 身份变化 | fstat 比对失败 | 全体停止 + DENY |

### 2.10 失败后恢复与拒绝策略

- 身份变化 → 全体停止，审计记录，DENY。
- 锁文件丢失（被 unlink）→ 需人工重建基准身份，重新初始化锁目录；**不得**自动删除"陈旧锁文件"来清理。
- 无法保证所有参与者锁定同一对象 → **DENY**（平台或部署环境不满足永久锁文件协议时）。

### 2.11 验收对照

| 验收项 | 结论 |
|--------|------|
| 不存在 A 锁 inode-A、B 锁 inode-B 却认为同一锁 | ✅ §2.4/§2.6（身份变化全体停止）|
| 不依赖目录权限防同用户 | ✅ §2.2 |
| 永久锁文件 + 禁 unlink/replace | ✅ §2.3 |
| 锁文件存在≠锁被持有 | ✅ §2.3 |
| 同用户多进程竞争分析 | ✅ §2.5 |
| POSIX/Windows 分别定义 | ✅ §2.7/§2.8 |
| 身份变化全体停止 | ✅ §2.6 |
| 不能证明则 DENY | ✅ §2.10 |

**P0-03 结论：PASS（设计级，跨进程锁一致性已证明）**，同用户恶意进程、Windows 身份语义 NOT_VERIFIED。

---

## 3. P0-04｜PREPARED 与 Candidate 发布顺序重构

### 3.1 缺陷定位（05.12 §4.6/§4.7）

05.12 §4.6 的状态机是 `CANDIDATE_DURABLE → PREPARED → COMMITTING`，但 §4.7 恢复矩阵中"发布前崩溃"的恢复动作依赖 PREPARED 已存在；若 PREPARED 的持久化时机与发布动作重叠，就会出现"恢复要求 PREPARED 证据，但崩溃点之前 PREPARED 尚未持久化"的顺序矛盾。

### 3.2 核心原则：PREPARED 必须先于任何发布动作持久化

**不变量 I-06**：PREPARED 必须在**任何**正式发布动作（rename/MoveFileEx）**之前**完成持久化（fsync 文件 + fsync 父目录）。发布前崩溃时，PREPARED 一定已经落盘。

### 3.3 新版 Candidate 提交时序（权威顺序）

```
阶段 A：Candidate 临时文件完整生成（同卷）
  A1. openat(cand_dirfd, .tmp.<random>, O_CREAT|O_EXCL) → 写入完整内容
  A2. fsync(tmp) + close(tmp)
  A3. 重新读回校验 sha256(tmp) == candidate_sha256
  A4. 无覆盖发布 tmp → 正式 candidate（renameat2 RENAME_NOREPLACE / MoveFileEx 不带 REPLACE）
  A5. fsync(cand 父目录)

阶段 B：PREPARED 持久化（★ 先于任何正式文档发布）
  B1. 写入 PREPARED journal（含 §3.4 全字段）
  B2. fsync(PREPARED 文件) + fsync(父目录)
  B3. 复核 PREPARED 已持久化（读回校验）

阶段 C：正式文档发布（在 PREPARED 持久化之后）
  C1. 复制 candidate → 正式文档同目录 tmp
  C2. 校验 sha256(tmp) == formal_doc_target_sha256
  C3. fsync(tmp)
  C4. 无覆盖/受保护 CAS 发布 tmp → 正式文档
  C5. fsync(父目录)
  C6. 复核正式文档身份 + 摘要 + revision

阶段 D：COMMITTED 持久化
  D1. 写入 COMMITTED journal
  D2. fsync + fsync 父目录
```

**关键**：阶段 B（PREPARED）在阶段 C（发布）之前，且两者之间无重叠。发布前任何崩溃点，PREPARED 已持久化。

### 3.4 事务绑定字段（全量）

| 字段 | 说明 |
|------|------|
| transaction_id | 唯一事务 ID |
| authorization_id | 授权 ID |
| candidate_identity | 正式 Candidate 身份（dev+ino / volume+index）|
| candidate_sha256 | Candidate 内容 SHA-256 |
| old_revision | 旧 revision |
| target_revision | 目标 revision |
| formal_doc_target_sha256 | 正式文档目标摘要 |
| **commit_state** | 提交阶段（CANDIDATE_DURABLE / PREPARED / COMMITTING / COMMITTED）|

### 3.5 PREPARED 状态机（合法转换表）

```
CREATED → CANDIDATE_DURABLE → PREPARED → COMMITTING → COMMITTED
                                          ├→ ABORTED
                                          └→ INDETERMINATE
```

| 转换 | 触发 | 持久化点 | 前置证据 |
|------|------|---------|---------|
| CREATED→CANDIDATE_DURABLE | 阶段 A 完成 | fsync candidate + 父目录 | candidate_sha256 |
| CANDIDATE_DURABLE→PREPARED | 阶段 B 完成 | fsync PREPARED + 父目录 | candidate 完整 |
| PREPARED→COMMITTING | 开始阶段 C | — | PREPARED 已持久化 |
| COMMITTING→COMMITTED | 阶段 D 完成 | fsync COMMITTED + 父目录 | 正式文档已发布 |
| COMMITTING→ABORTED | 显式回滚 | fsync ABORTED | — |
| 任意→INDETERMINATE | 无法判定 | 保留证据 | — |

### 3.6 双事务竞争矩阵（防止丢失更新）

| 场景 | 处理 |
|------|------|
| 两事务同时发布同一正式文档 | 受保护 CAS（§3.7）串行，后到者 revision 冲突 → DENY |
| 两事务同 transaction_id | DENY（事务 ID 冲突）|
| 目标被抢先创建 | 无覆盖原语失败 → DENY |
| 丢失更新 | 通过"PREPARED 绑定 old_revision + 发布时 CAS 校验 revision 单调"防止 |

### 3.7 revision 与 CAS 一致性（不得只"先读后写"）

正式文档更新采用**受保护 CAS**：在持有授权锁的前提下，"读当前 revision → 校验 == old_revision → 无覆盖发布 → 写 COMMITTED"是一个**不可分割的持锁段**。发布原语保证目标已存在（revision 已被他人更新）时不覆盖 → CAS 失败。

```
assert_lock_owned(lease)                     # 持授权锁
cur_rev = read_formal_doc_revision()
if cur_rev != prepared.old_revision:
    return CAS_FAILED                        # 他人已更新 → DENY，不覆盖
无覆盖发布 tmp → formal_doc                  # 目标已存在 → 失败
验证 formal_doc revision == target_revision
写 COMMITTED
```

### 3.8 崩溃恢复矩阵（每个证据在崩溃点之前已生成）

| 崩溃点 | 恢复动作 | 恢复所需证据 | 证据生成时机 |
|--------|---------|-------------|-------------|
| 阶段 A 中（tmp 半写）| 删 tmp 重做 | 无 | — |
| 阶段 A 后、PREPARED 前 | 重新生成 candidate | candidate_sha256 | 阶段 A 已生成 |
| 阶段 B 中（PREPARED 半写）| 删半写 PREPARED 重写 | — | — |
| 阶段 B 后、发布前 | 继续阶段 C | PREPARED（已持久化）| 阶段 B ✓ |
| 阶段 C 中（发布半途）| 校验后重判 | PREPARED + candidate | 阶段 A/B ✓ |
| 阶段 C 后、COMMITTED 前 | 校验正式文档 → 补 COMMITTED | PREPARED + 正式文档 | 阶段 B + C ✓ |
| 阶段 D 中 | 删半写 COMMITTED 重写 | — | — |
| 目标被抢先创建 | DENY | 目标已存在 | — |

**关键保证**：每个恢复分支所需的证据（PREPARED、candidate、正式文档）都在对应崩溃点**之前**已经持久化，不存在"要求证据但证据尚未生成"的状态。

### 3.9 恢复不认领无法证明归属的文件

恢复程序**不得仅凭 Candidate 存在或 SHA256 相同就认领**，必须校验 transaction_id + candidate_identity + candidate_sha256 + old_revision + target_revision + formal_doc_target_sha256 全量匹配（§3.4）。无法证明归属 → DENY，不覆盖其他事务结果。

### 3.10 验收对照

| 验收项 | 结论 |
|--------|------|
| PREPARED 先于发布持久化 | ✅ §3.2/§3.3（阶段 B 先于 C）|
| 恢复证据先于崩溃点生成 | ✅ §3.8 |
| 事务绑定全字段含 commit_state | ✅ §3.4 |
| 双事务竞争防丢失更新 | ✅ §3.6/§3.7 |
| 不依赖"先读后写" | ✅ §3.7（受保护 CAS）|
| 恢复不认领他人文件 | ✅ §3.9 |

**P0-04 结论：PASS（设计级，PREPARED 顺序矛盾已消除）**，CAS/无覆盖发布语义 NOT_VERIFIED。

---

## 4. P0-05｜Quarantine 源对象内容一致性与事务绑定

### 4.1 缺陷定位（05.12 §5.3~5.5）

05.12 §5.5 的复制一致性检查是"复制前后 sha256(src) 比对"，但 §5.3/§5.4 中源对象的身份（dev+ino）被当成了"内容不变"的等价物。**inode 身份相同 ≠ 内容相同**——源文件可以在 inode 不变的情况下被并发改写（open 后 write，dev+ino 都不变）。因此"复制的是 H1，最后处理的却是 H2"的路径在 05.12 中未被彻底阻断。

### 4.2 核心原则：身份相同 ≠ 内容相同

**不变量 I-09/I-10 的强化**：quarantine 副本必须与源对象**内容**（而非仅身份）建立可验证关系；源对象处理前必须证明其**内容**仍符合预期。

### 4.3 源对象一致性协议（双哈希 + 内容校验）

```
阶段 Q1：QUARANTINE_INTENT 持久化（绑定源对象身份 + 源内容 SHA256）
  intent 字段：quarantine_id / transaction_id / source_identity(dev+ino)
              / source_sha256 / target_dir / operation_type=QUARANTINE
  fsync(intent) + fsync(父目录)

阶段 Q2：复制前校验
  读取 src 内容 → sha256_before = sha256(src)
  校验 sha256_before == intent.source_sha256（不一致 → 源已变化 → DENY）

阶段 Q3：复制 src → qtmp
  fsync(qtmp)

阶段 Q4：复制后校验（★ 关键，阻断 H1→H2）
  读取 qtmp 内容 → sha256_qtmp = sha256(qtmp)
  再次读取 src 内容 → sha256_after = sha256(src)
  校验：sha256_qtmp == sha256_after == intent.source_sha256
  （三者必须相等，否则 DENY，不发布 qfinal）

阶段 Q5：发布 qtmp → qfinal（无覆盖）
  fsync(quarantine 父目录)

阶段 Q6：COMPLETE 记录（绑定 quarantine_id + qfinal 身份 + qfinal sha256）
  fsync + fsync 父目录

--- 源对象处理是独立阶段（见 §4.5）---
```

### 4.4 复制期间并发修改分析

| 场景 | 检测 | 处理 |
|------|------|------|
| 复制前源已被改 | Q2 校验 sha256 != intent → DENY | 不复制 |
| 复制中源被改（复制的是 H1，源变 H2）| Q4 校验 sha256_qtmp(H1) != sha256_after(H2) → **DENY** | 不发布 qfinal，不处理源 |
| 复制后源被改（qtmp=H1=源当时内容，之后源变 H2）| Q4 时若源仍=H1 则通过；之后源变 H2 → 源处理阶段（§4.5）再次校验拦截 | 见 §4.5 |
| 外部进程不配合（绕过协议写源）| 无法阻止，但内容校验会暴露不一致 → DENY/INDETERMINATE | 不处理源 |

**关键**：Q4 的"qtmp 内容 == 复制后源内容 == intent"三方校验，确保了"复制的是 H1，发布的是 H1，intent 记录的也是 H1"。若复制期间源被改成 H2，则 Q4 校验失败，阻断。

### 4.5 quarantine 与 source-retire 双阶段状态机

05.12 用含义模糊的 QUARANTINE_INTENT 同时表达"隔离副本"和"处理源对象"，混淆。05.13 拆为**两个独立状态机**：

**阶段一：Quarantine（生成隔离副本）**
```
ACTIVE → QUARANTINE_INTENT → QFINAL_DURABLE → QUARANTINED
                                     └→ CORRUPT / INDETERMINATE
```

**阶段二：Source Retire（处理源对象，独立 intent）**
```
QUARANTINED → SOURCE_RETIRE_INTENT → SOURCE_RETIRED
                              └→ INDETERMINATE
```

**SOURCE_RETIRE_INTENT**（独立持久化，不复用 QUARANTINE_INTENT）绑定：
- retire_id（独立 ID）
- quarantine_id（关联）
- source_identity + source_sha256（处理前必须再校验，见 §4.6）

### 4.6 源对象处理前的内容再校验

进入 Source Retire 阶段前，**必须再次校验源对象内容**：

```
写 SOURCE_RETIRE_INTENT 前：
  再次 sha256(src)
  校验 sha256(src) == intent.source_sha256（== qfinal 内容）
  不一致 → 源对象内容已变 → INDETERMINATE，不处理源
一致 → 写 SOURCE_RETIRE_INTENT → rename(src → src.retired) → fsync
```

### 4.7 qtmp/qfinal/source/Journal 四方绑定规则

| 规则 | 校验 |
|------|------|
| qtmp.sha256 == source_sha256 | Q4 |
| qfinal.sha256 == qtmp.sha256 | Q5 后 |
| source 处理前 sha256 == qfinal.sha256 | §4.6 |
| 四方绑定同一 quarantine_id | intent 字段 |
| 身份相同 ≠ 内容相同 | 全程内容哈希校验 |

### 4.8 崩溃恢复矩阵

| 崩溃点 | 恢复动作 |
|--------|---------|
| Q1 intent 写前 | 无副作用，可重来 |
| Q1 intent 写中 | 删半写重写 |
| Q2/Q4 校验失败 | DENY（源已变化）|
| Q3 复制中（qtmp 半写）| 删 qtmp 重做 |
| Q4 后、Q5 发布前 | qtmp 完整、无 qfinal → 重新发布（qtmp 已通过三方校验）|
| Q5 发布后、Q6 COMPLETE 前 | qfinal 存在 + intent 存在 → 补 COMPLETE |
| 源处理阶段（retire）崩溃 | SOURCE_RETIRE_INTENT 先持久化 → 恢复按 intent 重判 |
| qfinal 孤立（有 qfinal 无 intent）| INDETERMINATE，不覆盖不删除 |
| 源内容变化（处理前校验失败）| INDETERMINATE |

### 4.9 无法证明源内容一致的拒绝策略

- 源对象内容与 intent/qfinal 不一致 → **DENY/INDETERMINATE，不处理源对象**。
- 无法阻止外部写入 + 无法证明快照一致性 → **不处理源对象，INDETERMINATE**。
- 不把"身份相同"当作"内容相同"的证据。

### 4.10 验收对照

| 验收项 | 结论 |
|--------|------|
| 不出现复制 H1 处理 H2 | ✅ §4.3 Q4 三方校验 + §4.6 处理前再校验 |
| 身份相同 ≠ 内容相同 | ✅ §4.2 |
| 源对象一致性协议 | ✅ §4.3 |
| 复制期间并发修改分析 | ✅ §4.4 |
| 双阶段状态机 | ✅ §4.5 |
| 四方绑定 | ✅ §4.7 |
| 崩溃恢复矩阵 | ✅ §4.8 |
| 无法证明 → DENY/INDETERMINATE | ✅ §4.9 |

**P0-05 结论：PASS（设计级，H1→H2 路径已阻断）**，外部写入场景 NOT_VERIFIED。

---

## 5. P0-06｜Tombstone 删除归因证据重构

### 5.1 缺陷定位（05.12 §6.5~6.7）

05.12 §6.5 的恢复矩阵中，"unlink 后、REMOVED 前崩溃 → UNLINK_INTENT 已持久化 → 补写 REMOVED"。但 **UNLINK_INTENT 只证明了"删除意图"，不能证明"unlink 确实由本事务执行"**。外部进程可能在 UNLINK_INTENT 持久化后、本事务 unlink 前删除了 tombstone——此时"UNLINK_INTENT 存在 + tombstone 不存在"被错误归因为本事务删除成功。

这是**归因过度**：把删除意图冒充删除事实。

### 5.2 核心原则：删除意图 ≠ 删除事实

**不变量 I-11/I-12 的强化**：删除归因必须具有**真实、可恢复的证据**，不能仅凭"UNLINK_INTENT 存在 + 对象不存在"推断本事务删除成功。

### 5.3 关键认识：底层文件系统无法提供可恢复的删除动作归因

`unlink`/`DeleteFile` 是**不可归因的破坏性操作**——文件系统不记录"是哪个进程删除了哪个 inode"。一旦 unlink 完成，tombstone 消失，**没有任何底层机制能证明"是本事务执行的 unlink"还是"外部进程删的"**。

因此必须**承认归因的边界**，而非伪造确定性。

### 5.4 新版 tombstone 状态机（区分意图与事实）

```
PENDING → DELETE_INTENT → TOMBSTONED → UNLINK_INTENT → [unlink] → REMOVED_BY_THIS_CLEANUP → CLEANED
                                                        │
                                                        └→ INDETERMINATE（unlink 后无法证明是本事务删的）
```

**关键改变**：`unlink` 这一步之后，**不再直接进入 REMOVED_BY_THIS_CLEANUP**，而是进入一个需要**归因判定**的节点：

| 状态 | 证据 | 含义 |
|------|------|------|
| DELETE_INTENT | intent 记录 fsync | 意图持久化 |
| TOMBSTONED | tombstone 记录 fsync | 已 rename 到 tombstone |
| UNLINK_INTENT | unlink 意图 fsync | 即将 unlink |
| （unlink 动作）| 无持久化证据 | **不可归因的破坏性动作** |
| REMOVED_BY_THIS_CLEANUP | 需额外证据（§5.6）| 仅当有真实归因证据 |
| ALREADY_ABSENT | 无本事务 intent | 执行前已不存在 |
| INDETERMINATE | 证据不足 | 无法归因 |
| CLEANED | 全部终态 + 归档 | — |

### 5.5 UNLINK 前后崩溃恢复矩阵（诚实处理归因边界）

| 崩溃点 | 恢复判定 | 依据 |
|--------|---------|------|
| UNLINK_INTENT 写前、tombstone 存在 | TOMBSTONED，可继续 | tombstone 在 |
| UNLINK_INTENT 后、unlink 前 | 继续 unlink | 意图已持久化 |
| **unlink 后、REMOVED 前崩溃** | **INDETERMINATE**（除非有 §5.6 的归因证据）| 无法证明是本事务 unlink |
| unlink 后、REMOVED 前、且 tombstone 被外部删除 | **INDETERMINATE** | 外部删除与本事务 unlink 无法区分 |

**诚实结论**：在无额外归因机制时，`unlink 后 + REMOVED 前` 的崩溃恢复**不能**补写 REMOVED_BY_THIS_CLEANUP，只能 INDETERMINATE。

### 5.6 是否存在可靠的删除动作归因机制

| 机制 | 是否可行 | 说明 |
|------|---------|------|
| 依赖 unlink 自身 | 否 | 文件系统不记录删除者 |
| tombstone 内容写入"删除令牌" | 否 | unlink 后令牌随 tombstone 消失 |
| 先写"删除完成"再 unlink | 否 | 顺序反了，unlink 可能失败 |
| **降低结论等级** | **是** | 承认无法可靠归因，用 INDETERMINATE |

**结论：本设计不声称存在可靠的 unlink 动作归因机制。** 因此：

- 若采用"先 rename 到 tombstone → 再 unlink"方案，则 **unlink 之后到 REMOVED 之前的窗口，恢复只能 INDETERMINATE**。
- 若要获得确定的 REMOVED_BY_THIS_CLEANUP，必须**改变删除方式**：不直接 unlink，而是**保留 tombstone 不删**（tombstone 作为"已删除"的持久化标记，永久保留或延迟到审计归档后批量清理）。这样"tombstone 存在"就是删除完成的持久化证据，无需 unlink 归因。

### 5.7 推荐方案：tombstone 不 unlink（保留为删除证据）

**消除归因困境的确定性方案**：tombstone **不 unlink**，而是永久保留（或延迟到 CLEANED 后批量清理，且清理本身也走 tombstone 协议）。

```
PENDING → DELETE_INTENT → TOMBSTONED → REMOVED_BY_THIS_CLEANUP → CLEANED
```

- `TOMBSTONED` 状态：对象已 rename 到 tombstone，**tombstone 的存在本身就是"对象已从正式路径移除"的持久化证据**。
- 不再有 unlink 步骤 → 不再有"unlink 后无法归因"的窗口。
- `REMOVED_BY_THIS_CLEANUP` 的判定：tombstone 存在 + 身份/哈希匹配 + REMOVED 记录已持久化。三者都有持久化证据，无归因困境。

**代价**：tombstone 占用磁盘。处理：CLEANED 后由独立归档/回收流程处理 tombstone（同样走"先 rename 到 archive → 记录"的协议，不直接 unlink）。

### 5.8 三态归因决策表（诚实版）

| 观测 | 判定 |
|------|------|
| 对象存在 + 无本事务 intent | 未开始，不归因 |
| 对象存在 + DELETE_INTENT | 待删 |
| tombstone 存在 + TOMBSTONED | 已 rename，待确认 |
| tombstone 存在 + 身份/哈希匹配 + REMOVED | **REMOVED_BY_THIS_CLEANUP** |
| 对象不存在 + 无任何 intent | **ALREADY_ABSENT** |
| 对象不存在 + 有 intent 但 tombstone 也消失 | **INDETERMINATE**（不能证明是本事务删除）|
| 身份/哈希不匹配 | INDETERMINATE |
| 证据不足 | INDETERMINATE |

### 5.9 终态审计快照：记录实际证据，不记推断

`TerminalStateSnapshot` 记录**实际观测到的证据**（tombstone 身份、哈希、REMOVED 记录），**不记录推断**（如"本事务已 unlink"）。快照字段见 §5.4 状态定义，每个字段对应一个已持久化的证据文件。

### 5.10 验收对照

| 验收项 | 结论 |
|--------|------|
| 删除意图不冒充删除事实 | ✅ §5.2 |
| unlink 后无法归因 → INDETERMINATE | ✅ §5.5/§5.6 |
| 不补写 REMOVED 以获得确定结论 | ✅ §5.5 |
| 对象不存在 ≠ 本次删除成功 | ✅ §5.8 |
| 三态严格区分 | ✅ §5.8 |
| tombstone 不 unlink 方案 | ✅ §5.7 |
| 审计快照记实际证据 | ✅ §5.9 |

**P0-06 结论：PASS（设计级，归因过度已纠正）**，tombstone 保留方案、并发删除 NOT_VERIFIED。

---

## 6. 设计不变量集中证明（I-01～I-15）

| 编号 | 不变量 | 证明章节 |
|------|--------|---------|
| I-01 | 没有有效锁不得执行受保护操作 | P0-02 §1.6（子进程 DENY）|
| I-02 | 所有进程锁定同一稳定锁对象 | P0-03 §2.3/§2.4 |
| I-03 | fork 后子进程不使用失效继承同步对象 | P0-02 §1.3/§1.4 |
| I-04 | 子进程清理不解除父进程 OFD 锁 | P0-02 §1.4（close 不 LOCK_UN）|
| I-05 | 正式 Candidate 不暴露半成品 | P0-04 §3.3 阶段 A |
| I-06 | PREPARED 先于正式发布持久化 | P0-04 §3.2/§3.3 |
| I-07 | 正式目标不被其他事务覆盖 | P0-04 §3.7 CAS |
| I-08 | 恢复不认领无法证明归属的文件 | P0-04 §3.9 |
| I-09 | quarantine 副本与源内容及事务可验证 | P0-05 §4.3/§4.7 |
| I-10 | 源对象处理前证明内容符合预期 | P0-05 §4.6 |
| I-11 | 删除归因有真实可恢复证据 | P0-06 §5.6/§5.7 |
| I-12 | 对象不存在 ≠ 本次删除成功 | P0-06 §5.8 |
| I-13 | 无法证明 → DENY/INDETERMINATE | 贯穿全部恢复矩阵 |
| I-14 | 设计级通过 ≠ 运行时验证 | 全文 NOT_VERIFIED/NOT_TESTED |
| I-15 | 平台保证限定到平台/版本/API | P0-03 §2.8、P0-05 §4.9 |

---

## 7. 端到端交叉审查

见 CROSS-REVIEW.md（八条链路逐条推演）。

核心结论预览：P0-02~P0-06 复用 P0-01 锁接口，未引入第二套锁语义；五项修复相互引用前序确定契约；无跨章节矛盾。

---

## 8. P1 与既有阻断项状态

| 编号 | 状态 |
|------|------|
| P1-01 锁文件替换/TOCTOU | 设计级闭合（P0-03），运行时 NOT_VERIFIED |
| P1-02 analyze_file 行为等价 | NOT_TESTED |
| P1-03 A/B 三文件合并 | 保持原状态，不执行 |
| P0-BLOCK-1 存储事务实现+故障注入 | 未解决 |
| P0-BLOCK-2 analyze_file 一致性 | 未解决 |
| P1-BLOCK-3 A/B 三文件分叉 | 未解决 |
| F1 / P0.8 / F2 | 不解除 |
| IMPLEMENT-01 | STOP |

---

## 9. IMPLEMENT-01 准入结论

**PASS-WITH-ISSUES（维持阻断）。**

五项 P0 缺陷已消除：

| 编号 | 缺陷 | 修复结果 |
|------|------|---------|
| P0-02 | child 回调死锁 | 回调读不可变 tuple，禁碰锁，ForkGate 重建 |
| P0-03 | 跨进程锁 inode 不一致 | 永久锁文件 + 身份变化全体 DENY |
| P0-04 | PREPARED 顺序矛盾 | PREPARED 先持久化再发布 |
| P0-05 | 身份相同≠内容相同 | 双哈希三方校验 + SOURCE_RETIRE_INTENT |
| P0-06 | 删除意图冒充事实 | tombstone 不 unlink，归因边界诚实处理 |

但均为设计级契约，运行时 NOT_VERIFIED / NOT_TESTED。三个阻断项未解决。

**本工单不授权实现、不授权测试、不解除 IMPLEMENT-01 STOP。F1/P0.8/F2 未解除。三个阻断项未解除。**

---

## 10. 交付物索引

| 文件 | 内容 |
|------|------|
| `USIT-STAGE-B-DESIGN-05.13.md` | 本文（主设计）|
| `USIT-STAGE-B-DESIGN-05.13-ISSUES.md` | 问题清单 |
| `USIT-STAGE-B-DESIGN-05.13-DIFF.md` | DESIGN-05.12 → 05.13 差异 |
| `USIT-STAGE-B-DESIGN-05.13-CROSS-REVIEW.md` | 端到端交叉审查记录 |
| `USIT-STAGE-B-DESIGN-05.13-EXECUTION-REPORT.md` | 执行报告 |
| `USIT-STAGE-B-DESIGN-05.13-HASH-MANIFEST.md` | 哈希清单 |
| `USIT-STAGE-B-DESIGN-05.13-HASH-RECEIPT.txt` | 哈希收据 |
| `USIT-STAGE-B-DESIGN-05.13-交付物.zip` | 七份文档打包 |
