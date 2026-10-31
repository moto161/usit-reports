# USIT-STAGE-B-DESIGN-05.18｜Fork协议重构＋Sealed退役协议重构＋持久化与Tombstone补强

> 阶段 B · DESIGN-05.18 · READ-ONLY DESIGN REVISION ONLY
> 作者：USIT 主开发 · 生成：2026-10-31（GMT+8）
> 前置审查：DESIGN-05.17 独立复审 FAIL
> 工作区：`C:\Users\Administrator\WorkBuddy\2026-10-15-22-09-47\usit_p0`
> 交付目录：`C:\Users\Administrator\WorkBuddy\2026-10-15-22-09-47\usit_p0_05.18_reports`（**项目外**）
> 状态：**STOP — 等待独立复审**（未授权 IMPLEMENT-01）

---

## 0. 缺陷追踪表（缺陷→历史证据→05.17失败原因→05.18修订→安全证明→反例→拒绝策略→验收）

| # | 历史证据 | 05.17 失败原因 | 05.18 修订位置 | 安全证明 | 反例 | 拒绝策略 | 验收 |
|---|---------|---------------|---------------|---------|------|---------|------|
| P0-02 | 05.17 §1.2 用 `os._fork()` 持锁 fork | 绕过 CPython 生命周期后未证明运行时恢复安全；未证明主线程/子解释器约束 | §1 | 承认无法证明 os._fork 后 Python 运行时安全 → 改为**不绕过 CPython** 的受控 fork | 持锁 fork 后 child 执行 Python 崩溃 | 无法证明则 DENY | DESIGN_PARTIAL |
| P0-05 | 05.17 §3.2 声称 `+i` 保持可 rename | **chattr(1) 权威文档证实 `+i` 阻止 rename/delete**；且 BUGS 节证实 `+i` **不阻止已 open 写 FD** | §3 | 推翻 `+i` 方案，Source Retire 在所有本地平台 DENY | 旧 FD 写 / rename 被 +i 拒 | 默认 DENY | DESIGN_PARTIAL→DENY |
| P0-04 | 05.17 §2.2 缺目录项 fsync 分步 | 未同步 Candidate/PREPARED 所在目录的目录项 | §2 | 补目录项 fsync + 分步崩溃矩阵 | 目录项未持久化丢失 | 强制 fsync 目录 | DESIGN_PROVEN（POSIX）|
| P0-06 | 05.17 §4.4 ReplaceFileW 部分失败未定义 | 假设失败时占位必然在原位，未按错误码分析 | §4 | 按错误码矩阵 + 恢复状态机 | 部分失败状态不确定 | INDETERMINATE | DESIGN_PARTIAL |

**本节是全文总纲。本轮核心结论：P0-05 的 `chattr +i` 方案被权威文档推翻，Source Retire 在所有本地文件系统平台必须 DENY；P0-02 的 `os._fork` 方案无法证明 CPython 运行时安全，降级为 DESIGN_PARTIAL。**

---

## 1. P0-02｜Fork / FD 生命周期协议重新设计

### 1.1 缺陷定位：05.17 用 `os._fork()` 绕过 CPython 生命周期，未证明安全

05.17 §1.2 用 `os._fork()` 在持锁窗口内调用裸 fork，以规避 `register_at_fork` 回调的重入死锁。**但这一方案有一个未被 05.17 证明的根本问题**：

`os._fork()` 是 Python 暴露的**底层 fork 系统调用**，它**不执行** CPython 的 fork 生命周期钩子。具体地说，CPython 的 `PyOS_BeforeFork` / `PyOS_AfterFork_Parent` / `PyOS_AfterFork_Child` 是**在 `os.fork()` 内部调用**的（通过 `os.register_at_fork` 注册的钩子，以及 CPython 内建的运行时锁重置逻辑）。

- `os.fork()`：调用 `PyOS_BeforeFork`（获取 GIL、获取 import 锁等）→ `fork()` → 父进程 `PyOS_AfterFork_Parent` / 子进程 `PyOS_AfterFork_Child`（**重置 threading 锁、random seed、import 锁等**）。
- `os._fork()`：**跳过这些钩子**，直接调 `fork()`。

**结论（不再回避）**：`os._fork()` 之后，子进程的 CPython 运行时状态（尤其是 threading 模块的内部锁、import 锁、GIL 相关状态）**没有经过 `PyOS_AfterFork_Child` 的重置**。在此状态下继续执行 Python 代码（包括本库的 child 重建逻辑）**是不安全的**——可能死锁、崩溃或数据竞争。**05.17 未证明这一点，其 PASS 结论不成立。**

### 1.2 `os.fork()` vs `os._fork()` 的实际差异（依据 CPython 源码语义）

| 维度 | `os.fork()` | `os._fork()` |
|------|------------|--------------|
| 触发 `PyOS_BeforeFork` | 是 | 否 |
| 触发 `PyOS_AfterFork_Parent` | 是 | 否 |
| 触发 `PyOS_AfterFork_Child` | 是（重置 runtime 锁）| 否 |
| 触发 `register_at_fork` 用户钩子 | 是 | 否 |
| child 内继续执行 Python 的安全边界 | 经 runtime 重置后相对安全 | **未重置，不安全** |
| 用途 | 正常 fork | 仅用于"fork 后立即 exec，不再执行 Python"的场景 |

**关键**：`os._fork()` 的合法用途是"fork 后立即 `os.exec*`，不在 child 中执行任何 Python 字节码"。而 05.17 用 `os._fork()` 后**继续在 child 中执行 Python 重建逻辑**（§1.4），这违反了 `os._fork()` 的安全边界。

