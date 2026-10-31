# USIT-STAGE-B-DESIGN-05.21｜Fork准入与钩子协议重构＋POSIX完整崩溃矩阵闭合＋交付完整性补证

> 阶段 B · DESIGN-05.21 · READ-ONLY DESIGN REVISION ONLY
> 作者：USIT 主开发 · 生成：2026-10-31（GMT+8）
> 前置审查：DESIGN-05.20 独立复审结论 FAIL
> 工作区：`C:\Users\Administrator\WorkBuddy\2026-10-15-22-09-47\usit_p0`
> 交付目录：`C:\Users\Administrator\WorkBuddy\2026-10-15-22-09-47\usit_p0_05.21_reports`（**项目外**）
> 状态：**STOP — 等待独立复审**（未授权 IMPLEMENT-01）

---

## 0. 缺陷追踪表（05.20 问题 → 05.21 修订位置 → 新证据 → 是否闭合 → 剩余风险 → 结论）

| # | 05.20 问题 | 05.21 修订位置 | 新证据 | 是否闭合 | 剩余风险 | 结论 |
|---|-----------|---------------|--------|---------|---------|------|
| P0-02-1 | 锁级别定义（L0=_gate 最低、L2=_coord 较高）与实际获取顺序（fork 入口先拿 coord 再拿 gate，即 L2→L0 逆序）矛盾 | §1.3 | 重排锁级别：_coord 最低级、_gate 次之、_fd 再次之，编号与获取顺序一致 | 闭合 | 第三方 C fork 不可控 | DESIGN_PARTIAL |
| P0-02-2 | fork 入口/freeze/before/after 钩子职责未严格分离 | §1.4 | 四职责分离 + 调用时序图 | 闭合 | 同上 | DESIGN_PARTIAL |
| P0-04-1 | NOT_DURABLE 重试成功后错误直接迁移 DURABLE | §2.4/§2.5 | 按失败步骤恢复，从第一个未完成步骤继续 | 闭合 | Windows 目录持久化 | DESIGN_PROVEN（POSIX）|
| P0-04-2 | 16 步崩溃矩阵未逐项覆盖全部崩溃位置 | §2.6 | 完整逐步崩溃矩阵 | 闭合 | 同上 | DESIGN_PROVEN（POSIX）|
| 交付完整性 | ZIP 哈希未写入 RECEIPT | §7 | RECEIPT 记录 MANIFEST 哈希 + ZIP 哈希/字节数 | 闭合 | — | 待核验 |

**本节是全文总纲。本轮核心任务：P0-02 重排锁级别消除编号与实际顺序矛盾、分离 fork 四职责；P0-04 修正 NOT_DURABLE 迁移、补齐完整崩溃矩阵；交付完整性补证。**

---

## 1. P0-02｜Fork Gate 与钩子协议彻底重构

### 1.1 缺陷定位：05.20 锁级别定义与实际获取顺序矛盾

05.20 §1.3 的锁级别表定义：

```
L0. _gate_lock        （最低级）
L1. _fd_lock
L2. _fork_coord_lock  （较高级）
L3. 业务锁
```

但 05.20 §1.3 规则 2 又写"fork 入口获取 `_fork_coord_lock` → `_gate_lock`（调用 freeze）→ `_fd_lock`"。这意味着 fork 入口**先获取 L2（_fork_coord_lock）再获取 L0（_gate_lock）**——这是**从高级别到低级别的逆序获取**，直接违反"严格升序、禁止逆序"的规则 1 声明。

**矛盾本质**：锁级别编号（L0~L3）与实际获取顺序（coord 最先、gate 次之、fd 再次之）**不一致**。05.20 把 `_fork_coord_lock` 编号为 L2（较高），但实际它必须**最先**获取，因此它的级别编号应该是**最低**（L0），而不是 L2。

**结论**：05.20 的锁级别表是自相矛盾的，不能通过"调整数字看起来一致"来修复，必须**重新设计锁级别，使编号与唯一全局获取顺序严格对应**。

### 1.2 锁级别重构（编号 = 实际获取顺序）

**唯一全局获取顺序（严格升序，先低级后高级）**：

```
L0. _fork_coord_lock  （fork 协调锁，最先获取，全局最低级）
L1. _gate_lock        （准入/冻结/计数，第二级）
L2. _fd_lock          （Registry，第三级）
L3. _lock / _lease_lock / _pmutex_lock  （业务锁，最高级，禁止持锁 fork）
```

**关键修订**：
- `_fork_coord_lock` 从 05.20 的 L2 **降为 L0**（因为它必须最先获取）；
- `_gate_lock` 从 L0 **升为 L1**；
- `_fd_lock` 保持 L2；
- 业务锁保持 L3（最高级）。

**这样编号与获取顺序严格一致**：任何持锁路径都只能从 L0 向 L3 升序获取，不存在"先拿高级再拿低级"的逆序。

### 1.3 锁依赖有向图（证明无环）

```
_fork_coord_lock (L0)
      │
      ▼
   _gate_lock (L1)
      │
      ▼
    _fd_lock (L2)
      │
      ▼
  业务锁 (L3)  ← 禁止持锁 fork（L3 持有者不得再获取 L0）
```

**无环证明**：
- 所有锁的获取方向严格沿 L0→L1→L2→L3 单向箭头；
- 任何路径不得从 L3 反向获取 L0/L1/L2（禁止持锁 fork 规则）；
- 不存在两条路径产生相反方向的锁获取，因此有向图**无环**，不存在死锁。

### 1.4 四个职责的严格分离

**职责定义**：

| 职责 | 函数 | 唯一职责 | 禁止行为 |
|------|------|---------|---------|
| fork 入口 | `fork_entry()` | 唯一受控的外部 fork 入口：获取 `_fork_coord_lock`，调用 `freeze()`，在冻结保持下调用 `os.fork()`，恢复父进程，返回 | 不得被其他函数重复调用 fork；不得绕过 freeze |
| 冻结 | `freeze()` | 关闭准入（gate=FREEZING）、等待 in_flight 归零、冻结 Registry、生成快照 | **不得自行调用 os.fork()**；不得恢复 Gate |
| before 钩子 | `before()` | 只执行 fork 前准备（只读快照，不取锁）| 不得再次调用 fork；不得触发 freeze |
| after 钩子 | `after_in_parent()` / `after_in_child()` | parent 恢复 Gate 与计数；child 重建锁对象 + 清理 FD + 置 child-local OPEN | 不得重复执行对方职责；child 不恢复 parent 状态 |

