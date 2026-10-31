# USIT-STAGE-B-DESIGN-05.20｜Fork准入原子性补证＋POSIX持久化状态机修订＋Windows ReplaceFileW错误矩阵纠正

> 阶段 B · DESIGN-05.20 · READ-ONLY DESIGN REVISION ONLY
> 作者：USIT 主开发 · 生成：2026-10-31（GMT+8）
> 前置审查：DESIGN-05.19 独立复审结论 FAIL
> 工作区：`C:\Users\Administrator\WorkBuddy\2026-10-15-22-09-47\usit_p0`
> 交付目录：`C:\Users\Administrator\WorkBuddy\2026-10-15-22-09-47\usit_p0_05.20_reports`（**项目外**）
> 状态：**STOP — 等待独立复审**（未授权 IMPLEMENT-01）

---

## 0. 缺陷追踪表（05.19 原问题 → 05.20 修订位置 → 新证据 → 是否闭合 → 剩余风险 → 结论）

| # | 05.19 原问题 | 05.20 修订位置 | 新证据 | 是否闭合 | 剩余风险 | 结论 |
|---|-------------|---------------|--------|---------|---------|------|
| P0-02 | 准入"先检查 Gate 状态、稍后增加 in_flight"存在分离步骤竞态窗口 | §1 | 原子准入协议（Gate 状态检查与 in_flight 登记合并为单一原子步骤）| 协议层闭合 | 第三方 C 扩展直接 FD 操作不可控 | DESIGN_PARTIAL |
| P0-04 | 05.19 状态机将"current 已持久化但回执未持久"粗糙并入 COMMIT_INDETERMINATE | §2 | 六态状态机，新增 COMMIT_DURABLE_AUDIT_PENDING；统一步骤编号 | 协议层闭合 | Windows 目录持久化 NOT_VERIFIED | DESIGN_PROVEN（POSIX）|
| P0-05 | 保持 DENY | §4 | 无需新修复，复核边界表述 | 保持 DENY | 无本地强制不可变原语 | DENY |
| P0-06 | 05.19 §3.3 把 1176（未指定备份）的"替换文件在其原始名"误写为"以原目标名存在" | §3 | Microsoft Learn 官方原文 "the replacement file exists under its original name" | 闭合 | ReplaceFileW 部分失败中间态 NOT_VERIFIED | DESIGN_PARTIAL |

**本节是全文总纲。本轮核心任务：P0-02 补证准入原子性；P0-04 状态机细化出 COMMIT_DURABLE_AUDIT_PENDING 并统一步骤编号；P0-06 纠正 1176 路径语义错误；P0-05 保持 DENY。**

---

## 1. P0-02｜Fork Gate 准入与在途计数原子性证明

### 1.1 缺陷定位：05.19 的"检查 Gate 后再增加计数"存在分离步骤竞态

05.19 §1.3 的 FREEZING 态方案虽然覆盖了 fork 窗口，但**准入路径本身**存在一个未证明的窗口。05.19 隐含的准入流程是：

```text
线程 A：检查 Gate == OPEN          # 步骤 1（读）
线程 A：暂停（未增加 in_flight）   # 竞态窗口！
线程 B：Gate = FREEZING            # 步骤 2（写）
线程 B：观察 in_flight == 0        # 步骤 3
线程 B：冻结 Registry + 快照 + fork # 步骤 4
线程 A：恢复，增加 in_flight        # 步骤 5（太晚了）
线程 A：执行 open/close            # 步骤 6（绕过冻结）
```

**问题**：线程 A 的"检查 Gate 状态"（步骤 1）与"增加 in_flight 计数"（步骤 5）是**两个分离的操作**，之间没有原子性。线程 B 的 FREEZING 可能在两者之间介入，导致线程 A 在 FREEZING 之后才登记 in_flight，而线程 B 已经观察到 `in_flight == 0` 并完成快照与 fork。结果是**快照与实际继承 FD 表不一致**。

**结论**：05.19 的准入协议**未证明原子性**，本轮必须补证。

### 1.2 原子准入协议（核心修订）

**关键设计原则**：Gate 状态检查与 in_flight 计数登记必须**合并为一个原子操作**（等价线性化），不得采用"先检查、后计数"的分离步骤。

采用**单一互斥量 + 单一原子状态字**的方案：

```text
共享状态（由 _gate_lock 保护，或使用单个原子变量）：
  gate_state : { OPEN, FREEZING }    # 用 _gate_lock 保护，或用 std::atomic 单字
  in_flight  : int                    # 在途 FD 操作计数，由 _gate_lock 保护

【原子准入协议 —— 伪代码】
fd_op_enter():
    acquire(_gate_lock)               # 单一锁，覆盖状态检查 + 计数登记
    if gate_state != OPEN:
        release(_gate_lock)
        # FREEZING 中：本次 FD 操作必须等待，或直接拒绝
        wait_until_open()             # 阻塞直到 Gate 回到 OPEN（条件变量）
        acquire(_gate_lock)
        # 重新检查（可能再次 FREEZING）
        if gate_state != OPEN:
            release(_gate_lock); return DENY   # 或重试
    in_flight += 1                    # 【原子】状态检查与计数登记在同一临界区
    release(_gate_lock)
    # —— 至此，本操作已登记在途，FREEZING 不能再观察到 in_flight==0 而误判 ——
    try:
        do_fd_operation()             # 实际 syscall（open/close/dup/...）
        update_registry()             # Registry 登记（在 _fd_lock 保护下）
    finally:
        acquire(_gate_lock)
        in_flight -= 1
        if in_flight == 0:
            signal(freezing_done_cv)  # 唤醒等待排空的 FREEZING
        release(_gate_lock)

【FREEZING 协议 —— 伪代码】
freeze():
    acquire(_gate_lock)
    gate_state = FREEZING             # 原子写状态
    # 等待在途操作排空
    while in_flight > 0:
        wait(freezing_done_cv, _gate_lock)   # 条件变量等待，释放锁
    # —— 此处 in_flight == 0，且 gate_state == FREEZING，任何新准入都会被拒绝 ——
    release(_gate_lock)
    # 冻结 Registry + 生成快照（在 _fd_lock 下，此时无在途 FD 操作）
    acquire(_fd_lock)
    snapshot = freeze_registry()
    release(_fd_lock)
    os.fork()                          # FREEZING 态保持覆盖 fork 窗口
    # parent 恢复
    acquire(_gate_lock)
    gate_state = OPEN
    signal_all(gate_open_cv)
    release(_gate_lock)
```

**线性化点（linearization point）定义**：
- **准入操作**的线性化点是 `in_flight += 1` 这条原子指令（在 `_gate_lock` 临界区内）；
- **FREEZING 操作**的线性化点是 `gate_state = FREEZING` 这条原子指令；
- 两者都通过 `_gate_lock` 串行化，因此**线性化点之间存在全序关系**：
  - 若 `in_flight += 1` 先于 `gate_state = FREEZING` → 该操作已登记，FREEZING 会等待它完成；
  - 若 `gate_state = FREEZING` 先于 `in_flight += 1` → 该操作被拒绝或阻塞，不会在冻结后执行。

**结论**：状态检查与计数登记在同一临界区，不存在"检查后暂停、计数前 FREEZING 介入"的竞态窗口。此协议可证明线性一致（linearizable）。

### 1.3 锁、原子变量、条件变量职责与内存可见性

| 原语 | 职责 | 内存可见性 |
|------|------|-----------|
| `_gate_lock`（互斥量）| 串行化准入登记、FREEZING 状态切换、in_flight 增减 | 互斥量提供 acquire/release 语义，临界区内的写对后续读可见 |
| `in_flight`（int）| 在途 FD 操作计数 | 由 `_gate_lock` 保护，无裸读写 |
| `freezing_done_cv`（条件变量）| FREEZING 等待 in_flight 归零 | 条件变量 wait 原子释放锁并等待 signal |
| `gate_open_cv`（条件变量）| 被拒准入等待 Gate 回到 OPEN | 同上 |
| `_fd_lock` | 保护 Registry 的登记/冻结 | 与 `_gate_lock` 分层，顺序固定 |

