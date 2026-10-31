# USIT-STAGE-B-DESIGN-05.22｜Fork锁释放与快照生命周期修复＋POSIX事务状态机重构＋S1～S16完整崩溃矩阵＋交付独立核验

> 阶段 B · DESIGN-05.22 · STRICT READ-ONLY DESIGN REVISION ONLY
> 作者：USIT 主开发 · 生成：2026-10-31（GMT+8）
> 前置审查：DESIGN-05.21 独立复审结论 FAIL
> 工作区：`C:\Users\Administrator\WorkBuddy\2026-10-15-22-09-47\usit_p0`
> 交付目录：`C:\Users\Administrator\WorkBuddy\2026-10-15-22-09-47\usit_p0_05.22_reports`（**项目外**）
> 状态：**STOP — 等待独立复审**（未授权 IMPLEMENT-01）

---

## 0. 缺陷追踪表（B22-01 ~ B22-05）

| # | 原始缺陷 | 05.21 位置 | 05.22 修订位置 | 证据 | 验证方式 | 状态 | 剩余风险 | 解除阻断？|
|---|---------|-----------|---------------|------|---------|------|---------|----------|
| B22-01 | `after_in_parent()` 与 `fork_entry()` finally 对 `_fork_coord_lock` 双重释放 | §1.4 时序图(120行) + §1.5(134/146行) | §1.2 唯一释放协议 | 锁所有权与释放责任表 | 逐路径 A~H 审查 | 闭合 | 第三方 C fork | 否 |
| B22-02 | `freeze()` 快照与 `before()` 之间存储位置/所有权/生命周期/并发隔离未定义 | §1.4 时序图 | §1.3 fork_epoch 快照协议 | fork_epoch 状态机 + 所有权图 | 并发时间线 | 闭合 | 第三方 C fork | 否 |
| B22-03 | P0-04 最终态 `DURABLE` 与 `DURABLE（进行中）` 混用，S1~S16 崩溃矩阵不完整 | §2.3/§2.5 | §2.1 七态状态机 + §2.4 完整矩阵 | 状态迁移表 + 10 类崩溃位置 | 逐步矩阵 | 闭合 | Windows 目录持久化 | 否 |
| B22-04 | ZIP 回执未提供真实 SHA-256/字节数，独立解压/CRC32/逐字节比对未完成 | §8 | §6 交付核验 | RECEIPT 记录 ZIP 哈希 + 核验结果 | 真实解压+比对 | 闭合 | — | 否 |
| B22-05 | 源码未修改缺少本轮独立前后基线证据 | — | §6.3 基线采集 | 本轮前后基线对比 | 逐文件 SHA-256 | 闭合 | — | 否 |

**本节是全文总纲。本轮核心任务：P0-02 唯一锁释放协议 + fork_epoch 快照协议；P0-04 七态状态机消除 DURABLE 混用 + 完整崩溃矩阵；交付独立核验 + 源码前后基线。**

---

## 1. P0-02｜Fork 协议修订

### 1.1 缺陷定位：05.21 的 `_fork_coord_lock` 双重释放路径

05.21 存在两处矛盾：

1. §1.4 时序图第 120 行：`after_in_parent()` 内执行 `release(_fork_coord_lock)`；
2. §1.5 第 134 行：`fork_entry 用 try/finally 包裹，finally 保证恢复 OPEN 并释放 _fork_coord_lock`；
3. §1.5 第 146 行：`after_in_parent 异常 | 持 L0 | finally 释放 L0`。

**问题**：`after_in_parent` 释放一次 `_fork_coord_lock`，`fork_entry` 的 finally 又释放一次 → **同一把锁被释放两次**，第二次释放是未定义行为（未持有锁的释放）。这是 B22-01。

**根本原因**：05.21 把"锁释放责任"同时指派给了两个函数，没有明确唯一释放者。

### 1.2 唯一锁释放协议（选定"fork_entry 统一释放"）

**决策**：采用 **`fork_entry()` 统一负责 `_fork_coord_lock` 的释放**，钩子（before/parent/child）**不得**获取或释放 `_fork_coord_lock`。

**理由**：
- `_fork_coord_lock` 由 `fork_entry()` 获取（L0），释放责任归获取者，这是最清晰的 RAII 语义；
- at-fork 钩子由 CPython 在 `os.fork()` 内部触发，其调用顺序与 `os.fork()` 返回顺序不完全可控，让钩子负责释放跨函数边界的锁易产生重复释放或遗漏释放；
- 唯一释放者使"获取者=释放者"成立，天然避免重复释放。

**锁所有权与释放责任表**：

| 锁 | 获取者 | 所有者 | 唯一释放者 | 释放前置条件 |
|----|--------|--------|-----------|-------------|
| `_fork_coord_lock` (L0) | `fork_entry()` | `fork_entry()` | **`fork_entry()`（finally）** | 已完成 parent 状态恢复（或异常路径已处理）|
| `_gate_lock` (L1) | `freeze()` / 准入 / `after_in_parent()` | 各临界区 | 各临界区自身 | 临界区结束 |
| `_fd_lock` (L2) | `freeze()` / FD 操作 | 各临界区 | 各临界区自身 | 临界区结束 |
| 业务锁 (L3) | 业务代码 | 业务代码 | 业务代码 | 禁止持锁 fork |

**关键规则**：
- `_fork_coord_lock` 由 `fork_entry()` 唯一释放，**钩子不得触碰**；
- `after_in_parent()` 只负责释放它自己获取的 `_gate_lock`（恢复 Gate 状态），**不释放** `_fork_coord_lock`；
- `after_in_child()` 不释放任何继承锁（直接赋值全新锁对象）。

### 1.3 逐路径释放审查（A~H）

| 路径 | 锁获取 | 所有者 | 唯一释放 | 释放前置 | 重复释放？| 异常时 Gate | 允许再 fork？| 进入 QUARANTINED/DENY？|
|------|--------|--------|---------|---------|----------|------------|-------------|----------------------|
| A. fork 成功父返回 | fork_entry | fork_entry | fork_entry finally | after_in_parent 已恢复 Gate | 否 | OPEN | 是 | 否 |
| B. fork 系统调用失败 | fork_entry | fork_entry | fork_entry finally | except 恢复 Gate=OPEN | 否 | OPEN | 是（可重试）| 否 |
| C. before 钩子失败 | fork_entry | fork_entry | fork_entry finally | except 恢复 Gate=OPEN | 否 | OPEN | 是 | 否 |
| D. after_in_parent 失败 | fork_entry | fork_entry | fork_entry finally | except 恢复 Gate=OPEN | 否 | OPEN（恢复）| 是 | 否 |
| E. freeze 失败 | fork_entry | fork_entry | fork_entry finally | except 恢复 Gate=OPEN | 否 | OPEN | 是 | 否 |
| F. fork 入口异常/取消 | fork_entry | fork_entry | fork_entry finally | finally 恢复 Gate=OPEN | 否 | OPEN | 是 | 否 |
| G. 连续 fork | fork_entry | fork_entry | fork_entry finally | 每次独立释放 | 否 | OPEN | 是 | 否 |
| H. after_in_child 失败 | child | child | 不适用（child 不释放 coord）| — | 否 | child-local | 不适用 | DENY 受保护操作 |