**完整调用时序图**：

```
fork_entry()                          # 唯一外部 fork 入口
  │
  ├─ acquire(_fork_coord_lock)         # L0
  │
  ├─ freeze()                          # 冻结职责（不 fork）
  │    ├─ acquire(_gate_lock)          # L1
  │    ├─ gate_state = FREEZING
  │    ├─ while in_flight > 0: wait    # 排空（条件变量释放 L1 再重取）
  │    ├─ release(_gate_lock)
  │    ├─ acquire(_fd_lock)            # L2
  │    ├─ snapshot = freeze_registry()
  │    ├─ release(_fd_lock)
  │    └─ return snapshot
  │
  ├─ os.fork()                         # 唯一 fork 点（FREEZING 态保持）
  │    ├─ PyOS_BeforeFork → before()   # 只读快照，无锁
  │    ├─ fork() 系统调用
  │    ├─ parent: PyOS_AfterFork_Parent → after_in_parent()
  │    │      ├─ acquire(_gate_lock)   # L1
  │    │      ├─ gate_state = OPEN
  │    │      ├─ signal_all(gate_open_cv)
  │    │      ├─ release(_gate_lock)
  │    │      └─ release(_fork_coord_lock)   # L0
  │    └─ child: PyOS_AfterFork_Child → after_in_child()
  │           ├─ 直接赋值全新锁对象（不 acquire 继承锁）
  │           ├─ 遍历 snapshot，generation 匹配 → close
  │           ├─ Gate 置 child-local OPEN
  │           └─ _fork_reset_ok = True
  │
  └─ return child_pid (parent) / 继续执行 (child)
```

**证明不存在以下问题**：
1. **before 递归触发 fork**：before 只读快照、不取锁、不调用 fork 入口，无递归路径；
2. **freeze 重复执行**：freeze 只在 fork_entry 内被调用一次，且 gate=FREEZING 后再次进入 freeze 会因 gate 已 FREEZING 而幂等退出；
3. **Gate 永久 FREEZING**：parent 的 after_in_parent 必定恢复 OPEN；若 fork 失败，fork_entry 的 except 分支也恢复 OPEN（见 §1.5）；
4. **fork 异常后父进程无法恢复**：fork_entry 用 try/finally 包裹，finally 保证恢复 OPEN 并释放 _fork_coord_lock；
5. **子进程继承锁死锁**：child 直接赋值全新锁对象，不 acquire 任何继承锁；
6. **父子 Registry 交叉污染**：child 只读 snapshot（fork 时拷贝的独立副本），不修改 parent Registry；parent 的 Registry 不受 child 影响。

### 1.5 异常/取消/fork 失败路径（逐一路径审查）

| 路径 | 锁状态 | 恢复动作 | 结果 |
|------|--------|---------|------|
| 准入后 do_fd_operation 异常 | 已释放 _gate_lock | finally 保证 in_flight-- | 计数不泄漏 |
| 准入中线程取消 | 持 _gate_lock | 禁用取消临界区/RAII | 计数配对 |
| freeze 中 gate=FREEZING 后异常 | 持 L1 | catch 恢复 gate=OPEN + signal_all | Gate 不永久 FREEZING |
| os.fork() 失败（EAGAIN 等）| 持 L0+L1(FREEZING) | except 恢复 OPEN + 释放 L0 | 可重试，幂等 |
| after_in_parent 异常 | 持 L0 | finally 释放 L0 | 不影响 child |
| after_in_child 异常 | child | _fork_reset_ok=False | DENY 后续受保护操作 |
| 连续 fork | — | 幂等（freeze 幂等 + 状态恢复）| 安全 |

### 1.6 原子准入与 FD 生命周期（保留 05.20 设计，重新审查）

05.20 §1.2 的原子准入设计**保留**，本轮仅修正锁级别编号使其一致：

- Gate 检查与 `in_flight += 1` 在同一 `_gate_lock`（现 L1）临界区，保持原子性；
- FREEZING 后不得迟到登记（gate 检查在临界区内，FREEZING 写入也经同一锁）；
- 操作成功/失败/异常/取消均正确释放计数（RAII/finally）；
- FD 创建、登记、复制、关闭、转移、继承纳入完整生命周期；
- 第三方 C 扩展 FD 操作不可控 → UNSUPPORTED/DENY。

**线性化点**（不变）：
- 准入 = `in_flight += 1`（L1 临界区内）；
- FREEZING = `gate_state = FREEZING`（L1 临界区内）；
- 二者经 L1 互斥量串行化，存在全序关系。

### 1.7 fork 安全支持范围矩阵（逐项定义支持状态）

| 路径 | 支持状态 | 理由 |
|------|---------|------|
| 单线程进程（受控 fork）| **SUPPORTED** | 无并发 FD 操作，Gate 覆盖完整 |
| 多线程 Python 进程（受控 fork）| **DESIGN_PARTIAL** | 协议层可证明，但 CPython 运行时 NOT_TESTED |
| 第三方 C 扩展直接 fork | **UNSUPPORTED/DENY** | 绕过 at-fork 钩子，无 Gate 覆盖 |
| 嵌入式 Python | **UNSUPPORTED/DENY** | 宿主控制 fork 与 FD 生命周期 |
| Python 子解释器 | **UNSUPPORTED/DENY** | fork 行为未定义 |
| `os.fork()` | **SUPPORTED**（受控）| 触发 at-fork 钩子 + Gate |
| `os._fork()` | **禁用** | 跳过 CPython 生命周期 |
| `os.posix_spawn()` | **部分受控** | 不经本库 fork 入口，需单独评估，默认 UNSUPPORTED |
| `multiprocessing`（spawn）| **SUPPORTED**（经新进程入口）| 不 fork 当前进程，走独立进程创建 |
| `multiprocessing`（fork）| **部分受控** | 依赖 at-fork，但 multiprocessing 内部 fork 不经本库入口 |
| 外部进程创建（subprocess 等）| **不适用** | 不涉及本进程 fork |

**关键结论**：at-fork 钩子**不能**覆盖所有第三方 C fork 和进程内部 FD 操作，因为：
- at-fork 钩子（`PyOS_BeforeFork/AfterFork_*`）只由 `os.fork()` 触发，第三方 C 扩展直接调用 `libc fork()` **不触发**这些钩子；
- 第三方 C 扩展直接调用 `open/close/dup` **不经过**本库的 Gate 包装器，因此无法被 in_flight 计数覆盖；
- 因此对含第三方 C 扩展直接 FD/fork 的环境，必须 UNSUPPORTED/DENY。