**锁顺序（严格升序，禁止逆序）**：
```
L0. _gate_lock        （准入/冻结/计数）
L1. _fd_lock          （Registry）
L2. _fork_coord_lock  （fork 协调，仅 fork 入口使用）
L3. _lock / _lease_lock / _pmutex_lock  （业务锁，禁止持锁 fork）
```

规则：
1. 准入只获取 `_gate_lock`（短暂），FD 操作期间可获取 `_fd_lock`，顺序恒为 gate → fd；
2. fork 入口获取 `_fork_coord_lock` → `_gate_lock`（调用 freeze）→ `_fd_lock`，顺序恒为 coord → gate → fd；
3. **禁止持锁 fork**（持有 L3 业务锁的线程不得 fork）；
4. before/parent/child 钩子**不获取任何锁**。

**无死锁证明**：所有锁获取严格遵循 coord → gate → fd 的升序，无逆序获取，无跨锁循环等待；条件变量等待均原子释放所持锁，不形成持锁等待环。

**异常退出与取消处理**：
- 准入在 `in_flight += 1` 后若 `do_fd_operation()` 抛异常，`finally` 块保证 `in_flight -= 1` 必定执行（RAII 语义），不泄漏计数；
- 线程取消（thread cancellation）若在 `finally` 前触发，需用禁用取消的临界区（POSIX `pthread_setcancelstate`）包裹计数增减，或采用 RAII guard 对象，确保计数严格配对；
- fork 前异常退出：freeze() 若在 `gate_state = FREEZING` 后异常，必须在 catch 中恢复 `gate_state = OPEN` 并 signal_all，否则 Gate 永久 FREEZING（死锁）。

### 1.4 FD 操作生命周期图

```
fd_op_enter()（gate 临界区：检查状态 + in_flight++）
   │
   ▼
do_fd_operation()（syscall：open/close/dup/...）
   │
   ▼
update_registry()（_fd_lock 下登记/注销）
   │
   ▼
fd_op_exit()（gate 临界区：in_flight--，归零则 signal）
```

**关键**：`update_registry()` 与 `do_fd_operation()` 之间若存在窗口（syscall 成功但 Registry 尚未登记），该窗口内 FREEZING 会等待 `in_flight > 0` 排空（因为 `fd_op_enter` 已登记 in_flight，直到 `fd_op_exit` 才归零），因此**不会**在 syscall 成功但未登记时冻结 Registry。这是原子准入带来的额外保证。

### 1.5 Fork 入口矩阵（区分 Python API 保证 / CPython 实现 / POSIX 语义 / 第三方）

| 入口 | Python API 保证 | CPython 实现行为 | POSIX 语义 | 第三方扩展行为 | 本库 Gate 覆盖 | 结论 |
|------|----------------|-----------------|-----------|---------------|---------------|------|
| `os.fork()` | 触发 register_at_fork 钩子 | 调用 PyOS_BeforeFork/AfterFork_Parent/AfterFork_Child | fork() | — | 是（before 钩子无锁）| **受控 fork 采用** |
| `os._fork()` | 无钩子 | 跳过 PyOS_* 生命周期 | fork() | — | 是但不安全 | **禁用**（仅 fork+exec）|
| `os.register_at_fork()` | 用户钩子注册点 | 由 os.fork 触发 | — | — | 注册 before/parent/child | 使用 |
| `PyOS_BeforeFork/AfterFork_*` | C API | CPython 内部生命周期 | — | — | 由 os.fork 触发 | 依赖 |
| 第三方 C 扩展直接 `fork()` | 无 | 绕过 | fork() | 直接调 libc fork | **否** | **DENY / UNSUPPORTED** |
| 嵌入 Python 宿主 fork | 无 | 宿主控制 | fork() | 宿主进程 fork | **否** | **DENY / UNSUPPORTED** |
| 子解释器/多解释器 | 不确定 | 未定义 fork 行为 | fork() | — | **否** | **DENY / UNSUPPORTED** |

**无法确保 Gate 覆盖的入口必须拒绝运行，不宣称通用安全。**

### 1.6 FD 入口覆盖矩阵（逐项分类）

| 入口 | 分类 | 说明 |
|------|------|------|
| `open/openat/openat2` | 完全受控（经本库包装器）| 包装器内走 fd_op_enter → syscall → registry |
| `socket/socketpair` | 完全受控 | 同上 |
| `accept/accept4` | 完全受控 | 同上 |
| `pipe/pipe2` | 完全受控 | 同上 |
| `dup/dup2/dup3` | 完全受控 | 同上 |
| `fcntl(F_DUPFD/F_DUPFD_CLOEXEC)` | 完全受控 | 同上 |
| `close/close_range` | 完全受控 | 同上 |
| `detach`（socket 转移）| 完全受控 | 转移时注销 Registry |
| `fileno`（只读查询）| 完全受控（只读）| 不改变 FD 表 |
| 继承 FD（fork 继承）| 由 fork 门禁覆盖 | 见 §1.2 |
| 第三方库直接创建 FD | **无法保证受控** | Python 层无法拦截 C 扩展直接 open/socket |
| 第三方 C 扩展 | **无法保证受控** | 同上级 |
| 嵌入 Python | **UNSUPPORTED** | 宿主控制 FD 生命周期 |
| 子解释器 | **UNSUPPORTED** | fork 行为未定义 |

**核心结论**：本库自有 FD 入口**完全受控**（经统一包装器）；**第三方 C 扩展直接 FD 操作无法保证受控**，属 UNSUPPORTED/DENY 环境。**不得把 Python 层包装器覆盖等同于系统级 FD 操作覆盖**——系统级（第三方 C 扩展直接调 libc）的 FD 操作不在本库 Gate 之内。

### 1.7 20 条并发时间线

| # | 场景 | 初始状态 | 动作顺序 | 计数值 | Gate 状态 | 最终结果 | 拒绝策略 |
|---|------|---------|---------|--------|----------|---------|---------|
| T1 | 准入前暂停 | OPEN, in_flight=0 | A 暂停（未进入）→ B freeze→fork | 0 | FREEZING | A 恢复后被拒 | 阻塞/拒绝 |
| T2 | 检查 Gate 后暂停 | OPEN | A 检查 OPEN→暂停→B FREEZING→A 恢复 | 0 | FREEZING | **A 计数登记被 FREEZING 串行化，A 被拒** | 原子准入 |
| T3 | 计数增加后暂停 | OPEN | A in_flight++→暂停→B 观察 in_flight=1 等待 | 1 | FREEZING | B 等待 A 完成 | 排空 |
| T4 | FD syscall 执行期间暂停 | OPEN | A 执行 open 中→B freeze 等待 | 1 | FREEZING | B 等 A 完成 | 排空 |
| T5 | syscall 成功 Registry 未登记 | OPEN | A open 成功、未登记→B freeze 等待 | 1 | FREEZING | B 等 A 登记完 | in_flight 覆盖 |
| T6 | Registry 更新期间暂停 | OPEN | A 更新中→B freeze 等待 | 1 | FREEZING | B 等 A 完成 | 排空 |
| T7 | 计数减少前暂停 | OPEN | A 操作完未 in_flight--→B 等待 | 1 | FREEZING | B 等 A 归零 | 排空 |
| T8 | FREEZING 与准入同时竞争 | OPEN | A 与 B 同时进 gate 临界区 | — | — | 互斥量串行化，二者有序 | 线性化点 |
| T9 | FREEZING 后迟到的 FD 请求 | FREEZING | A 迟到请求 | 0 | FREEZING | A 被拒/阻塞 | 拒绝 |
| T10 | fork 等待归零期间新请求到达 | FREEZING | B 等待，A 新请求到达 | 0 | FREEZING | A 被拒 | 拒绝 |
| T11 | fork 前异常退出 | FREEZING | freeze 异常，未恢复 | 0 | FREEZING | **catch 恢复 OPEN** | 恢复+signal |
| T12 | fork 失败 | FREEZING | fork 返回 -1 | 0 | FREEZING | parent 恢复 OPEN | 幂等 |
| T13 | 父进程恢复 | FREEZING | parent 置 OPEN+signal_all | 0 | OPEN | 新请求放行 | 恢复 |
| T14 | 子进程 FD 重建失败 | child | child 重建抛异常 | — | child-local | _fork_reset_ok=False | DENY |
| T15 | 多个 fork 请求同时到达 | OPEN | A、B 同时 fork | — | — | _fork_coord_lock 串行化 | 串行 |
| T16 | 线程取消/信号打断 | OPEN | 准入后取消 | 1→0 | OPEN | finally 保证归零 | RAII |
| T17 | 计数泄漏（异常） | OPEN | do_fd_operation 抛异常 | 1→0 | OPEN | finally 归零 | 不泄漏 |
| T18 | 第三方 C open | 任意 | 绕过 gate 直接 open | — | — | 快照外 FD | child 标 UNKNOWN_FD |
| T19 | 第三方 C fork | 任意 | 绕过 gate 直接 fork | — | — | 状态未知 | DENY |
| T20 | 连续 freeze/恢复 | OPEN | 两次 fork 依次 | 0 | OPEN | 幂等 | — |

