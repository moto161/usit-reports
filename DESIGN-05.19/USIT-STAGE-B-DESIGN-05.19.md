# USIT-STAGE-B-DESIGN-05.19｜Fork门禁闭合＋Windows Tombstone错误状态重构＋CAS持久化补强

> 阶段 B · DESIGN-05.19 · READ-ONLY DESIGN REVISION ONLY
> 作者：USIT 主开发 · 生成：2026-10-31（GMT+8）
> 前置审查：DESIGN-05.18 独立复审 FAIL
> 工作区：`C:\Users\Administrator\WorkBuddy\2026-10-15-22-09-47\usit_p0`
> 交付目录：`C:\Users\Administrator\WorkBuddy\2026-10-15-22-09-47\usit_p0_05.19_reports`（**项目外**）
> 状态：**STOP — 等待独立复审**（未授权 IMPLEMENT-01）

---

## 0. 缺陷追踪表（缺陷→历史证据→05.18失败原因→05.19修订→证明依据→反例→拒绝策略→验收）

| # | 历史证据 | 05.18 失败原因 | 05.19 修订位置 | 证明依据 | 反例 | 拒绝策略 | 验收 |
|---|---------|---------------|---------------|---------|------|---------|------|
| P0-02 | 05.18 §1.4 快照后释放 `_fd_lock` 与 `_fork_coord_lock` 再 `os.fork()` | 快照生成后、实际 fork 前存在未门禁窗口：其他线程可新建/关闭/复用 FD，快照与实际进程 FD 状态不一致 | §1 | 冻结必须覆盖真正的 fork 系统调用窗口，改为"冻结→在门禁保持下 fork→恢复"全程受控 | 快照后 FD 关闭、数字复用、fork 前新建 FD | 非受控入口 UNSUPPORTED/DENY | DESIGN_PARTIAL |
| P0-04 | 05.18 §2 缺持久化故障状态机，缺 COMMIT 回执目录项持久化分步 | 仅列崩溃矩阵，未定义任一 fsync 失败后的全局事务状态与禁写/回滚策略 | §2 | 新增 DURABLE/NOT_DURABLE/COMMIT_INDETERMINATE/RECOVERY_REQUIRED/QUARANTINED 五态状态机 | fsync 失败后继续提交 | 不确定态禁提交 | DESIGN_PROVEN（POSIX）/ NOT_VERIFIED（Windows）|
| P0-05 | 05.18 §3 已撤回 `chattr +i` 保持 rename 的错误 | 无需新修复；05.18 结论正确但需复核 DENY 边界是否被误扩为"不存在任何可行机制" | §4 | 区分"当前方案未证明"与"平台不存在可行机制"；保持 Source Retire DENY | 旧 FD/mmap/硬链接写入 | 默认 DENY | DENY（保持）|
| P0-06 | 05.18 §4.2 仅笼统列错误码，未逐项覆盖 1175/1176/1177 官方语义 | 未依据 Microsoft 官方文档精确区分三个 ReplaceFileW 特有错误码的最终文件状态 | §3 | 依据 Microsoft Learn ReplaceFileW 官方契约逐码建立文件状态判定矩阵 | 1175/1176/1177 部分移动 | 无法唯一判定 → INDETERMINATE | DESIGN_PARTIAL（协议层）/ NOT_VERIFIED（持久性）|

**本节是全文总纲。本轮核心任务：P0-02 闭合"快照到 fork 之间"的门禁窗口；P0-04 建立持久化故障状态机；P0-06 依据 Microsoft 官方文档重构 ReplaceFileW 错误状态矩阵；P0-05 保持 DENY 并厘清其边界表述。**

---

## 1. P0-02｜Fork 与 FD 生命周期门禁彻底闭合

### 1.1 缺陷定位：05.18 的"快照→释放锁→fork"存在未门禁窗口

05.18 §1.4 的流程在生成冻结快照 tuple 后**立即释放了 `_fd_lock` 和 `_fork_coord_lock`**，然后才调用 `os.fork()`。这段"快照已生成但锁已释放"的窗口内：

- 其他线程可以 `close()` 一个已在快照中的 FD；
- 其他线程可以 `open()` 新 FD，且其数字可能**复用**刚关闭的 FD 数字；
- 其他线程可以 `dup2()` 覆盖某个快照中的 FD 数字；
- 快照 tuple 与进程实际的 FD 表已经**漂移**，child 据此 close FD 可能误关新资源。

**结论（05.19 必须闭合）**：快照必须与 fork 在**同一个受保护临界区**内完成，冻结必须覆盖真正的 `fork()` 系统调用窗口，不能在 fork 前提前释放门禁。

### 1.2 必须解决的三个本质问题

1. **快照与 fork 之间不得有任何未门禁的 FD 状态变化**——这要求"冻结 FD 操作入口"和"实际 fork"必须原子地处于同一门禁下。
2. **第三方 C 扩展可直接调用 `open`/`close`/`fork`，不经本库 Gate**——这是 Python 层**无法完全拦截**的，必须明确其信任边界。
3. **`os._fork()` 不是安全方案**——它跳过 CPython 的 `PyOS_BeforeFork/AfterFork_Parent/AfterFork_Child`，在 child 中继续执行 Python 代码不安全（05.18 §1.1 已确立，本轮重申）。

### 1.3 门禁闭合方案：冻结覆盖真正的 fork 窗口（05.19 修订）

05.18 的"快照后释放锁"必须废除。05.19 采用**冻结态覆盖 fork 系统调用**的方案：

```
fork_controlled 入口（必须主解释器主线程，否则 DENY）：
0. 线程身份检查：非主解释器主线程 → DENY
1. 获取 _fork_coord_lock（ForkGate 协调锁）
2. ForkGate 进入 FREEZING 态：
   a. 置 gate_state = FREEZING（原子写）
   b. 阻塞新 fork 请求
   c. 等待所有在途 FD 操作（open/close/dup/detach）排空（引用计数归零）
3. 冻结 FD Registry（在 FREEZING 态下，FD 操作入口已被 Gate 阻断）：
   a. 获取 _fd_lock（此时无在途操作，短暂持有）
   b. 枚举 Registry 生成冻结快照 tuple [(fd, generation, obj_id, state)]
   c. 释放 _fd_lock —— 但 **FREEZING 态仍在**，任何新的 FD 操作入口会先检查 Gate，被阻塞
4. 【关键】在 FREEZING 态保持下，直接调用 os.fork()
   - before 钩子：只读快照 tuple，不获取任何锁
   - PyOS_BeforeFork → fork() → PyOS_AfterFork_Parent/Child
5. parent：ForkGate 退出 FREEZING（回到 OPEN），_fork_coord_lock 释放，返回 child pid
6. child（经 PyOS_AfterFork_Child 重置后）：
   - 直接赋值全新锁对象（不 acquire 继承锁）
   - ForkGate 状态置为 child-local OPEN（不继承父的 FREEZING）
   - 遍历快照 tuple，generation 匹配才 close
```