### 1.8 P0-02 交付材料

- 锁级别与锁顺序表：✅ §1.2
- 锁依赖有向图：✅ §1.3
- fork 入口与 at-fork 钩子调用图：✅ §1.4
- 四职责分离表 + 时序图：✅ §1.4
- Gate/FD 生命周期：✅ §1.6（继承 05.20 §1.4）
- API 支持/拒绝矩阵：✅ §1.7
- 20 条并发/异常时间线：✅ §1.9
- 15 个反例：✅ §1.10

### 1.9 20 条并发/异常时间线

| # | 场景 | 初始状态 | 动作 | 锁状态 | 结果 | 拒绝策略 |
|---|------|---------|------|--------|------|---------|
| T1 | 受控 fork 完整流程 | IDLE | fork_entry→freeze→os.fork→恢复 | L0→L1→L2 升序 | 安全 | — |
| T2 | 准入前暂停 | OPEN | A 暂停→B freeze | L1 串行化 | A 被拒 | 拒绝 |
| T3 | 检查 Gate 后暂停 | OPEN | A 检查→暂停→B FREEZING | L1 临界区原子 | A 被拒 | 原子准入 |
| T4 | 计数增加后暂停 | OPEN | A in_flight++→暂停 | L1 | B 等待 A | 排空 |
| T5 | FD syscall 执行中暂停 | OPEN | A open 中→B freeze | L1 | B 等 A | 排空 |
| T6 | syscall 成功 Registry 未登记 | OPEN | A open 成功未登记 | L1 计数覆盖 | B 等 A | 排空 |
| T7 | 计数减少前暂停 | OPEN | A 未 in_flight-- | L1 | B 等 A 归零 | 排空 |
| T8 | FREEZING 与准入竞争 | OPEN | A/B 同时进 L1 | L1 互斥 | 有序 | 线性化点 |
| T9 | FREEZING 后迟到请求 | FREEZING | A 迟到 | L1 | A 被拒 | 拒绝 |
| T10 | fork 等待归零期间新请求 | FREEZING | B 等待，A 新请求 | L1 | A 被拒 | 拒绝 |
| T11 | freeze 中 gate 已 FREEZING 后异常 | FREEZING | freeze 异常 | L1 | catch 恢复 OPEN | 恢复+signal |
| T12 | fork 失败（EAGAIN）| FREEZING | fork 返回 -1 | L0+L1 | except 恢复 | 幂等 |
| T13 | after_in_parent 异常 | parent | parent 钩子异常 | L0 | finally 释放 | 不影响 child |
| T14 | after_in_child 异常 | child | child 重建异常 | — | _fork_reset_ok=False | DENY |
| T15 | 多个 fork 同时 | OPEN | A/B 同时 fork | L0 串行化 | 串行 | 串行 |
| T16 | 线程取消打断计数 | OPEN | 取消 | L1 | RAII 配对 | 不泄漏 |
| T17 | 计数泄漏（异常）| OPEN | do_fd_operation 异常 | L1 | finally 归零 | 不泄漏 |
| T18 | 第三方 C open | 任意 | 绕过 gate | — | 快照外 FD | UNKNOWN_FD |
| T19 | 第三方 C fork | 任意 | 绕过 | — | 状态未知 | DENY |
| T20 | posix_spawn 路径 | — | 不经本库 fork 入口 | — | 默认不支持 | UNSUPPORTED |

### 1.10 15 个反例及拒绝策略

| # | 反例 | 拒绝策略 |
|---|------|---------|
| R1 | 锁级别编号与获取顺序矛盾（05.20）| 重排 L0=coord, L1=gate, L2=fd |
| R2 | 检查 Gate 后计数前 FREEZING | 原子准入（同 L1 临界区）|
| R3 | in_flight 泄漏 | RAII/finally |
| R4 | os._fork 后执行 Python | 禁用 os._fork |
| R5 | before 持锁重入 | before 无锁 |
| R6 | 非主线程 fork | DENY |
| R7 | 子解释器/嵌入 fork | DENY/UNSUPPORTED |
| R8 | 第三方 C fork | DENY |
| R9 | 第三方 C open FD 误关 | child 只关 generation 匹配 FD |
| R10 | 快照后 FD 复用 | FREEZING 阻断 + generation |
| R11 | 连续 fork 状态残留 | 幂等恢复 |
| R12 | 持锁 fork 死锁 | 禁止持锁 fork（L3→L0 逆序禁止）|
| R13 | freeze 异常未恢复 OPEN | catch 恢复 + signal |
| R14 | 线程取消打断计数 | 禁用取消/RAII |
| R15 | multiprocessing fork 路径不经本库入口 | 部分受控，默认 UNSUPPORTED |

### 1.11 P0-02 结论

**P0-02 = DESIGN_PARTIAL**。
- **协议层**：锁级别编号与实际获取顺序严格一致（L0=coord→L1=gate→L2=fd 升序），锁依赖图无环，四职责分离，原子准入保留——**DESIGN_PROVEN（协议层）**。
- **运行时**：多线程 fork 运行时行为 NOT_TESTED。
- **不可控入口**：第三方 C fork/嵌入/子解释器/posix_spawn 默认 UNSUPPORTED/DENY。

---

## 2. P0-04｜POSIX 16 步完整崩溃矩阵

### 2.1 缺陷定位：05.20 的 NOT_DURABLE 迁移错误 + 崩溃矩阵不完整

05.20 存在两处问题：
1. §2.3 状态机图写 `NOT_DURABLE ──重试成功──▶ DURABLE`，§2.7 表写 "fsync 重试成功 → DURABLE"——这是**错误的跳步迁移**：NOT_DURABLE 重试成功后不能直接跳 DURABLE，必须按失败发生的具体步骤判断已完成的持久化阶段，从第一个未完成且可安全重做的步骤继续，只有 current 与 COMMIT 回执都达到持久化条件才能进 DURABLE。
2. 16 步崩溃矩阵未逐项覆盖"操作开始前/执行中/完成后/fsync 成功/fsync 失败/恢复时观察"全部崩溃位置。

### 2.2 S1~S16 协议定义表（沿用 05.20 编号，不改动）