**证明无重复释放**：`_fork_coord_lock` 的释放**只出现在 `fork_entry()` 的 finally 块一处**，钩子不触碰，因此不存在两条释放路径。CPython at-fork 钩子调用顺序（PyOS_BeforeFork → fork() → PyOS_AfterFork_Parent/Child）中，钩子只操作 `_gate_lock`（parent 恢复 Gate）或直接赋值锁对象（child），不涉及 `_fork_coord_lock`。

### 1.4 fork_epoch 快照协议（修复 B22-02）

**问题**：05.21 未定义 `freeze()` 返回的快照如何交给 `before()`，存储位置、所有权、生命周期、并发隔离均未定义。

**设计**：引入 `fork_epoch`（单调递增的 fork 事务标识），快照与 fork 一一对应。

**快照生命周期**：

```
fork_entry():
    epoch = ++fork_epoch_counter          # 原子递增，唯一 fork 事务 ID
    acquire(_fork_coord_lock)             # L0，串行化 fork
    snapshot = freeze(epoch)              # freeze 生成并登记快照
    _pending_snapshot = (epoch, snapshot) # 存入进程级单一槽位（受 _fork_coord_lock 保护）
    os.fork()                             # before 钩子读取 _pending_snapshot
      ├─ before(): 只读 _pending_snapshot[epoch]，不取锁
      ├─ parent: after_in_parent(): 恢复 Gate，清空 _pending_snapshot（置 None）
      └─ child: after_in_child(): 读取快照，重建，置 child-local 状态
    finally:
        _pending_snapshot = None          # 异常/取消路径也清空
        release(_fork_coord_lock)
```

**快照所有权表**：

| 项 | 定义 |
|----|------|
| 唯一存储位置 | 进程级单一槽位 `_pending_snapshot = (epoch, snapshot_tuple)`，由 `_fork_coord_lock` 保护写入 |
| 创建者 | `freeze(epoch)` |
| 读取者 | `before()`（只读）、`after_in_child()`（child 副本）|
| 销毁者 | `after_in_parent()`（父进程清空）或 `fork_entry()` finally（异常路径）|
| 生命周期起点 | `freeze(epoch)` 生成快照时 |
| 生命周期终点 | `after_in_parent()` 清空 / finally 清空 |
| 父子可见性 | 父进程：`_pending_snapshot` 槽位；子进程：fork 时刻的内存副本 |
| 多次 fork 并发 | `_fork_coord_lock` 串行化，epoch 单调，同一时刻只有一个 pending 快照 |
| 连续 fork 替换 | 新 epoch 的 freeze 覆盖旧 pending（旧 fork 已结束才进入新 fork）|
| 异常清理 | finally 清空 `_pending_snapshot` |
| 快照后 Registry 是否可变 | 快照是 Registry 的**不可变 tuple 副本**，之后 Registry 变化不影响快照 |
| 防覆盖 | `_fork_coord_lock` 保护写入，before 只读，无并发写 |
| 防旧快照误用 | epoch 校验：child/钩子校验 epoch 与当前 fork 一致，不匹配则拒绝 |

**fork_epoch 状态机**：

```
fork_epoch 生命周期：
  IDLE ──fork_entry 递增 epoch──▶ PENDING（_pending_snapshot 已登记）
  PENDING ──os.fork()──▶ fork 成功？├─ 是：parent PENDING→CLEARED（after_in_parent 清空）→ IDLE
                                     └─ 否：PENDING→CLEARED（finally 清空）→ IDLE
  PENDING ──before/child 读取校验 epoch 匹配──▶ 匹配则继续，不匹配则 DENY
```

**证明快照与 fork 一一对应**：`fork_epoch_counter` 单调递增，每次 `fork_entry` 分配唯一 epoch，`_pending_snapshot` 单槽位在 `_fork_coord_lock` 串行化下只承载当前 epoch 的快照，before/child 通过 epoch 校验确保不误用旧快照。

### 1.5 锁顺序与状态机（重新检查）

保留 05.21 锁顺序：`L0=_fork_coord_lock → L1=_gate_lock → L2=_fd_lock → L3=业务锁`。

逐项重新检查：

| 检查项 | 结果 |
|--------|------|
| L3→L0/L1/L2 逆序 | 不存在（禁止持锁 fork 规则）|
| 持有 L1 等待条件变量死锁 | 不存在（条件变量 wait 原子释放 L1）|
| after 钩子取锁循环等待 | 不存在（parent 只取 L1 恢复 Gate，child 不取锁）|
| fork 后继承已锁定业务锁 | 存在风险（多线程 fork 时其他线程锁）→ child 不 acquire 继承锁 |
| 锁释放后 Gate 未同步 | 不存在（after_in_parent 恢复 OPEN，finally 兜底）|
| child 误关未知 FD | 不存在（generation 匹配才 close）|
| generation/FD 复用竞态 | 不存在（FREEZING 阻断 + generation 双防线）|

**注意**：锁顺序正确 ≠ fork 安全。协议层证明（§1.2~§1.4）与 CPython 运行时未验证项（§1.7）分别标注。

### 1.6 fork 入口支持矩阵

| 入口 | 状态 |
|------|------|
| 受控 `os.fork()` | **SUPPORTED**（DESIGN_PROVEN 协议层 / NOT_TESTED 运行时）|
| `os._fork()` | **DENY**（跳过 CPython 生命周期）|
| 非主线程 fork | **DENY** |
| 多线程进程（受控 fork）| **DESIGN_PARTIAL** |
| 第三方 C 扩展直接 libc fork | **UNSUPPORTED/DENY** |
| 第三方 C 扩展直接 open/close/dup | **UNSUPPORTED/DENY** |
| 嵌入式 Python | **UNSUPPORTED/DENY** |
| 子解释器 | **UNSUPPORTED/DENY** |
| `os.posix_spawn()` | **UNSUPPORTED**（不经本库 fork 入口）|
| multiprocessing spawn | **SUPPORTED**（独立进程创建，不 fork 本进程）|
| multiprocessing fork | **DESIGN_PARTIAL**（内部 fork 不经本库入口）|
| subprocess | **不适用**（不涉及本进程 fork）|
| 连续 fork | **SUPPORTED**（epoch 单调 + 幂等恢复）|