### 1.8 15 个反例及逐一处理

| # | 反例 | 处理 |
|---|------|------|
| R1 | 检查 Gate 后、计数前 FREEZING 介入 | 原子准入（同临界区），消除 |
| R2 | in_flight 计数泄漏 | finally/RAII 保证配对 |
| R3 | os._fork 后 child 执行 Python | 禁用 os._fork |
| R4 | before 钩子持锁重入 | before 无锁 |
| R5 | 非主线程 fork | DENY |
| R6 | 子解释器 fork | DENY/UNSUPPORTED |
| R7 | 第三方 C fork 绕过 | DENY |
| R8 | 第三方 C open 的 FD 被误关 | child 只关 generation 匹配的 FD |
| R9 | 快照后 FD 数字复用 | FREEZING 阻断 + generation 双防线 |
| R10 | 连续 fork 状态残留 | 幂等恢复 |
| R11 | 在途操作未排空 | FREEZING 条件变量等待 |
| R12 | 持锁 fork 死锁 | 禁止持锁 fork |
| R13 | freeze 异常未恢复 OPEN | catch 恢复 + signal_all |
| R14 | 线程取消打断计数 | 禁用取消临界区/RAII |
| R15 | 嵌入 Python 宿主 fork | DENY/UNSUPPORTED |

### 1.9 P0-02 验收与结论

- 原子准入伪代码：✅ §1.2
- 线性化点定义：✅ §1.2
- Gate 状态机 + FD 生命周期图：✅ §1.2/§1.4
- 锁顺序图 + 无死锁证明：✅ §1.3
- 20 条并发时间线：✅ §1.7
- 15 个反例：✅ §1.8
- FD 入口覆盖矩阵 + Fork 入口矩阵：✅ §1.6/§1.5
- 平台/API 保证矩阵：✅ §1.5

**P0-02 结论：DESIGN_PARTIAL**。
- **协议层（受控 FD 入口 + 受控 fork）**：原子准入（状态检查与计数登记同一临界区）+ FREEZING 态覆盖 fork 窗口，可证明线性一致，无误关、无死锁、快照一致 —— **DESIGN_PROVEN（协议层）**。
- **运行时**：CPython 多线程 fork 运行时行为依赖 CPython 实现，NOT_TESTED。
- **不可控入口**：第三方 C 扩展直接 FD 操作 / 直接 fork / 嵌入 Python / 子解释器，Python 层无法拦截 → **UNSUPPORTED/DENY**。

---

## 2. P0-04｜POSIX CAS 与持久化状态机修订

### 2.1 缺陷定位：05.19 状态机语义过于粗糙

05.19 §2.4 将"current 目录 fsync 已成功、但 COMMIT 回执尚未持久化"统一归入 COMMIT_INDETERMINATE，语义过粗。实际上 current 已经持久化时，事务的**数据层提交已完成**，仅**审计层（回执）未完成**——这不应与"无法确定 current 是否已持久化"的 COMMIT_INDETERMINATE 混为一谈。

### 2.2 六态持久化状态机（修订）

```
         ┌───────────────────────────────────────────────────────┐
         ▼                                                       │
   DURABLE ──current已持久+回执已持久──▶ （事务完全完成）        │
         │                                                       │
         ├─current已持久，回执未持久──▶ COMMIT_DURABLE_AUDIT_PENDING
         │                                                       │
         ├─提交前关键fsync失败──▶ NOT_DURABLE ──重试成功──▶ DURABLE
         │                                                       │
         ├─无法确定current是否持久化──▶ COMMIT_INDETERMINATE      │
         │                                                       │
         ├─恢复操作需要执行──▶ RECOVERY_REQUIRED                 │
         │                                                       │
         └─证据冲突/身份校验失败──▶ QUARANTINED                   │
```

### 2.3 六态语义表（进入条件 / 退出条件 / 允许 / 禁止 / 读 / 新事务 / 自动恢复 / 人工确认）

| 状态 | 进入条件 | 退出条件 | 允许操作 | 禁止操作 | 允许读？| 允许新事务？| 允许自动恢复？| 需人工确认？|
|------|---------|---------|---------|---------|--------|-----------|-------------|-----------|
| DURABLE | current 已持久化 + 回执已持久化 | — | 后续事务 | — | 是 | 是 | — | 否 |
| COMMIT_DURABLE_AUDIT_PENDING | current 已持久化，回执未持久 | 补写回执并 fsync 成功 | 补写回执、读 current | 覆盖 current、回滚 | 是 | **否**（先补回执）| 可自动补回执 | 否（自动补即可）|
| NOT_DURABLE | 提交前关键 fsync 失败 | 重试 fsync 成功 | 重试 fsync、读 | 提交、回滚 | 是 | 否 | 可重试 | 否（重试即可）|
| COMMIT_INDETERMINATE | 无法确定 current 是否已持久化 | 重新读取校验确认 | 读 + 校验身份/哈希 | 覆盖、回滚、提交 | 是 | 否 | **否** | **是** |
| RECOVERY_REQUIRED | 磁盘错误/重试失败 | 人工恢复完成 | 读 + 校验 | 提交、回滚 | 是 | 否 | 否 | 是 |
| QUARANTINED | 证据冲突/身份校验失败 | 人工裁决 | 只读 | 一切写操作 | 受限 | 否 | 否 | 是 |

**核心修订**：
- **COMMIT_DURABLE_AUDIT_PENDING** 从 COMMIT_INDETERMINATE 中拆出——current 已持久化（数据层提交完成），仅审计回执未完成，可自动补回执，**不需要**人工确认，也**不禁止**后续事务（补完回执即可）；
- **COMMIT_INDETERMINATE** 严格限定为"无法确定 current 是否已持久化"，此时**禁止**自动恢复、**禁止**新事务，需人工确认。

### 2.4 六字段 Candidate 与五校验谓词（逐项：来源/类型/比较对象/时机/锁/TOCTOU/拒绝策略）

| 字段 | 来源 | 类型 | 比较对象 | 比较时机 | 锁保护范围 | 失败拒绝策略 | 是否 TOCTOU | 是否重生成 |
|------|------|------|---------|---------|-----------|-------------|------------|-----------|
| base_revision | 生成 Candidate 时读 current.revision | int | current.revision | 提交锁内 | 全局提交锁 | 拒绝 + 重生成 | 否（锁内重读）| 是 |
| base_txn_id | 生成 Candidate 时读 current.txn_id | str | current.txn_id | 提交锁内 | 全局提交锁 | 拒绝 + 重生成 | 否 | 是 |
| base_hash | 生成 Candidate 时读 current.hash | str(sha256) | current.hash | 提交锁内 | 全局提交锁 | 拒绝 + 重生成 | 否 | 是 |
| new_revision | base_revision + 1 | int | base_revision + 1（断言）| 提交锁内 | 全局提交锁 | 拒绝 | 否 | 是 |
| new_txn_id | 本次事务唯一 ID | str | 唯一性（O_EXCL）| 生成时 | 目录项 O_EXCL | 拒绝（重复）| 否 | 是 |
| candidate_hash | Candidate 文件 SHA-256 | str | PREPARED 记录哈希 | 提交锁内 | 全局提交锁 | 拒绝 + 重生成 | 否 | 是 |