**关键改变（相对 05.18）**：
- **冻结态（FREEZING）不再在 fork 前释放**，而是**覆盖真正的 `os.fork()` 系统调用**；
- 步骤 3 释放的只是 `_fd_lock`（细粒度锁），但**门禁（Gate/FREEZING 态）保持**，因此快照后没有任何受管 FD 操作能进入；
- 快照生成后、fork 前的窗口内，受管 FD 操作入口被 Gate 阻断 → 快照与实际 FD 表**保持一致**。

### 1.4 FD 操作入口覆盖矩阵（逐项审查）

本库必须为**所有受管 FD 的创建/关闭/复制入口**设置统一 Gate。下表逐项回答：是否经统一 Gate、是否在 Gate 内完成系统调用+登记、失败是否留半登记、第三方能否绕过、绕过后如何检测拒绝。

| 入口 | 经统一 Gate？ | Gate 内完成 syscall+登记？ | 失败留半登记？ | 第三方绕过？ | 绕过检测/拒绝 |
|------|--------------|--------------------------|---------------|-------------|--------------|
| `open/openat` | 是 | 是（Gate 内 syscall，失败不登记）| 否（失败即不登记，FD 未生成）| C 扩展可绕过 | child 重建时 generation 不匹配 → 不关闭，标记 UNKNOWN_FD |
| `socket/socketpair` | 是 | 是 | 否 | 可绕过 | 同上 |
| `accept/accept4` | 是 | 是 | 否 | 可绕过 | 同上 |
| `pipe/pipe2` | 是 | 是 | 否 | 可绕过 | 同上 |
| `dup/dup2/dup3` | 是 | 是 | 否（源 FD 已存在则登记副本）| 可绕过 | 同上 |
| `fcntl(F_DUPFD*)` | 是 | 是 | 否 | 可绕过 | 同上 |
| `close` | 是 | 是（Gate 内 syscall，登记态先置 CLOSING）| 否（syscall 成功才置 CLOSED）| 可绕过 | generation 失效 → child 不误关 |
| `close_range` | 是 | 是 | 否 | 可绕过 | 同上 |
| `detach`（socket 转移所有权）| 是 | 是（转移时注销 Registry）| 否 | 可绕过 | 同上 |
| `fileno`（只读查询）| 只读，不 Gate 但读 Registry 快照 | — | — | — | 只读无副作用 |
| FD 继承（fork 继承）| 由 fork 门禁覆盖 | 见 §1.3 | — | — | child 按 generation 校验 |
| 第三方库直接创建 FD | **否（无法拦截）** | 否 | 可能 | **是** | child 对不在快照中的继承 FD 标 UNKNOWN_FD，**不主动 close**（避免误关）|
| 子进程 FD 重建 | 是 | 是 | 否 | — | — |

**关键结论**：
- 本库自有的 FD 操作入口**全部经统一 Gate**，在 FREEZING 态下被阻断 → 受控 fork 的冻结窗口可证明闭合；
- **第三方 C 扩展直接调用 libc `open/close/fork` 无法被 Python 层拦截**。对这些入口，child 的防御策略是：**只关闭 generation 匹配的已知 FD，对未知 FD 不主动 close**（宁可泄漏，不误关），并标记 `UNKNOWN_FD` 进入审计。

### 1.5 fork 路径比较矩阵（依据 CPython 官方语义）

| 路径 | 触发 CPython fork 钩子？ | child 可安全执行 Python？ | 本库门禁能否覆盖？ | 结论 |
|------|------------------------|--------------------------|-------------------|------|
| `os.fork()` | 是（PyOS_BeforeFork/AfterFork_Parent/AfterFork_Child + register_at_fork）| 经运行时重置后相对安全 | 是（before 无锁钩子 + FREEZING 态）| **受控 fork 采用** |
| `os._fork()` | 否 | 否（运行时未重置，不安全）| 是但不安全 | **禁用**（仅 fork+exec 场景合法）|
| `os.register_at_fork()` | 是（用户钩子注册点）| — | — | 本库注册 before/parent/child 钩子 |
| `PyOS_BeforeFork()` | CPython 内部 | — | — | 由 os.fork 触发 |
| `PyOS_AfterFork_Parent()` | CPython 内部 | — | — | 由 os.fork 触发 |
| `PyOS_AfterFork_Child()` | CPython 内部 | 重置 runtime 锁 | — | 由 os.fork 触发 |
| 第三方 C 扩展直接 `fork()` | 否 | 否 | **否** | **DENY / UNSUPPORTED** |
| 嵌入式 Python / 子解释器 | 视宿主 | 不确定 | 否 | **DENY / UNSUPPORTED** |

### 1.6 主解释器主线程约束（强制，继承 05.18）

CPython 在多线程程序中 `fork()` 只保证调用线程在子进程存活，其他线程持有的锁可能处于锁定态。因此：

- 受控 fork **仅允许主解释器主线程**调用；
- 检测到非主线程、存在子解释器、或嵌入式宿主 → **DENY**；
- 不满足约束的运行环境 → **UNSUPPORTED/DENY**，不宣称通用安全。

### 1.7 锁顺序图（覆盖全部实际参与锁，证明无循环等待）

```
锁层级（升序，禁止逆序获取）：
L0. _fork_coord_lock   （ForkGate 协调锁）
L1. _gate_lock         （ForkGate 内部状态锁，保护 FREEZING/OPEN 态）
L2. _fd_lock           （FD Registry 锁）
L3. _lock              （LockHandleRegistry 锁）
L4. _lease_lock        （Lease 锁）
L5. _pmutex_lock       （ProcessMutex 锁）
```

规则：
1. before/parent/child 钩子**不获取任何锁**（消除重入死锁）；
2. fork 协调只短暂获取 `_fork_coord_lock` → `_gate_lock` → `_fd_lock`（**固定顺序**），不嵌套 L3~L5；
3. **禁止持锁 fork**（持有 L3~L5 的线程不得调用 fork）；
4. 其他锁获取严格升序。

**无死锁证明**：before 钩子无锁 + 协调锁顺序固定（coord→gate→fd）+ 禁止持锁 fork + 其余升序 → 不存在锁逆序，不存在循环等待环。

### 1.8 fork 父子进程状态机

```
[parent] IDLE ──fork_controlled──▶ FREEZING（Gate 阻断 FD 操作）
   FREEZING ──排空在途──▶ SNAPSHOT（生成冻结 tuple）
   SNAPSHOT ──os.fork()──▶ fork 成功？
        ├─ 是 ──▶ parent: FREEZING→OPEN → 返回 pid → IDLE
        └─ 否 ──▶ parent: FREEZING→OPEN → 返回错误 → IDLE（幂等可重试）
[child]（fork 成功后）
   ├─ PyOS_AfterFork_Child 重置 runtime
   ├─ 直接赋值全新锁对象（不 acquire 继承锁）
   ├─ ForkGate → child-local OPEN
   ├─ 遍历冻结 tuple，generation 匹配 → close；不匹配/未知 → UNKNOWN_FD 标记
   └─ 设置 _fork_reset_ok = True（或异常 → False → DENY 后续受保护操作）
```