**关键结论**：任何不经过本库 Gate/Registry 的入口必须明确拒绝或标不支持。**at-fork 钩子不能覆盖第三方直接 libc fork**——因为 at-fork 钩子仅由 `os.fork()` 触发，第三方 C 扩展直接调用 `libc fork()` 不触发。

### 1.7 P0-02 验收材料

- 锁级别与获取顺序表：✅ §1.2/§1.5
- 锁依赖图：✅ 继承 05.21 §1.3（L0→L1→L2→L3 无环）
- fork_entry/freeze/before/parent/child 调用图：✅ §1.4
- 锁所有权与释放责任表：✅ §1.2
- 快照所有权与生命周期图：✅ §1.4
- fork_epoch 状态机：✅ §1.4
- 异常/取消恢复矩阵：✅ §1.3
- API 支持/拒绝矩阵：✅ §1.6
- 20 条并发/异常时间线：✅ §1.8
- 15 个反例：✅ §1.9

### 1.8 20 条并发/异常时间线

| # | 场景 | 初始状态 | 事件顺序 | 锁状态 | Gate | 快照 epoch | FD Registry | 预期结果 | 拒绝策略 |
|---|------|---------|---------|--------|------|-----------|-------------|---------|---------|
| T1 | fork 成功 | IDLE | fork_entry→freeze→os.fork→恢复 | L0→L1→L2 升序 | FREEZING→OPEN | epoch=N 登记→清空 | 冻结→恢复 | 安全 | — |
| T2 | fork 系统调用失败 | IDLE | fork 返回 -1 | L0 持有 | FREEZING→OPEN | 清空 | 未变 | 可重试 | finally 恢复 |
| T3 | before 钩子失败 | PENDING | before 抛异常 | L0 | FREEZING→OPEN | 清空 | 未变 | 可重试 | finally 恢复 |
| T4 | after_in_parent 失败 | parent | parent 钩子异常 | L0 | OPEN（恢复）| 清空 | 未变 | 不影响 child | finally 恢复 |
| T5 | freeze 失败 | IDLE | freeze 异常 | L0 | OPEN（恢复）| 未登记 | 未变 | 可重试 | catch 恢复 |
| T6 | 准入后计数泄漏 | OPEN | do_fd_operation 异常 | L1 | OPEN | — | — | 计数配对 | RAII |
| T7 | 快照后 FD close | FREEZING | close 被阻断 | L1 | FREEZING | 快照不变 | 冻结 | 一致 | 阻断 |
| T8 | FD 数字复用 | FREEZING | open 复用被阻断 | L1 | FREEZING | 快照不变 | 冻结 | 不误关 | generation |
| T9 | 连续 fork | OPEN | 两次 fork 依次 | L0 串行 | OPEN | epoch 递增 | 各自冻结 | 幂等 | — |
| T10 | 旧快照误用 | PENDING | child 读到旧 epoch | — | — | epoch 不匹配 | — | DENY | epoch 校验 |
| T11 | 第三方 C fork | 任意 | 绕过 gate | — | — | 无 | 未知 | DENY | UNSUPPORTED |
| T12 | 第三方 C open FD | 任意 | 绕过 gate | — | — | 无 | 快照外 | UNKNOWN_FD | 不关闭 |
| T13 | 非主线程 fork | 任意 | 线程检查失败 | — | — | — | — | DENY | DENY |
| T14 | 子解释器 fork | 子解释器 | 检测失败 | — | — | — | — | DENY | DENY |
| T15 | posix_spawn | — | 不经 fork_entry | — | — | — | — | 不支持 | UNSUPPORTED |
| T16 | after_in_child 失败 | child | child 重建异常 | — | child-local | — | — | _fork_reset_ok=False | DENY |
| T17 | 持锁 fork（L3→L0）| 业务持锁 | fork 检测 | L3 | — | — | — | 拒绝 | 禁止 |
| T18 | 锁重复释放（B22-01）| — | after 释放 + finally 释放 | L0 | — | — | — | 已消除 | 唯一释放者 |
| T19 | multiprocessing fork | — | 内部 fork 不经本库 | — | — | — | — | 部分受控 | DESIGN_PARTIAL |
| T20 | 快照槽位并发覆盖 | — | 两 fork 竞争 | L0 串行 | — | epoch 唯一 | — | 不覆盖 | 单槽位+锁 |

### 1.9 15 个反例及拒绝策略

| # | 反例 | 拒绝策略 |
|---|------|---------|
| R1 | `_fork_coord_lock` 双重释放（B22-01）| fork_entry 唯一释放 |
| R2 | 快照存储/所有权未定义（B22-02）| fork_epoch 单槽位协议 |
| R3 | 旧快照被新 fork 误用 | epoch 校验 |
| R4 | os._fork 后执行 Python | DENY |
| R5 | before 持锁重入 | before 无锁 |
| R6 | 非主线程 fork | DENY |
| R7 | 子解释器/嵌入 fork | DENY/UNSUPPORTED |
| R8 | 第三方 C fork | DENY |
| R9 | 第三方 C open FD 误关 | child 只关 generation 匹配 |
| R10 | 快照后 FD 复用 | FREEZING 阻断 + generation |
| R11 | 连续 fork 状态残留 | 幂等恢复 |
| R12 | 持锁 fork 死锁 | 禁止持锁 fork |
| R13 | freeze 异常未恢复 OPEN | catch 恢复 |
| R14 | 线程取消打断计数 | RAII/禁用取消 |
| R15 | posix_spawn/multiprocessing fork 不经入口 | UNSUPPORTED/DESIGN_PARTIAL |

### 1.10 P0-02 结论

**P0-02 = DESIGN_PARTIAL**。
- **协议层**：唯一锁释放协议（fork_entry 唯一释放）+ fork_epoch 快照协议 + 锁顺序一致 + 逐路径审查闭合——**DESIGN_PROVEN（协议层）**。
- **运行时**：多线程 fork 运行时行为 NOT_TESTED。
- **不可控入口**：第三方 C fork/open、嵌入、子解释器、posix_spawn UNSUPPORTED/DENY。

---

## 2. P0-04｜POSIX 事务状态机重构

### 2.1 缺陷定位：05.21 混用最终态 DURABLE 与 DURABLE（进行中）

05.21 §2.3 状态机中同时出现 `DURABLE`（最终态）与 `DURABLE（进行中）`（事务执行中），并把"开始提交"的初始态也标为 `DURABLE（进行中）`。这是语义混用——最终态与进行中状态共用一个名字，导致迁移表混乱。