### 1.3 05.18 的修正：不绕过 CPython 生命周期

**核心决策：放弃"持锁调用裸 fork"的思路，回到 `os.fork()`，接受其生命周期管理，重新设计锁协调以避免重入死锁。**

**重入死锁的根源**：05.17 之所以要用 `os._fork()`，是因为持锁调用 `os.fork()` 时，`before` 钩子会再尝试获取同一把锁 → 重入死锁。

**05.18 的解法：`before` 钩子不获取任何锁，锁协调在 `before` 之外完成。**

### 1.4 受控 fork 协议（05.18，完整）

```
fork_controlled 入口：
0. 线程身份检查：必须是主解释器主线程（否则 DENY，见 §1.5）
1. 进入 fork 协调（获取 ForkGate 的协调锁 _fork_coord_lock）
2. 阻止新 fork（ForkGate 计数 +1）
3. 等待在途 fork 完成（若已有 fork 在进行 → 阻塞）
4. 注册/准备 before 快照：
   - 在 _fork_coord_lock 保护下，枚举 FD Registry（持 _fd_lock 短暂获取）
   - 生成冻结快照 tuple：[(fd, generation, open_file_id, state)]
   - 释放 _fd_lock（快照已生成，不可变 tuple）
5. 释放 _fork_coord_lock
6. 调用 os.fork()                       # 正常 fork，触发 CPython 生命周期
   - before 钩子（register_at_fork）：只读快照 tuple，不获取任何锁
   - PyOS_BeforeFork → fork → PyOS_AfterFork_Parent/Child
7. parent：恢复 ForkGate 计数 -1，返回 child pid
8. child（经 PyOS_AfterFork_Child 重置后）：
   - 直接赋值全新锁对象（不 acquire 继承锁）
   - 遍历快照 tuple，generation 匹配才 close
   - 设置 _fork_reset_ok
```

**关键改变**：
- 用 `os.fork()`（不是 `os._fork()`），让 CPython 完成运行时重置；
- `before` 钩子**只读快照 tuple，不获取任何锁**，因此不会与持锁状态冲突；
- 冻结快照在**步骤 4** 用 `_fd_lock` 短暂保护生成，快照是**不可变 tuple**，之后无需再持锁。

### 1.5 主解释器主线程约束（强制）

**CPython 官方文档（os.fork 的可用性说明与 CPython C API）明确**：在多线程程序中，`fork()` 只保证**调用 fork 的线程**在子进程中存活，其他线程在子进程中**消失**，但它们持有的锁可能处于锁定状态。因此：

- **受控 fork 只允许在主解释器、主线程调用**；
- 若检测到当前不是主线程，或存在子解释器 → **DENY**（不 fork）。

**不能满足此约束时 DENY**，不声称"多线程 fork 安全"。

### 1.6 锁顺序（覆盖所有实际参与锁）

```
L0. _fork_coord_lock（ForkGate 协调锁）
L1. _fd_lock（FD Registry）
L2. _gate_lock（ForkGate 内部锁）
L3. _lock（LockHandleRegistry）
L4. _lease_lock（Lease）
L5. _pmutex_lock（ProcessMutex）
```

**规则**：
1. before 钩子**不获取任何锁**（消除重入）；
2. fork 协调（步骤 1~4）只短暂获取 `_fork_coord_lock` 和 `_fd_lock`，且**顺序固定**（coord → fd），不嵌套其他锁；
3. 禁止持锁 fork（任何持有 L2~L5 的线程不得 fork）；
4. 其他锁获取严格升序。

**无死锁证明**：before 无锁 + 协调锁获取顺序固定 + 禁止持锁 fork + 其余升序 → 无循环等待。

### 1.7 fork 来源与保证边界矩阵

| 来源 | 保证 | 结论 |
|------|------|------|
| 本库受控 fork（主线程）| CPython 生命周期 + 协调 + 快照 | DESIGN_PROVEN（协议层）|
| 本库受控 fork（非主线程）| 无法保证 | DENY |
| Python `os.fork()`（直接）| CPython 生命周期 + before 只读钩子 + child self-check | DESIGN_PARTIAL |
| 第三方 C `fork()` | 不触发 atfork | DENY |
| 其他库/运行时触发 fork | 无法追踪 | DENY |
| 外部进程 fork | 不适用 | 不适用 |

### 1.8 8 个最小反例

| # | 反例 | 结论 |
|---|------|------|
| R1 | os._fork 后 child 执行 Python（threading 锁未重置）| 05.17 方案缺陷，05.18 改用 os.fork |
| R2 | before 钩子持锁导致重入死锁 | 05.18 before 无锁 |
| R3 | 非主线程 fork，其他线程锁消失 | DENY |
| R4 | 子解释器 fork | DENY |
| R5 | 快照后 FD 数字复用 | generation 绑定 |
| R6 | 第三方 C fork 绕过 | DENY |
| R7 | 连续 fork 状态未恢复 | ForkGate 计数 + 幂等 |
| R8 | child 回调异常 | fail-closed |

### 1.9 12 条并发与 fork 时间线

| # | 场景 | 推演 | 结果 |
|---|------|------|------|
| F1 | 主线程受控 fork | 协调→快照→os.fork→child 重置 | 安全 |
| F2 | 非主线程 fork | 线程检查失败 | DENY |
| F3 | before 只读快照 | 不取锁，无重入 | 安全 |
| F4 | 快照后 FD close | generation 失效 | 不误关 |
| F5 | FD 数字复用 | 新 generation | 不误关 |
| F6 | fork 前新建 FD | 协调锁保护 | 快照后阻塞 |
| F7 | fork 失败 | 恢复协调态 | 安全 |
| F8 | parent 回调异常 | 审计，不影响结果 | 安全 |
| F9 | child 回调异常 | _fork_reset_ok=False | DENY |
| F10 | 连续 fork | 幂等 | 安全 |
| F11 | 第三方 C fork | 绕过 | DENY |
| F12 | 进程退出 | OS 释放 | 安全 |