五校验谓词（**全部在全局提交锁内**）：

```
P1: current.revision == base_revision          # 父版本未变
P2: current.txn_id == base_txn_id              # 父事务 ID 未变
P3: current.hash == base_hash                  # 父内容哈希未变
P4: new_revision == base_revision + 1          # 版本单调 +1
P5: sha256(candidate_content) == candidate_hash # Candidate 未被篡改，与 PREPARED 一致
```

**TOCTOU 说明**：P1~P3 的"比较对象"（current）与"Candidate 生成时读取的值"之间，通过在**全局提交锁内重新读取 current** 消除 TOCTOU——即校验时读的 current 与锁保护的 current 是同一份，无检查-使用分离。P4 是纯断言（基于已读到的 base_revision）。P5 校验 Candidate 文件内容，若 Candidate 文件在提交锁外被篡改，则 P5 失败 → 拒绝 + 重生成。

**任一谓词失败 → 拒绝提交，必须基于最新 current 重新生成 Candidate，禁止只改 revision 数字后重试。**

### 2.5 完整持久化步骤表（统一步骤编号，消除"13 步 vs 16 步"不一致）

统一定义 **16 步**协议（编号唯一，后续矩阵均引用此编号）：

| 步骤 | 操作 | 目标文件/目录 | 文件内容持久？| 目录项持久？| current 状态 |
|------|------|--------------|-------------|------------|-------------|
| S1 | 写 Candidate 版本文件（O_EXCL）| versions/<txn_id>.<rev> | 否 | 否 | 旧 rev |
| S2 | fsync(candidate_fd) | Candidate 文件 | 是 | 否 | 旧 rev |
| S3 | fsync(versions_dirfd) | versions/ 目录 | 是 | 是 | 旧 rev |
| S4 | 重读 + SHA-256 校验 | Candidate | — | — | 旧 rev |
| S5 | 写 PREPARED | prepared/<txn_id> | 否 | 否 | 旧 rev |
| S6 | fsync(prepared_fd) | PREPARED 文件 | 是 | 否 | 旧 rev |
| S7 | fsync(prepared_dirfd) | prepared/ 目录 | 是 | 是 | 旧 rev |
| S8 | flock(commit_lock, LOCK_EX) | 锁文件 | — | — | 旧 rev |
| S9 | 读 current + 五条件校验（§2.4）| current | — | — | 旧 rev |
| S10 | 写 current.<txn_id> | current 临时文件 | 否 | 否 | 旧 rev |
| S11 | fsync(current_tmp_fd) | current 临时文件 | 是 | 否 | 旧 rev |
| S12 | rename(current.<txn_id> → current) | current | 是 | 否（内存）| **新 rev** |
| S13 | fsync(parent_dirfd) | current 父目录 | — | 是 | 新 rev |
| S14 | 写 COMMIT 回执 + fsync(commit_fd) | commit/<txn_id> | 是 | 否 | 新 rev |
| S15 | fsync(commit_dirfd) | commit/ 目录 | — | 是 | 新 rev |
| S16 | flock LOCK_UN | 锁文件 | — | — | 新 rev |

### 2.6 每步崩溃矩阵（证据 / 恢复 / 重试 / 新事务 / 人工确认）

| 崩溃点 | current 可观察状态 | 可观察证据 | 状态机 | 恢复动作 | 允许重试？| 允许新事务？| 人工确认？|
|--------|------------------|-----------|--------|---------|----------|-----------|----------|
| S1 后 | 旧 rev | Candidate 半成品 | DURABLE（未写）| 删半成品，重试 | 是 | 是 | 否 |
| S2 前 | 旧 rev | Candidate 内容未持久 | NOT_DURABLE | 重试 fsync | 是 | 否 | 否 |
| S3 前 | 旧 rev | Candidate 目录项未持久 | NOT_DURABLE | 重试 fsync | 是 | 否 | 否 |
| S7 前 | 旧 rev | PREPARED 目录项未持久 | NOT_DURABLE | 校验 PREPARED，重试 | 是 | 否 | 否 |
| S12 前 | 旧 rev | Candidate+PREPARED 完整 | DURABLE | 可重试 | 是 | 是 | 否 |
| S13 前 | 新 rev | current 目录项未持久 | **COMMIT_INDETERMINATE** | 重新读校验 current 身份 | 否 | 否 | 是 |
| S14 前 | 新 rev | current 已持久，回执未写 | **COMMIT_DURABLE_AUDIT_PENDING** | 补写回执 + fsync | 否 | 否（先补回执）| 否 |
| S15 前 | 新 rev | 回执内容已持久，目录项未持久 | COMMIT_DURABLE_AUDIT_PENDING | 补 fsync 回执目录 | 否 | 否（先补）| 否 |
| S16 后 | 新 rev | 全部持久 | DURABLE | — | — | 是 | 否 |

**关键修订**：
- S13 前（rename 已发生但 current 目录项未持久）→ **COMMIT_INDETERMINATE**（无法确定 current 是否已持久化，需人工确认）；
- S14 后（current 已持久但回执未持久）→ **COMMIT_DURABLE_AUDIT_PENDING**（数据层完成，仅审计未完成，可自动补，不需人工确认）。

### 2.7 fsync 失败恢复协议（区分失败类型）

| 失败类型 | 状态机 | 恢复前校验 | 恢复动作 |
|---------|--------|-----------|---------|
| fsync 返回失败（rename 前）| NOT_DURABLE | 重读文件身份 + 哈希 | 重试 fsync |
| fsync 重试成功 | DURABLE | — | 继续 |
| fsync 重试失败 | RECOVERY_REQUIRED | 校验文件身份 + 目录项 | 人工确认 |
| rename 前失败 | NOT_DURABLE | 校验 Candidate/PREPARED | 重试 |
| rename 后、目录 fsync 前失败 | COMMIT_INDETERMINATE | **重读 current 身份 + 哈希 + 版本** | 人工确认 |
| 目录 fsync 成功、COMMIT 回执前崩溃 | COMMIT_DURABLE_AUDIT_PENDING | 校验 current 已持久 | 补回执 |
| 回执写入成功但回执目录 fsync 失败 | COMMIT_DURABLE_AUDIT_PENDING | 校验回执内容 | 补 fsync 目录 |
| 断电后 current 与 PREPARED/COMMIT 不一致 | COMMIT_INDETERMINATE / QUARANTINED | **重读所有证据文件** | 人工裁决 |

**核心规则**：
- 恢复前必须**重新读取并校验**文件身份、内容哈希、事务 ID、版本号和目录项状态；
- **不得盲目回滚**；
- **不得仅根据内存中的事务状态判断提交成功**——必须以磁盘上的 current 文件为准。

### 2.8 Windows 独立协议（保留 NOT_VERIFIED）

- 不得假装等价 POSIX `fsync(dirfd)`；
- Windows 目录项断电持久性 **NOT_VERIFIED**（无权威 API 文档明确支持并证明完整语义）；
- `FILE_FLAG_BACKUP_SEMANTICS`、`FILE_FLAG_WRITE_THROUGH`、`FlushFileBuffers`、`ReplaceFileW` **均不得单独解释为完整的目录项断电持久性保证**；
- 需独立设计 CreateFileW + LockFileEx + ReplaceFileW + FlushFileBuffers 组合，但持久性结论保留 NOT_VERIFIED。

### 2.9 20 条故障时间线 + 15 个反例