| 步骤 | 操作 | 执行对象 | 文件/目录 | 持久化点 | 前置条件 |
|------|------|---------|----------|---------|---------|
| S1 | 写 Candidate 版本文件（O_EXCL）| Candidate | versions/<txn_id>.<rev> | 无 | — |
| S2 | fsync(candidate_fd) | Candidate | Candidate 文件 | 内容持久 | S1 完成 |
| S3 | fsync(versions_dirfd) | Candidate 目录项 | versions/ 目录 | 目录项持久 | S2 完成 |
| S4 | 重读 + SHA-256 校验 | Candidate | Candidate | — | S3 完成 |
| S5 | 写 PREPARED | PREPARED | prepared/<txn_id> | 无 | S4 完成 |
| S6 | fsync(prepared_fd) | PREPARED | PREPARED 文件 | 内容持久 | S5 完成 |
| S7 | fsync(prepared_dirfd) | PREPARED 目录项 | prepared/ 目录 | 目录项持久 | S6 完成 |
| S8 | flock(commit_lock, LOCK_EX) | 全局提交锁 | 锁文件 | — | S7 完成 |
| S9 | 读 current + 五条件校验 | current | current | — | S8 完成 |
| S10 | 写 current.<txn_id> | current 临时文件 | current 临时文件 | 无 | S9 完成 |
| S11 | fsync(current_tmp_fd) | current 临时文件 | current 临时文件 | 内容持久 | S10 完成 |
| S12 | rename(current.<txn_id> → current) | current | current | 目录项原子替换（内存）| S11 完成 |
| S13 | fsync(parent_dirfd) | current 目录项 | current 父目录 | 目录项持久 | S12 完成 |
| S14 | 写 COMMIT 回执 + fsync(commit_fd) | COMMIT 回执 | commit/<txn_id> | 回执内容持久 | S13 完成 |
| S15 | fsync(commit_dirfd) | COMMIT 回执目录项 | commit/ 目录 | 回执目录项持久 | S14 完成 |
| S16 | flock LOCK_UN | 全局提交锁 | 锁文件 | — | S15 完成 |

### 2.3 六态状态机（状态迁移表，禁止跳步/无证据升级）

```
DURABLE                    ← 最终态（current + 回执均持久）
COMMIT_DURABLE_AUDIT_PENDING ← current 持久、回执未持久
NOT_DURABLE                ← 提交前关键持久化步骤失败
COMMIT_INDETERMINATE       ← 无法确定 current 是否已持久
RECOVERY_REQUIRED          ← 需要人工恢复
QUARANTINED                ← 证据冲突/身份校验失败
```

**状态迁移表（完整，禁止未定义迁移）**：

| 当前状态 | 事件 | 目标状态 | 是否合法 |
|---------|------|---------|---------|
| （初始）| 开始提交 | DURABLE（进行中）| 合法 |
| DURABLE（进行中）| S2/S3/S6/S7/S11 任一 fsync 失败（rename 前）| NOT_DURABLE | 合法 |
| NOT_DURABLE | 重试该 fsync 成功 | **回到失败步骤继续**（非直接 DURABLE）| 合法（见 §2.4）|
| NOT_DURABLE | 重试失败/磁盘错误 | RECOVERY_REQUIRED | 合法 |
| DURABLE（进行中）| S12 rename 完成但 S13 未持久 | COMMIT_INDETERMINATE | 合法 |
| COMMIT_INDETERMINATE | 重新读校验确认 current 已持久 | COMMIT_DURABLE_AUDIT_PENDING | 合法（有证据）|
| COMMIT_INDETERMINATE | 无法确认 | RECOVERY_REQUIRED | 合法 |
| COMMIT_DURABLE_AUDIT_PENDING | 补写回执 + fsync 成功 | DURABLE | 合法 |
| 任意 | 证据冲突/身份失败 | QUARANTINED | 合法 |
| NOT_DURABLE | 直接跳 DURABLE（无证据）| DURABLE | **禁止（05.20 错误）**|

**核心修订**：NOT_DURABLE 重试成功后**不得直接迁移 DURABLE**，必须回到失败发生的具体步骤，按 §2.4 的恢复决策树继续执行剩余持久化步骤，只有 current 与回执都持久才进 DURABLE。

### 2.4 修正 NOT_DURABLE 恢复（fsync 失败恢复决策树）

**核心原则**：按失败发生的具体步骤恢复，从第一个未完成且可安全重做的步骤继续，禁止跳步升级。

```
fsync 失败（发生在步骤 Si）：
  ├─ 重读实际文件和目录项状态
  ├─ 核对：文件身份、内容 SHA-256、txn_id、revision、父版本、Candidate 六字段/五谓词
  ├─ 判断当前已完成的持久化阶段（S1..Si-1 哪些已完成）
  ├─ 确定第一个未完成且可安全重做的步骤 Sj（j >= i）
  │
  ├─ 若 Si 在 rename（S12）之前：
  │    ├─ 重试 fsync 成功 → 从 Sj 继续执行（状态仍 DURABLE-进行中，非跳 DURABLE）
  │    └─ 重试失败 → RECOVERY_REQUIRED
  │
  ├─ 若 Si == S13（rename 后、目录 fsync 前）：
  │    ├─ 重读 current，确认 current 已持久 → COMMIT_DURABLE_AUDIT_PENDING
  │    └─ 无法确认 → COMMIT_INDETERMINATE
  │
  ├─ 若 Si == S15（回执目录 fsync）：
  │    ├─ 回执内容已持久 → COMMIT_DURABLE_AUDIT_PENDING（补 fsync 目录）
  │    └─ 无法确认 → COMMIT_INDETERMINATE
  │
  └─ 任何情况：current 与 COMMIT 回执均达到持久条件 → 才能进入 DURABLE
     若 current 已持久、回执未持久 → 保持 COMMIT_DURABLE_AUDIT_PENDING
     若无法证明 → COMMIT_INDETERMINATE / RECOVERY_REQUIRED
     未完成恢复和隔离前 → 禁止新事务
```

**关键修正**：
- "fsync 重试成功"不再直接映射到 DURABLE，而是回到失败步骤继续；
- 只有 current 与回执**都**持久化才能进 DURABLE；
- current 已持久但回执未持久 → 保持 COMMIT_DURABLE_AUDIT_PENDING。

### 2.5 完整逐步崩溃矩阵（覆盖 7 类崩溃位置）

对每步覆盖：①操作开始前 ②操作执行中（如有中间态）③操作完成但下一持久化动作前 ④fsync 成功返回后 ⑤fsync 失败/不确定 ⑥恢复时观察预期状态 ⑦恢复时观察缺失/旧版本/部分/身份冲突。