### 1.10 CPython 生命周期调用顺序图

```
os.fork() 内部：
  PyOS_BeforeFork()
    ├─ 获取 GIL
    ├─ 获取 import 锁
    ├─ 调用 register_at_fork 的 before 钩子（本库：只读快照，无锁）
    └─ 其他运行时准备
  fork() 系统调用
  父进程：PyOS_AfterFork_Parent()
    ├─ 释放 import 锁 / GIL
    └─ 调用 register_at_fork 的 parent 钩子（本库：恢复计数）
  子进程：PyOS_AfterFork_Child()
    ├─ 重置 GIL（新进程）
    ├─ 重置 threading 内部锁
    ├─ 重置 import 锁
    ├─ 重置 random seed
    └─ 调用 register_at_fork 的 child 钩子（本库：重建锁对象 + close FD）
```

### 1.11 FD 完整生命周期状态图

```
创建 → 登记(ACTIVE) → 使用 → 关闭(CLOSING→CLOSED) → 注销(移除)
                           ↘ fork冻结(快照tuple) → parent恢复(ACTIVE)
                                                   ↘ child重建(close匹配FD + 重建锁)
exec → 关闭非CLOEXEC → 退出
进程退出 → OS 释放全部 FD
```

### 1.12 P0-02 验收与结论

- 12 条时间线：✅ §1.9
- 8 个反例：✅ §1.8
- fork 来源矩阵：✅ §1.7
- CPython 生命周期调用图：✅ §1.10
- FD 生命周期状态图：✅ §1.11
- 不绕过 CPython：✅ §1.3（改用 os.fork）

**P0-02 结论：DESIGN_PARTIAL**。
- 协议层（协调 + 快照 + generation + 主线程约束 + before 无锁）可证明无死锁、无误关。
- **但**：CPython 多线程 fork 的运行时行为（threading 锁重置、子解释器、GIL）依赖 CPython 实现细节，**未在运行时验证**，标记 NOT_TESTED。
- 非主线程 fork、子解释器 fork、第三方 C fork：**DENY**。
- 不声称"整个 Python 运行时安全"，只声称"本库受控 fork 协议层安全"。

---

## 2. P0-04｜Candidate 持久化协议补强

### 2.1 缺陷定位：05.17 §2.2 缺目录项 fsync

05.17 补了 `fsync(candidate_fd)` 和 `fsync(prepared_fd)`，但**没有同步 Candidate/PREPARED 所在目录的目录项**。文件内容 fsync 只保证数据块落盘，**目录项（文件名→inode 映射）的持久化需要 `fsync(dirfd)`**。

### 2.2 完整 POSIX 协议（文件 + 目录项双重持久化）

```
1. 写 Candidate 版本文件 versions/<txn_id>.<rev>（O_EXCL）
2. fsync(candidate_fd)                       # 文件内容持久化
3. fsync(versions_dirfd)                     # Candidate 目录项持久化  ← 05.17 缺失
4. 重读 + SHA-256 校验
5. 写 PREPARED prepared/<txn_id>
6. fsync(prepared_fd)                        # PREPARED 内容持久化
7. fsync(prepared_dirfd)                     # PREPARED 目录项持久化  ← 05.17 缺失
8. flock(fd_lock, LOCK_EX)
9. 读 current → 父版本三元组
10. 五条件校验（§2.3）
11. 写 current.<txn_id>（新指针内容）
12. fsync(current_tmp_fd)                    # 指针临时文件内容持久化
13. rename(current.<txn_id> → current)       # 目录项原子替换
14. fsync(parent_dirfd)                      # current 父目录项持久化
15. 写 COMMIT 回执 + fsync(commit_fd) + fsync(commit_dirfd)
16. flock LOCK_UN
```

### 2.3 五条件校验（保留 05.17）

`base_revision`/`base_txn_id`/`base_hash`/`new_revision`/`new_txn_id`/`candidate_hash` 绑定；提交前五条件校验，任一不匹配拒绝 + 重生成。

### 2.4 崩溃持久化矩阵（分步，含目录项）

| 崩溃点 | current | Candidate | 目录项状态 | 恢复动作 |
|--------|---------|-----------|-----------|---------|
| 步骤 2 前 | 旧 rev | 内容未持久 | 目录项未持久 | 删除半成品，重试 |
| 步骤 3 前 | 旧 rev | 内容已持久 | **目录项未持久** | 重试（目录项可能丢失）|
| 步骤 7 前 | 旧 rev | 完整 | PREPARED 目录项未持久 | 校验 PREPARED，重试 |
| 步骤 13 前 | 旧 rev | 完整+PREPARED | 已持久 | 可重试 |
| 步骤 14 前 | **新 rev** | 已发布 | current 目录项未持久 | 按 current 确认 + 补 fsync |
| 步骤 15 前 | 新 rev | 已发布 | 已持久 | current 为准确认 |

**关键澄清**：`fsync(dirfd)` 是目录项持久化的必要条件。**rename 的原子性 ≠ 断电持久性**——rename 原子地改了内存中的目录项，但目录项落盘需要 `fsync(dirfd)`。

### 2.5 15 条崩溃时间线