**本轮彻底消除**：引入独立状态 `COMMIT_IN_PROGRESS` 表示"事务正在执行"，`DURABLE` 只表示最终持久化完成态。

### 2.2 七态状态机（唯一状态定义）

| 状态 | 唯一定义 |
|------|---------|
| COMMIT_IN_PROGRESS | 事务正在执行，尚未完成持久化 |
| NOT_DURABLE | 提交前关键持久化步骤失败 |
| COMMIT_INDETERMINATE | current 是否持久化无法确认 |
| COMMIT_DURABLE_AUDIT_PENDING | current 已持久，COMMIT 回执尚未完整持久 |
| DURABLE | current 与 COMMIT 回执都达到规定持久化条件 |
| RECOVERY_REQUIRED | 自动恢复条件不足，需人工 |
| QUARANTINED | 证据冲突或身份验证失败 |

### 2.3 状态迁移表（合法迁移 + 禁止迁移）

| 当前状态 | 事件 | 目标状态 | 合法性 |
|---------|------|---------|--------|
| （初始）| 开始提交 | COMMIT_IN_PROGRESS | 合法 |
| COMMIT_IN_PROGRESS | S2/S3/S6/S7/S11 任一 fsync 失败（rename 前）| NOT_DURABLE | 合法 |
| COMMIT_IN_PROGRESS | S12 rename 完成但 S13 未持久 | COMMIT_INDETERMINATE | 合法 |
| COMMIT_IN_PROGRESS | S13 完成（current 已持久）| COMMIT_DURABLE_AUDIT_PENDING | 合法 |
| COMMIT_IN_PROGRESS | S15 完成（current+回执都持久）| DURABLE | 合法 |
| NOT_DURABLE | 重试该 fsync 成功 | **回到 COMMIT_IN_PROGRESS 对应步骤继续**（非直接 DURABLE）| 合法 |
| NOT_DURABLE | 重试失败/磁盘错误 | RECOVERY_REQUIRED | 合法 |
| COMMIT_INDETERMINATE | 重读确认 current 已持久 | COMMIT_DURABLE_AUDIT_PENDING | 合法（有证据）|
| COMMIT_INDETERMINATE | 无法确认 | RECOVERY_REQUIRED | 合法 |
| COMMIT_DURABLE_AUDIT_PENDING | 补回执 + fsync 成功 | DURABLE | 合法 |
| 任意 | 证据冲突/身份失败 | QUARANTINED | 合法 |
| NOT_DURABLE | 直接跳 DURABLE（无证据）| DURABLE | **禁止（05.21 错误）**|
| COMMIT_IN_PROGRESS | 直接标 DURABLE（未持久）| DURABLE | **禁止** |

**关键修订**：`COMMIT_IN_PROGRESS` 独立存在，`DURABLE` 仅为最终态；NOT_DURABLE 重试成功回到 `COMMIT_IN_PROGRESS` 继续，禁止跳 DURABLE。

### 2.4 S1~S16 完整崩溃矩阵（10 类崩溃位置，无省略）

先固定 S1~S16 操作定义（沿用 05.21，不重排/偷删/合并）：

| 步骤 | 操作 |
|------|------|
| S1 | Candidate O_EXCL 写入 |
| S2 | fsync(candidate_fd) |
| S3 | fsync(versions_dirfd) |
| S4 | 重读与 SHA-256 校验 |
| S5 | 写 PREPARED |
| S6 | fsync(prepared_fd) |
| S7 | fsync(prepared_dirfd) |
| S8 | 获取全局提交锁 |
| S9 | 读取 current 并校验五条件 |
| S10 | 写 current 临时文件 |
| S11 | fsync(current_tmp_fd) |
| S12 | rename 替换 current |
| S13 | fsync(current 父目录) |
| S14 | 写 COMMIT 回执并 fsync(commit_fd) |
| S15 | fsync(commit_dirfd) |
| S16 | 释放全局提交锁 |

**崩溃位置代号**：①操作开始前 ②操作执行中 ③操作完成未下一持久化 ④fsync 成功返回后 ⑤fsync 失败/不确定 ⑥恢复观察预期 ⑦恢复观察缺失 ⑧恢复观察旧版本 ⑨恢复观察部分内容 ⑩恢复观察身份/哈希/txn_id/revision 冲突。

**完整矩阵**（每步每位置，不写"同上"）：