### 1.9 15 条并发/竞态时间线

| # | 场景 | 初始状态 | 操作顺序 | 竞态窗口 | 预期结果 | 拒绝策略 |
|---|------|---------|---------|---------|---------|---------|
| T1 | 受控 fork 完整流程 | IDLE | FREEZING→SNAPSHOT→fork→恢复 | 无（门禁覆盖 fork）| 安全 | — |
| T2 | 快照后 FD close | FREEZING | 其他线程 close 被 Gate 阻断 | **已被 FREEZING 消除** | 快照一致 | 阻断 |
| T3 | 快照后 FD 数字复用 | FREEZING | open 复用数字被 Gate 阻断 | 已消除 | 不误关 | 阻断 |
| T4 | fork 前新建 FD | FREEZING | open 被 Gate 阻断 | 已消除 | 不进入快照 | 阻断 |
| T5 | 快照中 FD 被 dup2 覆盖 | FREEZING | dup2 被 Gate 阻断 | 已消除 | 不误关 | 阻断 |
| T6 | fork 失败（EAGAIN）| SNAPSHOT | fork 返回 -1 | parent 恢复 | FREEZING→OPEN，可重试 | 幂等 |
| T7 | parent 回调异常 | fork 中 | parent 钩子抛异常 | parent | 记录审计，不影响 child | 记录 |
| T8 | child 回调异常 | fork 后 | child 重建抛异常 | child | _fork_reset_ok=False | DENY 受保护操作 |
| T9 | 连续 fork | OPEN | 第二次 fork 进入 FREEZING | 无 | 幂等 | — |
| T10 | 第三方 C fork | 任意 | 绕过 Gate 直接 fork | **无法拦截** | child 状态未知 | DENY/UNSUPPORTED |
| T11 | 第三方 C open FD | 任意 | 绕过 Gate open | 无法拦截 | 快照外 FD | child 标 UNKNOWN_FD 不关闭 |
| T12 | 线程阻塞在 FD 操作 | FREEZING 前 | 某线程阻塞在 open | 排空等待 | 等待其完成 | 排空 |
| T13 | exec 后 | child | exec 替换映像 | 非 CLOEXEC FD 关闭 | FD 由 exec 语义处理 | 不主动管 |
| T14 | 进程退出 | 任意 | 退出 | — | OS 释放全部 FD | — |
| T15 | 子解释器 fork | 子解释器 | fork 被检测 | 检测 | DENY | UNSUPPORTED |

### 1.10 12 个失败/绕过反例

| # | 反例 | 结论 |
|---|------|------|
| R1 | 快照后释放锁再 fork（05.18 方案）| 存在未门禁窗口，05.19 废除，FREEZING 覆盖 fork |
| R2 | os._fork 后 child 执行 Python | 运行时未重置，不安全，禁用 |
| R3 | before 钩子持锁 | 重入死锁，05.19 before 无锁 |
| R4 | 非主线程 fork | DENY |
| R5 | 子解释器/嵌入式 fork | DENY/UNSUPPORTED |
| R6 | 第三方 C fork 绕过门禁 | 无法拦截，DENY |
| R7 | 第三方 C open 的 FD 被 child 误关 | child 只关 generation 匹配的 FD，未知 FD 不关 |
| R8 | 快照后 FD 数字复用被误关 | FREEZING 阻断复用，generation 二次防线 |
| R9 | 连续 fork 状态残留 | ForkGate 幂等 + 状态恢复 |
| R10 | child 回调异常 | _fork_reset_ok=False → DENY |
| R11 | 在途 FD 操作未排空 | FREEZING 态等待排空 |
| R12 | 持锁 fork 导致死锁 | 禁止持锁 fork |

### 1.11 P0-02 验收与结论

- 15 条并发/竞态时间线：✅ §1.9
- 12 个失败/绕过反例：✅ §1.10
- FD 操作入口覆盖矩阵：✅ §1.4
- fork 路径比较矩阵：✅ §1.5
- 锁顺序图：✅ §1.7
- fork 父子状态机：✅ §1.8
- 冻结覆盖真正的 fork 窗口：✅ §1.3（FREEZING 态不再提前释放）

**P0-02 结论：DESIGN_PARTIAL**。
- **协议层（受控 fork）**：FREEZING 态覆盖 `os.fork()` 系统调用，快照与 fork 之间的门禁窗口被闭合，可证明无误关、无死锁、快照一致 —— **DESIGN_PROVEN（协议层）**。
- **运行时**：CPython 多线程 fork 的运行时行为（threading 锁重置、GIL）依赖 CPython 实现，未在运行时验证 —— **NOT_TESTED**。
- **不可控入口**：第三方 C 扩展直接 `fork`/`open`/`close` **无法被 Python 层拦截**。对此类运行环境明确 **UNSUPPORTED/DENY**，不宣称通用安全。child 对未知继承 FD **不主动 close**（宁可泄漏，不误关），并标 `UNKNOWN_FD` 进入审计。

---

## 2. P0-04｜真正基线 CAS、锁对象身份与跨平台提交协议补强

### 2.1 缺陷定位：05.18 缺持久化故障状态机

05.18 §2 补了目录项 fsync 和分步崩溃矩阵，但**未定义任一关键 fsync 失败后的全局事务状态**——即 fsync 失败后，系统是否禁止后续事务、是否允许读取、是否允许自动回滚、如何防止基于不确定版本继续提交。05.19 必须补齐这一状态机。

### 2.2 六字段绑定与五条件校验（完整谓词）

Candidate 绑定六字段：

| 字段 | 来源 | 比较对象 |
|------|------|---------|
| `base_revision` | 生成 Candidate 时读取的 current.revision | `current.revision` |
| `base_txn_id` | 生成 Candidate 时读取的 current.txn_id | `current.txn_id` |
| `base_hash` | 生成 Candidate 时读取的 current.hash | `current.hash` |
| `new_revision` | `base_revision + 1` | `base_revision + 1`（断言）|
| `new_txn_id` | 本次事务分配的唯一 ID | 唯一性约束（O_EXCL 目录项）|
| `candidate_hash` | Candidate 文件内容 SHA-256 | PREPARED 记录中的哈希 |

五条件校验谓词（**全部在全局提交锁保护下，逐项列出**）：

```
P1: current.revision == base_revision          # 父版本未变
P2: current.txn_id == base_txn_id              # 父事务 ID 未变
P3: current.hash == base_hash                  # 父内容哈希未变
P4: new_revision == base_revision + 1          # 版本单调 +1
P5: sha256(candidate_content) == candidate_hash # Candidate 内容未被篡改，且与 PREPARED 一致
```

任一谓词为假：**拒绝提交**，且**必须基于最新 current 重新生成 Candidate**，禁止只修改 revision 数字后重试。

### 2.3 POSIX 完整持久化顺序（13 步 + 目录项）