| # | 场景 | 推演 | 结果 |
|---|------|------|------|
| G1 | Candidate 内容未 fsync | 步骤 2 强制 | 拒绝跳过 |
| G2 | Candidate 目录项未 fsync 崩溃 | 目录项可能丢 | 标 NOT_DURABLE |
| G3 | PREPARED 内容未 fsync | 步骤 6 强制 | 拒绝跳过 |
| G4 | PREPARED 目录项未 fsync | 步骤 7 强制 | 拒绝跳过 |
| G5 | rename 后目录未 fsync | 步骤 14 强制 | NOT_DURABLE |
| G6 | fsync 失败 | 标 NOT_DURABLE | 不回滚 |
| G7 | 磁盘错误 | 放弃 + INDETERMINATE | 保留现场 |
| G8 | 重复 txn_id | O_EXCL 失败 | 拒绝 |
| G9 | 双事务同父版本 | flock 串行化 | 一胜一弃 |
| G10 | Candidate 过期 | 五条件失败 | 重生成 |
| G11 | revision 篡改 | 断言失败 | 拒绝 |
| G12 | PREPARED 半写入 | 校验失败忽略 | 无副作用 |
| G13 | Candidate 篡改 | hash 失败 | 拒绝 |
| G14 | rename 后 COMMIT 前崩溃 | current 为准 | 确认 |
| G15 | 锁路径替换 | 身份校验 | DENY |

### 2.6 Windows 独立协议（不声称等价）

`CreateFileW` + `LockFileEx` + `ReplaceFileW` + `FlushFileBuffers`。**注意**：Windows 的 `FlushFileBuffers` 对**目录句柄**的适用条件有限（目录句柄需 `FILE_FLAG_BACKUP_SEMANTICS` 打开才能用于 FlushFileBuffers，且其持久化保证弱于 POSIX `fsync(dirfd)`），**不得宣称等价**。Windows 断电恢复能力 **NOT_VERIFIED**。

### 2.7 P0-04 验收与结论

- 文件内容 + 目录项双重持久化：✅ §2.2
- 分步崩溃矩阵：✅ §2.4
- 15 条时间线：✅ §2.5
- POSIX/Windows 分开：✅ §2.2/§2.6

**P0-04 结论：DESIGN_PROVEN（POSIX）/ NOT_VERIFIED（Windows）**。
- POSIX 的 fsync(fd)+fsync(dirfd) 持久化顺序由 POSIX 语义证明。
- Windows 目录持久化（FlushFileBuffers on directory handle）NOT_VERIFIED。
- rename 原子性 ≠ 断电持久性，已明确。

---

## 3. P0-05｜Sealed / Source Retire 协议重新设计（推翻 +i 方案）

### 3.1 缺陷定位：05.17 的 `chattr +i` 方案被权威文档双重推翻

05.17 §3.2/§3.4 的核心假设是"`chattr +i` 保持生效时仍可 rename 文件，因为 rename 只改目录项"。**这一假设被 chattr(1) 官方 man page 直接否定**：

> **chattr(1) ATTRIBUTES, `i` 属性原文**："A file with the 'i' attribute cannot be modified: **it cannot be deleted or renamed**, no link can be created to this file, most of the file's metadata can not be modified, and the file can not be opened in write mode."

即 `+i` **同时阻止 rename 和 delete**。05.17 的"保持 +i 执行 rename"在真实 Linux 语义下会返回 **EPERM**，退役操作无法完成。

**更严重的第二点**，chattr(1) 的 BUGS AND LIMITATIONS 节明确：

> "Setting 'a' and 'i' attributes **will not affect the ability to write to already existing file descriptors**."

即 `+i` **不阻止已 open 的写 FD 继续写入**。这推翻了 05.16/05.17 反复引用的"`+i` 让旧 FD 写返回 EPERM"这一核心假设。

**综合结论**：`chattr +i` 方案**两个方向都失效**——
1. 它**不能**阻止旧可写 FD 写（旧 FD 写照常成功）；
2. 它**不能**在保持保护下完成 rename（rename 被 EPERM 拒绝）。

### 3.2 严格区分的对象与操作（不再混用）

| 概念 | 定义 |
|------|------|
| inode 内容 | 文件数据块 |
| inode 元数据 | 权限、时间戳、属性（含 immutable flag）|
| 目录项 | 目录中的"名字→inode"映射 |
| 路径名 | 到达 inode 的路径字符串 |
| 文件描述符 | 进程内引用 inode 的句柄 |
| 硬链接 | 指向同一 inode 的额外目录项 |
| mmap 映射 | 直接映射 inode 页面的内存 |
| 只读句柄 | O_RDONLY 打开的 fd |
| 不可变属性 | 阻止修改/删除/rename 的 flag |
| 目录权限 | 控制目录项创建/删除 |
| 挂载边界 | 文件系统边界 |
| 退役 | 从活跃命名空间移除 |
| 清理 | 释放资源 |
| 数据销毁 | 安全擦除内容 |

**核心澄清**："内容不可变"与"路径可移动"是**两个正交维度**，但 Linux `+i` 恰好**同时锁定两者**（既禁内容修改，又禁路径移动）。05.17 错误地认为 `+i` 只锁内容、不锁路径。

### 3.3 候选方案评估（逐项回答 12 个问题）