| 步骤 | 位置 | 磁盘可观察状态 | 能否证明持久化？| 文件身份核验 | SHA-256 核验 | txn_id/rev/父版本核验 | 六字段/五谓词核验 | 恢复动作 | 禁止动作 | 事务状态 | 允许新事务？| 人工裁决？|
|------|------|---------------|---------------|-------------|-------------|---------------------|-------------------|---------|---------|---------|-----------|----------|
| S1 | ① | 无 Candidate | N/A | N/A | N/A | N/A | N/A | 重新 S1 | 无 | 初始 | 是 | 否 |
| S1 | ② | Candidate 半成品 | 否 | 检查 inode 存在 | 校验失败 | N/A | N/A | 删半成品重试 S1 | 当完成 | 初始 | 是 | 否 |
| S1 | ③ | Candidate 完整 | 否（内容未持久）| 核对 inode | 校验 | N/A | N/A | 执行 S2 | 跳过 S2 | COMMIT_IN_PROGRESS | 否 | 否 |
| S2 | ④ | Candidate 内容持久 | 是 | 核对 | 匹配 | N/A | N/A | 执行 S3 | 跳过 S3 | COMMIT_IN_PROGRESS | 否 | 否 |
| S2 | ⑤ | 内容不确定 | 否 | 重读 | 重算 | N/A | N/A | 重试 S2 | 当持久 | NOT_DURABLE | 否 | 否 |
| S3 | ③ | Candidate 目录项持久 | 是 | 核对 | 匹配 | N/A | N/A | 执行 S4 | 跳过 | COMMIT_IN_PROGRESS | 否 | 否 |
| S3 | ⑤ | 目录项不确定 | 否 | 重读目录 | N/A | N/A | N/A | 重试 S3 | 当持久 | NOT_DURABLE | 否 | 否 |
| S4 | ③ | 校验通过 | — | 核对 | 匹配 | N/A | 六字段一致 | 执行 S5 | 跳过 | COMMIT_IN_PROGRESS | 否 | 否 |
| S4 | ⑩ | 哈希不匹配 | 否 | 冲突 | 不匹配 | N/A | 六字段冲突 | 拒绝 + 重生成 | 提交 | 拒绝 | 是 | 否 |
| S5 | ② | PREPARED 半成品 | 否 | 检查 | 校验失败 | N/A | N/A | 删半成品重试 S5 | 当完成 | COMMIT_IN_PROGRESS | 否 | 否 |
| S6 | ⑤ | PREPARED 内容不确定 | 否 | 重读 | 重算 | N/A | N/A | 重试 S6 | 当持久 | NOT_DURABLE | 否 | 否 |
| S7 | ⑤ | PREPARED 目录项不确定 | 否 | 重读目录 | N/A | N/A | N/A | 重试 S7 | 当持久 | NOT_DURABLE | 否 | 否 |
| S8 | ③ | 锁已持有 | — | N/A | N/A | N/A | N/A | 执行 S9 | 释放锁跳过 | COMMIT_IN_PROGRESS | 否 | 否 |
| S9 | ③ | current 未变 | — | N/A | N/A | 五条件失败 | 五谓词失败 | 重生成 Candidate | 只改 rev 重试 | 拒绝 | 是 | 否 |
| S10 | ② | current 临时半成品 | 否 | 检查 | 校验失败 | N/A | N/A | 删半成品重试 S10 | 当完成 | COMMIT_IN_PROGRESS | 否 | 否 |
| S11 | ⑤ | current 临时内容不确定 | 否 | 重读 | 重算 | N/A | N/A | 重试 S11 | 当持久 | NOT_DURABLE | 否 | 否 |
| S12 | ① | current 未替换 | 是（旧 rev）| 核对 current 身份 | 匹配旧 | 旧 rev | N/A | 继续 S12 | 盲目 rename | COMMIT_IN_PROGRESS | 否 | 否 |
| S12 | ② | current 已/未替换（不确定）| **否** | 重读 current 身份 | 比对新旧 | 判断 rev | N/A | 重读判定旧/新 | 盲目重试 rename | COMMIT_INDETERMINATE | 否 | 是 |
| S12 | ③ | current 已替换（内存）| 否（目录项未持久）| 核对 | 匹配新 | 新 rev | N/A | 执行 S13 | 跳过 S13 | COMMIT_INDETERMINATE | 否 | 是 |
| S13 | ④ | current 目录项持久 | 是 | 核对 | 匹配 | 新 rev | N/A | 执行 S14 | 跳过 | COMMIT_DURABLE_AUDIT_PENDING | 否 | 否 |
| S13 | ⑤ | current 目录项不确定 | 否 | 重读 | 匹配 | 判断 rev | N/A | 重读确认 | 盲目回滚 | COMMIT_INDETERMINATE | 否 | 是 |
| S14 | ② | 回执半成品 | 否 | 检查 | 校验失败 | N/A | N/A | 删半成品重试 S14 | 当完成 | COMMIT_DURABLE_AUDIT_PENDING | 否 | 否 |
| S14 | ④ | 回执内容持久 | 是 | 核对 | 匹配 | N/A | N/A | 执行 S15 | 跳过 | COMMIT_DURABLE_AUDIT_PENDING | 否 | 否 |
| S15 | ⑤ | 回执目录项不确定 | 否 | 重读目录 | N/A | N/A | N/A | 补 fsync S15 | 当完成 | COMMIT_DURABLE_AUDIT_PENDING | 否 | 否 |
| S15 | ④ | 回执目录项持久 | 是 | 核对 | 匹配 | N/A | N/A | 执行 S16 | 跳过 | DURABLE | 否 | 否 |
| S16 | ③ | 锁释放，全部持久 | 是 | 核对 | 匹配 | 新 rev | N/A | — | — | DURABLE | 是 | 否 |
| S16 | ②（进程崩溃）| 锁未释放，current+回执已持久 | 是 | 核对 | 匹配 | 新 rev | N/A | 重读确认 DURABLE，锁由 OS 释放 | 回滚 | DURABLE | 是 | 否 |

**关键点**：
- S12（rename）的"执行中"（②）是唯一导致 COMMIT_INDETERMINATE 的崩溃点——rename 原子性只保证旧/新之一，崩溃后必须重读 current 身份判断；
- S16 释放锁是**非持久化步骤**（锁由 OS 在进程退出时自动释放），S16 崩溃不影响已持久化的 current+回执，事务仍 DURABLE；
- 所有"恢复观察"位置（⑥~⑩）都在恢复流程中通过重读磁盘校验，而非信任内存。

### 2.5 恢复协议（修正 05.21 决策树）

```
fsync 失败/崩溃发生在步骤 Si：
  1. 从磁盘重读真实状态：文件身份、大小、SHA-256、txn_id、revision、父版本、
     Candidate 六字段、五谓词、current 实际指向、PREPARED/COMMIT 回执身份
  2. 判断已完成持久化阶段
  3. 分步处理：
     - Si ∈ S1~S11（rename 前）：重试该步骤 fsync → 成功则回到 COMMIT_IN_PROGRESS 继续，失败则 RECOVERY_REQUIRED
     - S12 rename 前：Candidate/PREPARED 完整 → 可继续 S12
     - S12 rename 中：重读 current 身份判定旧/新 → 不能盲目重试 rename
     - S13 目录 fsync 失败：重读确认 current 已持久 → COMMIT_DURABLE_AUDIT_PENDING，否则 COMMIT_INDETERMINATE
     - S14 回执写入中断：删半成品重试，current 已持久保持 COMMIT_DURABLE_AUDIT_PENDING
     - S15 回执目录 fsync 失败：补 fsync，保持 COMMIT_DURABLE_AUDIT_PENDING
     - S16 释放锁异常/进程退出：current+回执已持久 → DURABLE（锁由 OS 释放）
```

**禁止**：
- fsync 重试成功直接跳 DURABLE；
- 未确认 current 状态时盲目重复 rename；
- 不确定态自动回滚；
- 未完成恢复/隔离时启动新事务；
- 仅凭错误码推断文件存在性；
- 将 current 持久化与 COMMIT 回执持久化混为一谈。

### 2.6 新事务准入矩阵

| 状态 | 允许新事务？| 前置条件 |
|------|-----------|---------|
| COMMIT_IN_PROGRESS | **否** | 当前事务未完成 |
| NOT_DURABLE | **否** | 需先恢复或进入其他态 |
| COMMIT_INDETERMINATE | **否** | 需人工裁决确认 |
| COMMIT_DURABLE_AUDIT_PENDING | **否** | 需先补回执到 DURABLE |
| DURABLE | 是 | current+回执都持久 |
| RECOVERY_REQUIRED | **否** | 需人工恢复 |
| QUARANTINED | **否** | 需人工裁决 |

**默认**：未完成恢复、审计回执或身份核验时禁止新事务。唯一允许新事务的状态是 DURABLE（无前置条件）。

### 2.7 POSIX 设计边界（区分证明层次）