| # | 时间线 | 推演 | 结果 |
|---|--------|------|------|
| G1 | Candidate 内容未 fsync | NOT_DURABLE | 拒绝跳过 |
| G2 | Candidate 目录项未 fsync | NOT_DURABLE | 拒绝跳过 |
| G3 | PREPARED 内容未 fsync | NOT_DURABLE | 拒绝跳过 |
| G4 | PREPARED 目录项未 fsync | NOT_DURABLE | 拒绝跳过 |
| G5 | rename 后目录未 fsync | COMMIT_INDETERMINATE | 人工确认 |
| G6 | current 已持久但回执未写 | COMMIT_DURABLE_AUDIT_PENDING | 自动补回执 |
| G7 | fsync 失败 | NOT_DURABLE | 禁提交 |
| G8 | fsync 重试失败 | RECOVERY_REQUIRED | 人工确认 |
| G9 | 证据被篡改 | QUARANTINED | 人工裁决 |
| G10 | 重复 txn_id | O_EXCL 失败 | 拒绝 |
| G11 | 双事务同父版本 | flock 串行化 | 一胜一弃 |
| G12 | Candidate 过期 | P1~P3 失败 | 重生成 |
| G13 | revision 篡改 | P4 失败 | 拒绝 |
| G14 | Candidate 内容篡改 | P5 失败 | 拒绝 |
| G15 | rename 后 COMMIT 前崩溃 | current 为准 | COMMIT_DURABLE_AUDIT_PENDING |
| G16 | 断电后 current 与 PREPARED 不一致 | 重读证据 | COMMIT_INDETERMINATE |
| G17 | 回执目录 fsync 失败 | 补 fsync | COMMIT_DURABLE_AUDIT_PENDING |
| G18 | 锁路径替换 | 身份校验 | DENY |
| G19 | 恶意同 UID | 无强制 | NOT_VERIFIED |
| G20 | Windows 目录持久化 | FlushFileBuffers 弱保证 | NOT_VERIFIED |

15 个反例：Candidate 内容/目录项未持久、PREPARED 内容/目录项未持久、rename 后目录未持久、回执未写、fsync 失败/重试失败、证据篡改、重复 txn、双事务、Candidate 过期/篡改、revision 篡改、断电不一致、锁替换、恶意同 UID、Windows 目录持久化。

### 2.10 P0-04 验收与结论

- 六态状态机：✅ §2.2/§2.3
- 状态迁移表：✅ §2.3
- 六字段 + 五谓词逐项：✅ §2.4
- 16 步持久化表（统一步骤编号）：✅ §2.5
- 每步崩溃矩阵：✅ §2.6
- fsync 失败恢复协议：✅ §2.7
- 20 条时间线 + 15 反例：✅ §2.9
- Windows 边界表：✅ §2.8

**P0-04 结论：DESIGN_PROVEN（POSIX）/ NOT_VERIFIED（Windows）**。
- POSIX 六态状态机闭合，COMMIT_DURABLE_AUDIT_PENDING 从 COMMIT_INDETERMINATE 拆出，步骤编号统一为 16 步。
- Windows 目录持久化 NOT_VERIFIED。

---

## 3. P0-06｜Windows ReplaceFileW 错误码矩阵纠正

### 3.1 缺陷定位：05.19 §3.3 把 1176（未指定备份）的替换文件路径语义写错

05.19 §3.3 将 1176（未指定备份文件名）描述为"替换文件**以原目标名存在**"，并在 §3.3 矩阵中写"替换文件以原目标名存在（替换文件已改名）"。

**这与 Microsoft 官方契约不一致**。官方原文（Microsoft Learn《ReplaceFileW function (winbase.h)》）：

> **1176 `ERROR_UNABLE_TO_MOVE_REPLACEMENT`**："The replacement file could not be renamed. If lpBackupFileName was specified, the replaced and replacement files retain their original file names. Otherwise, the replaced file no longer exists and the replacement file exists **under its original name**."

其中 "its original name" 指代的是 **替换文件（replacement file）自己的原始文件名**，**不是**原目标文件名。即：

- 指定备份名 → 原目标文件与替换文件**都保持各自原名**；
- 未指定备份名 → 原目标文件**不再存在**，替换文件**仍在其自己的原名下**（即替换文件没有被改名成目标名，而是保持在 `lpReplacementFileName` 路径）。

05.19 错误地把"替换文件的原始路径"写成了"原目标路径"，必须纠正。

### 3.2 官方原文与链接（保留关键语义）

**来源**：Microsoft Learn《ReplaceFileW function (winbase.h)》
**链接**：`https://learn.microsoft.com/en-us/windows/win32/api/winbase/nf-winbase-replacefilew`
**版本**：Last updated 2023-06-01

关键原文（Return value 节）：

| 错误码 | 官方原文（关键语义）|
|--------|-------------------|
| 1175 `ERROR_UNABLE_TO_REMOVE_REPLACED` | "The replaced file could not be deleted. The replaced and replacement files **retain their original file names**." |
| 1176 `ERROR_UNABLE_TO_MOVE_REPLACEMENT` | "The replacement file could not be renamed. If lpBackupFileName was specified, the replaced and replacement files retain their original file names. Otherwise, the replaced file no longer exists and the replacement file exists **under its original name**." |
| 1177 `ERROR_UNABLE_TO_MOVE_REPLACEMENT_2` | "The replacement file could not be moved. The replacement file **still exists under its original name**; however, it has **inherited the file streams and attributes** from the file it is replacing. The file to be replaced **still exists with a different name**. If lpBackupFileName is specified, it will be the name of the replaced file." |
| 其他错误（如 `ERROR_INVALID_PARAMETER`）| "the replaced and replacement files will **retain their original file names**. In this scenario, a backup file does not exist and it is not guaranteed that the replacement file will have inherited all of the attributes and streams." |

其他关键约束（Remarks 节）：
- "The resulting file has the same file ID as the replacement file."
- "The backup file, replaced file, and replacement file must all reside on the same volume."

### 3.3 1175/1176/1177 逐码纠正矩阵

| 错误码 | 原目标文件（lpReplacedFileName）| 替换文件（lpReplacementFileName）| 备份文件 | 部分移动？| 可安全重试？| 恢复策略 |
|--------|-------------------------------|--------------------------------|---------|----------|-----------|---------|
| 1175 REMOVE_REPLACED | 存在（原名）| 存在（原名）| 未创建 | 否 | 是（幂等）| 重新读校验后重试 |
| 1176 MOVE_REPLACEMENT（指定备份）| 存在（原名）| 存在（原名）| 可能已创建 | 否 | 是 | 校验后重试 |
| 1176 MOVE_REPLACEMENT（未指定备份）| **不存在** | **存在（替换文件自己的原名）**| 无 | **是（目标已消失）**| **否（先判定）**| INDETERMINATE，按内容判定 |
| 1177 MOVE_REPLACEMENT_2 | 存在（不同名/备份名）| 存在（替换文件自己的原名，已继承流属性）| 存在（即目标名，若指定）| **是** | **否** | INDETERMINATE，按内容判定 |

**关键纠正（相对 05.19）**：
- 1176（未指定备份）的替换文件是"**仍在其自己的原始文件名下**"，**不是**"以原目标名存在"。05.19 的"替换文件以原目标名存在（替换文件已改名）"是**错误**的——恰恰相反，替换文件**没有**被改名成目标名。
- 1177 的替换文件也是"**仍在其自己的原始文件名下**"，但已继承目标文件的流和属性；目标文件以不同名存在。

### 3.4 其他错误码保守解释矩阵（禁止从错误码推断文件状态）

| 错误码 | 官方明确保证 | 官方未保证 | 不得推断 | 必须重读 | 可重试？| INDETERMINATE？|
|--------|-------------|-----------|---------|---------|--------|---------------|
| ERROR_FILE_NOT_FOUND | 无（不保证是哪个文件缺失）| 哪个文件不存在 | 不能推断"一定是替换文件缺失" | 是 | 否（先判定）| 是 |
| ERROR_PATH_NOT_FOUND | 无 | 路径/目录状态 | 不能推断具体路径 | 是 | 否 | 是 |
| ERROR_SHARING_VIOLATION | 无 | 两个文件是否都存在 | 不能推断"两个文件都仍在" | 是 | 否（先判定）| 是 |
| ERROR_ACCESS_DENIED | 无 | 文件位置/存在性 | 不能推断 | 是 | 否 | 是 |
| ERROR_INVALID_PARAMETER | 两文件保留原名 | 是否继承属性 | 不能推断属性继承 | 是 | 是（校验后）| 否 |

**核心规则**：
- **禁止**仅凭 `ERROR_FILE_NOT_FOUND` 推断一定是替换文件缺失；
- **禁止**仅凭 `ERROR_SHARING_VIOLATION` 推断两个文件都仍然存在；
- 文件存在性/位置**只能来自实际重新读取**，不得从错误码推断。