| 方案 | 旧FD能写？| 硬链接绕过？| mmap改？| append改？| 目录项替换？| 非特权干扰？| 同UID干扰？| root绕过？| 校验-退役竞态？| 崩溃恢复？| 退役被拒？| 权威文档？|
|------|----------|-----------|--------|---------|-----------|-----------|-----------|----------|-------------|----------|---------|---------|
| A. +i 保持 + rename | 能（+i 不阻旧FD）| — | 能 | 能 | rename 被 +i 拒 | — | — | — | — | — | **是（EPERM）** | 否 |
| B. 专用受控目录+独立权限主体 | 能 | 能 | 能 | 能 | 可能 | 可能 | 可能 | 能 | 有 | 不确定 | 否 | 无 |
| C. 不可变版本对象+发布指针 | 副本不可变，源仍可变 | — | — | — | — | — | — | — | 有（源可变）| — | 否 | 部分 |
| D. 权限隔离/挂载边界/专用服务 | 视实现 | 视实现 | 视实现 | 视实现 | 视实现 | 视实现 | 视实现 | 视实现 | 视实现 | 视实现 | 视实现 | 需专用实现 |
| E. 其他 | — | — | — | — | — | — | — | — | — | — | — | — |

**逐项结论**：
- **方案 A（05.17 的 +i 方案）**：**被权威文档否决**——+i 既不能阻止旧 FD 写，又阻止 rename。**废弃**。
- **方案 B**：本地文件系统无法为"同 UID 进程"建立强制边界，无法阻止旧 FD/硬链接/mmap 写。
- **方案 C**：只读副本的不可变性**不能**代替源对象不可变，源仍可变，竞态仍在。
- **方案 D**：需要**独立权限主体**（跨信任边界服务 / 独立守护进程 / 特权命名空间），本工单 READ-ONLY **不实现**。
- **方案 E**：无已知的、无需特权的本地强制不可变原语能同时满足"阻止旧 FD 写"和"允许路径移动"。

### 3.4 平台独立矩阵

| 平台 | 强制不可变原语 | 阻止旧 FD 写？| 允许 rename？| 结论 |
|------|--------------|--------------|-------------|------|
| Linux ext4/xfs/btrfs `+i` | `chattr +i` | **否**（BUGS 节）| **否**（EPERM）| **DENY** |
| Linux 不支持 immutable 的 FS | 无 | — | — | **DENY** |
| macOS `uchg` | `chflags uchg` | 不保证 | 语义弱 | **DENY** |
| Windows | 无强制原语 | — | — | **DENY** |
| 网络文件系统 | 无 | — | — | **DENY** |
| 无特权环境 | 无法施加 +i | — | — | **DENY** |

### 3.5 Source Retire 状态机与 DENY 条件

```
WRITABLE → SEALING → SEALED → QUARANTINED → RETIRED → PURGED
                          ↘ INDETERMINATE
```

**DENY 条件（强制）**：
1. 无法证明旧 FD/mmap/硬链接写入被阻断 → **DENY**；
2. 无法证明退役操作可成功完成 → **DENY**；
3. 无法证明最终校验至退役完成期间内容稳定 → **DENY**；
4. 无法证明权限主体边界 → **DENY**。

**在 05.18 的结论下**：本地文件系统**无法同时满足**上述 1~4（`+i` 既不能阻旧 FD 写、又不能 rename），因此 **Source Retire 在所有本地文件系统平台默认 DENY**。唯一理论可行的是方案 D（独立特权服务），NOT_IMPLEMENTED。

### 3.6 15 条并发/崩溃时间线 + 10 个反例

| # | 场景 | 推演 | 结果 |
|---|------|------|------|
| H1 | 旧 FD 写 + i 已设 | +i 不阻旧 FD → 写成功 | **竞态（推翻 05.17）**|
| H2 | rename + i 已设 | rename EPERM | 退役失败 |
| H3 | mmap 写 + i | +i 不阻已映射页 | 竞态 |
| H4 | 硬链接写 + i | +i 不阻已 open 句柄 | 竞态 |
| H5 | append + i | +i 不阻已 open FD | 竞态 |
| H6 | 最终校验后旧 FD 写 | 旧 FD 写成功 | 竞态 |
| H7 | 路径替换 | unlink 后 recreate | 身份变化检测 |
| H8 | 退役前崩溃 | 源仍在 | 可重试 |
| H9 | 退役后崩溃 | 回执未写 | 按隔离目录确认 |
| H10 | 权限被撤销 | 无法施加 +i | DENY |
| H11 | 无 +i 平台 | 无原语 | DENY |
| H12 | 同 UID 恶意进程 | 无法强制 | NOT_VERIFIED |
| H13 | root 绕过 | root 可 -i | NOT_VERIFIED |
| H14 | 网络 FS | 无原语 | DENY |
| H15 | 无法证明内容稳定 | 证据不足 | DENY |

10 个最小反例：旧 FD 写、rename EPERM、mmap 写、硬链接写、append 写、校验后写、路径替换、同 UID、root -i、网络 FS。

### 3.7 P0-05 验收与结论

- 推翻 +i 方案（权威文档）：✅ §3.1
- 12 个对象/操作严格区分：✅ §3.2
- 方案 A~E 逐项评估：✅ §3.3
- 平台矩阵：✅ §3.4
- 状态机 + DENY 条件：✅ §3.5
- 15 条时间线 + 10 反例：✅ §3.6

**P0-05 结论：Source Retire 在所有本地文件系统平台 DENY（DESIGN_PARTIAL，最终 DENY）**。
- `chattr +i` 方案被 chattr(1) 权威文档双重推翻（不阻旧 FD 写 + 阻止 rename）。
- 本地 FS 无机制同时满足"阻止旧 FD 写"和"允许路径移动"。
- 唯一理论可行的是独立特权服务（方案 D），NOT_IMPLEMENTED。
- **诚实结论：Source Retire 无法在本工单范围内证明安全，必须 DENY，而非 PASS。**