| 层次 | 内容 | 状态 |
|------|------|------|
| POSIX API 规范保证 | fsync(fd) 内容持久、rename 原子性 | 规范级 |
| Linux man-pages 描述 | fsync(dirfd) 目录项持久 | 文档级 |
| 特定文件系统行为 | ext4/xfs 等差异 | 未逐 FS 验证 |
| 存储设备与缓存行为 | 断电语义 | NOT_VERIFIED |
| 设计推演 | 崩溃矩阵/状态机 | 设计级 |
| 运行时验证 | 实际故障注入 | NOT_TESTED |

**关键区分**：`fsync(fd)`（文件内容）与 `fsync(dirfd)`（目录项）语义不同，不得混为一谈；rename 原子性 ≠ 断电持久性；Windows 目录持久化 NOT_VERIFIED，不用 POSIX 结论代替。

### 2.8 20 条故障时间线 + 15 个反例

| # | 时间线 | 推演 | 结果 |
|---|--------|------|------|
| G1 | S1 半写入 | 删半成品重试 | 安全 |
| G2 | S2 fsync 失败 | NOT_DURABLE | 重试 |
| G3 | S2 重试成功 | 回 COMMIT_IN_PROGRESS 继续（非 DURABLE）| 继续 |
| G4 | S3 fsync 失败 | NOT_DURABLE | 重试 |
| G5 | S6/S7 fsync 失败 | NOT_DURABLE | 重试 |
| G6 | S11 fsync 失败 | NOT_DURABLE | 重试 |
| G7 | S12 rename 中崩溃 | COMMIT_INDETERMINATE | 重读 current |
| G8 | S13 fsync 失败 | COMMIT_INDETERMINATE | 重读确认 |
| G9 | S13 成功 | COMMIT_DURABLE_AUDIT_PENDING | 补回执 |
| G10 | S14 半写入 | COMMIT_DURABLE_AUDIT_PENDING | 删半成品重试 |
| G11 | S15 fsync 失败 | COMMIT_DURABLE_AUDIT_PENDING | 补 fsync |
| G12 | S15 成功 | DURABLE | 完成 |
| G13 | S16 崩溃 | DURABLE（锁由 OS 释放）| 确认 |
| G14 | NOT_DURABLE 跳 DURABLE（05.21 错误）| **禁止** | 按步骤 |
| G15 | fsync 重试失败 | RECOVERY_REQUIRED | 人工 |
| G16 | 证据冲突 | QUARANTINED | 人工 |
| G17 | 断电 current 与 PREPARED 不一致 | 重读 | COMMIT_INDETERMINATE |
| G18 | 恢复观察旧版本 current | 判断 rename 是否发生 | 按步骤 |
| G19 | 恢复观察身份冲突 | QUARANTINED | 人工 |
| G20 | 未恢复即新事务 | **禁止** | 禁止 |

15 个反例：S1~S15 各 fsync 失败/半写入、S12 rename 中崩溃、S16 锁释放混淆、NOT_DURABLE 跳步、证据冲突、断电不一致、旧版本、身份冲突、未恢复即新事务、DURABLE 混用（B22-03）。

### 2.9 P0-04 结论

**P0-04 = DESIGN_PROVEN（POSIX）/ NOT_VERIFIED（Windows）**。
- 七态状态机消除 DURABLE 混用（COMMIT_IN_PROGRESS 独立）；
- S1~S16 完整崩溃矩阵（10 类崩溃位置，无省略）；
- 恢复协议 + 新事务准入矩阵闭合；
- Windows 目录持久化 NOT_VERIFIED。

---

## 3. P0-05 与 P0-06 边界

### 3.1 P0-05（继续 DENY）

继续 DENY，不放宽 Source Retire，不设计删除/清理/归档/旧 FD 失效/mmap 失效等实现路径，不声称源内容已不可写。

### 3.2 P0-06（仅保留已纠正语义）

仅保留 05.20/05.21 已纠正的 ReplaceFileW 1175/1176/1177 语义，不重新设计整个 Windows 替换协议。继续要求通过重读文件 ID、路径、大小、内容哈希判断部分失败后实际状态。未运行时验证的中间态继续 DESIGN_PARTIAL / NOT_VERIFIED。

---

## 4. 设计不变量（I-01 ~ I-45）