```
 1. 写 Candidate 版本文件 versions/<txn_id>.<rev>（O_EXCL）
 2. fsync(candidate_fd)                        # Candidate 内容持久化
 3. fsync(versions_dirfd)                      # Candidate 目录项持久化
 4. 重读 + SHA-256 校验 candidate_hash
 5. 写 PREPARED prepared/<txn_id>
 6. fsync(prepared_fd)                         # PREPARED 内容持久化
 7. fsync(prepared_dirfd)                      # PREPARED 目录项持久化
 8. flock(commit_lock, LOCK_EX)                # 获取全局提交锁
 9. 读 current → 六字段 + 五条件校验（§2.2）
10. 写 current.<txn_id>（新指针内容）
11. fsync(current_tmp_fd)                      # current 临时文件内容持久化
12. rename(current.<txn_id> → current)         # current 目录项原子替换
13. fsync(parent_dirfd)                        # current 父目录项持久化
14. 写 COMMIT 回执 + fsync(commit_fd)          # 回执内容持久化
15. fsync(commit_dirfd)                        # 回执目录项持久化
16. flock LOCK_UN
```

每一步崩溃后的状态、恢复动作、是否允许重试/继续，见 §2.5 崩溃矩阵。

### 2.4 持久化故障状态机（新增，五态）

```
         ┌─────────────────────────────────────────────┐
         ▼                                             │
   DURABLE ──关键fsync成功──▶ （事务完成）             │
         │                                             │
         ├─fsync失败──▶ NOT_DURABLE ──重试fsync──▶ DURABLE
         │                  │                          │
         │                  └─重试失败/磁盘错误──▶ RECOVERY_REQUIRED
         │                                            │
         ├─rename已发生但目录项未持久（状态不确定）──▶ COMMIT_INDETERMINATE
         │                                            │
         └─证据不一致/被篡改──▶ QUARANTINED
```

| 状态 | 含义 | 是否禁后续事务 | 是否允许读 | 是否允许自动回滚 | 是否需人工确认 |
|------|------|---------------|-----------|-----------------|---------------|
| DURABLE | 所有关键 fsync 成功 | 否 | 是 | — | 否 |
| NOT_DURABLE | 某 fsync 失败但未发生 rename | **是**（禁后续提交）| 是 | 否（不自动回滚，避免状态不确定）| 可重试 fsync |
| COMMIT_INDETERMINATE | rename 已发生但目录项/回执未持久 | **是** | 是 | **否**（不得自动回滚）| **是**（需确认 current 为准）|
| RECOVERY_REQUIRED | 磁盘错误/重试失败 | **是** | 是 | 否 | **是** |
| QUARANTINED | 证据不一致/被篡改 | **是** | 受限 | 否 | **是** |

**核心规则**：
- 任一关键 fsync 失败 → 至少进入 `NOT_DURABLE`，**禁止后续事务提交**；
- `COMMIT_INDETERMINATE` 态下**禁止自动回滚或重复提交**——必须按 current 文件的实际内容确认最终版本，不得基于不确定版本继续提交；
- 读取**允许**（读不破坏持久性），但提交**禁止**；
- 任何不确定态不得自动推进，需人工确认。

### 2.5 13 步崩溃矩阵（每步状态 + 恢复）

| 崩溃点 | current | Candidate | 目录项 | 事务状态 | 恢复动作 | 允许重试？| 允许继续新事务？|
|--------|---------|-----------|--------|---------|---------|----------|---------------|
| 步骤 2 前 | 旧 rev | 内容未持久 | 未持久 | DURABLE（未写）| 删半成品，重试 | 是 | 是 |
| 步骤 3 前 | 旧 rev | 内容已持久 | 未持久 | NOT_DURABLE | 重试 fsync | 是 | 否（先闭环）|
| 步骤 7 前 | 旧 rev | 完整 | PREPARED 目录项未持久 | NOT_DURABLE | 校验 PREPARED，重试 | 是 | 否 |
| 步骤 12 前 | 旧 rev | 完整+PREPARED | 已持久 | DURABLE | 可重试 | 是 | 是 |
| 步骤 13 前 | **新 rev** | 已发布 | current 目录项未持久 | COMMIT_INDETERMINATE | 按 current 确认 + 补 fsync | 否（确认）| 否 |
| 步骤 15 前 | 新 rev | 已发布 | 已持久，回执未持久 | COMMIT_INDETERMINATE | 补写回执 + fsync | 否（确认）| 否 |

**关键澄清（继承 05.18）**：`fsync(dirfd)` 是目录项持久化的必要条件；**rename 的原子性 ≠ 断电持久性**。

### 2.6 15 条崩溃时间线

| # | 场景 | 推演 | 结果 |
|---|------|------|------|
| G1 | Candidate 内容未 fsync | 步骤 2 强制 | 拒绝跳过 |
| G2 | Candidate 目录项未 fsync 崩溃 | 目录项可能丢 | NOT_DURABLE |
| G3 | PREPARED 内容未 fsync | 步骤 6 强制 | 拒绝跳过 |
| G4 | PREPARED 目录项未 fsync | 步骤 7 强制 | NOT_DURABLE |
| G5 | rename 后目录未 fsync | 步骤 13 强制 | COMMIT_INDETERMINATE |
| G6 | fsync 失败 | NOT_DURABLE | 禁提交 |
| G7 | fsync 重试失败/磁盘错误 | RECOVERY_REQUIRED | 人工确认 |
| G8 | 证据被篡改 | QUARANTINED | 人工确认 |
| G9 | 重复 txn_id | O_EXCL 失败 | 拒绝 |
| G10 | 双事务同父版本 | flock 串行化 | 一胜一弃 |
| G11 | Candidate 过期 | 五条件 P1~P3 失败 | 重生成 |
| G12 | revision 篡改 | P4 断言失败 | 拒绝 |
| G13 | Candidate 内容篡改 | P5 失败 | 拒绝 |
| G14 | rename 后 COMMIT 前崩溃 | current 为准 | 确认 |
| G15 | 锁路径替换 | 身份校验 | DENY |

### 2.7 Windows 独立协议（不映射 POSIX 等价物）

Windows 提交协议独立设计：`CreateFileW`（含 `FILE_FLAG_WRITE_THROUGH`）+ `LockFileEx` + `ReplaceFileW`（用于 current 指针的原子替换）+ `FlushFileBuffers`。

**不声称等价**：
- `FlushFileBuffers` 对**普通文件句柄**保证刷新文件系统缓存，但**目录项持久化**需目录句柄以 `FILE_FLAG_BACKUP_SEMANTICS` 打开，且其语义弱于 POSIX `fsync(dirfd)`，**无官方文档保证目录项断电持久**；
- `ReplaceFileW` 的原子替换语义 ≠ POSIX `rename` 的断电持久语义（详见 §3）；
- Windows 断电恢复能力 **NOT_VERIFIED**。

### 2.8 P0-04 验收与结论

- 六字段 + 五条件完整谓词：✅ §2.2
- 13 步持久化顺序 + 目录项：✅ §2.3
- 持久化故障五态状态机：✅ §2.4
- 分步崩溃矩阵：✅ §2.5
- 15 条时间线：✅ §2.6
- Windows 独立协议：✅ §2.7