---

## 4. P0-06｜Tombstone / Windows 恢复协议补强

### 4.1 缺陷定位：05.17 未分析 ReplaceFileW 部分失败

05.17 §4.4 假设 `ReplaceFileW` 失败时占位文件必然仍在原位置。**实际上 `ReplaceFileW` 可能部分完成**（目标已改名、替换文件已移动、备份已创建等中间态），失败时不能假设任何文件位置。

### 4.2 ReplaceFileW 错误状态矩阵

| 错误/状态 | 原目标文件 | 替换文件 | 备份文件 | 判定 |
|----------|-----------|---------|---------|------|
| 成功 | 已被替换为新内容 | 已消耗 | 已创建（若指定）| PUBLISHED |
| ERROR_FILE_NOT_FOUND | 存在 | 不存在 | — | 可重试 |
| ERROR_ACCESS_DENIED | 未知 | 未知 | 未知 | INDETERMINATE |
| ERROR_SHARING_VIOLATION | 存在 | 存在 | — | 可重试（占位仍在）|
| 部分失败（进程崩溃）| 未知 | 未知 | 未知 | **INDETERMINATE，按内容判定** |

**核心**：ReplaceFileW 失败或崩溃后，**不能假设任何文件位置**，必须**按内容校验**判定（占位标记 vs 有效元数据），无法判定 → INDETERMINATE。

### 4.3 占位 vs 有效元数据（保留 + 强化）

- PLACEHOLDER ≠ PUBLISHED；
- 仅凭文件存在不得判定成功；
- 元数据绑定 txn_id/cleanup_id/源身份/哈希；
- 不完整或身份不匹配 → INDETERMINATE；
- 恢复幂等；
- 证据不足不删现场。

### 4.4 FlushFileBuffers 对目录句柄的适用条件

`FlushFileBuffers` 需目录句柄以 `FILE_FLAG_BACKUP_SEMANTICS` 打开才可调用，且其语义是"刷新文件系统缓存"，**不保证等价于 POSIX `fsync(dirfd)` 的目录项持久化**。Windows 断电后目录项是否持久 **NOT_VERIFIED**。

### 4.5 恢复状态机

```
恢复入口（读到 tombstone 目标文件）
  ├─ 内容 == 占位标记 → PLACEHOLDER → 删除占位，可安全重试
  ├─ 内容 == 有效元数据（字段全 + 哈希匹配 + txn 归属）→ PUBLISHED → 认领
  ├─ 内容异常/不完整/哈希不匹配 → INDETERMINATE → 保留现场
  └─ 无法判定（文件不存在但回执存在等）→ INDETERMINATE
```

### 4.6 15 条崩溃/恢复时间线

| # | 场景 | 推演 | 结果 |
|---|------|------|------|
| J1 | 临时文件写一半 | 校验失败删除 | 安全 |
| J2 | 临时 fsync 失败 | 不发布 | 拒绝 |
| J3 | 占位创建后崩溃 | 占位标记 → 删除重试 | 可重试 |
| J4 | ReplaceFileW 失败（NOT_FOUND）| 目标存在 → 重试 | 可重试 |
| J5 | ReplaceFileW 部分失败 | 内容判定 | INDETERMINATE |
| J6 | ReplaceFileW 成功 Flush 前崩溃 | 元数据哈希匹配 → 补 Flush | 可继续 |
| J7 | Flush 失败 | 目录项不持久 | INDETERMINATE |
| J8 | 系统崩溃（断电）| 内容判定 | 占位/有效/INDETERMINATE |
| J9 | 重复 txn | CREATE_NEW 失败 | 拒绝 |
| J10 | 源同名重建 | 源路径又存在 | INDETERMINATE |
| J11 | 隔离对象被替换 | 身份变化 | INDETERMINATE |
| J12 | 元数据被替换/删除 | 不匹配/消失 | INDETERMINATE |
| J13 | 并发清理 | 隔离 rename 竞争 | 一个成功 |
| J14 | 审计写失败 | REMOVED 失败 | INDETERMINATE |
| J15 | 归因证据不足 | 无元数据 | INDETERMINATE |

### 4.7 归档独立建模（保留）

归档独立事务，不复用 tombstone 结论；归档前不销毁原证据；失败 INDETERMINATE。

### 4.8 P0-06 验收与结论

- ReplaceFileW 错误状态矩阵：✅ §4.2
- 占位/有效区分：✅ §4.3
- FlushFileBuffers 适用条件：✅ §4.4
- 恢复状态机 + 幂等：✅ §4.5
- 15 条时间线：✅ §4.6

**P0-06 结论：DESIGN_PARTIAL**。
- 恢复状态机与占位/有效区分闭合（协议层）。
- ReplaceFileW 部分失败的中间态依赖 Windows 具体行为，NOT_VERIFIED。
- FlushFileBuffers 对目录句柄的持久化保证 NOT_VERIFIED。

---

## 5. 设计不变量（I-01 ~ I-35）