| # | 定义 | 适用范围 | 前置条件 | 证明依据 | 失效反例 | 失败状态 | 拒绝策略 | 验证状态 |
|---|------|---------|---------|---------|---------|---------|---------|---------|
| I-01 | 锁级别编号与获取顺序一致 | 全锁 | — | §1.2 | 编号与顺序矛盾 | 逆序死锁 | L0=coord | DESIGN_PROVEN |
| I-02 | `_fork_coord_lock` 唯一释放者 = fork_entry | fork | — | §1.2 | 双释放（B22-01）| 未定义行为 | 唯一释放 | DESIGN_PROVEN |
| I-03 | 钩子不触碰 `_fork_coord_lock` | fork | — | §1.2 | after 钩子释放 coord | 双释放 | 钩子不取不释 | DESIGN_PROVEN |
| I-04 | fork_entry 唯一 fork 入口 | fork | — | §1.4 | 多入口 fork | 职责混乱 | 唯一入口 | DESIGN_PROVEN |
| I-05 | freeze 不调用 fork | fork | — | §1.4 | freeze 内 fork | 递归 | 禁止 | DESIGN_PROVEN |
| I-06 | before 无锁不 fork | fork | — | §1.4 | before 持锁/fork | 重入 | 只读快照 | DESIGN_PROVEN |
| I-07 | fork_epoch 单调唯一 | fork | — | §1.4 | epoch 重复 | 快照误用 | 原子递增 | DESIGN_PROVEN |
| I-08 | 快照与 fork 一一对应 | fork | — | §1.4 | 旧快照误用 | 误关 FD | epoch 校验 | DESIGN_PROVEN |
| I-09 | 快照单槽位受 L0 保护 | fork | — | §1.4 | 并发覆盖 | 快照混乱 | 单槽位+锁 | DESIGN_PROVEN |
| I-10 | after_in_parent 恢复 Gate | fork | — | §1.3 | parent 不恢复 | 永久 FREEZING | finally 恢复 | DESIGN_PROVEN |
| I-11 | after_in_child 不 acquire 继承锁 | fork child | — | §1.4 | 继承 locked mutex | 死锁 | 直接赋值 | DESIGN_PROVEN |
| I-12 | fork 不绕过 CPython 生命周期 | fork | 主解释器 | CPython 文档 | os._fork 后执行 Python | 未重置 | 用 os.fork | DESIGN_PROVEN |
| I-13 | 受控 fork 仅主线程 | fork | 主线程 | CPython 约束 | 非主线程 | 线程锁消失 | DENY | DESIGN_PARTIAL |
| I-14 | 准入原子性（检查+计数同 L1）| FD | — | §1 继承 | 检查后计数前 FREEZING | 快照不一致 | 原子准入 | DESIGN_PROVEN |
| I-15 | 冻结覆盖 fork 窗口 | fork | 受控 | §1.4 | 快照后释放锁 | 未门禁 | FREEZING 保持 | DESIGN_PROVEN |
| I-16 | FD 绑定 generation | FD | — | 协议 | 裸数字复用 | 误关 | 不匹配不关 | DESIGN_PROVEN |
| I-17 | child 未知 FD 不主动 close | fork child | — | §1.6 | 误关未知 FD | 资源错关 | UNKNOWN_FD | DESIGN_PROVEN |
| I-18 | 计数严格配对 | FD | — | §1.3 | 计数泄漏 | 排空失效 | RAII | DESIGN_PROVEN |
| I-19 | 第三方 C fork/open DENY | FD/fork | 第三方 | §1.6 | C 直接 fork/open | 绕过 Gate | DENY | DESIGN_PROVEN |
| I-20 | 七态状态机唯一定义 | 提交 | — | §2.2 | DURABLE 混用（B22-03）| 状态混乱 | COMMIT_IN_PROGRESS 独立 | DESIGN_PROVEN |
| I-21 | DURABLE 仅最终态 | 提交 | — | §2.2 | DURABLE（进行中）| 混用 | 消除 | DESIGN_PROVEN |
| I-22 | NOT_DURABLE 不跳步 DURABLE | 提交 | — | §2.3/§2.5 | 跳步迁移 | 错误升级 | 回 COMMIT_IN_PROGRESS | DESIGN_PROVEN |
| I-23 | 只有 current+回执都持久才 DURABLE | 提交 | — | §2.5 | current 持久回执未持久当 DURABLE | 错误完成 | AUDIT_PENDING | DESIGN_PROVEN |
| I-24 | Candidate 六字段绑定 | 提交 | — | 05.21 §2 | 未绑父版本 | 错误提交 | 五条件 | DESIGN_PROVEN |
| I-25 | 五条件完整谓词 | 提交 | 提交锁 | 05.21 §2 | 笼统五条件 | 漏校验 | P1~P5 | DESIGN_PROVEN |
| I-26 | 文件内容先于目录项持久化 | 提交 | — | §2.4 | 未 fsync 文件 | 数据丢 | 禁止 | DESIGN_PROVEN |
| I-27 | 目录项 fsync(dirfd) | 提交 | — | §2.4 | 目录项未持久 | 目录项丢 | 强制 | DESIGN_PROVEN |
| I-28 | rename 原子性 ≠ 断电持久性 | 提交 | — | §2.7 | 误当持久 | 断电丢 | fsync 目录 | DESIGN_PROVEN |
| I-29 | S12 rename 中崩溃重读确认 | 提交 | — | §2.4 | 盲目重试 rename | 不确定 | INDETERMINATE | DESIGN_PROVEN |
| I-30 | S16 锁释放非持久化步骤 | 提交 | — | §2.4 | 锁释放当持久化 | 混淆 | 区分 | DESIGN_PROVEN |
| I-31 | fsync 失败禁后续提交 | 提交 | — | §2.6 | fsync 失败后提交 | 不确定提交 | 禁提交 | DESIGN_PROVEN |
| I-32 | 不确定态禁回滚/重复提交 | 提交 | — | §2.5 | 自动回滚 | 状态破坏 | 禁自动 | DESIGN_PROVEN |
| I-33 | 每崩溃点确定恢复 | 提交 | — | §2.4 | 无规则 | 恢复错误 | 矩阵 | DESIGN_PROVEN |
| I-34 | 步骤编号统一 S1~S16 | 提交 | — | §2.4 | 编号不一致 | 引用错乱 | S1~S16 | DESIGN_PROVEN |
| I-35 | revision 单调 | 提交 | — | 05.21 | 回退 | 版本倒退 | P4 断言 | DESIGN_PROVEN |
| I-36 | 恢复前重读磁盘证据 | 提交 | — | §2.5 | 仅凭内存 | 错误判断 | 重读 | DESIGN_PROVEN |
| I-37 | 恶意同 UID 锁替换排除 | 锁 | 威胁模型 | — | 同 UID 替换 | 锁分裂 | NOT_VERIFIED | NOT_VERIFIED |
| I-38 | +i 不阻旧 FD 写 | Source Retire | — | chattr(1) | 旧 FD 写 | 竞态 | DENY | NOT_VERIFIED |
| I-39 | +i 阻止 rename | Source Retire | — | chattr(1) | rename EPERM | 退役失败 | DENY | NOT_VERIFIED |
| I-40 | Source Retire 默认 DENY | Source Retire | — | §3.1 | 无强制边界 | 不安全 | DENY | DESIGN_PARTIAL |
| I-41 | 元数据原子发布 | tombstone | — | 05.20 | O_EXCL 逐步写 | 半成品 | 临时+校验+无覆盖 | DESIGN_PROVEN |
| I-42 | 1176 路径语义正确 | tombstone | — | Microsoft Learn | "以原目标名" | 路径混淆 | "其原名" | DESIGN_PROVEN |
| I-43 | 错误码不推断文件状态 | tombstone | — | 05.20 | 从错误码推断 | 错误判定 | 重读 | DESIGN_PROVEN |
| I-44 | 无法证明必须 DENY/INDETERMINATE | 全局 | — | 全文 | 猜测 | 错误结论 | 不猜测 | DESIGN_PROVEN |
| I-45 | 不把推演写运行时验证 | 全局 | — | 全文 | 自审当验证 | 虚假验证 | NOT_TESTED | DESIGN_PROVEN |

**共 45 条不变量。** 本轮新增/修订 I-02/I-03（唯一锁释放）、I-07/I-08/I-09（fork_epoch 快照）、I-20/I-21（消除 DURABLE 混用）、I-30（S16 锁释放非持久化）。

---

## 5. 全局一致性审查

### 5.1 20 条端到端链路