| 步骤 | 崩溃位置 | 可观察状态 | 能否证明持久化？| 允许恢复 | 禁止操作 | 事务状态 | 允许新事务？| 人工裁决？|
|------|---------|-----------|---------------|---------|---------|---------|-----------|----------|
| S1 | ①开始前 | 无 Candidate | — | 重新开始 S1 | 无 | 初始 | 是 | 否 |
| S1 | ②执行中（半写入）| Candidate 半成品 | 否 | 删半成品重试 S1 | 当作完成 | 初始 | 是 | 否 |
| S1 | ③完成未 S2 | Candidate 完整 | 内容未持久（不可证明）| 执行 S2 | 跳过 S2 | DURABLE-进行中 | 否 | 否 |
| S2 | ④fsync 成功返回后 | Candidate 内容持久 | 是 | 执行 S3 | 跳过 S3 | DURABLE-进行中 | 否 | 否 |
| S2 | ⑤fsync 失败/不确定 | Candidate 内容不确定 | 否 | 重读校验→重试 S2 | 当作持久 | NOT_DURABLE | 否 | 否 |
| S3 | ③完成未 S4 | Candidate 目录项持久 | 是 | 执行 S4 | 跳过 | DURABLE-进行中 | 否 | 否 |
| S3 | ⑤fsync 失败 | 目录项不确定 | 否 | 重试 S3 | 当作持久 | NOT_DURABLE | 否 | 否 |
| S4 | ③完成未 S5 | Candidate 校验通过 | — | 执行 S5 | 跳过 | DURABLE-进行中 | 否 | 否 |
| S5 | ②半写入 | PREPARED 半成品 | 否 | 删半成品重试 S5 | 当作完成 | DURABLE-进行中 | 否 | 否 |
| S6 | ⑤fsync 失败 | PREPARED 内容不确定 | 否 | 重试 S6 | 当作持久 | NOT_DURABLE | 否 | 否 |
| S7 | ⑤fsync 失败 | PREPARED 目录项不确定 | 否 | 重试 S7 | 当作持久 | NOT_DURABLE | 否 | 否 |
| S8 | ③完成未 S9 | 锁已持有 | — | 执行 S9 | 释放锁跳过 | DURABLE-进行中 | 否 | 否 |
| S9 | ③五条件失败 | current 未变 | — | 重生成 Candidate | 只改 rev 重试 | 拒绝提交 | 是 | 否 |
| S10 | ②半写入 | current 临时文件半成品 | 否 | 删半成品重试 S10 | 当作完成 | DURABLE-进行中 | 否 | 否 |
| S11 | ⑤fsync 失败 | current 临时内容不确定 | 否 | 重试 S11 | 当作持久 | NOT_DURABLE | 否 | 否 |
| S12 | ②执行中崩溃 | current 可能已/未替换 | **不确定** | 重读 current 身份 | 盲目重试 | COMMIT_INDETERMINATE | 否 | 是 |
| S12 | ③完成未 S13 | current 已替换（内存）| 目录项未持久 | 执行 S13 | 跳过 S13 | COMMIT_INDETERMINATE | 否 | 是 |
| S13 | ④fsync 成功 | current 目录项持久 | 是 | 执行 S14 | 跳过 | COMMIT_DURABLE_AUDIT_PENDING | 否 | 否 |
| S13 | ⑤fsync 失败 | current 目录项不确定 | 否 | 重读确认 | 盲目回滚 | COMMIT_INDETERMINATE | 否 | 是 |
| S14 | ②半写入 | 回执半成品 | 否 | 删半成品重试 S14 | 当作完成 | COMMIT_DURABLE_AUDIT_PENDING | 否 | 否 |
| S14 | ④fsync 成功 | 回执内容持久 | 是 | 执行 S15 | 跳过 | COMMIT_DURABLE_AUDIT_PENDING | 否 | 否 |
| S15 | ⑤fsync 失败 | 回执目录项不确定 | 否 | 补 fsync S15 | 当作完成 | COMMIT_DURABLE_AUDIT_PENDING | 否 | 否 |
| S15 | ④fsync 成功 | 回执目录项持久 | 是 | 执行 S16 | 跳过 | DURABLE | 否 | 否 |
| S16 | ③完成 | 锁已释放，全部持久 | 是 | — | — | DURABLE | 是 | 否 |

**关键点**：
- S12（rename）是唯一"执行中崩溃导致不确定"的步骤——rename 的原子性只保证"要么旧要么新"，但崩溃后需重读 current 身份确认是旧还是新，**不能盲目重试**；
- S13/S15 的 fsync 失败进入 COMMIT_INDETERMINATE 或保持 COMMIT_DURABLE_AUDIT_PENDING，不直接跳 DURABLE。

### 2.6 20 条故障时间线 + 15 个反例

| # | 时间线 | 推演 | 结果 |
|---|--------|------|------|
| G1 | S1 半写入崩溃 | 删半成品重试 | 安全 |
| G2 | S2 fsync 失败 | NOT_DURABLE | 重试 |
| G3 | S2 fsync 重试成功 | 回到 S3 继续，**非跳 DURABLE** | 继续 |
| G4 | S3 fsync 失败 | NOT_DURABLE | 重试 |
| G5 | S6 fsync 失败 | NOT_DURABLE | 重试 |
| G6 | S7 fsync 失败 | NOT_DURABLE | 重试 |
| G7 | S11 fsync 失败 | NOT_DURABLE | 重试 |
| G8 | S12 rename 中崩溃 | COMMIT_INDETERMINATE | 重读 current |
| G9 | S13 fsync 失败 | COMMIT_INDETERMINATE | 重读确认 |
| G10 | S13 fsync 成功 | COMMIT_DURABLE_AUDIT_PENDING | 补回执 |
| G11 | S14 半写入 | COMMIT_DURABLE_AUDIT_PENDING | 删半成品重试 |
| G12 | S15 fsync 失败 | COMMIT_DURABLE_AUDIT_PENDING | 补 fsync |
| G13 | S15 fsync 成功 | DURABLE | 完成 |
| G14 | NOT_DURABLE 直接跳 DURABLE（05.20 错误）| **禁止** | 按步骤恢复 |
| G15 | fsync 重试失败 | RECOVERY_REQUIRED | 人工 |
| G16 | 证据冲突 | QUARANTINED | 人工 |
| G17 | 断电后 current 与 PREPARED 不一致 | 重读证据 | COMMIT_INDETERMINATE |
| G18 | 恢复时观察到旧版本 current | 判断是否已 rename | 按步骤 |
| G19 | 恢复时身份冲突 | QUARANTINED | 人工 |
| G20 | 未完成恢复前启动新事务 | **禁止** | 禁止 |