| # | 定义 | 适用范围 | 证明位置 | 失效反例 | 拒绝策略 | 验证状态 |
|---|------|---------|---------|---------|---------|---------|
| I-01 | fork 不绕过 CPython 生命周期 | fork | §1.3 | os._fork 后执行 Python | 用 os.fork | DESIGN_PARTIAL |
| I-02 | before 钩子不获取锁 | fork | §1.4/§1.6 | before 持锁重入 | 只读快照 | DESIGN_PROVEN |
| I-03 | 受控 fork 仅主线程 | fork | §1.5 | 非主线程 fork | DENY | DESIGN_PARTIAL |
| I-04 | FD 绑定 generation | FD | §1.4 | 裸数字复用 | 不匹配不关 | DESIGN_PROVEN |
| I-05 | child 不获取继承锁 | fork child | §1.4 | 继承 locked mutex | 直接赋值 | DESIGN_PROVEN |
| I-06 | 禁止持锁 fork | fork | §1.6 | 持锁 fork | 拒绝 | DESIGN_PROVEN |
| I-07 | 锁获取升序无死锁 | 全锁 | §1.6 | 逆序 | 禁止 | DESIGN_PROVEN |
| I-08 | 第三方 C fork DENY | fork | §1.7 | C 直接 fork | DENY | DESIGN_PROVEN |
| I-09 | 子解释器 fork DENY | fork | §1.5 | 子解释器 | DENY | DESIGN_PARTIAL |
| I-10 | Candidate 绑定六字段 | 提交 | §2.3 | 未绑父版本 | 五条件 | DESIGN_PROVEN |
| I-11 | 文件内容先于目录项持久化 | 提交 | §2.2 | 未 fsync 文件 | 禁止 | DESIGN_PROVEN |
| I-12 | 目录项必须 fsync(dirfd) | 提交 | §2.2 | 目录项未持久 | 强制 fsync | DESIGN_PROVEN |
| I-13 | rename 原子性 ≠ 断电持久性 | 提交 | §2.4 | 误当持久 | fsync 目录 | DESIGN_PROVEN |
| I-14 | 每崩溃点确定恢复 | 提交 | §2.4 | 无规则 | 按矩阵 | DESIGN_PROVEN |
| I-15 | revision 单调 | 提交 | §2.3 | 回退 | 断言拒绝 | DESIGN_PROVEN |
| I-16 | 恶意同 UID 锁替换排除 | 锁 | §2（继承）| 同 UID 替换 | 边界 NOT_VERIFIED | NOT_VERIFIED |
| I-17 | +i 不阻止旧 FD 写 | Source Retire | §3.1 | 旧 FD 写成功 | 承认 → DENY | NOT_VERIFIED（文档证实）|
| I-18 | +i 阻止 rename | Source Retire | §3.1 | rename EPERM | 承认 → DENY | NOT_VERIFIED（文档证实）|
| I-19 | 源内容稳定无法本地证明 | Source Retire | §3.3/§3.4 | 旧 FD/mmap/硬链接写 | DENY | DESIGN_PARTIAL |
| I-20 | Source Retire 默认 DENY | Source Retire | §3.5 | 无强制边界 | DENY | DESIGN_PARTIAL |
| I-21 | 只读副本 ≠ 源不可变 | Source Retire | §3.3 | 副本当源证据 | 不混用 | DESIGN_PROVEN |
| I-22 | inode 身份 ≠ 内容不可变 | Source Retire | §3.3 | 身份当不可变 | 不混用 | DESIGN_PROVEN |
| I-23 | 特权主体撤销 +i 排除 | Source Retire | §3.4 | root -i | 威胁模型排除 | NOT_VERIFIED |
| I-24 | 元数据原子发布（临时+校验+无覆盖）| tombstone | §4 | O_EXCL 逐步写 | 废弃 | DESIGN_PROVEN |
| I-25 | 占位 ≠ 有效 tombstone | tombstone | §4.3 | 占位误认 | 内容区分 | DESIGN_PARTIAL |
| I-26 | ReplaceFileW 部分失败按内容判定 | tombstone | §4.2 | 假设文件位置 | INDETERMINATE | DESIGN_PARTIAL |
| I-27 | 恢复幂等 | tombstone | §4.5 | 非幂等 | 状态机 | DESIGN_PROVEN |
| I-28 | 归档独立建模 | 归档 | §4.7 | 复用结论 | 单独协议 | DESIGN_PROVEN |
| I-29 | 归档前不销毁证据 | 归档 | §4.7 | 提前销毁 | 禁止 | DESIGN_PROVEN |
| I-30 | 移除 ≠ 销毁 | 删除 | §4 | 仅源消失 | INDETERMINATE | DESIGN_PROVEN |
| I-31 | 无法证明必须 DENY/INDETERMINATE | 全局 | 全文 | 猜测 | 不猜测 | DESIGN_PROVEN |
| I-32 | 不把设计推演写运行时验证 | 全局 | 全文 | 自审当验证 | 标 NOT_TESTED | DESIGN_PROVEN |
| I-33 | 平台能力不跨平台推广 | 全局 | §3.4 | 推广 FS 行为 | 逐平台 | DESIGN_PROVEN |
| I-34 | 主解释器主线程约束 | fork | §1.5 | 违反 | DENY | DESIGN_PARTIAL |
| I-35 | 不绕过 CPython 后执行 Python | fork | §1.1/§1.3 | os._fork+Python | 禁用 os._fork | DESIGN_PARTIAL |

**共 35 条不变量，每条含定义/范围/证明位置/失效反例/拒绝策略/验证状态。** 其中 I-17/I-18 是**本轮新发现**——由 chattr(1) 权威文档证实，直接推翻了 05.16/05.17 的 `+i` 假设。

---

## 6. 全局一致性审查

### 6.1 15 条端到端链路