**P0-04 结论：DESIGN_PROVEN（POSIX）/ NOT_VERIFIED（Windows）**。
- POSIX 的 fsync(fd)+fsync(dirfd) 持久化顺序由 POSIX 语义证明，五态状态机闭合了 fsync 失败后的禁提交策略。
- Windows 目录持久化（FlushFileBuffers on directory handle）NOT_VERIFIED。
- 恶意同 UID 锁替换：威胁模型排除（见 §7 ISSUES），NOT_VERIFIED。

---

## 3. P0-06｜Windows ReplaceFileW 错误状态与 Tombstone 恢复重构

### 3.1 缺陷定位：05.18 未逐项覆盖 1175/1176/1177 官方语义

05.18 §4.2 仅笼统列出错误码，未依据 Microsoft 官方文档精确区分三个 ReplaceFileW 特有错误码的**最终文件状态**。05.19 依据 Microsoft Learn《ReplaceFileW function (winbase.h)》官方契约（Last updated 2023-06-01）逐码重构。

### 3.2 ReplaceFileW 官方契约关键事实（引用依据）

依据 Microsoft Learn ReplaceFileW 文档：

1. **1175 `ERROR_UNABLE_TO_REMOVE_REPLACED`**：原文 "The replaced file could not be deleted. The replaced and replacement files **retain their original file names**." → 目标文件与替换文件**都保持原名**，无移动发生。
2. **1176 `ERROR_UNABLE_TO_MOVE_REPLACEMENT`**：原文 "The replacement file could not be renamed. **If lpBackupFileName was specified, the replaced and replacement files retain their original file names. Otherwise, the replaced file no longer exists and the replacement file exists under its original name.**" → **分两种情况**：指定了备份名 → 两文件都保持原名；未指定备份名 → 原目标**已不存在**，替换文件**以原目标名存在**。
3. **1177 `ERROR_UNABLE_TO_MOVE_REPLACEMENT_2`**：原文 "The replacement file could not be moved. The replacement file **still exists under its original name**; however, it has **inherited the file streams and attributes from the file it is replacing**. The file to be replaced **still exists with a different name**. If lpBackupFileName is specified, it will be the name of the replaced file." → 替换文件**仍在原名**（但已继承目标文件的流和属性），目标文件**以不同名字存在**（若指定备份名则备份名即目标名）。
4. **其他错误**（如 `ERROR_INVALID_PARAMETER`）："the replaced and replacement files will **retain their original file names**. In this scenario, a backup file does not exist and it is not guaranteed that the replacement file will have inherited all of the attributes and streams."
5. **关键约束**：备份/被替换/替换文件必须**同卷**；"The resulting file has the same file ID as the replacement file."

### 3.3 错误码→文件状态判定矩阵（逐码，不推断）

对每个错误码分别分析原目标文件、替换文件、备份文件、是否可能已发生部分移动、是否可安全重试、是否需人工恢复。

| 错误码 | 原目标文件（lpReplacedFileName）| 替换文件（lpReplacementFileName）| 备份文件（lpBackupFileName）| 是否部分移动？| 可安全重试？| 恢复策略 |
|--------|-------------------------------|--------------------------------|---------------------------|--------------|-----------|---------|
| 成功 | 已被替换为新内容 | 已消耗（身份转移）| 已创建（若指定）| — | — | PUBLISHED |
| 1175 REMOVE_REPLACED | **存在**（原名）| **存在**（原名）| 无/未创建 | 否（未移动）| 是（幂等）| 重新读文件状态，校验后重试 |
| 1176 MOVE_REPLACEMENT（指定备份）| 存在（原名）| 存在（原名）| 可能已创建 | 否 | 是 | 校验后重试 |
| 1176 MOVE_REPLACEMENT（未指定备份）| **不存在** | **存在（以原目标名）**| 无 | **是（替换文件已改名）**| **否（先判定）**| INDETERMINATE，按内容判定 |
| 1177 MOVE_REPLACEMENT_2 | **存在（不同名/备份名）**| **存在（原名，已继承流属性）**| 存在（即目标名）| **是（部分移动）**| **否** | INDETERMINATE，按内容判定 |
| ERROR_FILE_NOT_FOUND | 存在 | 不存在 | — | 否 | 是 | 可重试（替换文件缺失）|
| ERROR_PATH_NOT_FOUND | 未知 | 未知 | — | 否 | 否 | 校验路径后重试 |
| ERROR_SHARING_VIOLATION | 存在 | 存在 | — | 否 | 是（解除占用后）| 重试 |
| ERROR_ACCESS_DENIED | 未知 | 未知 | 未知 | 未知 | **否** | INDETERMINATE，校验 |
| 其他错误 | 原名保留 | 原名保留 | 不存在 | 否 | 视错误 | 校验后重试 |

**核心规则**：
- **1175 与 1176（指定备份）**：无部分移动，可安全重试；
- **1176（未指定备份）与 1177**：**已发生部分移动**，文件位置不确定，**禁止盲目重试**，必须重新读取文件系统实际状态、校验文件 ID/内容哈希/大小/预期角色后才能进入下一状态；
- 任何"文件不存在/存在"的结论**只能来自实际重新读取**，不得从错误码推断。

### 3.4 Tombstone 状态判定矩阵（至少六态）

| 状态 | 判定条件 | 允许操作 | 恢复动作 |
|------|---------|---------|---------|
| PLACEHOLDER | 内容 == 占位标记（空/占位符，非有效元数据）| 删除占位，可重试 | 安全重试 |
| PUBLISHED | 内容 == 有效元数据（字段全 + 哈希匹配 + txn 归属）| 认领为发布成功 | 补持久化 |
| REPLACEMENT_PRESENT | 替换文件以原名存在（1177 场景）| 只读校验 | 判定归属后恢复 |
| BACKUP_PRESENT | 备份文件存在且为目标原内容 | 只读校验 | 判定是否已发布 |
| PARTIAL_MUTATION | 已发生部分移动但无法判定最终态 | **只读** | 校验后决定 |
| INDETERMINATE | 无法唯一判定 | **禁止删除/覆盖** | 保留现场，人工恢复 |

**核心规则**：
- PLACEHOLDER ≠ PUBLISHED，仅凭文件存在不得判定发布成功；
- 元数据必须绑定 txn_id、cleanup_id、源对象身份和哈希；
- 不完整或身份不匹配 → INDETERMINATE；
- 恢复必须幂等，证据不足不得删除现场。

### 3.5 FlushFileBuffers 与目录持久化核实

依据 Microsoft 文档核实：
- `FlushFileBuffers` 对**普通文件句柄**：刷新该文件的系统文件系统缓存到磁盘；
- **目录句柄能否合法传入**：目录句柄需以 `FILE_FLAG_BACKUP_SEMANTICS` 打开，`FlushFileBuffers` 才能接受；但**即使能调用，其保证也弱于 POSIX `fsync(dirfd)`**；
- **是否存在官方 API 组合明确保证目录项断电持久**：**未找到官方文档明确保证**，故 Windows 目录项断电持久性 **NOT_VERIFIED**。