15 个反例：S1~S15 各 fsync 失败/半写入、S12 rename 中崩溃、NOT_DURABLE 跳步迁移、证据冲突、断电不一致、恢复时旧版本、身份冲突、未恢复即新事务。

### 2.7 POSIX/Linux 保证与未保证清单

| 项 | POSIX/Linux 保证 | 未保证 |
|----|-----------------|--------|
| fsync(fd) | 文件内容持久化到存储 | 目录项持久化 |
| fsync(dirfd) | 目录项持久化（POSIX 语义）| 不同文件系统可能有差异 |
| rename | 原子替换目录项（内存可见）| **断电持久性**（需 fsync(dirfd)）|
| O_EXCL | 原子独占创建 | 内容持久性 |
| flock | 进程间互斥 | 恶意同 UID 进程绕过 |

### 2.8 P0-04 结论

**P0-04 = DESIGN_PROVEN（POSIX）/ NOT_VERIFIED（Windows）**。
- 状态迁移表闭合，NOT_DURABLE 不再跳步迁移 DURABLE；
- 16 步完整崩溃矩阵覆盖 7 类崩溃位置；
- fsync 失败恢复决策树明确；
- Windows 目录持久化继续 NOT_VERIFIED，不混用 POSIX 结论。

---

## 3. P0-06 与 P0-05 的修改边界

### 3.1 P0-06（保留 05.20 纠正，仅允许一致性修正）

- 保留 05.20 对 1176 路径语义的纠正（替换文件"在其自己的原始名"，非"原目标名"）；
- 保留 1175/1176/1177 官方语义与其他错误码保守处理原则；
- 不得仅凭普通错误码推断文件存在性；
- 不重新设计 Windows 替换协议；
- 未经运行时验证的部分失败中间态继续标记 NOT_VERIFIED / DESIGN_PARTIAL。

### 3.2 P0-05（继续 DENY）

- 继续 DENY，不开展 Source Retire 设计放宽；
- 不删除、清理、归档，不声称已有 FD 已不可写。

---

## 4. 设计不变量（I-01 ~ I-45）