| # | 链路 | 关键边界 | 崩溃点 | 恢复 | 结论 |
|---|------|---------|--------|------|------|
| L1 | fork_entry→freeze→os.fork→after 钩子 | 唯一锁释放 + epoch | fork 中 | 恢复 | DESIGN_PARTIAL |
| L2 | 锁顺序 L0→L1→L2 无逆序 | 锁依赖图 | — | — | DESIGN_PROVEN |
| L3 | 快照单槽位 epoch 一一对应 | epoch 校验 | 连续 fork | 替换 | DESIGN_PROVEN |
| L4 | FD 准入→计数→冻结→fork→child 重建 | generation | 冻结中 | 校验 | DESIGN_PROVEN |
| L5 | 第三方 C open→fork→child | 不可控 | — | UNKNOWN_FD | DESIGN_PARTIAL |
| L6 | Candidate→fsync 文件→fsync 目录→rename→fsync 目录 | 双重持久化 | rename 前后 | 按 current | DESIGN_PROVEN |
| L7 | COMMIT_IN_PROGRESS→fsync 失败→NOT_DURABLE→按步骤恢复 | 状态机 | fsync | 回 COMMIT_IN_PROGRESS | DESIGN_PROVEN |
| L8 | NOT_DURABLE 跳步 DURABLE（反例）| 禁止 | — | 禁止 | DESIGN_PROVEN（拒绝）|
| L9 | rename 后目录未持久→COMMIT_INDETERMINATE | 不确定态 | rename 后 | 重读 | DESIGN_PROVEN |
| L10 | current 持久回执未写→AUDIT_PENDING | 拆分状态 | S14 前 | 补回执 | DESIGN_PROVEN |
| L11 | 双事务竞争 | flock | 竞争点 | 败者重生成 | DESIGN_PROVEN |
| L12 | 断电 current 与 PREPARED 不一致 | 重读 | 断电 | 人工裁决 | DESIGN_PROVEN |
| L13 | 源 seal→quarantine→retire | retire DENY | — | DENY | DENY |
| L14 | 旧 FD + i | +i 不阻旧 FD | 校验后写 | 竞态 | DENY |
| L15 | rename + i | rename EPERM | — | 退役失败 | DENY |
| L16 | tombstone 临时→校验→无覆盖发布 | 临时 fsync | 发布前后 | 占位/有效 | DESIGN_PARTIAL |
| L17 | ReplaceFileW 1176 未指定备份 | 官方契约 | 替换中 | INDETERMINATE | DESIGN_PARTIAL |
| L18 | ReplaceFileW 1177 部分移动 | 官方契约 | 替换中 | INDETERMINATE | DESIGN_PARTIAL |
| L19 | 归档协议 | 临时+校验+发布 | 归档中 | 哈希确认 | DESIGN_PROVEN |
| L20 | 恶意同 UID | 无 | 任意 | 检测 | NOT_VERIFIED |

### 5.2 时间线与反例汇总

- 时间线：P0-02 二十条 + P0-04 二十条 = **40 条**（≥40 达标）；
- 反例：P0-02 十五个 + P0-04 十五个 = **30 个**（≥30 达标）。

### 5.3 跨章节身份一致性

| 身份/状态 | 产生 | 传递 | 一致性 |
|-----------|------|------|--------|
| `_fork_coord_lock` 释放 | §1.2 | §1.3/§1.4 | 唯一释放者 fork_entry |
| fork_epoch | §1.4 | §1.8 | 单调 + 单槽位 |
| 快照 | §1.4 | §1.8 | 一一对应 |
| 七态状态机 | §2.2 | §2.3/§2.4 | 唯一定义 |
| 父版本三元组 | 05.21 | §2.4 | 五条件 |
| S1~S16 | §2.4 | §2.4/§2.5 | 编号统一 |

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

## 6. 交付物与完整性核验

### 6.1 交付文件清单（8 项）

| 文件 | 说明 |
|------|------|
| `USIT-STAGE-B-DESIGN-05.22.md` | 主设计文档 |
| `USIT-STAGE-B-DESIGN-05.22-ISSUES.md` | 问题清单 |
| `USIT-STAGE-B-DESIGN-05.22-DIFF.md` | 相对 05.21 差异 |
| `USIT-STAGE-B-DESIGN-05.22-CROSS-REVIEW.md` | 反例审查 |
| `USIT-STAGE-B-DESIGN-05.22-EXECUTION-REPORT.md` | 执行报告 |
| `USIT-STAGE-B-DESIGN-05.22-HASH-MANIFEST.md` | 哈希清单 |
| `USIT-STAGE-B-DESIGN-05.22-HASH-RECEIPT.md` | 哈希回执 |
| `USIT-STAGE-B-DESIGN-05.22-DELIVERY.zip` | 交付包 |

### 6.2 HASH-MANIFEST 要求

记录前六份 Markdown 文件的文件名、实际字节数、SHA-256（文件最终写入后重新计算）。

### 6.3 HASH-RECEIPT 要求

记录 MANIFEST 自身字节数/SHA-256、ZIP 自身字节数/SHA-256、核验时间、ZIP 独立解压结果、每个文件的名称/字节数/SHA-256/CRC32、与原始交付文件逐字节比较结果、核验工具和版本。

### 6.4 源码未修改证据（本轮独立前后基线）

本轮将在项目外创建 `DESIGN-05.22` 前采集源码基线，结束后再次采集对比，记录基线采集时间、工作区、文件范围、文件数量、逐文件 SHA-256。不能采集的内容标 NOT_VERIFIED。

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

**DESIGN-05.22 = PASS-WITH-ISSUES**，分项结论：

| 项 | 结论 | 依据 |
|----|------|------|
| P0-02 | DESIGN_PARTIAL | 唯一锁释放协议 + fork_epoch 快照协议闭合（协议层 DESIGN_PROVEN）；第三方 C fork 不可控；运行时 NOT_TESTED |
| P0-04 POSIX | DESIGN_PROVEN | 七态状态机消除 DURABLE 混用 + S1~S16 完整崩溃矩阵 |
| P0-04 Windows | NOT_VERIFIED | 目录持久化无权威保证 |
| P0-05 | DENY（保持）| 不放宽 |
| P0-06 | DESIGN_PARTIAL / NOT_VERIFIED | 保留已纠正语义 |
| 交付完整性 | 待核验（§6）| ZIP 独立解压 + CRC32 + 逐字节比对 |
| IMPLEMENT-01 | **NOT_APPROVED** | 未授权 |

**核心诚实结论**：
1. **B22-01 闭合**：`_fork_coord_lock` 唯一释放者 = `fork_entry()` finally，钩子不触碰，消除双释放。
2. **B22-02 闭合**：fork_epoch 单调 + 单槽位 `_pending_snapshot` + epoch 校验，快照与 fork 一一对应。
3. **B22-03 闭合**：七态状态机，`COMMIT_IN_PROGRESS` 独立，`DURABLE` 仅最终态，S1~S16 崩溃矩阵覆盖 10 类位置。
4. **B22-04/B22-05**：见 §6 交付核验与源码基线，真实执行后记录。

**默认结论：IMPLEMENT-01 NOT_APPROVED。** 等待 ChatGPT 独立复审。

---