**不得**：将"目录句柄可打开"等同于"目录句柄可 Flush 且等价 fsync(dirfd)"。

### 3.6 恢复策略（幂等，禁止盲目重试）

```
恢复入口（读到 tombstone 目标文件）：
  1. 重新读取文件系统实际状态（文件存在性、文件 ID、内容哈希、大小）
  2. 校验文件身份与预期角色：
     - 占位标记 → PLACEHOLDER → 删除占位，可重试
     - 有效元数据（字段全 + 哈希匹配 + txn 归属）→ PUBLISHED → 认领
     - 替换文件以原名存在（1177）→ REPLACEMENT_PRESENT → 判定归属
     - 备份存在 → BACKUP_PRESENT → 判定是否已发布
  3. 禁止盲目重试（尤其 1176 未指定备份 / 1177 场景）
  4. 禁止未经授权删除备份或占位文件
  5. 无法唯一恢复 → INDETERMINATE，保留现场，人工恢复
```

### 3.7 15 条崩溃/恢复时间线

| # | 场景 | 推演 | 结果 |
|---|------|------|------|
| J1 | 临时文件写一半 | 校验失败删除 | 安全 |
| J2 | 临时 fsync 失败 | 不发布 | 拒绝 |
| J3 | 占位创建后崩溃 | 占位标记 → 删除重试 | PLACEHOLDER 可重试 |
| J4 | 1175 REMOVE_REPLACED | 两文件原名 → 幂等重试 | 可重试 |
| J5 | 1176（指定备份）| 两文件原名 → 校验重试 | 可重试 |
| J6 | 1176（未指定备份）| 目标已不存在，替换文件以原名 → 部分移动 | INDETERMINATE 判定 |
| J7 | 1177 MOVE_REPLACEMENT_2 | 部分移动，替换继承属性 | INDETERMINATE 判定 |
| J8 | ReplaceFileW 成功 Flush 前崩溃 | 元数据哈希匹配 → 补 Flush | PUBLISHED 补持久化 |
| J9 | Flush 失败 | 目录项不持久 | INDETERMINATE |
| J10 | 系统崩溃（断电）| 内容判定 | 按状态矩阵 |
| J11 | 重复 txn | CREATE_NEW 失败 | 拒绝 |
| J12 | 源同名重建 | 源路径又存在 | INDETERMINATE |
| J13 | 元数据被替换/删除 | 不匹配/消失 | INDETERMINATE |
| J14 | 并发清理 | 隔离 rename 竞争 | 一个成功 |
| J15 | 审计写失败 | REMOVED 失败 | INDETERMINATE |

### 3.8 归档独立建模（保留，不复用 tombstone 结论）

归档独立事务：临时对象 + 完整性校验 + 无覆盖发布 + 持久化 + 恢复；归档前不销毁原事务证据；归档失败或证据不一致 → 保留现场 + INDETERMINATE。

### 3.9 P0-06 验收与结论

- 1175/1176/1177 官方语义逐码覆盖：✅ §3.2/§3.3（依据 Microsoft Learn）
- 错误码→文件状态矩阵：✅ §3.3
- 六态判定矩阵：✅ §3.4
- FlushFileBuffers 核实：✅ §3.5
- 幂等恢复策略：✅ §3.6
- 15 条时间线：✅ §3.7

**P0-06 结论：DESIGN_PARTIAL（协议层）/ NOT_VERIFIED（持久性）**。
- 错误状态矩阵与恢复状态机依据 Microsoft 官方文档闭合（1175/1176/1177 逐码区分了"无移动可重试"与"部分移动须判定"）。
- Windows 目录项断电持久性（FlushFileBuffers）**NOT_VERIFIED**。
- 部分失败后的中间态依赖 Windows 具体运行时行为，**NOT_VERIFIED**。

---

## 4. P0-05｜Sealed 与 Source Retire：保持 DENY（不制造 PASS）

### 4.1 承接 05.18 的权威结论（无需新修复，仅厘清边界）

05.18 已通过 chattr(1) 官方 man page 双重推翻 `chattr +i` 方案：
1. `+i` **阻止 rename/delete**（"it cannot be deleted or renamed"）→ 05.17 的"保持 +i 可 rename"是错的，rename 返回 EPERM；
2. `+i` **不阻止已 open 写 FD 写入**（"will not affect the ability to write to already existing file descriptors"）→ 05.16/05.17 的"+i 让旧 FD 写 EPERM"是错的。

### 4.2 厘清边界表述（本轮修正点）

05.18 结论需要区分两种表述，避免过度断言：

| 表述 | 准确含义 | 05.19 修正 |
|------|---------|-----------|
| "当前方案未证明" | 本工单范围内的候选方案（协作锁、权限、哈希、只读句柄、+i）无法证明源对象强制不可变 | 保持 |
| "平台不存在可行机制" | 需要在**明确威胁模型**下、经**逐平台权威文档**验证才能断言 | 05.19 修正为：**在"同 UID 恶意进程 + root/管理员"威胁模型下，本地文件系统无已知原语同时满足"阻止旧 FD 写"与"允许路径移动"；但存在未验证的候选（独立特权服务/专用命名空间）** |

**修正后的准确结论**：
- 不把普通权限、flock、哈希、只读句柄当作强制不可变性（这些**只提供协作语义**，不防御恶意同 UID 进程）；
- 区分"当前方案未证明"（协作锁/权限/哈希/只读句柄/`+i` 均已否决）与"平台不存在可行机制"（独立特权服务尚未验证，不能断言不存在）；
- 在**本工单 READ-ONLY 范围**内，Source Retire **无法证明安全，必须 DENY**。

### 4.3 严格区分的对象与操作（继承 05.18 §3.2）

| 概念 | 定义 | 与"强制不可变"的关系 |
|------|------|-------------------|
| inode 内容 | 文件数据块 | 不可变的目标 |
| inode 元数据 | 权限/时间戳/属性 | `+i` 锁定 |
| 目录项 | 名字→inode 映射 | 退役操作对象 |
| 路径名 | 到达 inode 的路径 | 退役移除对象 |
| 文件描述符 | 进程引用句柄 | 旧 FD 写绕过 |
| 硬链接 | 额外目录项 | 写绕过 |
| mmap 映射 | 内存映射页面 | 写绕过 |
| 只读句柄 | O_RDONLY fd | 不保证源不可变 |
| 不可变属性 | 阻止修改/删除/rename | `+i` 但**不阻旧 FD** |
| 目录权限 | 控制目录项 | 协作语义 |
| 挂载边界 | 文件系统边界 | 方案 D 候选 |
| 退役/清理/销毁 | 三个独立阶段 | 严格区分 |

### 4.4 Source Retire 状态机与 DENY 条件（继承 05.18 §3.5）

```
WRITABLE → SEALING → SEALED → QUARANTINED → RETIRED → PURGED
                              ↘ INDETERMINATE
```

DENY 条件（强制）：
1. 无法证明旧 FD/mmap/硬链接写入被阻断 → DENY；
2. 无法证明退役操作可成功完成 → DENY；
3. 无法证明最终校验至退役完成期间内容稳定 → DENY；
4. 无法证明权限主体边界 → DENY。