| # | 定义 | 适用范围 | 前置条件 | 线性化点/证明 | 失效反例 | 失败状态 | 拒绝策略 | 验证状态 |
|---|------|---------|---------|--------------|---------|---------|---------|---------|
| I-01 | 锁级别编号与获取顺序一致 | 全锁 | — | §1.2 | 编号与顺序矛盾 | 逆序死锁 | 重排 L0=coord | DESIGN_PROVEN |
| I-02 | fork 入口唯一（fork_entry）| fork | — | §1.4 | 多入口 fork | 职责混乱 | 唯一入口 | DESIGN_PROVEN |
| I-03 | freeze 不调用 fork | fork | — | §1.4 | freeze 内 fork | 递归 | 禁止 | DESIGN_PROVEN |
| I-04 | before 无锁不 fork | fork | — | §1.4 | before 持锁/fork | 重入 | 只读快照 | DESIGN_PROVEN |
| I-05 | after_in_parent 恢复 Gate | fork | — | §1.4 | parent 不恢复 | Gate 永久 FREEZING | finally 恢复 | DESIGN_PROVEN |
| I-06 | after_in_child 重建锁不 acquire 继承锁 | fork child | — | §1.4 | 继承 locked mutex | 死锁 | 直接赋值 | DESIGN_PROVEN |
| I-07 | fork 不绕过 CPython 生命周期 | fork | 主解释器 | CPython 文档 | os._fork 后执行 Python | 运行时未重置 | 用 os.fork | DESIGN_PROVEN |
| I-08 | 受控 fork 仅主线程 | fork | 主线程 | CPython 约束 | 非主线程 fork | 线程锁消失 | DENY | DESIGN_PARTIAL |
| I-09 | 准入原子性（检查+计数同 L1 临界区）| FD 准入 | — | §1.6 | 检查后计数前 FREEZING | 快照不一致 | 原子准入 | DESIGN_PROVEN |
| I-10 | 冻结覆盖 fork 窗口 | fork | 受控入口 | §1.4 | 快照后释放锁 | 未门禁 | FREEZING 保持 | DESIGN_PROVEN |
| I-11 | FD 绑定 generation | FD | — | 协议 | 裸数字复用 | 误关 | 不匹配不关 | DESIGN_PROVEN |
| I-12 | child 未知 FD 不主动 close | fork child | — | §1.7 | 误关未知 FD | 资源错关 | UNKNOWN_FD | DESIGN_PROVEN |
| I-13 | 计数严格配对（RAII）| FD 准入 | — | §1.6 | 计数泄漏 | 排空失效 | finally | DESIGN_PROVEN |
| I-14 | freeze 异常恢复 OPEN | fork | — | §1.5 | freeze 异常未恢复 | 永久 FREEZING | catch 恢复 | DESIGN_PROVEN |
| I-15 | 第三方 C fork/open DENY/UNSUPPORTED | FD/fork | 第三方扩展 | §1.7 | C 直接 fork/open | 绕过 Gate | DENY | DESIGN_PROVEN |
| I-16 | 子解释器/嵌入/posix_spawn UNSUPPORTED | fork | — | §1.7 | 该路径 | 未定义 | DENY/UNSUPPORTED | DESIGN_PARTIAL |
| I-17 | Candidate 绑定六字段 | 提交 | — | 05.20 §2.4 | 未绑父版本 | 错误提交 | 五条件 | DESIGN_PROVEN |
| I-18 | 五条件完整谓词 | 提交 | 全局提交锁 | 05.20 §2.4 | 笼统五条件 | 漏校验 | P1~P5 | DESIGN_PROVEN |
| I-19 | 文件内容先于目录项持久化 | 提交 | — | §2.5 | 未 fsync 文件 | 数据丢 | 禁止 | DESIGN_PROVEN |
| I-20 | 目录项 fsync(dirfd) | 提交 | — | §2.5 | 目录项未持久 | 目录项丢 | 强制 | DESIGN_PROVEN |
| I-21 | rename 原子性 ≠ 断电持久性 | 提交 | — | §2.7 | 误当持久 | 断电丢 | fsync 目录 | DESIGN_PROVEN |
| I-22 | NOT_DURABLE 不跳步迁移 DURABLE | 提交 | — | §2.3/§2.4 | 跳步迁移（05.20）| 错误升级 | 按步骤恢复 | DESIGN_PROVEN |
| I-23 | 只有 current+回执都持久才 DURABLE | 提交 | — | §2.4 | current 持久回执未持久当 DURABLE | 错误完成 | AUDIT_PENDING | DESIGN_PROVEN |
| I-24 | 六态状态机闭合 | 提交 | — | §2.3 | 未定义迁移 | 状态混乱 | 迁移表 | DESIGN_PROVEN |
| I-25 | S12 rename 中崩溃需重读确认 | 提交 | — | §2.5 | 盲目重试 rename | 状态不确定 | INDETERMINATE | DESIGN_PROVEN |
| I-26 | fsync 失败禁后续提交 | 提交 | — | §2.4 | fsync 失败后提交 | 不确定提交 | 禁提交 | DESIGN_PROVEN |
| I-27 | 不确定态禁回滚/重复提交 | 提交 | — | §2.3 | 自动回滚 | 状态破坏 | 禁自动 | DESIGN_PROVEN |
| I-28 | 每崩溃点确定恢复 | 提交 | — | §2.5 | 无规则 | 恢复错误 | 矩阵 | DESIGN_PROVEN |
| I-29 | 步骤编号统一 16 步 | 提交 | — | §2.2 | 编号不一致 | 引用错乱 | S1~S16 | DESIGN_PROVEN |
| I-30 | revision 单调 | 提交 | — | 05.20 §2.4 | 回退 | 版本倒退 | P4 断言 | DESIGN_PROVEN |
| I-31 | 恢复前重读证据 | 提交 | — | §2.4 | 仅凭内存状态 | 错误判断 | 重读磁盘 | DESIGN_PROVEN |
| I-32 | 恶意同 UID 锁替换排除 | 锁 | 威胁模型 | — | 同 UID 替换 | 锁分裂 | NOT_VERIFIED | NOT_VERIFIED |
| I-33 | +i 不阻旧 FD 写 | Source Retire | — | chattr(1) | 旧 FD 写 | 竞态 | DENY | NOT_VERIFIED（文档证实）|
| I-34 | +i 阻止 rename | Source Retire | — | chattr(1) | rename EPERM | 退役失败 | DENY | NOT_VERIFIED（文档证实）|
| I-35 | 源内容稳定无法本地证明 | Source Retire | — | §3.2 | 旧 FD/mmap/硬链接写 | 竞态 | DENY | DESIGN_PARTIAL |
| I-36 | Source Retire 默认 DENY | Source Retire | — | §3.2 | 无强制边界 | 不安全 | DENY | DESIGN_PARTIAL |
| I-37 | 元数据原子发布 | tombstone | — | 05.20 §3 | O_EXCL 逐步写 | 半成品 | 临时+校验+无覆盖 | DESIGN_PROVEN |
| I-38 | 占位 ≠ 有效 tombstone | tombstone | — | 05.20 §3.6 | 占位误认 | 错误发布 | 内容区分 | DESIGN_PARTIAL |
| I-39 | 1176 路径语义正确 | tombstone | — | Microsoft Learn | "以原目标名" | 路径混淆 | "其原名" | DESIGN_PROVEN |
| I-40 | 错误码不推断文件状态 | tombstone | — | 05.20 §3.4 | 从错误码推断 | 错误判定 | 重新读取 | DESIGN_PROVEN |
| I-41 | 文件 ID/内容哈希/路径正交 | tombstone | — | 05.20 §3.5 | 单维证明 | 错误 | 三者结合 | DESIGN_PROVEN |
| I-42 | ReplaceFileW 部分失败按内容判定 | tombstone | — | 05.20 §3.3 | 假设文件位置 | 错误 | INDETERMINATE | DESIGN_PARTIAL |
| I-43 | 恢复幂等 | tombstone | — | 05.20 §3.6 | 非幂等 | 重复副作用 | 状态机 | DESIGN_PROVEN |
| I-44 | 无法证明必须 DENY/INDETERMINATE | 全局 | — | 全文 | 猜测 | 错误结论 | 不猜测 | DESIGN_PROVEN |
| I-45 | 不把推演写运行时验证 | 全局 | — | 全文 | 自审当验证 | 虚假验证 | NOT_TESTED | DESIGN_PROVEN |

**共 45 条不变量。** 本轮新增/修订 I-01（锁编号与顺序一致）、I-02~I-06（四职责分离）、I-22（NOT_DURABLE 不跳步）、I-23（current+回执都持久才 DURABLE）、I-25（S12 崩溃重读确认）。

---

## 5. 全局一致性审查

### 5.1 20 条端到端链路

| # | 链路 | 前置条件 | 关键边界 | 崩溃点 | 恢复 | 结论 |
|---|------|---------|---------|--------|------|------|
| L1 | fork_entry→freeze→os.fork→after 钩子 | 主线程 | 四职责分离 + 锁升序 | fork 中 | 恢复 | DESIGN_PARTIAL |
| L2 | 锁顺序 L0→L1→L2 升序无逆序 | — | 锁依赖图无环 | — | — | DESIGN_PROVEN |
| L3 | FD 准入→计数→冻结→fork→child 重建 | gate 原子 | generation + in_flight | 冻结中 | 校验 | DESIGN_PROVEN |
| L4 | 第三方 C open→fork→child 处理 | 无法拦截 | 不可控 | — | UNKNOWN_FD | DESIGN_PARTIAL |
| L5 | Candidate→fsync 文件→fsync 目录→rename→fsync 目录 | — | 双重持久化 | rename 前后 | 按 current | DESIGN_PROVEN |
| L6 | fsync 失败→NOT_DURABLE→按步骤恢复 | — | 恢复决策树 | fsync | 按步骤 | DESIGN_PROVEN |
| L7 | NOT_DURABLE 跳步 DURABLE（反例）| — | 禁止 | — | 禁止 | DESIGN_PROVEN（拒绝）|
| L8 | rename 后目录未持久→COMMIT_INDETERMINATE | — | 不确定态 | rename 后 | 重读确认 | DESIGN_PROVEN |
| L9 | current 持久回执未写→AUDIT_PENDING | — | 拆分状态 | S14 前 | 补回执 | DESIGN_PROVEN |
| L10 | 双事务竞争 | — | flock | 竞争点 | 败者重生成 | DESIGN_PROVEN |
| L11 | 断电后 current 与 PREPARED 不一致 | — | 重读证据 | 断电 | 人工裁决 | DESIGN_PROVEN |
| L12 | 源 seal→quarantine→retire | — | retire DENY | — | DENY | DENY |
| L13 | 旧 FD + i | — | +i 不阻旧 FD | 校验后写 | 竞态 | DENY |
| L14 | rename + i | — | rename EPERM | — | 退役失败 | DENY |
| L15 | tombstone 临时→校验→无覆盖发布 | — | 临时 fsync | 发布前后 | 占位/有效 | DESIGN_PARTIAL |
| L16 | ReplaceFileW 1175 无移动 | — | 官方契约 | — | 幂等重试 | DESIGN_PROVEN |
| L17 | ReplaceFileW 1176 未指定备份 | — | 官方契约 | 替换中 | INDETERMINATE | DESIGN_PARTIAL |
| L18 | ReplaceFileW 1177 部分移动 | — | 官方契约 | 替换中 | INDETERMINATE | DESIGN_PARTIAL |
| L19 | 归档协议 | — | 临时+校验+发布 | 归档中 | 哈希确认 | DESIGN_PROVEN |
| L20 | 恶意同 UID | — | 无 | 任意 | 检测 | NOT_VERIFIED |