### 3.5 文件身份验证（重新读取时核对项）

重新读取时至少核对：

| 核对项 | 能证明什么 | 不能证明什么 |
|--------|-----------|-------------|
| 路径 | 当前命名空间位置 | 内容/身份（路径可被替换）|
| 文件存在性 | 该路径是否有目录项 | 内容正确性/身份 |
| 文件 ID（file ID）| 对象身份（同一 inode/file record）| 内容（同 ID 内容可被改写）|
| 卷 ID（volume ID）| 文件所在卷 | 内容/身份（仅卷归属）|
| 文件大小 | 字节数 | 内容正确性（同大小不同内容）|
| 内容 SHA-256 | 内容完整性 | 路径/身份（同内容可复制）|
| 预期角色 | 设计意图 | 实际状态 |
| txn_id / cleanup_id | 事务归属 | 内容完整性 |
| 源对象身份 | 关联的源对象 | 内容 |
| 备份身份 | 备份文件归属 | 内容 |

**核心结论**：文件 ID 证明"身份"、内容哈希证明"内容"、路径证明"命名空间位置"，三者**正交**，任何一个都不能单独证明全部。判定必须三者结合，且以**内容哈希 + 文件 ID** 为准，路径仅作辅助。

### 3.6 Tombstone 状态机（六态，严格进入条件）

| 状态 | 严格进入条件 | 证据要求 | 允许操作 | 禁止操作 | 恢复步骤 | 人工确认 |
|------|-------------|---------|---------|---------|---------|---------|
| PLACEHOLDER | 内容 == 占位标记 | 内容哈希 == 占位哈希 | 删除占位，重试 | 当作 PUBLISHED | 删占位 → 重试 | 否 |
| PUBLISHED | 内容 == 有效元数据且字段全 + 哈希匹配 + txn 归属 | 元数据哈希 + txn_id + cleanup_id + 源身份 | 认领为发布成功，补持久化 | 删除/覆盖 | 认领 + 补 Flush | 否 |
| REPLACEMENT_PRESENT | 替换文件仍在其原始名（1176 未指定备份 / 1177）| 文件 ID + 内容哈希 + 预期角色 | 只读校验 | 当作已发布 | 判定归属 | 是 |
| BACKUP_PRESENT | 备份文件存在且为目标原内容 | 备份内容哈希 + 目标身份 | 只读校验 | 删除备份 | 判定是否已发布 | 是 |
| PARTIAL_MUTATION | 已部分移动但无法判定最终态 | 文件 ID + 路径 + 内容哈希冲突 | 只读 | 一切写 | 校验后决定 | 是 |
| INDETERMINATE | 无法唯一判定 | 证据不足/冲突 | 只读 | **删除/覆盖** | 保留现场，人工恢复 | 是 |

**核心规则**：不得将"文件存在"直接认定为 PUBLISHED；元数据必须绑定 txn_id、cleanup_id、源对象身份和哈希；不完整或身份不匹配 → INDETERMINATE；恢复幂等；证据不足不删现场。

### 3.7 20 条恢复时间线 + 15 个反例

| # | 时间线 | 推演 | 结果 |
|---|--------|------|------|
| J1 | 临时文件写一半 | 校验失败删除 | 安全 |
| J2 | 临时 fsync 失败 | 不发布 | 拒绝 |
| J3 | 占位创建后崩溃 | 占位标记 → 删重试 | PLACEHOLDER |
| J4 | 1175 REMOVE_REPLACED | 两文件原名 → 幂等重试 | 可重试 |
| J5 | 1176（指定备份）| 两文件原名 → 校验重试 | 可重试 |
| J6 | 1176（未指定备份）| 目标已消失，替换文件**在其原名** | INDETERMINATE 判定 |
| J7 | 1177 MOVE_REPLACEMENT_2 | 部分移动，替换继承属性 | INDETERMINATE 判定 |
| J8 | ReplaceFileW 成功 Flush 前崩溃 | 元数据哈希匹配 → 补 Flush | PUBLISHED 补持久化 |
| J9 | Flush 失败 | 目录项不持久 | INDETERMINATE |
| J10 | 系统崩溃（断电）| 内容判定 | 按状态矩阵 |
| J11 | 重复 txn | CREATE_NEW 失败 | 拒绝 |
| J12 | 源同名重建 | 源路径又存在 | INDETERMINATE |
| J13 | 元数据被替换/删除 | 不匹配/消失 | INDETERMINATE |
| J14 | 并发清理 | 隔离 rename 竞争 | 一个成功 |
| J15 | 审计写失败 | REMOVED 失败 | INDETERMINATE |
| J16 | ERROR_FILE_NOT_FOUND | 不能推断哪个缺失 | 重读判定 |
| J17 | ERROR_SHARING_VIOLATION | 不能推断两文件都在 | 重读判定 |
| J18 | 文件 ID 与内容哈希冲突 | 身份 vs 内容矛盾 | QUARANTINED |
| J19 | 替换文件在其原名（1177）误当已发布 | 需判定归属 | REPLACEMENT_PRESENT |
| J20 | 备份存在但发布状态不明 | 判定是否已发布 | BACKUP_PRESENT |

15 个反例：临时写一半、fsync 失败、占位误认、1175 幂等、1176 指定/未指定备份、1177 部分移动、Flush 前崩溃、Flush 失败、断电、重复 txn、源重建、元数据替换、并发清理、审计失败、错误码推断文件状态。

### 3.8 P0-06 验收与结论

- 官方原文 + 链接：✅ §3.2
- 1175/1176/1177 逐码纠正矩阵：✅ §3.3（**纠正 1176 路径语义**）
- 其他错误码保守解释：✅ §3.4
- 文件身份校验流程：✅ §3.5
- Tombstone 六态状态机：✅ §3.6
- 20 条时间线 + 15 反例：✅ §3.7

**P0-06 结论：DESIGN_PARTIAL（协议层）/ NOT_VERIFIED（持久性）**。
- 1176 路径语义已纠正（替换文件"在其自己的原始名"而非"原目标名"）；
- ReplaceFileW 部分失败的中间态、Windows 目录持久化 NOT_VERIFIED。

---

## 4. P0-05｜Source Retire 继续保持 DENY

### 4.1 承接 05.18/05.19 的权威结论（无需新修复）

05.18 已通过 chattr(1) 官方 man page 双重推翻 `chattr +i` 方案；05.19 已厘清"当前方案未证明"与"平台不存在可行机制"的边界。本轮**不试图解决或批准 Source Retire**，仅复核边界表述。

### 4.2 必须保留并严格区分的对象与操作

| 概念 | 定义 | 与强制不可变的关系 |
|------|------|-------------------|
| 已有 FD | 进程内已打开的写句柄 | 旧 FD 写绕过 |
| mmap | 内存映射页面 | 写绕过 |
| 硬链接 | 额外目录项 | 写绕过 |
| 路径与目录项 | 命名空间引用 | 退役移除对象 |
| inode / 文件 ID | 对象身份 | 身份 ≠ 内容不可变 |
| 内容不可变性 | 内容不再变化 | 目标性质 |
| 权限与挂载边界 | 访问控制 | 仅协作语义 |
| 同 UID 攻击模型 | 同 UID 恶意进程 | 无法强制防御 |
| 进程外写入 | 外部进程写 | 无法拦截 |
| 旧 FD 写入 | 旧句柄写 | 无法拦截 |

**严格区分五个维度**（不得混为一谈）：
1. "路径不可见" ≠ "目录项已移除"；
2. "目录项已移除" ≠ "内容不可写"；
3. "内容不可写" ≠ "已有 FD 不可写"；
4. "已有 FD 不可写" ≠ "强制不可变"；
5. 只读权限、flock、哈希、rename、普通句柄限制 **均不等于强制不可变**。

### 4.3 P0-05 结论

**P0-05 = DENY（保持）**。不解除阻断，不制造 PASS。

---

## 5. 设计不变量（I-01 ~ I-45）