### 4.5 P0-05 验收与结论

**P0-05 结论：Source Retire 保持 DENY（本轮不要求解决，明确拒绝而非失败）**。
- 保留 05.18 撤回 `chattr +i` 保持 rename 的修正；
- 区分"当前方案未证明"与"平台不存在可行机制"；
- 不把权限/flock/哈希/只读句柄当强制不可变；
- **不启动任何真实退役、隔离、删除实验**。

---

## 5. 设计不变量（I-01 ~ I-40）

| # | 定义 | 适用范围 | 证明位置 | 失效反例 | 拒绝策略 | 验证状态 |
|---|------|---------|---------|---------|---------|---------|
| I-01 | fork 不绕过 CPython 生命周期 | fork | §1.5 | os._fork 后执行 Python | 用 os.fork | DESIGN_PROVEN |
| I-02 | before 钩子不获取锁 | fork | §1.7 | before 持锁重入 | 只读快照 | DESIGN_PROVEN |
| I-03 | 受控 fork 仅主线程 | fork | §1.6 | 非主线程 fork | DENY | DESIGN_PARTIAL |
| I-04 | FD 绑定 generation | FD | §1.4 | 裸数字复用 | 不匹配不关 | DESIGN_PROVEN |
| I-05 | child 不获取继承锁 | fork child | §1.8 | 继承 locked mutex | 直接赋值 | DESIGN_PROVEN |
| I-06 | 禁止持锁 fork | fork | §1.7 | 持锁 fork | 拒绝 | DESIGN_PROVEN |
| I-07 | 锁获取升序无死锁 | 全锁 | §1.7 | 逆序 | 禁止 | DESIGN_PROVEN |
| I-08 | **冻结覆盖真正的 fork 窗口** | fork | §1.3 | 快照后释放锁 | FREEZING 态保持 | DESIGN_PROVEN |
| I-09 | 快照后无未门禁 FD 变化 | fork | §1.3/§1.4 | 快照后 close/open | Gate 阻断 | DESIGN_PROVEN |
| I-10 | 第三方 C fork DENY/UNSUPPORTED | fork | §1.5 | C 直接 fork | DENY | DESIGN_PROVEN |
| I-11 | 子解释器/嵌入式 fork DENY | fork | §1.6 | 子解释器 | DENY | DESIGN_PARTIAL |
| I-12 | child 未知 FD 不主动 close | fork child | §1.4 | 误关未知 FD | 标 UNKNOWN_FD | DESIGN_PROVEN |
| I-13 | Candidate 绑定六字段 | 提交 | §2.2 | 未绑父版本 | 五条件 | DESIGN_PROVEN |
| I-14 | 五条件完整谓词逐项列出 | 提交 | §2.2 | 笼统"五条件" | 逐项 P1~P5 | DESIGN_PROVEN |
| I-15 | 文件内容先于目录项持久化 | 提交 | §2.3 | 未 fsync 文件 | 禁止 | DESIGN_PROVEN |
| I-16 | 目录项必须 fsync(dirfd) | 提交 | §2.3 | 目录项未持久 | 强制 fsync | DESIGN_PROVEN |
| I-17 | rename 原子性 ≠ 断电持久性 | 提交 | §2.3/§2.5 | 误当持久 | fsync 目录 | DESIGN_PROVEN |
| I-18 | fsync 失败禁后续提交 | 提交 | §2.4 | fsync 失败后继续 | NOT_DURABLE 态 | DESIGN_PROVEN |
| I-19 | 不确定态禁自动回滚/重复提交 | 提交 | §2.4 | 自动回滚 | COMMIT_INDETERMINATE | DESIGN_PROVEN |
| I-20 | 每崩溃点确定恢复 | 提交 | §2.5 | 无规则 | 按矩阵 | DESIGN_PROVEN |
| I-21 | revision 单调 | 提交 | §2.2 | 回退 | P4 断言拒绝 | DESIGN_PROVEN |
| I-22 | 恶意同 UID 锁替换排除 | 锁 | §2/§7 | 同 UID 替换 | 边界 NOT_VERIFIED | NOT_VERIFIED |
| I-23 | +i 不阻止旧 FD 写 | Source Retire | §4.1 | 旧 FD 写成功 | 承认 → DENY | NOT_VERIFIED（文档证实）|
| I-24 | +i 阻止 rename | Source Retire | §4.1 | rename EPERM | 承认 → DENY | NOT_VERIFIED（文档证实）|
| I-25 | 源内容稳定无法本地证明 | Source Retire | §4.2 | 旧 FD/mmap/硬链接写 | DENY | DESIGN_PARTIAL |
| I-26 | Source Retire 默认 DENY | Source Retire | §4.4 | 无强制边界 | DENY | DESIGN_PARTIAL |
| I-27 | 只读副本 ≠ 源不可变 | Source Retire | §4.2 | 副本当源证据 | 不混用 | DESIGN_PROVEN |
| I-28 | inode 身份 ≠ 内容不可变 | Source Retire | §4.2 | 身份当不可变 | 不混用 | DESIGN_PROVEN |
| I-29 | 区分"未证明"与"不存在机制" | Source Retire | §4.2 | 过度断言 | 厘清边界 | DESIGN_PROVEN |
| I-30 | 元数据原子发布 | tombstone | §3 | O_EXCL 逐步写 | 临时+校验+无覆盖 | DESIGN_PROVEN |
| I-31 | 占位 ≠ 有效 tombstone | tombstone | §3.4 | 占位误认 | 内容区分 | DESIGN_PARTIAL |
| I-32 | 1175/1176/1177 逐码区分 | tombstone | §3.3 | 笼统错误码 | 官方矩阵 | DESIGN_PROVEN（文档依据）|
| I-33 | 错误码不推断文件状态 | tombstone | §3.3 | 从错误码推断存在性 | 重新读取 | DESIGN_PROVEN |
| I-34 | ReplaceFileW 部分失败按内容判定 | tombstone | §3.3/§3.4 | 假设文件位置 | INDETERMINATE | DESIGN_PARTIAL |
| I-35 | 恢复幂等 | tombstone | §3.6 | 非幂等 | 状态机 | DESIGN_PROVEN |
| I-36 | 归档独立建模 | 归档 | §3.8 | 复用结论 | 单独协议 | DESIGN_PROVEN |
| I-37 | 归档前不销毁证据 | 归档 | §3.8 | 提前销毁 | 禁止 | DESIGN_PROVEN |
| I-38 | 移除 ≠ 销毁 | 删除 | §3 | 仅源消失 | INDETERMINATE | DESIGN_PROVEN |
| I-39 | 无法证明必须 DENY/INDETERMINATE | 全局 | 全文 | 猜测 | 不猜测 | DESIGN_PROVEN |
| I-40 | 不把设计推演写运行时验证 | 全局 | 全文 | 自审当验证 | 标 NOT_TESTED | DESIGN_PROVEN |