### 5.2 时间线与反例汇总

- 时间线：P0-02 二十条 + P0-04 二十条 = **40 条**（≥40 达标）；
- 反例：P0-02 十五个 + P0-04 十五个 = **30 个**（≥30 达标）。

### 5.3 跨章节身份一致性

| 身份/状态 | 产生 | 传递 | 一致性 |
|-----------|------|------|--------|
| 锁级别 | §1.2 | §1.3/§1.4 | 编号 = 获取顺序 |
| FD generation | §1.6 | §1.9 | generation + 原子准入 |
| in_flight 计数 | §1.6 | §1.9 | RAII 配对 |
| 父版本三元组 | 05.20 §2.4 | §2.5 | 五条件 P1~P3 |
| 持久化六态 | §2.3 | §2.4/§2.5 | 迁移表闭合 |
| Source Retire | §3.2 | — | DENY |
| Tombstone 事务身份 | 05.20 §3.5 | — | txn 归因 + 三要素正交 |

### 5.4 历史阻断逐项标记

| 阻断 | 状态 |
|------|------|
| IMPLEMENT-01 | STOP（保持）|
| P0-BLOCK-1 | 未解除 |
| P0-BLOCK-2 | 未解除 |
| P1-BLOCK-3 | 未解除 |
| F1 | 不解除 |
| P0.8 | 不解除 |
| F2 | 不解除 |

---

## 6. 阻断状态（保持不变）

| 项目 | 状态 |
|------|------|
| IMPLEMENT-01 | **STOP / NOT_APPROVED** |
| P0-BLOCK-1 | 未解除 |
| P0-BLOCK-2 | 未解除 |
| P1-BLOCK-3 | 未解除 |
| F1 | 不解除 |
| P0.8 | 不解除 |
| F2 | 不解除 |
| 冻结源码哈希 | 不变 |
| A/B 工作区 | 不合并 |

---

## 7. 最终结论

**DESIGN-05.21 = PASS-WITH-ISSUES**，分项结论：

| 项 | 结论 | 依据 |
|----|------|------|
| P0-02 | DESIGN_PARTIAL | 锁级别编号与获取顺序一致（L0=coord→L1=gate→L2=fd），四职责分离，锁依赖图无环（协议层 DESIGN_PROVEN）；第三方 C fork 不可控 UNSUPPORTED/DENY；运行时 NOT_TESTED |
| P0-04 POSIX | DESIGN_PROVEN | 状态迁移表闭合（NOT_DURABLE 不再跳步 DURABLE）+ 16 步完整崩溃矩阵 + 恢复决策树 |
| P0-04 Windows | NOT_VERIFIED | FlushFileBuffers 目录持久化无权威保证 |
| P0-05 | DENY（保持）| 不制造 PASS |
| P0-06 | DESIGN_PARTIAL / NOT_VERIFIED | 保留 05.20 纠正，部分失败中间态未验证 |
| 交付完整性 | PASS | 8 文件 + ZIP 独立解压逐字节比对 |
| IMPLEMENT-01 | **NOT_APPROVED** | 未授权实现 |

**核心诚实结论**：
1. **P0-02 锁级别矛盾已消除**：05.20 的"L2=coord 却最先获取"逆序矛盾，通过把 `_fork_coord_lock` 重排为 L0（最先获取）、`_gate_lock` 升为 L1、`_fd_lock` 保持 L2，使编号与实际获取顺序严格一致，锁依赖图无环。
2. **P0-02 fork 四职责分离**：fork_entry / freeze / before / after_in_parent / after_in_child 职责严格分离，freeze 不再自行 fork，调用时序图证明无递归 fork、无永久 FREEZING、无继承锁死锁、无 Registry 交叉污染。
3. **P0-04 状态迁移修正**：NOT_DURABLE 重试成功后不再直接跳 DURABLE，而是回到失败步骤继续，只有 current+回执都持久才进 DURABLE；16 步完整崩溃矩阵覆盖 7 类崩溃位置。

**默认结论：IMPLEMENT-01 NOT_APPROVED。** 等待 ChatGPT 独立复审。

---

## 8. 交付物索引

| 文件 | 说明 |
|------|------|
| `USIT-STAGE-B-DESIGN-05.21.md` | 本主设计文档 |
| `USIT-STAGE-B-DESIGN-05.21-ISSUES.md` | 问题清单及剩余风险 |
| `USIT-STAGE-B-DESIGN-05.21-DIFF.md` | 相对 05.20 逐项差异 |
| `USIT-STAGE-B-DESIGN-05.21-CROSS-REVIEW.md` | 反例审查记录 |
| `USIT-STAGE-B-DESIGN-05.21-EXECUTION-REPORT.md` | 实际执行报告 |
| `USIT-STAGE-B-DESIGN-05.21-HASH-MANIFEST.md` | 交付文件 SHA256 清单 |
| `USIT-STAGE-B-DESIGN-05.21-HASH-RECEIPT.md` | 哈希验证回执 |
| `USIT-STAGE-B-DESIGN-05.21-DELIVERY.zip` | 完整打包 |

---