| # | 定义 | 适用范围 | 前置条件 | 证明/证据 | 失效反例 | 失败状态 | 拒绝策略 | 验证状态 |
|---|------|---------|---------|----------|---------|---------|---------|---------|
| I-01 | fork 不绕过 CPython 生命周期 | fork | 主解释器 | CPython 文档 | os._fork 后执行 Python | 运行时未重置 | 用 os.fork | DESIGN_PROVEN |
| I-02 | before 钩子不取锁 | fork | — | 协议 §1.2 | before 持锁 | 重入死锁 | 只读快照 | DESIGN_PROVEN |
| I-03 | 受控 fork 仅主线程 | fork | 主线程 | CPython 约束 | 非主线程 fork | 线程锁消失 | DENY | DESIGN_PARTIAL |
| I-04 | FD 绑定 generation | FD | — | 协议 | 裸数字复用 | 误关 | 不匹配不关 | DESIGN_PROVEN |
| I-05 | child 不获取继承锁 | fork child | — | 协议 | 继承 locked mutex | 死锁 | 直接赋值 | DESIGN_PROVEN |
| I-06 | 禁止持锁 fork | fork | — | 锁序 | 持锁 fork | 死锁 | 拒绝 | DESIGN_PROVEN |
| I-07 | 锁获取升序 | 全锁 | — | 锁序图 | 逆序 | 死锁 | 禁止 | DESIGN_PROVEN |
| I-08 | **准入原子性（检查+计数同临界区）** | FD 准入 | — | §1.2 | 检查后计数前 FREEZING | 快照不一致 | 原子准入 | DESIGN_PROVEN |
| I-09 | 冻结覆盖真正的 fork 窗口 | fork | 受控入口 | §1.2 | 快照后释放锁 | 未门禁窗口 | FREEZING 态保持 | DESIGN_PROVEN |
| I-10 | 第三方 C FD 操作 DENY/UNSUPPORTED | FD | 第三方扩展 | §1.6 | C 直接 open/fork | 绕过 Gate | DENY | DESIGN_PROVEN |
| I-11 | 子解释器/嵌入 fork DENY | fork | — | §1.5 | 子解释器 fork | 未定义行为 | DENY | DESIGN_PARTIAL |
| I-12 | child 未知 FD 不主动 close | fork child | — | §1.6 | 误关未知 FD | 资源错关 | 标 UNKNOWN_FD | DESIGN_PROVEN |
| I-13 | 计数严格配对（RAII）| FD 准入 | — | §1.3 | 计数泄漏 | 排空失效 | finally/RAII | DESIGN_PROVEN |
| I-14 | freeze 异常恢复 OPEN | fork | — | §1.3 | freeze 异常未恢复 | Gate 永久 FREEZING | catch 恢复 | DESIGN_PROVEN |
| I-15 | Candidate 绑定六字段 | 提交 | — | §2.4 | 未绑父版本 | 错误提交 | 五条件 | DESIGN_PROVEN |
| I-16 | 五条件完整谓词 | 提交 | 全局提交锁 | §2.4 | 笼统五条件 | 漏校验 | 逐项 P1~P5 | DESIGN_PROVEN |
| I-17 | 文件内容先于目录项持久化 | 提交 | — | §2.5 | 未 fsync 文件 | 数据丢 | 禁止 | DESIGN_PROVEN |
| I-18 | 目录项 fsync(dirfd) | 提交 | — | §2.5 | 目录项未持久 | 目录项丢 | 强制 | DESIGN_PROVEN |
| I-19 | rename 原子性 ≠ 断电持久性 | 提交 | — | §2.6 | 误当持久 | 断电丢 | fsync 目录 | DESIGN_PROVEN |
| I-20 | 状态机六态语义 | 提交 | — | §2.3 | 状态混用 | 错误恢复 | 六态区分 | DESIGN_PROVEN |
| I-21 | COMMIT_DURABLE_AUDIT_PENDING 独立于 INDETERMINATE | 提交 | — | §2.3 | 回执未持久当不确定 | 过度人工确认 | 拆出独立态 | DESIGN_PROVEN |
| I-22 | fsync 失败禁后续提交 | 提交 | — | §2.3 | fsync 失败后提交 | 不确定提交 | NOT_DURABLE 禁 | DESIGN_PROVEN |
| I-23 | 不确定态禁回滚/重复提交 | 提交 | — | §2.3 | 自动回滚 | 状态破坏 | 禁自动 | DESIGN_PROVEN |
| I-24 | 每崩溃点确定恢复 | 提交 | — | §2.6 | 无规则 | 恢复错误 | 按矩阵 | DESIGN_PROVEN |
| I-25 | 步骤编号统一（16 步）| 提交 | — | §2.5 | 编号不一致 | 引用错乱 | 统一 S1~S16 | DESIGN_PROVEN |
| I-26 | revision 单调 | 提交 | — | §2.4 | 回退 | 版本倒退 | P4 断言 | DESIGN_PROVEN |
| I-27 | 恢复前重读证据 | 提交 | — | §2.7 | 仅凭内存状态 | 错误判断 | 重读磁盘 | DESIGN_PROVEN |
| I-28 | 恶意同 UID 锁替换排除 | 锁 | 威胁模型 | — | 同 UID 替换 | 锁分裂 | 边界 NOT_VERIFIED | NOT_VERIFIED |
| I-29 | +i 不阻旧 FD 写 | Source Retire | — | chattr(1) man page | 旧 FD 写成功 | 竞态 | DENY | NOT_VERIFIED（文档证实）|
| I-30 | +i 阻止 rename | Source Retire | — | chattr(1) man page | rename EPERM | 退役失败 | DENY | NOT_VERIFIED（文档证实）|
| I-31 | 源内容稳定无法本地证明 | Source Retire | — | §4.2 | 旧 FD/mmap/硬链接写 | 竞态 | DENY | DESIGN_PARTIAL |
| I-32 | Source Retire 默认 DENY | Source Retire | — | §4.3 | 无强制边界 | 不安全 | DENY | DESIGN_PARTIAL |
| I-33 | 五个维度严格区分 | Source Retire | — | §4.2 | 维度混用 | 错误结论 | 区分 | DESIGN_PROVEN |
| I-34 | 只读副本 ≠ 源不可变 | Source Retire | — | §4.2 | 副本当源证据 | 错误 | 不混用 | DESIGN_PROVEN |
| I-35 | 元数据原子发布 | tombstone | — | §3 | O_EXCL 逐步写 | 半成品 | 临时+校验+无覆盖 | DESIGN_PROVEN |
| I-36 | 占位 ≠ 有效 tombstone | tombstone | — | §3.6 | 占位误认 | 错误发布 | 内容区分 | DESIGN_PARTIAL |
| I-37 | 1176 路径语义正确 | tombstone | — | Microsoft Learn 原文 | "以原目标名存在" | 路径混淆 | 纠正为"其原名" | DESIGN_PROVEN（文档依据）|
| I-38 | 错误码不推断文件状态 | tombstone | — | §3.4 | 从错误码推断 | 错误判定 | 重新读取 | DESIGN_PROVEN |
| I-39 | 文件 ID/内容哈希/路径三者正交 | tombstone | — | §3.5 | 单维证明全部 | 错误 | 三者结合 | DESIGN_PROVEN |
| I-40 | ReplaceFileW 部分失败按内容判定 | tombstone | — | §3.3/§3.6 | 假设文件位置 | 错误 | INDETERMINATE | DESIGN_PARTIAL |
| I-41 | 恢复幂等 | tombstone | — | §3.6 | 非幂等 | 重复副作用 | 状态机 | DESIGN_PROVEN |
| I-42 | 归档独立建模 | 归档 | — | 继承 05.19 | 复用结论 | 错误 | 单独协议 | DESIGN_PROVEN |
| I-43 | 移除 ≠ 销毁 | 删除 | — | §3 | 仅源消失 | 误当销毁 | INDETERMINATE | DESIGN_PROVEN |
| I-44 | 无法证明必须 DENY/INDETERMINATE | 全局 | — | 全文 | 猜测 | 错误结论 | 不猜测 | DESIGN_PROVEN |
| I-45 | 不把推演写运行时验证 | 全局 | — | 全文 | 自审当验证 | 虚假验证 | 标 NOT_TESTED | DESIGN_PROVEN |