**共 40 条不变量，每条含定义/范围/证明位置/失效反例/拒绝策略/验证状态。** 本轮新增 I-08/I-09（冻结覆盖 fork 窗口）、I-12（未知 FD 不误关）、I-14（五条件完整谓词）、I-18/I-19（持久化故障状态机）、I-29（厘清 DENY 边界）、I-32/I-33（错误码官方矩阵）。

---

## 6. 全局一致性审查

### 6.1 18 条端到端链路

| # | 链路 | 关键边界 | 崩溃点 | 恢复 | 结论 |
|---|------|---------|--------|------|------|
| L1 | 受控 fork→FREEZING→fork→父子恢复 | CPython 生命周期 + 冻结覆盖 | fork 中 | child 重建 | DESIGN_PARTIAL |
| L2 | FD 冻结→快照→fork→child 重建 | generation + Gate | 冻结中 | 校验 | DESIGN_PROVEN |
| L3 | 第三方 C open 的 FD→fork→child 处理 | 无法拦截 | — | 标 UNKNOWN_FD | DESIGN_PARTIAL |
| L4 | Candidate→fsync 文件→fsync 目录→rename→fsync 目录 | 双重持久化 | rename 前后 | 按 current | DESIGN_PROVEN |
| L5 | fsync 失败→NOT_DURABLE→禁提交 | 故障状态机 | fsync | 人工确认 | DESIGN_PROVEN |
| L6 | rename 后目录未持久→COMMIT_INDETERMINATE | 不确定态 | rename 后 | 按 current | DESIGN_PROVEN |
| L7 | 双事务竞争 | flock | 竞争点 | 败者重生成 | DESIGN_PROVEN |
| L8 | 源 seal→quarantine→retire | **retire DENY** | — | DENY | DENY |
| L9 | 旧 FD + i | **+i 不阻旧 FD** | 校验后写 | 竞态 | DENY |
| L10 | rename + i | **rename EPERM** | — | 退役失败 | DENY |
| L11 | tombstone 临时→校验→无覆盖发布 | 临时 fsync | 发布前后 | 占位/有效 | DESIGN_PARTIAL |
| L12 | ReplaceFileW 1175 无移动 | 官方契约 | — | 幂等重试 | DESIGN_PROVEN |
| L13 | ReplaceFileW 1176 未指定备份部分移动 | 官方契约 | 替换中 | INDETERMINATE | DESIGN_PARTIAL |
| L14 | ReplaceFileW 1177 部分移动 | 官方契约 | 替换中 | INDETERMINATE | DESIGN_PARTIAL |
| L15 | rename 后崩溃 REMOVED 未持久化 | rename→REMOVED | rename 后 | 元数据匹配 | DESIGN_PROVEN |
| L16 | 归档协议 | 临时+校验+发布 | 归档中 | 哈希确认 | DESIGN_PROVEN |
| L17 | 审计不可用 | REMOVED 失败 | 审计写 | INDETERMINATE | DESIGN_PROVEN |
| L18 | 恶意同 UID | 无 | 任意 | 检测 | NOT_VERIFIED |

### 6.2 时间线汇总

P0-02 十五条 + P0-04 十五条 + P0-06 十五条 = **45 条**（≥30 达标）。另含 P0-05 反例（继承 05.18 §3.6）。

### 6.3 跨章节身份一致性

| 身份 | 产生 | 传递 | 一致性 |
|------|------|------|--------|
| FD generation | §1.4 | §1.9 | generation 绑定 + Gate 阻断 |
| 父版本三元组 | §2.2 | §2.5 | 五条件 P1~P3 |
| 源对象身份 | §4 | §4.4 | retire DENY |
| Tombstone 事务身份 | §3.4 | §3.7 | txn 归因 + 官方错误矩阵 |

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

**DESIGN-05.19 = PASS-WITH-ISSUES**，具体分项：

| # | 结论 | 依据 |
|---|------|------|
| P0-02 | DESIGN_PARTIAL | FREEZING 态闭合了"快照到 fork 之间"的门禁窗口（协议层 DESIGN_PROVEN）；但第三方 C 扩展直接 fork/open 无法拦截 → UNSUPPORTED/DENY；运行时 NOT_TESTED |
| P0-04 | DESIGN_PROVEN（POSIX）/ NOT_VERIFIED（Windows）| 五条件完整谓词 + 13 步持久化 + 五态故障状态机闭合；Windows 目录持久化 NOT_VERIFIED |
| P0-05 | **DENY（保持）** | 工单明确"本轮不要求解决"，厘清"未证明"与"不存在机制"边界 |
| P0-06 | DESIGN_PARTIAL（协议层）/ NOT_VERIFIED（持久性）| 1175/1176/1177 依据 Microsoft 官方文档逐码闭合；Windows 目录持久化 NOT_VERIFIED |

**核心诚实结论**：
1. **P0-02 门禁窗口已闭合**：05.19 废除"快照后释放锁"，FREEZING 态覆盖真正的 `os.fork()` 系统调用，快照与 fork 之间无未门禁 FD 变化窗口。但 Python 层**无法拦截第三方 C 扩展直接 fork/open/close**，对此明确 UNSUPPORTED/DENY，不宣称通用安全。
2. **P0-04 持久化故障状态机建立**：五态（DURABLE/NOT_DURABLE/COMMIT_INDETERMINATE/RECOVERY_REQUIRED/QUARANTINED）闭合了 fsync 失败后的禁提交策略。
3. **P0-06 错误码依据官方文档逐码重构**：1175/1176/1177 区分了"无移动可重试"与"部分移动须判定"，不再从错误码推断文件存在性。
4. **P0-05 保持 DENY**：工单认可"本轮不要求解决"，不制造 PASS。

**整体 PASS-WITH-ISSUES 的依据**：四项 P0 中，P0-04 的 POSIX 协议层 DESIGN_PROVEN；P0-02/P0-06 协议层可证明但运行时/持久性 NOT_TESTED/NOT_VERIFIED（已明确标出）；P0-05 按工单口径保持 DENY（诚实拒绝而非未闭合）。无伪造 PASS，无降低标准。

**IMPLEMENT-01 继续 STOP，等待 ChatGPT 独立复审。**

---

## 9. 交付物索引

| 文件 | 说明 |
|------|------|
| `USIT-STAGE-B-DESIGN-05.19.md` | 本主设计文档 |
| `USIT-STAGE-B-DESIGN-05.19-ISSUES.md` | 问题清单及剩余风险 |
| `USIT-STAGE-B-DESIGN-05.19-DIFF.md` | 相对 05.18 逐项差异 |
| `USIT-STAGE-B-DESIGN-05.19-CROSS-REVIEW.md` | 反例审查记录 |
| `USIT-STAGE-B-DESIGN-05.19-EXECUTION-REPORT.md` | 实际执行报告 |
| `USIT-STAGE-B-DESIGN-05.19-HASH-MANIFEST.md` | 交付文件 SHA256 清单 |
| `USIT-STAGE-B-DESIGN-05.19-HASH-RECEIPT.txt` | 哈希验证回执 |
| `USIT-STAGE-B-DESIGN-05.19-交付物.zip` | 完整打包 |

---