| # | 链路 | 关键边界 | 崩溃点 | 恢复 | 结论 |
|---|------|---------|--------|------|------|
| L1 | 主线程受控 fork→父子退出 | CPython 生命周期 | fork 中 | child 重建 | DESIGN_PARTIAL |
| L2 | FD 生命周期→冻结→child 重建 | generation | 协调中 | 校验 | DESIGN_PROVEN |
| L3 | Candidate→fsync 文件→fsync 目录→rename→fsync 目录 | 双重持久化 | rename 前后 | 按 current | DESIGN_PROVEN |
| L4 | 双事务竞争 | flock | 竞争点 | 败者重生成 | DESIGN_PROVEN |
| L5 | 源 seal→quarantine→retire | **retire DENY** | — | DENY | DENY |
| L6 | 旧 FD + i | **+i 不阻旧 FD** | 校验后写 | 竞态 | DENY |
| L7 | rename + i | **rename EPERM** | — | 退役失败 | DENY |
| L8 | tombstone 临时→校验→无覆盖发布 | 临时 fsync | 发布前后 | 占位/有效 | DESIGN_PARTIAL |
| L9 | Windows ReplaceFileW 部分失败 | 内容判定 | 替换中 | INDETERMINATE | DESIGN_PARTIAL |
| L10 | rename 后崩溃 REMOVED 未持久化 | rename→REMOVED | rename 后 | 元数据匹配 | DESIGN_PROVEN |
| L11 | 归档协议 | 临时+校验+发布 | 归档中 | 哈希确认 | DESIGN_PROVEN |
| L12 | 审计不可用 | REMOVED 失败 | 审计写 | INDETERMINATE | DESIGN_PROVEN |
| L13 | 恶意同 UID | 无 | 任意 | 检测 | NOT_VERIFIED |
| L14 | 特权 -i | +i 可撤销 | 撤销后 | 威胁排除 | NOT_VERIFIED |
| L15 | 第三方 C fork | 无 | 任意 | DENY | DESIGN_PROVEN |

### 6.2 25 条时间线（汇总）

P0-02 十二条 + P0-04 十五条 + P0-05 十五条 + P0-06 十五条 = **57 条**。

### 6.3 跨章节身份一致性

| 身份 | 产生 | 传递 | 一致性 |
|------|------|------|--------|
| FD generation | §1.4 | §1.9 | generation 绑定 |
| 父版本三元组 | §2.3 | §2.4 | 五条件 |
| 源对象身份 | §3 | §3.6 | +i 无意义，retire DENY |
| Tombstone 事务身份 | §4.3 | §4.6 | txn 归因 |

### 6.4 历史阻断逐项标记

| 阻断 | 状态 |
|------|------|
| IMPLEMENT-01 | STOP（保持）|
| P0-BLOCK-1 | 未解除（保持）|
| P0-BLOCK-2 | 未解除（保持）|
| P1-BLOCK-3 | 未解除（保持）|
| F1 | 不解除（保持）|
| P0.8 | 不解除（保持）|
| F2 | 不解除（保持）|

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
| 冻结源码哈希 | 不变 |
| A/B 工作区 | 不合并 |

---

## 8. 最终结论

**DESIGN-05.18 = FAIL（整体）**，具体分项：

| # | 结论 | 依据 |
|---|------|------|
| P0-02 | DESIGN_PARTIAL | 协议层可证明，但 CPython 多线程 fork 运行时 NOT_TESTED；os._fork 方案已废弃，改用 os.fork + before 无锁 |
| P0-04 | DESIGN_PROVEN（POSIX）/ NOT_VERIFIED（Windows）| 补目录项 fsync，分步矩阵闭合；Windows 目录持久化 NOT_VERIFIED |
| P0-05 | **DENY** | chattr(1) 权威文档证实 +i 既不阻旧 FD 写、又阻止 rename，Source Retire 无法在本工单证明安全 |
| P0-06 | DESIGN_PARTIAL | ReplaceFileW 部分失败按内容判定；Windows 具体行为 NOT_VERIFIED |

**核心诚实结论**：
1. **P0-05 是阻断项，本轮无法闭合**——`chattr +i` 方案被权威文档（chattr(1) man page）双重推翻。Source Retire 在本地文件系统上**无法证明安全**，必须 **DENY**。这推翻了 05.16/05.17 连续两轮的错误假设。
2. **P0-02 降级为 DESIGN_PARTIAL**——`os._fork()` 绕过 CPython 生命周期后执行 Python 代码**无法证明安全**，05.17 的 PASS 结论不成立；05.18 改回 `os.fork()`，但多线程 fork 运行时行为仍 NOT_TESTED。
3. P0-04/P0-06 补强到位（目录项 fsync、ReplaceFileW 错误矩阵），但 Windows 持久性仍 NOT_VERIFIED。

**由于 P0-05 是工单明确标记的阻断项，且本轮诚实结论是 DENY（无法证明），整体不得 PASS。**

**IMPLEMENT-01 继续 STOP，等待 ChatGPT 独立复审。**

---

## 9. 交付物索引

| 文件 | 说明 |
|------|------|
| `USIT-STAGE-B-DESIGN-05.18.md` | 本主设计文档 |
| `USIT-STAGE-B-DESIGN-05.18-ISSUES.md` | 问题清单及剩余风险 |
| `USIT-STAGE-B-DESIGN-05.18-DIFF.md` | 相对 05.17 逐项差异 |
| `USIT-STAGE-B-DESIGN-05.18-CROSS-REVIEW.md` | 反例审查记录 |
| `USIT-STAGE-B-DESIGN-05.18-EXECUTION-REPORT.md` | 实际执行报告 |
| `USIT-STAGE-B-DESIGN-05.18-HASH-MANIFEST.md` | 交付文件 SHA256 清单 |
| `USIT-STAGE-B-DESIGN-05.18-HASH-RECEIPT.txt` | 哈希验证回执 |
| `USIT-STAGE-B-DESIGN-05.18-交付物.zip` | 完整打包 |

---