**共 45 条不变量。** 本轮新增 I-08（准入原子性）、I-13（计数 RAII 配对）、I-14（freeze 异常恢复）、I-21（COMMIT_DURABLE_AUDIT_PENDING 独立态）、I-25（步骤编号统一）、I-33（五维度区分）、I-37（1176 路径语义正确）、I-39（身份/内容/路径正交）。

---

## 6. 全局一致性审查

### 6.1 20 条端到端链路

| # | 链路 | 前置条件 | 关键边界 | 崩溃点 | 恢复 | 结论 |
|---|------|---------|---------|--------|------|------|
| L1 | 受控 fork→原子准入→FREEZING→fork→恢复 | 主线程 | CPython 生命周期 + 冻结覆盖 | fork 中 | child 重建 | DESIGN_PARTIAL |
| L2 | FD 准入→计数→冻结→快照→fork→child 重建 | gate 原子 | generation + in_flight | 冻结中 | 校验 | DESIGN_PROVEN |
| L3 | 第三方 C open→fork→child 处理 | 无法拦截 | 不可控 | — | 标 UNKNOWN_FD | DESIGN_PARTIAL |
| L4 | Candidate→fsync 文件→fsync 目录→rename→fsync 目录 | — | 双重持久化 | rename 前后 | 按 current | DESIGN_PROVEN |
| L5 | fsync 失败→NOT_DURABLE→禁提交 | — | 故障状态机 | fsync | 重试/人工 | DESIGN_PROVEN |
| L6 | rename 后目录未持久→COMMIT_INDETERMINATE | — | 不确定态 | rename 后 | 人工确认 | DESIGN_PROVEN |
| L7 | current 已持久但回执未写→AUDIT_PENDING | — | 拆分状态 | S14 前 | 自动补回执 | DESIGN_PROVEN |
| L8 | 双事务竞争 | — | flock | 竞争点 | 败者重生成 | DESIGN_PROVEN |
| L9 | 断电后 current 与 PREPARED 不一致 | — | 重读证据 | 断电 | 人工裁决 | DESIGN_PROVEN |
| L10 | 源 seal→quarantine→retire | — | retire DENY | — | DENY | DENY |
| L11 | 旧 FD + i | — | +i 不阻旧 FD | 校验后写 | 竞态 | DENY |
| L12 | rename + i | — | rename EPERM | — | 退役失败 | DENY |
| L13 | tombstone 临时→校验→无覆盖发布 | — | 临时 fsync | 发布前后 | 占位/有效 | DESIGN_PARTIAL |
| L14 | ReplaceFileW 1175 无移动 | — | 官方契约 | — | 幂等重试 | DESIGN_PROVEN |
| L15 | ReplaceFileW 1176 未指定备份 | — | 官方契约（目标消失）| 替换中 | INDETERMINATE | DESIGN_PARTIAL |
| L16 | ReplaceFileW 1177 部分移动 | — | 官方契约 | 替换中 | INDETERMINATE | DESIGN_PARTIAL |
| L17 | 文件 ID 与内容哈希冲突 | — | 身份 vs 内容 | — | QUARANTINED | DESIGN_PROVEN |
| L18 | 归档协议 | — | 临时+校验+发布 | 归档中 | 哈希确认 | DESIGN_PROVEN |
| L19 | 审计不可用 | — | REMOVED 失败 | 审计写 | INDETERMINATE | DESIGN_PROVEN |
| L20 | 恶意同 UID | — | 无 | 任意 | 检测 | NOT_VERIFIED |

### 6.2 时间线与反例汇总

- 时间线：P0-02 二十条 + P0-04 二十条 + P0-06 二十条 = **60 条**（≥40 达标）；
- 反例：P0-02 十五个 + P0-04 十五个 + P0-06 十五个 = **45 个**（≥30 达标）。

### 6.3 跨章节身份一致性

| 身份 | 产生 | 传递 | 一致性 |
|------|------|------|--------|
| FD generation | §1.6 | §1.7 | generation 绑定 + gate 原子准入 |
| in_flight 计数 | §1.2 | §1.7 | RAII 严格配对 |
| 父版本三元组 | §2.4 | §2.6 | 五条件 P1~P3 |
| 持久化状态 | §2.3 | §2.6 | 六态统一 |
| 源对象身份 | §4 | §4.2 | retire DENY |
| Tombstone 事务身份 | §3.5/§3.6 | §3.7 | txn 归因 + 身份/内容/路径正交 |

### 6.4 术语统一

| 术语 | 统一定义 |
|------|---------|
| 步骤编号 | S1~S16（唯一）|
| 状态机六态 | DURABLE / COMMIT_DURABLE_AUDIT_PENDING / NOT_DURABLE / COMMIT_INDETERMINATE / RECOVERY_REQUIRED / QUARANTINED |
| Tombstone 六态 | PLACEHOLDER / PUBLISHED / REPLACEMENT_PRESENT / BACKUP_PRESENT / PARTIAL_MUTATION / INDETERMINATE |
| 1176 路径 | 替换文件"在其自己的原始名"（非"原目标名"）|

### 6.5 历史阻断逐项标记

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

## 7. 阻断状态（保持不变）

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

## 8. 最终结论

**DESIGN-05.20 = PASS-WITH-ISSUES**，分项结论：

| 项 | 结论 | 依据 |
|----|------|------|
| P0-02 | DESIGN_PARTIAL | 原子准入（检查+计数同临界区）补证了线性一致性（协议层 DESIGN_PROVEN）；第三方 C FD/fork 不可控 UNSUPPORTED/DENY；运行时 NOT_TESTED |
| P0-04 POSIX | DESIGN_PROVEN | 六态状态机（拆出 COMMIT_DURABLE_AUDIT_PENDING）+ 16 步统一编号 + 五谓词逐项 + TOCTOU 消除 |
| P0-04 Windows | NOT_VERIFIED | FlushFileBuffers 目录项持久性无权威保证 |
| P0-05 | DENY（保持）| 不制造 PASS，五个维度严格区分 |
| P0-06 | DESIGN_PARTIAL（协议层）/ NOT_VERIFIED（持久性）| 1176 路径语义已纠正（替换文件"在其原名"）；部分失败中间态 NOT_VERIFIED |
| 交付完整性 | PASS | 8 文件 + ZIP 独立解压逐字节比对 |
| IMPLEMENT-01 | **NOT_APPROVED** | 未授权实现 |

**核心诚实结论**：
1. **P0-02 准入原子性已补证**：05.19 的"检查 Gate 后、计数前"竞态窗口通过"状态检查与 in_flight 登记合并为同一临界区"消除，线性化点明确。但第三方 C 扩展直接 FD 操作/fork 仍不可控，UNSUPPORTED/DENY。
2. **P0-04 状态机细化**：COMMIT_DURABLE_AUDIT_PENDING 从 COMMIT_INDETERMINATE 拆出，区分"数据层已提交、审计未完成"与"无法确定是否已持久化"，步骤编号统一为 16 步。
3. **P0-06 纠正 1176 语义**：替换文件"在其自己的原始名"而非"原目标名"，依据 Microsoft Learn 官方原文。
4. **P0-05 保持 DENY**。

**默认结论：IMPLEMENT-01 NOT_APPROVED。** 等待 ChatGPT 独立复审。

---

## 9. 交付物索引

| 文件 | 说明 |
|------|------|
| `USIT-STAGE-B-DESIGN-05.20.md` | 本主设计文档 |
| `USIT-STAGE-B-DESIGN-05.20-ISSUES.md` | 问题清单及剩余风险 |
| `USIT-STAGE-B-DESIGN-05.20-DIFF.md` | 相对 05.19 逐项差异 |
| `USIT-STAGE-B-DESIGN-05.20-CROSS-REVIEW.md` | 反例审查记录 |
| `USIT-STAGE-B-DESIGN-05.20-EXECUTION-REPORT.md` | 实际执行报告 |
| `USIT-STAGE-B-DESIGN-05.20-HASH-MANIFEST.md` | 交付文件 SHA256 清单 |
| `USIT-STAGE-B-DESIGN-05.20-HASH-RECEIPT.md` | 哈希验证回执 |
| `USIT-STAGE-B-DESIGN-05.20-DELIVERY.zip` | 完整打包 |

---



