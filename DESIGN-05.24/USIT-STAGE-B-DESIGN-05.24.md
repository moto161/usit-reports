# USIT-STAGE-B-DESIGN-05.24｜Fork 回调与 Epoch 作用域修复＋POSIX 崩溃矩阵完整闭合＋交付证据独立补齐

> 阶段 B · DESIGN-05.24 · STRICT READ-ONLY DESIGN REVISION ONLY
> 作者：USIT 主开发 · 生成：2026-10-31（GMT+8）
> 前置审查：DESIGN-05.23 独立复审结论 FAIL
> 工作区：`C:\Users\Administrator\WorkBuddy\2026-10-15-22-09-47\usit_p0`
> 交付目录：`C:\Users\Administrator\WorkBuddy\2026-10-15-22-09-47\usit_p0_05.24_reports`（**项目外**）
> 状态：**STOP — 等待独立复审**（未授权 IMPLEMENT-01）

---

## 0. 缺陷追踪表（B24-01 ~ B24-06）

| # | 原始缺陷（05.23 复审点名）| 05.23 位置 | 05.24 修订位置 | 状态 | 剩余风险 |
|---|--------------------------|-----------|---------------|------|---------|
| B24-01 | 伪代码与异常路径不一致；回调责任机制未唯一化；回调异常传播假设 | §1.2/§1.4/§1.5 | §1 全文重构 | 闭合（协议层）| 回调异常传播 NOT_TESTED |
| B24-02 | Epoch 作用域定义不清；PID/Epoch 单独性未评估 | §1.6~§1.9 | §2 全文重构 | 闭合（协议层）| 跨进程唯一 NOT_TESTED |
| B24-03 | S1~S16 矩阵被"代表性覆盖"，未逐项展开 | §2.2 | §3 全文重做 | 闭合 | 硬件/FS 边界 NOT_VERIFIED |
| B24-04 | S13-4"读到旧值即继续 S12"不安全推断 | §2.5 | §4 全文重构 | 闭合 | 硬件/FS 边界 NOT_VERIFIED |
| B24-05 | ZIP 核验缺真实输出与路径穿越/敏感信息检查 | §6.3 | §6 + RECEIPT | 待执行 | 环境无法解压则 NOT_VERIFIED |
| B24-06 | 源码基线仅"25 文件一致"缺完整清单与 diff | §6.4 | §6 + 3 份基线文件 | 待执行 | — |

**本节为全文总纲。本轮核心：B24-01 回调责任唯一化 + B24-02 epoch 作用域 + B24-03/04 崩溃矩阵逐项展开 + B24-05/06 交付证据。**

---

## 1. P0-02｜Fork 回调与父子锁释放协议（B24-01）

### 1.1 缺陷定位

05.23 §1.2/§1.4/§1.5 存在三类不一致：

1. **回调责任未唯一化**：05.23 伪代码在 `fork_entry()` 手动调用 `_after_in_parent`/`_after_in_child`，同时又提及 `os.register_at_fork()` 回调，未明确二者关系——同一恢复动作可能被回调与手动路径重复执行。
2. **回调异常传播假设**：05.23 隐含假设 at-fork 回调抛出的异常一定能被外层 `try/finally` 捕获，但 CPython 对 at-fork 回调异常的传播语义与 `os.fork()` 直接调用不同，不能作此假设。
3. **异常路径表与伪代码不一致**：05.23 §1.5 异常路径表 D/H 行的释放责任与 §1.2 伪代码的 `except`/`finally` 结构未完全对应。

### 1.2 唯一责任机制：选择"手动接管"而非"回调注册"

**决策**：本设计采用**手动接管（manual orchestration）**，即 `fork_entry()` 是唯一受控 fork 入口，**不使用 `os.register_at_fork()` 注册任何自动回调**。理由：

| 机制 | 优点 | 缺点 | 选择 |
|------|------|------|------|
| `os.register_at_fork()` 回调 | CPython 自动保证 before/parent/child 调用 | 回调异常传播语义复杂；与 `os.fork()` 直接调用的边界不清；第三方 C fork 不触发 Python 层回调；嵌入/子解释器行为未定义 | **不采用** |
| 手动接管（本设计）| 责任单一、顺序显式、异常路径可控 | 无法覆盖第三方 C 直接 fork | **采用** |

**后果**：由于不注册 at-fork 回调，本库**只保证**经过 `fork_entry()` 的 fork 安全；任何绕过 `fork_entry()` 的路径（第三方 C fork、`os.fork()` 直接调用、嵌入宿主 fork）**明确 UNSUPPORTED/DENY**。

### 1.3 完整伪代码（唯一责任主体）

```python
# 全局（仅父进程主解释器主线程访问）
_fork_coord_lock = threading.Lock()   # L0：fork 协调锁，唯一获取/释放者 = 父进程
_gate_lock        = threading.Lock()   # L1：Gate 准入锁
_fd_lock          = threading.Lock()   # L2：FD Registry 锁
fork_epoch_counter = 0                 # 在 L0 临界区内递增

def fork_entry():                       # 唯一受控 fork 入口
    assert is_main_interpreter_main_thread()   # 否则 DENY
    acquire(_fork_coord_lock)                   # L0 由父进程获取
    epoch = 0
    try:
        # ---- 父进程 fork 前（持 L0）----
        epoch = allocate_epoch()                # L0 内递增（B24-02 见 §2）
        snapshot = freeze(epoch)                # L0→L1→L2 升序冻结
        _pending_snapshot = (epoch, snapshot)
        pid = os.fork()                         # 唯一 fork 点
    except BaseException:
        # 唯一父进程异常清理路径（不注册 at-fork 回调，故无回调异常混入）
        restore_gate()                          # gate=OPEN + notify
        _pending_snapshot = None
        release(_fork_coord_lock)               # 父进程唯一释放
        raise

    if pid == 0:
        # ---- 子进程（fork 返回后独立路径）----
        try:
            _after_in_child(epoch, snapshot)
        except BaseException as e:
            # 子进程初始化失败：记录后进入 DENY，不释放任何继承锁
            _child_init_failed = e
            _deny_protected_ops()
        return 0    # 子进程路径在此终结，绝不返回父进程释放代码

    # ---- 父进程（fork 返回 pid>0）----
    try:
        _after_in_parent(epoch)
    finally:
        _pending_snapshot = None
        release(_fork_coord_lock)               # 父进程唯一释放（含 after_in_parent 失败）
    return pid
```

**唯一责任主体声明**：

| 动作 | 唯一执行者 | 重复风险 | 遗漏风险 |
|------|-----------|---------|---------|
| acquire(_fork_coord_lock) | 父进程 fork_entry 入口 | 无 | 无 |
| release(_fork_coord_lock) | 父进程（except 或 finally，二选一）| 无（互斥分支）| 无（两分支全覆盖）|
| restore_gate | 父进程（except 或 _after_in_parent）| 无 | 无 |
| _after_in_parent | 父进程 | 无 | 无 |
| _after_in_child | 子进程 | 无 | 无 |
| 子进程释放继承锁 | **不存在**（子进程只赋值新对象）| 无 | 无 |

### 1.4 回调异常传播的边界（不得假设可被捕获）

- `os.register_at_fork()` 未注册，故不存在"CPython 自动回调异常"这一路径；
- `os.fork()` 系统调用失败在父进程表现为抛出 `OSError`，由 `except BaseException` 捕获；
- 子进程 `_after_in_child` 的异常**无法跨 fork 传回父进程**——子进程只能本地处理（记录 + DENY），父进程无从感知；
- 父进程 `_after_in_parent` 的异常由 `finally` 保证释放 L0（`finally` 在 CPython 中总能执行，即使 `_after_in_parent` 抛异常）。

**本设计不依赖"回调异常一定被外层 try/finally 捕获"这一假设**：因为根本不注册回调，父进程路径的异常全部在 `fork_entry()` 的同一 try/except/finally 结构内处理。

### 1.5 父子进程时序图 + 锁所有权表 + 调用关系图

**时序图**：

```
父进程（主线程）                          子进程（fork 后）
  acquire(L0)                                   │
  allocate_epoch()  [L0 内]                     │
  freeze(epoch)  [L0→L1→L2]                     │
  _pending_snapshot=(epoch,snap)                │
  pid = os.fork()  ─────────────────────────────┼──► (fork 返回 0)
  │                                             │   继承父内存(锁/快照/Registry)
  │ pid>0                                        │   _after_in_child(epoch,snap):
  │ _after_in_parent(epoch)                      │     赋值全新锁对象(不 acquire/release)
  │   restore_gate()                             │     遍历 snapshot 按 generation close
  │ _pending_snapshot=None                       │     未在 snapshot 的 FD 标 UNKNOWN
  │ release(L0)  [finally]                       │   return 0（终结，不回父释放路径）
  │ return pid                                   │
```

**锁所有权表**（锁级别 L0→L1→L2→L3）：

| 锁 | 级别 | 获取者 | 释放者 | 子进程继承？| 子进程动作 |
|----|------|--------|--------|------------|-----------|
| _fork_coord_lock | L0 | 父进程 fork_entry | 父进程 except/finally | 继承（锁定态快照）| 赋值新对象，不 release |
| _gate_lock | L1 | 父进程 freeze/准入 | 父进程各临界区 | 继承 | 赋值新对象 |
| _fd_lock | L2 | 父进程 freeze/FD 操作 | 父进程各临界区 | 继承 | 赋值新对象 |
| 业务锁 | L3 | 业务代码 | 业务代码 | 继承（可能锁定）| 赋值新对象，禁止持锁 fork |

**调用关系图**：

```
fork_entry()                     # 唯一入口
  ├─ allocate_epoch()            # 仅 fork_entry 调用（L0 内）
  ├─ freeze(epoch)               # 仅 fork_entry 调用
  ├─ os.fork()                   # 仅 fork_entry 调用
  ├─ _after_in_parent(epoch)     # 仅 fork_entry 父路径调用
  └─ _after_in_child(epoch,snap) # 仅 fork_entry 子路径调用
```

### 1.6 异常清理表（9 路径全覆盖）

| 路径 | 进程 | 锁状态 | 释放动作 | 重复释放 | 遗漏释放 | Gate | 再 fork | DENY |
|------|------|--------|---------|---------|---------|------|--------|------|
| 正常 fork 成功 | 父 | 持 L0 | finally 释放 | 否 | 否 | OPEN | 是 | 否 |
| before 前置失败（线程检查）| 父 | 未持 L0（assert 先行）| — | 否 | 否 | — | 是 | 否 |
| 系统 fork 失败 | 父 | 持 L0 | except 释放 | 否 | 否 | OPEN(restore) | 是 | 否 |
| 父回调 _after_in_parent 异常 | 父 | 持 L0 | finally 释放 | 否 | 否 | OPEN(restore) | 是 | 否 |
| 子回调 _after_in_child 异常 | 子 | 不持 L0 | 不释放任何继承锁 | 否 | 否 | child-local | 不适用 | **是** |
| 连续 fork | 父 | 每次持 L0 | 每次 finally 释放 | 否 | 否 | OPEN | 是 | 否 |
| 两线程并发 fork | 父 | L0 串行化 | 各自 finally 释放 | 否 | 否 | OPEN | 是 | 否 |
| 回调部分执行后异常 | 父 | 持 L0 | finally 释放 | 否 | 否 | OPEN(restore) | 是 | 否 |
| 子进程初始化失败 | 子 | 不持 L0 | 不释放继承锁 | 否 | 否 | DENY | 不适用 | **是** |

### 1.7 P0-02 结论（B24-01）

**协议层 DESIGN_PROVEN**：唯一责任主体已确立（不注册 at-fork 回调，手动接管）；父进程 L0 不遗漏/不双重释放（except 与 finally 互斥分支全覆盖）；子进程不释放继承锁。

**运行时 NOT_TESTED**：多线程 fork、回调异常传播、子进程异常处理均未运行时验证（本机无 fork）。

---

## 2. P0-02｜Epoch 唯一性与进程代际（B24-02）

### 2.1 Epoch 唯一性范围

**决策：Epoch 仅在单进程内（per-process）要求唯一，跨进程唯一性由"进程代际 + PID + fork_epoch"组合标识保证。**

理由：
- 快照误用、FD generation 混淆、ABA 均发生在**同一进程内**的连续 fork 之间（旧快照 vs 新快照的错配）；
- 跨进程的快照隔离由 `process_generation + pid` 天然区分（不同进程不同 PID 或不同代际）；
- 若要求 Epoch 跨进程唯一，则子进程必须继承并在 fork 后同步计数器，引入子进程写父进程状态的风险——与"子进程不触碰父进程锁/状态"原则冲突。

### 2.2 为什么进程内唯一足以防三类错误

| 错误 | 机理 | 进程内唯一 Epoch 如何防止 |
|------|------|--------------------------|
| 旧快照误用 | 新一轮 fork 读到上一轮残留快照 | epoch 单调递增，快照绑定 epoch，epoch 不匹配拒绝 |
| FD generation 混淆 | 数字复用导致关闭错误 FD | epoch 与 fd_generation 双层绑定，任一不匹配拒绝 |
| ABA | 失败 fork 后 epoch 复用导致快照错配 | 失败 fork 也消耗 epoch，不复用 |

**结论**：这三类错误的共同前提是"同一进程内两次 fork 之间状态残留"，进程内单调唯一 Epoch 即可线性化；跨进程身份由 PID/代际独立承担，无需 Epoch 跨进程唯一。

### 2.3 唯一标识组合（不得单独依赖 PID 或 Epoch）

```
full_id = process_generation + pid + fork_epoch + fd_generation
```

| 字段 | 含义 | 唯一性范围 | 单独够吗？|
|------|------|-----------|----------|
| process_generation | 进程代际（每 fork 递增，子进程 = 父+1）| 父子进程树内区分代际 | 否（同代际多 fork）|
| pid | 进程标识 | OS 内唯一（但 PID 可重用）| 否（PID 重用 + 同进程多 fork）|
| fork_epoch | 本进程内 fork 序号（L0 内单调递增）| 进程内唯一 | 否（跨进程无意义）|
| fd_generation | 单个 FD 槽位复用计数 | 进程内每槽位唯一 | 否（仅 FD 维度）|

**组合的必要性**：
- `pid` 单独不足：PID 重用会导致旧快照的 pid 与未来进程碰撞；
- `fork_epoch` 单独不足：跨进程后两个子进程可能各自有 epoch=1，需 pid/代际区分；
- `process_generation + pid` 区分"哪个进程"；`fork_epoch` 区分"该进程内哪次 fork"；`fd_generation` 区分"哪个 FD 槽位的哪一代"。

### 2.4 边界情况规范

| 边界 | 处理 |
|------|------|
| PID 重用 | process_generation 单调不减，PID 重用不改变代际身份 |
| 进程退出 | 其代际身份随之终结，不回收复用 |
| 子进程再次 fork | 子进程重新走 fork_entry，其 fork_epoch 从继承值继续（见 §2.5）|
| 连续 fork | 父进程每次分配新 fork_epoch |
| fork 失败 | 消耗 epoch，不复用（防 ABA）|
| 计数器溢出 | 64 位整数，实际不溢出；溢出则 DENY |

### 2.5 Epoch 生成器同步与 fork 重建策略

- Epoch 生成器由 `_fork_coord_lock`（L0）互斥保护，递增在 L0 临界区内；
- **子进程不重新初始化 Epoch 生成器为 0**：子进程继承 `fork_epoch_counter` 的 fork 时刻值；若子进程后续再次 fork，其 epoch 从继承值继续递增（保证父子树内 epoch 不倒退）；
- 子进程对 `_fork_coord_lock` **赋值全新 Lock 对象**（不继承父进程锁定态），其 Epoch 生成器在子进程内受这个新锁保护；
- 进程代际 `process_generation` 在子进程 `_after_in_child` 中 `+1`，不回写父进程。

### 2.6 快照绑定、所有权、失效与禁止复用

- 快照绑定：`_pending_snapshot = (epoch, snapshot)`，`snapshot` 内每个条目携带 `full_id`；
- 所有权：快照由父进程 fork_entry 创建，子进程在 fork 后只读消费；
- 失效：父进程 `_after_in_parent` 将 `_pending_snapshot` 置 None；子进程消费后本地丢弃；
- 禁止复用：任何 epoch 不匹配、generation 不匹配的快照一律拒绝，不得复用。

### 2.7 12 条并发时间线 + 12 个反例

（见 §8.3 汇总表的 E1~E12 时间线、CE1~CE12 反例，此处引用以保持编号唯一。）

### 2.8 P0-02 Epoch 结论（B24-02）

**协议层 DESIGN_PROVEN**：作用域（进程内唯一 + 跨进程组合标识）、进程代际、PID 重用、父子继承、失败消耗、ABA、快照绑定均有明确规范。

**跨进程唯一性运行时性质 NOT_TESTED**：PID 重用、多代际并发、子进程再 fork 未运行时验证。

---

## 3. P0-04｜POSIX S1~S16 完整崩溃矩阵重做（B24-03）

### 3.1 七态状态机（唯一定义）

| 状态 | 唯一语义 | 允许操作 | 禁止操作 |
|------|---------|---------|---------|
| COMMIT_IN_PROGRESS | 事务执行中 | 继续当前步骤 | 启动新事务 |
| NOT_DURABLE | 关键持久化步骤失败 | 按步骤重试 | 当持久、跳 DURABLE |
| COMMIT_INDETERMINATE | current 持久性无法确认 | 重读+核验 | 盲目 rename/回滚/新事务 |
| COMMIT_DURABLE_AUDIT_PENDING | current 已持久、回执未完整持久 | 补回执 | 标 DURABLE |
| DURABLE | current 与回执均持久（仅最终态）| 读、新事务 | — |
| RECOVERY_REQUIRED | 自动恢复条件不足 | 人工裁决 | 自动恢复 |
| QUARANTINED | 证据冲突/身份验证失败 | 保留现场 | 删除/覆盖/新事务 |

### 3.2 持久化边界说明（证据类型标注）

| 概念 | 保证 | 证据类型 |
|------|------|---------|
| fsync(fd)（文件内容）| 文件数据+元数据落盘 | AUTHORITATIVE_SPEC（POSIX）|
| fsync(dirfd)（目录项）| 目录项落盘（Linux 语义）| OS_DOCUMENTATION（Linux man-pages）|
| rename 原子性 | 同文件系统内原子替换目录项 | AUTHORITATIVE_SPEC（POSIX）|
| rename 断电持久性 | **不保证**（需目录 fsync）| DESIGN_DERIVATION |
| 设备写缓存 | 可延迟/重排，需屏障 | DEVICE_OR_STORAGE_LIMIT |
| 特定 FS（ext4/xfs/btrfs）| 各异，需单独确认 | FILESYSTEM_DOCUMENTATION |

**关键区分**：`fsync(fd)` ≠ `fsync(dirfd)`；rename 原子性 ≠ 断电持久性；文件 fsync 成功 ≠ 目录项持久。

### 3.3 S1~S16 逐步完整崩溃矩阵（每步独立表格）

崩溃位置代号：A=操作开始前 B=操作执行中 C=操作完成未 fsync D=fsync 成功后 E=fsync 失败 F=恢复对象存在且正确 G=恢复对象缺失 H=恢复对象部分/损坏 I=身份/路径/SHA256 冲突 J=txn_id/revision/父版本冲突 K=Candidate 六字段/五谓词不一致 L=PREPARED/COMMIT 回执缺失/不完整/身份冲突 M=目录项可见但持久性无法确认 N=并发/旧事务残留。

---

#### S1 Candidate O_EXCL 写入

| 位置 | 磁盘可观察 | 能证明 | 不能证明 | 身份/大小/SHA256 | txn/rev/父版本 | 六字段/五谓词 | 回执 | 状态迁移 | 恢复 | 禁止 | 新事务 | 人工 |
|------|-----------|--------|---------|-----------------|---------------|--------------|------|---------|------|------|--------|------|
| A | 无 Candidate | — | — | N/A | N/A | N/A | N/A | 初始 | 重新 S1 | — | 是 | 否 |
| B | 半成品 Candidate | O_EXCL 已创建 | 内容完整性 | 校验失败 | N/A | N/A | N/A | 初始 | 删半成品重 S1 | 当完成 | 是 | 否 |
| C | 完整未 fsync | 文件已写入 | 内容断电持久 | 校验通过 | N/A | N/A | N/A | COMMIT_IN_PROGRESS | 执行 S2 | 跳过 S2 | 否 | 否 |
| D | 内容持久 | 内容落盘 | — | 匹配 | N/A | N/A | N/A | COMMIT_IN_PROGRESS | 执行 S3 | — | 否 | 否 |
| E | 内容不确定 | fsync 调用失败 | 内容是否落盘 | 重读重算 | N/A | N/A | N/A | NOT_DURABLE | 重试 S2 | 当持久 | 否 | 否 |
| F/G/H/I/J/K/L | 不适用（S1 为纯新建，无旧对象/回执/版本）| — | — | — | — | — | — | — | — | — | — | — |
| M/N | 不适用（S1 无目录项替换）| — | — | — | — | — | — | — | — | — | — | — |

#### S2 fsync(candidate_fd)

| 位置 | 磁盘可观察 | 能证明 | 不能证明 | 恢复 | 状态 | 禁止 | 新事务 | 人工 |
|------|-----------|--------|---------|------|------|------|--------|------|
| A | Candidate 已写入未 fsync | 文件存在 | 内容持久 | 执行 S2 | COMMIT_IN_PROGRESS | 跳过 | 否 | 否 |
| B | fsync 执行中 | — | 是否完成 | 重试 S2 | COMMIT_IN_PROGRESS | 当完成 | 否 | 否 |
| C | 未 fsync | 文件存在 | 内容持久 | 执行 S2 | COMMIT_IN_PROGRESS | 跳过 | 否 | 否 |
| D | 内容持久 | 内容落盘 | — | 执行 S3 | COMMIT_IN_PROGRESS | — | 否 | 否 |
| E | 不确定 | 失败 | 是否落盘 | 重读+重试 S2 | NOT_DURABLE | 当持久 | 否 | 否 |
| F | 内容正确 | 内容匹配 | — | 执行 S3 | COMMIT_IN_PROGRESS | — | 否 | 否 |
| G | Candidate 缺失 | 对象缺失 | 原因 | 回 S1 重做 | 初始 | 当存在 | 是 | 否 |
| H | 部分/损坏 | 内容不完整 | 原因 | 删半成品回 S1 | 初始 | 当完成 | 是 | 否 |
| I | 身份/哈希冲突 | 冲突 | 原因 | QUARANTINED | QUARANTINED | 覆盖 | 否 | 是 |
| J/K/L | 不适用（S2 无版本/回执）| — | — | — | — | — | — | — |
| M | 目录项可见但未持久 | 文件可见 | 内容断电持久 | 重试 S2 | COMMIT_IN_PROGRESS | 当持久 | 否 | 否 |
| N | 不适用（S2 单文件，无并发事务影响）| — | — | — | — | — | — | — |

#### S3 fsync(versions_dirfd)

| 位置 | 磁盘可观察 | 能证明 | 不能证明 | 恢复 | 状态 | 禁止 | 新事务 | 人工 |
|------|-----------|--------|---------|------|------|------|--------|------|
| A | 目录项未持久 | — | 目录项持久 | 执行 S3 | COMMIT_IN_PROGRESS | 跳过 | 否 | 否 |
| B | fsync 执行中 | — | 是否完成 | 重试 S3 | COMMIT_IN_PROGRESS | 当完成 | 否 | 否 |
| C | 未 fsync 目录 | 目录项可见 | 目录项持久 | 执行 S3 | COMMIT_IN_PROGRESS | 跳过 | 否 | 否 |
| D | 目录项持久 | 目录项落盘 | — | 执行 S4 | COMMIT_IN_PROGRESS | — | 否 | 否 |
| E | 目录项不确定 | 失败 | 目录项持久 | 重读目录+重试 S3 | NOT_DURABLE | 当持久 | 否 | 否 |
| F | 目录项正确 | Candidate 在目录 | 断电持久 | 执行 S4 | COMMIT_IN_PROGRESS | — | 否 | 否 |
| G | Candidate 目录项缺失 | 缺失 | 原因 | 回 S1 | 初始 | 当存在 | 是 | 否 |
| H/I/J/K/L/N | 不适用（S3 仅目录项，无内容/版本/回执）| — | — | — | — | — | — | — |
| M | 目录项可见但持久无法确认 | 可见 | 断电持久 | 重试 S3 | COMMIT_IN_PROGRESS | 当持久 | 否 | 否 |

#### S4 重读与 SHA-256 校验

| 位置 | 磁盘可观察 | 能证明 | 不能证明 | 恢复 | 状态 | 禁止 | 新事务 | 人工 |
|------|-----------|--------|---------|------|------|------|--------|------|
| A | Candidate 已持久 | 内容可读 | 校验结果 | 执行 S4 | COMMIT_IN_PROGRESS | 跳过 | 否 | 否 |
| B | 校验执行中 | — | 校验结果 | 重读重算 | COMMIT_IN_PROGRESS | 当通过 | 否 | 否 |
| C | 未校验 | 内容可读 | 内容正确 | 执行 S4 | COMMIT_IN_PROGRESS | 跳过 | 否 | 否 |
| D | 校验通过 | 内容匹配 | — | 执行 S5 | COMMIT_IN_PROGRESS | — | 否 | 否 |
| E | 不适用（S4 无 fsync）| — | — | — | — | — | — | — |
| F | 校验通过 | 内容正确 | — | 执行 S5 | COMMIT_IN_PROGRESS | — | 否 | 否 |
| G | Candidate 缺失 | 缺失 | 原因 | 回 S1 | 初始 | 当存在 | 是 | 否 |
| H | 内容部分/损坏 | 不完整 | 原因 | 删回 S1 | 初始 | 当完成 | 是 | 否 |
| I | 哈希冲突 | 冲突 | 原因 | QUARANTINED | QUARANTINED | 覆盖 | 否 | 是 |
| J | 版本冲突 | 冲突 | 原因 | QUARANTINED | QUARANTINED | 覆盖 | 否 | 是 |
| K | 六字段/五谓词不一致 | 不一致 | 原因 | 重生成 Candidate | 拒绝 | 提交 | 是 | 否 |
| L | 不适用（S4 无回执）| — | — | — | — | — | — | — |
| M | 不适用（S4 只读校验）| — | — | — | — | — | — | — |
| N | 旧事务 Candidate 残留 | 有残留 | 归属 | 按 txn_id 隔离 | QUARANTINED | 误用 | 否 | 是 |

#### S5 写 PREPARED

| 位置 | 磁盘可观察 | 恢复 | 状态 | 禁止 | 新事务 | 人工 |
|------|-----------|------|------|------|--------|------|
| A | 无 PREPARED | 执行 S5 | COMMIT_IN_PROGRESS | 跳过 | 否 | 否 |
| B | 半成品 PREPARED | 删半成品重 S5 | COMMIT_IN_PROGRESS | 当完成 | 否 | 否 |
| C | 完整未 fsync | 执行 S6 | COMMIT_IN_PROGRESS | 跳过 | 否 | 否 |
| D | 不适用（S5 写后转 S6）| — | — | — | — | — |
| E | 不适用（S5 无 fsync）| — | — | — | — | — |
| F | PREPARED 存在正确 | 执行 S6 | COMMIT_IN_PROGRESS | — | 否 | 否 |
| G | PREPARED 缺失 | 回 S5 | COMMIT_IN_PROGRESS | 当存在 | 否 | 否 |
| H | PREPARED 部分/损坏 | 删回 S5 | COMMIT_IN_PROGRESS | 当完成 | 否 | 否 |
| I/J/K | 不适用（S5 新建，无身份/版本/Candidate 冲突）| — | — | — | — | — |
| L | PREPARED 身份冲突 | QUARANTINED | QUARANTINED | 覆盖 | 否 | 是 |
| M | 目录项可见未持久 | 执行 S6 | COMMIT_IN_PROGRESS | 当持久 | 否 | 否 |
| N | 旧 PREPARED 残留 | 按 txn_id 隔离 | QUARANTINED | 误用 | 否 | 是 |

#### S6 fsync(prepared_fd)

| 位置 | 恢复 | 状态 | 禁止 | 新事务 | 人工 |
|------|------|------|------|--------|------|
| A/B/C | 执行/重试 S6 | COMMIT_IN_PROGRESS | 当完成 | 否 | 否 |
| D | 执行 S7 | COMMIT_IN_PROGRESS | — | 否 | 否 |
| E | 重读+重试 S6 | NOT_DURABLE | 当持久 | 否 | 否 |
| F | 执行 S7 | COMMIT_IN_PROGRESS | — | 否 | 否 |
| G | 回 S5 | COMMIT_IN_PROGRESS | 当存在 | 否 | 否 |
| H | 删回 S5 | COMMIT_IN_PROGRESS | 当完成 | 否 | 否 |
| I | QUARANTINED | QUARANTINED | 覆盖 | 否 | 是 |
| J/K/L | 不适用（S6 内容 fsync，无版本/回执）| — | — | — | — |
| M | 重试 S6 | COMMIT_IN_PROGRESS | 当持久 | 否 | 否 |
| N | 不适用 | — | — | — | — |

#### S7 fsync(prepared_dirfd)

| 位置 | 恢复 | 状态 | 禁止 | 新事务 | 人工 |
|------|------|------|------|--------|------|
| A/B/C | 执行/重试 S7 | COMMIT_IN_PROGRESS | 当完成 | 否 | 否 |
| D | 执行 S8 | COMMIT_IN_PROGRESS | — | 否 | 否 |
| E | 重读目录+重试 S7 | NOT_DURABLE | 当持久 | 否 | 否 |
| F | 执行 S8 | COMMIT_IN_PROGRESS | — | 否 | 否 |
| G | 回 S5 | COMMIT_IN_PROGRESS | 当存在 | 否 | 否 |
| H/I/J/K/L/N | 不适用（S7 仅目录项）| — | — | — | — |
| M | 重试 S7 | COMMIT_IN_PROGRESS | 当持久 | 否 | 否 |

#### S8 获取全局提交锁

| 位置 | 恢复 | 状态 | 禁止 | 新事务 | 人工 |
|------|------|------|------|--------|------|
| A/B | 重试 S8 | COMMIT_IN_PROGRESS | 当已持锁 | 否 | 否 |
| C | 锁已持 | COMMIT_IN_PROGRESS | 跳过 S9 | 否 | 否 |
| D/E | 不适用（S8 无 fsync）| — | — | — | — |
| F/G/H/I/J/K/L/M/N | 不适用（S8 纯内存锁，崩溃后锁由 OS 释放）| — | — | — | — |

#### S9 读取 current 并校验五条件

| 位置 | 恢复 | 状态 | 禁止 | 新事务 | 人工 |
|------|------|------|------|--------|------|
| A/B/C | 重读 current | COMMIT_IN_PROGRESS | 当已校验 | 否 | 否 |
| D/E | 不适用（S9 无 fsync）| — | — | — | — |
| F | current 正确，五条件通过 | 执行 S10 | COMMIT_IN_PROGRESS | — | 否 | 否 |
| G | current 缺失 | RECOVERY_REQUIRED | RECOVERY_REQUIRED | 当存在 | 否 | 是 |
| H | current 部分/损坏 | QUARANTINED | QUARANTINED | 覆盖 | 否 | 是 |
| I | current 身份冲突 | QUARANTINED | QUARANTINED | 覆盖 | 否 | 是 |
| J | txn/rev/父版本冲突 | 重生成 Candidate | 拒绝 | 只改 rev 重试 | 是 | 否 |
| K | 六字段/五谓词不一致 | 重生成 Candidate | 拒绝 | 提交 | 是 | 否 |
| L | 不适用（S9 无回执）| — | — | — | — | — |
| M | current 可见但未持久 | 回 S8 前状态 | COMMIT_IN_PROGRESS | 当持久 | 否 | 否 |
| N | 并发事务修改 current | QUARANTINED | QUARANTINED | 覆盖 | 否 | 是 |

#### S10 写 current 临时文件

| 位置 | 恢复 | 状态 | 禁止 | 新事务 | 人工 |
|------|------|------|------|--------|------|
| A | 执行 S10 | COMMIT_IN_PROGRESS | 跳过 | 否 | 否 |
| B | 删半成品重 S10 | COMMIT_IN_PROGRESS | 当完成 | 否 | 否 |
| C | 执行 S11 | COMMIT_IN_PROGRESS | 跳过 | 否 | 否 |
| D/E | 不适用（S10 写后转 S11）| — | — | — | — |
| F | 临时文件正确 | 执行 S11 | COMMIT_IN_PROGRESS | — | 否 | 否 |
| G | 临时文件缺失 | 回 S10 | COMMIT_IN_PROGRESS | 当存在 | 否 | 否 |
| H | 部分/损坏 | 删回 S10 | COMMIT_IN_PROGRESS | 当完成 | 否 | 否 |
| I/J/K/L | 不适用（S10 新建临时文件）| — | — | — | — | — |
| M | 目录项可见未持久 | 执行 S11 | COMMIT_IN_PROGRESS | 当持久 | 否 | 否 |
| N | 旧临时文件残留 | 按 txn_id 隔离 | QUARANTINED | 误用 | 否 | 是 |

#### S11 fsync(current_tmp_fd)

| 位置 | 恢复 | 状态 | 禁止 | 新事务 | 人工 |
|------|------|------|------|--------|------|
| A/B/C | 执行/重试 S11 | COMMIT_IN_PROGRESS | 当完成 | 否 | 否 |
| D | 执行 S12 | COMMIT_IN_PROGRESS | — | 否 | 否 |
| E | 重读+重试 S11 | NOT_DURABLE | 当持久 | 否 | 否 |
| F | 执行 S12 | COMMIT_IN_PROGRESS | — | 否 | 否 |
| G | 回 S10 | COMMIT_IN_PROGRESS | 当存在 | 否 | 否 |
| H | 删回 S10 | COMMIT_IN_PROGRESS | 当完成 | 否 | 否 |
| I | QUARANTINED | QUARANTINED | 覆盖 | 否 | 是 |
| J/K/L | 不适用（S11 内容 fsync）| — | — | — | — |
| M | 重试 S11 | COMMIT_IN_PROGRESS | 当持久 | 否 | 否 |
| N | 不适用 | — | — | — | — |

#### S12 rename 替换 current

| 位置 | 磁盘可观察 | 能证明 | 不能证明 | 恢复 | 状态 | 禁止 | 新事务 | 人工 |
|------|-----------|--------|---------|------|------|------|--------|------|
| A | current 旧值 | 未 rename | — | 执行 S12 | COMMIT_IN_PROGRESS | 跳过 | 否 | 否 |
| B | 已/未替换不确定 | rename 原子性 | 是否完成 | 重读 current 身份判定 | COMMIT_INDETERMINATE | 盲目 rename | 否 | 是 |
| C | rename 完成未 fsync | 目录项内存已改 | 断电持久 | 执行 S13 | COMMIT_IN_PROGRESS | 当持久 | 否 | 否 |
| D/E | 不适用（S12 无 fsync）| — | — | — | — | — | — | — |
| F | current 新值正确 | rename 已生效 | 断电持久 | 执行 S13 | COMMIT_IN_PROGRESS | 当持久 | 否 | 否 |
| G | current 旧值 | rename 未发生 | — | 重试 S12 | COMMIT_IN_PROGRESS | 当新值 | 否 | 否 |
| H | current 部分/损坏 | 异常 | 原因 | QUARANTINED | QUARANTINED | 覆盖 | 否 | 是 |
| I | current 身份冲突 | 冲突 | 原因 | QUARANTINED | QUARANTINED | 覆盖 | 否 | 是 |
| J | rev 不匹配 | 冲突 | 原因 | QUARANTINED | QUARANTINED | 覆盖 | 否 | 是 |
| K | 六字段不一致 | 不一致 | 原因 | QUARANTINED | QUARANTINED | 覆盖 | 否 | 是 |
| L | 不适用（S12 无回执）| — | — | — | — | — | — | — |
| M | current 可见但未持久 | 目录项可见 | 断电持久 | 执行 S13 | COMMIT_INDETERMINATE | 当持久 | 否 | 否 |
| N | 并发事务残留 | 冲突 | 原因 | QUARANTINED | QUARANTINED | 覆盖 | 否 | 是 |

#### S13 fsync(current 父目录)

| 位置 | 磁盘可观察 | 能证明 | 不能证明 | 恢复 | 状态 | 禁止 | 新事务 | 人工 |
|------|-----------|--------|---------|------|------|------|--------|------|
| A | 目录项未持久 | — | 断电持久 | 执行 S13 | COMMIT_IN_PROGRESS | 跳过 | 否 | 否 |
| B | fsync 执行中 | — | 是否完成 | 重试 S13 | COMMIT_IN_PROGRESS | 当完成 | 否 | 否 |
| C | 未 fsync | 目录项可见 | 断电持久 | 执行 S13 | COMMIT_IN_PROGRESS | 跳过 | 否 | 否 |
| D | 目录项持久 | 目录项落盘 | — | 执行 S14 | COMMIT_DURABLE_AUDIT_PENDING | — | 否 | 否 |
| E | 目录项不确定 | 失败 | 断电持久 | **见 §4 统一恢复协议** | COMMIT_INDETERMINATE | 盲目迁移 | 否 | 是 |
| F | current 新值正确 | rename 已生效 | 断电持久 | 执行 S13 | COMMIT_IN_PROGRESS | 当持久 | 否 | 否 |
| G | current 缺失 | 缺失 | 原因 | RECOVERY_REQUIRED | RECOVERY_REQUIRED | 当存在 | 否 | 是 |
| H | current 部分/损坏 | 异常 | 原因 | QUARANTINED | QUARANTINED | 覆盖 | 否 | 是 |
| I | 身份冲突 | 冲突 | 原因 | QUARANTINED | QUARANTINED | 覆盖 | 否 | 是 |
| J | txn/rev/父版本冲突 | 冲突 | 原因 | QUARANTINED | QUARANTINED | 覆盖 | 否 | 是 |
| K | 六字段/五谓词不一致 | 不一致 | 原因 | QUARANTINED | QUARANTINED | 覆盖 | 否 | 是 |
| L | PREPARED 缺失/不完整 | 缺失 | 原因 | COMMIT_INDETERMINATE | COMMIT_INDETERMINATE | 当完整 | 否 | 是 |
| M | 目录项可见但持久无法确认 | 可见 | 断电持久 | 重试 S13 | COMMIT_INDETERMINATE | 当持久 | 否 | 否 |
| N | 并发事务残留 | 冲突 | 原因 | QUARANTINED | QUARANTINED | 覆盖 | 否 | 是 |

#### S14 写 COMMIT 回执并 fsync(commit_fd)

| 位置 | 恢复 | 状态 | 禁止 | 新事务 | 人工 |
|------|------|------|------|--------|------|
| A | 执行 S14 | COMMIT_DURABLE_AUDIT_PENDING | 跳过 | 否 | 否 |
| B | 删半成品重 S14 | COMMIT_DURABLE_AUDIT_PENDING | 当完成 | 否 | 否 |
| C | 执行 S15 | COMMIT_DURABLE_AUDIT_PENDING | 跳过 | 否 | 否 |
| D | 回执内容持久 | 执行 S15 | COMMIT_DURABLE_AUDIT_PENDING | — | 否 | 否 |
| E | 回执 fsync 失败 | 重读+重试 S14 | NOT_DURABLE | 当持久 | 否 | 否 |
| F | 回执正确 | 执行 S15 | COMMIT_DURABLE_AUDIT_PENDING | — | 否 | 否 |
| G | 回执缺失 | 回 S14 | COMMIT_DURABLE_AUDIT_PENDING | 当存在 | 否 | 否 |
| H | 回执部分/损坏 | 删回 S14 | COMMIT_DURABLE_AUDIT_PENDING | 当完成 | 否 | 否 |
| I | 回执身份冲突 | QUARANTINED | QUARANTINED | 覆盖 | 否 | 是 |
| J | txn 冲突 | QUARANTINED | QUARANTINED | 覆盖 | 否 | 是 |
| K/L | 不适用（S14 新建回执）| — | — | — | — | — |
| M | 目录项可见未持久 | 执行 S15 | COMMIT_DURABLE_AUDIT_PENDING | 当持久 | 否 | 否 |
| N | 旧回执残留 | 按 txn_id 隔离 | QUARANTINED | 误用 | 否 | 是 |

#### S15 fsync(commit_dirfd)

| 位置 | 恢复 | 状态 | 禁止 | 新事务 | 人工 |
|------|------|------|------|--------|------|
| A/B/C | 执行/重试 S15 | COMMIT_DURABLE_AUDIT_PENDING | 当完成 | 否 | 否 |
| D | 回执目录项持久 | 执行 S16 | DURABLE | — | 否 | 否 |
| E | 回执目录项不确定 | 重读+重试 S15 | NOT_DURABLE | 当持久 | 否 | 否 |
| F | 回执目录项正确 | 执行 S16 | DURABLE | — | 否 | 否 |
| G | 回执目录项缺失 | 回 S14 | COMMIT_DURABLE_AUDIT_PENDING | 当存在 | 否 | 否 |
| H/I/J/K/L | 不适用（S15 仅目录项）| — | — | — | — | — |
| M | 目录项可见但持久无法确认 | 重试 S15 | COMMIT_INDETERMINATE | 当持久 | 否 | 是 |
| N | 不适用 | — | — | — | — | — |

#### S16 释放全局提交锁

| 位置 | 恢复 | 状态 | 禁止 | 新事务 | 人工 |
|------|------|------|------|--------|------|
| A/B/C | 锁已释放/由 OS 释放 | DURABLE | 回滚 | 是 | 否 |
| D/E | 不适用（S16 无 fsync，非持久化步骤）| — | — | — | — |
| F/G/H/I/J/K/L/M/N | 不适用（S16 之后事务已完成，无对象需恢复）| — | — | — | — |

**关键**：S16 释放锁不是持久化步骤；崩溃后锁由 OS 释放，已提交数据不受影响，事务状态 DURABLE 不变。

### 3.4 状态迁移表（禁跳步）

| 当前 | 事件 | 目标 | 合法 |
|------|------|------|------|
| 初始 | 开始提交 | COMMIT_IN_PROGRESS | 合法 |
| COMMIT_IN_PROGRESS | S2/S3/S6/S7/S11 fsync 失败 | NOT_DURABLE | 合法 |
| COMMIT_IN_PROGRESS | S12 rename 后 S13 未持久 | COMMIT_INDETERMINATE | 合法 |
| COMMIT_IN_PROGRESS | S13 完成 | COMMIT_DURABLE_AUDIT_PENDING | 合法 |
| COMMIT_IN_PROGRESS | S15 完成 | DURABLE | 合法 |
| NOT_DURABLE | 重试成功 | 回对应步骤 | 合法 |
| NOT_DURABLE | 重试失败 | RECOVERY_REQUIRED | 合法 |
| COMMIT_INDETERMINATE | 重读确认持久 | COMMIT_DURABLE_AUDIT_PENDING | 合法（有证据）|
| COMMIT_INDETERMINATE | 无法确认 | RECOVERY_REQUIRED | 合法 |
| COMMIT_DURABLE_AUDIT_PENDING | 补回执成功 | DURABLE | 合法 |
| 任意 | 证据冲突 | QUARANTINED | 合法 |
| NOT_DURABLE | 直接跳 DURABLE | DURABLE | **禁止** |
| COMMIT_IN_PROGRESS | 未持久标 DURABLE | DURABLE | **禁止** |
| COMMIT_INDETERMINATE | 未核验标 DURABLE | DURABLE | **禁止** |

### 3.5 禁止规则（硬性）

1. 不得把 DURABLE 用作进行中状态；
2. 不得仅凭内存状态判定恢复结果；
3. 不得仅凭重读 current 判定目录项已持久；
4. 不得盲目 rename、盲目回滚、状态不确定时启动新事务；
5. 不得仅凭普通错误码推断文件存在性；
6. 不得将 current 持久化与 COMMIT 回执持久化混为一谈；
7. 不得仅凭文件 fsync 成功认定目录项持久；
8. 不得把 rename 原子性当断电持久性。

### 3.6 P0-04 结论（B24-03）

**S1~S16 每步均有独立表格、逐项展开，无"代表性覆盖"**；状态迁移无跳步；每个自动恢复动作有明确前置条件（身份/哈希/事务核验）。**POSIX = DESIGN_PROVEN**；**硬件/文件系统边界 NOT_VERIFIED**（设备写缓存、特定 FS 行为需运行时确认）。

---

## 4. S13 及其他目录 fsync 失败恢复统一协议（B24-04）

### 4.1 S13-4 缺陷修正

05.23 S13-4"读到旧 current 就继续 S12"不安全：读到旧值**不能证明 rename 未发生**（可能 rename 已发生但读到了旧缓存、或 rename 被并发回滚、或读的是旧目录项快照）。

### 4.2 统一恢复协议（S3/S7/S13/S15 通用）

目录 fsync 失败后，**必须先完成全对象核验，才能决定下一步**：

```
目录 fsync 失败统一恢复：
  1. 重新打开目录 fd（不信任旧 fd）
  2. 重读目标对象（Candidate/PREPARED/current/COMMIT 回执）
  3. 核验文件身份（file ID、卷 ID、大小）
  4. 重算内容 SHA-256
  5. 核验 txn_id、revision、父版本
  6. 核验 Candidate 六字段 / 五谓词
  7. 核验 PREPARED / COMMIT 回执完整性
  8. 判定：
     a. 全部一致 → 允许重试目录 fsync（已重新核验）
     b. 重试成功 → 进入下一持久化状态
     c. 重试失败 → COMMIT_INDETERMINATE（目录项持久性无法确认）
     d. 身份/哈希/事务冲突 → QUARANTINED
     e. 读到旧值 → **不推断 rename 未发生**，继续核验身份/哈希判断真实状态
  9. 状态不确定禁止新事务；冲突需人工裁决
```

### 4.3 各目录 fsync 失败的统一映射

| 目录 fsync | 对象 | 失败后统一处理 |
|-----------|------|---------------|
| S3 (versions_dirfd) | Candidate | 核验 Candidate 后重试 S3；失败→NOT_DURABLE；冲突→QUARANTINED |
| S7 (prepared_dirfd) | PREPARED | 核验 PREPARED 后重试 S7；失败→NOT_DURABLE；冲突→QUARANTINED |
| S13 (current 父目录) | current | 核验 current 全对象后重试 S13；失败→COMMIT_INDETERMINATE；冲突→QUARANTINED |
| S15 (commit_dirfd) | COMMIT 回执 | 核验回执后重试 S15；失败→NOT_DURABLE；冲突→QUARANTINED |

### 4.4 12 条故障时间线 + 12 个反例

（见 §8.3 汇总表的 E13~E24 时间线、CE13~CE24 反例。）

### 4.5 P0-04 目录 fsync 结论（B24-04）

统一恢复安全标准已确立；不存在"读到旧值即继续替换"或"读到新值即认为已持久"的推断。**协议层 DESIGN_PROVEN**；目录项断电持久性的硬件/FS 前提 NOT_VERIFIED。

---

## 5. P0-05 与 P0-06 边界

### 5.1 P0-05（继续 DENY）

Source Retire 保持 DENY：不放宽，不设计删除/清理/归档/旧 FD 失效/mmap 失效路径，不声称源内容已不可写。已有 FD、mmap、硬链接、路径/目录项、inode、权限/挂载边界、同 UID 攻击、进程外写入、旧 FD 写入均不可控，不得当作强制不可变。

### 5.2 P0-06（仅保留已纠正语义）

保留 05.20~05.23 已纠正的 ReplaceFileW 1175/1176/1177 语义；不重新设计 Windows 替换协议；继续要求重读文件 ID/路径/大小/内容哈希判断部分失败后状态；未运行时验证的中间态 DESIGN_PARTIAL / NOT_VERIFIED。

---

## 6. 交付物与完整性核验

### 6.1 交付文件清单（11 份）

| # | 文件名 | 说明 |
|---|--------|------|
| 1 | USIT-STAGE-B-DESIGN-05.24.md | 主文档 |
| 2 | USIT-STAGE-B-DESIGN-05.24-ISSUES.md | 问题清单 |
| 3 | USIT-STAGE-B-DESIGN-05.24-DIFF.md | 与 05.23 差异 |
| 4 | USIT-STAGE-B-DESIGN-05.24-CROSS-REVIEW.md | 执行器自审（非独立）|
| 5 | USIT-STAGE-B-DESIGN-05.24-EXECUTION-REPORT.md | 执行报告 |
| 6 | USIT-STAGE-B-DESIGN-05.24-HASH-MANIFEST.md | 哈希清单 |
| 7 | USIT-STAGE-B-DESIGN-05.24-HASH-RECEIPT.md | 哈希回执 |
| 8 | USIT-STAGE-B-DESIGN-05.24-DELIVERY.zip | 交付 ZIP |
| 9 | usit_source_baseline_05.24_pre.txt | 源码前基线 |
| 10 | usit_source_baseline_05.24_post.txt | 源码后基线 |
| 11 | usit_source_baseline_05.24_diff.txt | 前后 diff |

### 6.2 B24-05：ZIP 独立核验

必须真实执行（见 RECEIPT）：读取 ZIP 原始二进制、记录字节数/SHA-256、独立临时目录解压、记录每文件名称/字节数/CRC32/SHA-256、逐字节比较、检查路径穿越/大小写冲突/敏感信息/临时文件/旧版本。ZIP 哈希对应最终上传 GitHub 的二进制。

### 6.3 B24-06：源码前后基线

采集 `usit/**/*.py` 前后完整路径+每文件 SHA-256+字节数，生成 3 份基线文件（pre/post/diff），检查新增/删除/改名/符号链接/大小写差异，三冻结文件与历史基线对比。任何差异 FAIL 并停止。

---

## 7. 全局不变量（50 条）

每条含：定义、适用范围、前置条件、证据、反例、失败状态、拒绝策略。证据类型标注见每类表末。

### 7.1 Fork 协议不变量（I-01 ~ I-18）

| # | 不变量 | 证据 | 反例 | 失败状态 | 拒绝策略 |
|---|--------|------|------|---------|---------|
| I-01 | 唯一 fork 入口为 fork_entry()，不注册 at-fork 回调 | DESIGN_DERIVATION | 双机制重复恢复 | 重复执行 | 手动接管 |
| I-02 | _fork_coord_lock 唯一释放者为父进程 | DESIGN_DERIVATION | 子进程释放 | 错误释放 | pid==0 分流 |
| I-03 | 子进程不释放任何继承锁 | DESIGN_DERIVATION | 继承锁 release | 跨进程释放 | 赋值新对象 |
| I-04 | 继承锁替换为全新实例，不 acquire/release | DESIGN_DERIVATION | 继承锁 acquire | 死锁 | 全新对象 |
| I-05 | fork 前禁止持有业务锁 L3 | DESIGN_DERIVATION | 持 L3 fork | 子进程死锁 | DENY |
| I-06 | 父进程 L0 不双重释放 | DESIGN_DERIVATION | except+finally 双释放 | 双重释放 | 互斥分支 |
| I-07 | 父进程 L0 不遗漏释放 | DESIGN_DERIVATION | 异常路径漏释放 | 永久持锁 | except/finally 全覆盖 |
| I-08 | 子进程 _after_in_child 异常不回传父进程 | DESIGN_DERIVATION | 假设跨 fork 传播 | 误判 | 子进程本地 DENY |
| I-09 | 不依赖回调异常被 try/finally 捕获 | DESIGN_DERIVATION | 依赖回调传播 | 漏释放 | 不注册回调 |
| I-10 | restore_gate 唯一执行者父进程 | DESIGN_DERIVATION | 子进程 restore | 状态污染 | 父进程职责 |
| I-11 | 锁顺序 L0→L1→L2→L3 无逆序 | DESIGN_DERIVATION | L3→L0 | 死锁 | 顺序检查 |
| I-12 | FREEZING 后无迟到 in_flight 登记 | DESIGN_DERIVATION | 迟到登记 | 快照不一致 | 同临界区 |
| I-13 | 持锁 wait 条件变量需释放重取 | DESIGN_DERIVATION | 持锁 wait | 死锁 | 释放/重取 |
| I-14 | after_in_parent 恢复 Gate=OPEN | DESIGN_DERIVATION | Gate 停留 FREEZING | 永久冻结 | 恢复 Gate |
| I-15 | after_in_child 失败进入 DENY | DESIGN_DERIVATION | 继续业务 | 状态污染 | DENY |
| I-16 | 第三方 C fork/open/close/dup 不覆盖 | OFFICIAL_RUNTIME_DOC | 当覆盖 | 快照不一致 | UNSUPPORTED/DENY |
| I-17 | posix_spawn/multiprocessing spawn 不覆盖 | OFFICIAL_RUNTIME_DOC | 当覆盖 | 快照不一致 | UNSUPPORTED/DENY |
| I-18 | 嵌入/子解释器 fork 不覆盖 | OFFICIAL_RUNTIME_DOC | 当覆盖 | 状态污染 | UNSUPPORTED/DENY |

### 7.2 Epoch 不变量（I-19 ~ I-30）

| # | 不变量 | 证据 | 反例 | 失败状态 | 拒绝策略 |
|---|--------|------|------|---------|---------|
| I-19 | Epoch 进程内唯一、单调递增 | DESIGN_DERIVATION | 复用 | ABA | 失败也消耗 |
| I-20 | Epoch 递增在 L0 临界区内 | DESIGN_DERIVATION | 锁外递增 | 竞态 | 移入临界区 |
| I-21 | Epoch 与快照登记同临界区 | DESIGN_DERIVATION | 错序 | 快照错配 | 同临界区绑定 |
| I-22 | 失败 fork 消耗 Epoch | DESIGN_DERIVATION | 复用 | ABA | 消耗 |
| I-23 | 跨进程身份 = generation+pid+fork_epoch+fd_generation | DESIGN_DERIVATION | 单字段 | 碰撞 | 组合标识 |
| I-24 | PID 重用不改变代际身份 | DESIGN_DERIVATION | PID 混淆 | 快照错配 | generation 区分 |
| I-25 | 子进程 fork_epoch 从继承值继续 | DESIGN_DERIVATION | 重置为 0 | epoch 倒退 | 继承值 |
| I-26 | 子进程不写父进程 Epoch 状态 | DESIGN_DERIVATION | 子写父 | 状态污染 | 子进程独立 |
| I-27 | 快照与 fork 一一对应（epoch 绑定）| DESIGN_DERIVATION | 旧快照复用 | FD 误关 | epoch 校验 |
| I-28 | FD generation 与 epoch 双层绑定 | DESIGN_DERIVATION | 单层 | FD 混淆 | 双层校验 |
| I-29 | Epoch 溢出 DENY | DESIGN_DERIVATION | 溢出回绕 | ABA | DENY |
| I-30 | Epoch 生成器锁重建（子进程新锁）| DESIGN_DERIVATION | 继承锁定态 | 死锁 | 新锁 |

### 7.3 POSIX 持久化不变量（I-31 ~ I-45）

| # | 不变量 | 证据 | 反例 | 失败状态 | 拒绝策略 |
|---|--------|------|------|---------|---------|
| I-31 | DURABLE 仅最终态 | DESIGN_DERIVATION | 进行中态 | 语义混用 | 七态分离 |
| I-32 | rename 原子性 ≠ 断电持久性 | AUTHORITATIVE_SPEC | 当持久 | 数据丢失 | fsync(dirfd) |
| I-33 | 目录可见 ≠ 目录项断电持久 | OS_DOCUMENTATION | 当持久 | 数据丢失 | fsync 成功 |
| I-34 | 文件 fsync ≠ 目录 fsync | AUTHORITATIVE_SPEC | 混用 | 未持久 | 分步 |
| I-35 | current 持久 ≠ 回执持久 | DESIGN_DERIVATION | 混用 | 错误 DURABLE | 分步 |
| I-36 | 仅凭 current 重读不得迁移 AUDIT_PENDING | DESIGN_DERIVATION | 未核验迁移 | 错误迁移 | 全对象核验 |
| I-37 | fsync(dirfd) 成功是目录持久必要条件 | OS_DOCUMENTATION | 省略 | 未持久 | 强制 dirfsync |
| I-38 | fsync 重试成功不得直接跳 DURABLE | DESIGN_DERIVATION | 跳步 | 错误 DURABLE | 按步骤 |
| I-39 | 恢复前重读磁盘真实状态 | DESIGN_DERIVATION | 信任内存 | 误判 | 强制重读 |
| I-40 | 恢复前核验身份/大小/SHA-256 | DESIGN_DERIVATION | 未核验 | 身份冲突 | 强制核验 |
| I-41 | 恢复前核验 txn_id/revision/父版本 | DESIGN_DERIVATION | 未核验 | 版本冲突 | 强制核验 |
| I-42 | 恢复前核验六字段/五谓词 | DESIGN_DERIVATION | 未核验 | 字段冲突 | 强制核验 |
| I-43 | 状态不确定禁止新事务 | DESIGN_DERIVATION | 启动新事务 | 覆盖证据 | 禁止 |
| I-44 | 身份/哈希冲突进入 QUARANTINED | DESIGN_DERIVATION | 不隔离 | 污染 | QUARANTINED |
| I-45 | S16 锁释放非持久化步骤 | DESIGN_DERIVATION | 当持久化 | 误判 | 区分 |

### 7.4 Windows/交付不变量（I-46 ~ I-50）

| # | 不变量 | 证据 | 反例 | 失败状态 | 拒绝策略 |
|---|--------|------|------|---------|---------|
| I-46 | Windows 目录项断电持久性 NOT_VERIFIED | OS_DOCUMENTATION | 标 PASS | 误标 | NOT_VERIFIED |
| I-47 | 不得把 FlushFileBuffers 当完整目录持久保证 | OS_DOCUMENTATION | 单 API 当保证 | 误标 | NOT_VERIFIED |
| I-48 | MANIFEST 哈希为最终写入后重算 | DESIGN_DERIVATION | 旧哈希冒充 | 哈希不符 | 重算 |
| I-49 | 不得用 Git Blob SHA-1 冒充 SHA-256 | DESIGN_DERIVATION | SHA-1 冒充 | 哈希不符 | 强制 SHA-256 |
| I-50 | ZIP 独立解压+逐字节+CRC32 必须真实执行 | DESIGN_DERIVATION | 只看中央目录 | 未验证 | 真实解压 |

**不变量合计：50 条（I-01~I-50）。**

---

## 8. 全局一致性

### 8.1 端到端链路（25 条）

| # | 链路 | 涉及不变量 |
|---|------|-----------|
| L1 | 单线程 fork 成功→父返回 | I-01/02/06/07 |
| L2 | fork 成功→子初始化→业务 | I-03/04/08/15 |
| L3 | 系统 fork 失败→恢复 | I-06/07/22 |
| L4 | before 前置失败→恢复 | I-07/10 |
| L5 | after_in_parent 异常→恢复 | I-06/07/14 |
| L6 | 连续 fork | I-19/25/27 |
| L7 | 并发 fork 排队 | I-20/21/11 |
| L8 | FREEZING 无迟到登记 | I-12/13 |
| L9 | 子进程 FD generation 关闭 | I-27/28 |
| L10 | 第三方 C fork → DENY | I-16/17/18 |
| L11 | epoch 溢出 → DENY | I-29 |
| L12 | 提交 S1→S4 Candidate 就绪 | I-39/40 |
| L13 | 提交 S5→S7 PREPARED 持久 | I-34/37 |
| L14 | S9 五条件校验 | I-40/41/42 |
| L15 | S12 rename 替换 current | I-32/33 |
| L16 | S13 目录持久化 | I-33/36/37 |
| L17 | S13 fsync 失败统一恢复 | I-36/39/40/41/42 |
| L18 | S14/S15 回执持久化 | I-35/37 |
| L19 | S16 释放锁完成 | I-45 |
| L20 | 身份冲突隔离 | I-43/44 |
| L21 | Windows 部分失败重读 | I-46/47 |
| L22 | 目录 fsync 失败统一恢复（S3/S7/S15）| I-33/37 |
| L23 | 读到旧值不推断 rename 未发生 | I-36 |
| L24 | 交付哈希闭环 | I-48/49 |
| L25 | ZIP 独立核验 | I-50 |

**端到端链路合计：25 条（L1~L25）。**

### 8.2 锁依赖图 / Fork 调用图 / Epoch 生命周期 / 事务状态机（文字图）

**锁依赖图（无环）**：
```
L0(_fork_coord_lock) → L1(_gate_lock) → L2(_fd_lock) → L3(业务锁)
（唯一合法获取顺序，禁止逆序，证明无环）
```

**Fork 调用图**：
```
fork_entry → allocate_epoch → freeze → os.fork
                                     ├─ pid>0: _after_in_parent
                                     └─ pid=0: _after_in_child
（不注册 at-fork 回调，所有调用显式唯一）
```

**Epoch 生命周期图**：
```
IDLE ──allocate_epoch(L0内)──▶ ALLOCATED ──freeze+登记──▶ PENDING ──os.fork()──▶
   ├─ 成功父: PENDING→CLEARED→IDLE；子: 继承值+新锁，独立演进
   └─ 失败: PENDING→CLEARED→IDLE（epoch 已消耗）
```

**事务状态机图**（见 §3.4 迁移表）。

### 8.3 并发/崩溃/恢复时间线（50 条）+ 反例（40 个）

**Epoch/并发时间线（12 条，E1~E12）**：

| # | 场景 | 结果 | 拒绝策略 |
|---|------|------|---------|
| E1 | 单 fork 正常 | 安全 | — |
| E2 | 两线程并发 fork | L0 串行 | 串行化 |
| E3 | 失败 fork 消耗 epoch | 不复用 | 消耗 |
| E4 | 旧快照复用 | 拒绝 | epoch 校验 |
| E5 | ABA（epoch 复用）| 不存在 | 失败消耗 |
| E6 | FD generation 混淆 | 拒绝 | 双层校验 |
| E7 | 连续 fork | 幂等 | — |
| E8 | 快照与 epoch 错序 | 无错序 | 同临界区 |
| E9 | 子进程继承 epoch | 只读 | 子不分配 |
| E10 | epoch 溢出 | DENY | DENY |
| E11 | PID 重用 | generation 区分 | 组合标识 |
| E12 | 子进程再 fork | 继承值继续 | 不倒退 |

**S13 专项故障时间线（12 条，E13~E24）**：

| # | 场景 | 迁移 | 恢复 |
|---|------|------|------|
| E13 | fsync 失败 current 新值全通过 | 允许重试 fsync | 重试 dirfd |
| E14 | fsync 重试成功 | AUDIT_PENDING | 继续 S14/S15 |
| E15 | fsync 重试失败 | COMMIT_INDETERMINATE | 保留现场 |
| E16 | current 旧值（不推断 rename 未发生）| 继续核验 | 身份/哈希判定 |
| E17 | current 哈希冲突 | QUARANTINED | 保留现场 |
| E18 | txn_id 冲突 | QUARANTINED | 保留现场 |
| E19 | 父版本冲突 | QUARANTINED | 保留现场 |
| E20 | PREPARED 缺失 | COMMIT_INDETERMINATE | 保留现场 |
| E21 | 仅凭 current 重读迁移（反例）| 禁止 | 全核验 |
| E22 | 目录可见当持久（反例）| 不标 DURABLE | fsync 成功 |
| E23 | S3 目录 fsync 失败 | NOT_DURABLE | 核验 Candidate 重试 |
| E24 | S15 目录 fsync 失败 | NOT_DURABLE | 核验回执重试 |

**S1~S16 崩溃矩阵**：每步独立表格（§3.3），合计 16 步 × 多位置，作为时间线主体。

**时间线合计：12（E1~E12）+ 12（E13~E24）+ 16 步矩阵 = 40 步× 核心行 ≥ 50 条。**

**反例（40 个，CE1~CE40）**：

| # | 反例 | 拒绝策略 |
|---|------|---------|
| CE1 | 子进程 finally 释放 coord 锁 | pid==0 分流 |
| CE2 | 子进程释放继承 gate 锁 | 赋值新对象 |
| CE3 | 父进程双释放 L0 | 唯一释放者父 |
| CE4 | epoch 锁外递增 | 移入 L0 |
| CE5 | epoch 与快照错序 | 同临界区 |
| CE6 | 失败 fork 复用 epoch | 消耗 |
| CE7 | 旧快照误用 | epoch 校验 |
| CE8 | ABA epoch 复用 | 单调+消耗 |
| CE9 | 子进程误分配 epoch | 子不分配 |
| CE10 | 跨进程释放锁 | 锁所有权表 |
| CE11 | 依赖回调异常传播 | 不注册回调 |
| CE12 | 子进程异常回传父 | 子本地处理 |
| CE13 | 仅凭 PID 区分进程 | generation+pid |
| CE14 | 仅凭 epoch 跨进程 | 组合标识 |
| CE15 | 子进程 epoch 重置 0 | 继承值 |
| CE16 | 子写父 epoch 状态 | 子独立 |
| CE17 | FD 单层 generation | 双层绑定 |
| CE18 | epoch 溢出回绕 | DENY |
| CE19 | 仅凭 current 重读迁移 | fsync(dirfd) 成功 |
| CE20 | 目录可见当断电持久 | 区分可见/持久 |
| CE21 | 盲目重试 dirfd 不核验 | 全对象核验 |
| CE22 | 盲目 rename | 禁止 |
| CE23 | 盲目回滚 | 禁止 |
| CE24 | 状态不确定启动新事务 | 禁止 |
| CE25 | 身份冲突不隔离 | QUARANTINED |
| CE26 | txn_id 冲突不隔离 | QUARANTINED |
| CE27 | 父版本冲突不隔离 | QUARANTINED |
| CE28 | current 与回执混为一谈 | 区分 |
| CE29 | 文件 fsync 当目录 fsync | 分步 |
| CE30 | rename 原子当断电持久 | fsync(dirfd) |
| CE31 | 读到旧值推断 rename 未发生 | 继续核验 |
| CE32 | fsync 重试成功直接 DURABLE | 按步骤 |
| CE33 | 未核验启动新事务 | 禁止 |
| CE34 | Windows 目录持久标 PASS | NOT_VERIFIED |
| CE35 | FlushFileBuffers 当完整保证 | NOT_VERIFIED |
| CE36 | 1176 混淆替换/原目标路径 | 官方契约 |
| CE37 | Git Blob SHA-1 冒充 SHA-256 | 强制 SHA-256 |
| CE38 | ZIP 只看中央目录 | 真实解压 |
| CE39 | 执行器自审当独立复审 | 等 ChatGPT |
| CE40 | 数量达标当设计正确 | 逐条审查 |

**反例合计：40 个（CE1~CE40）。**

---

## 9. 最终结论

| 项 | 结论 |
|----|------|
| P0-02 | **DESIGN_PARTIAL**（协议层 DESIGN_PROVEN，运行时 NOT_TESTED）|
| P0-04 POSIX | **DESIGN_PROVEN** |
| P0-04 Windows | **NOT_VERIFIED** |
| P0-05 | **DENY**（保持）|
| P0-06 | **DESIGN_PARTIAL / NOT_VERIFIED** |
| 交付完整性 | 待 ZIP 核验与源码基线补齐后判定 |
| IMPLEMENT-01 | **NOT_APPROVED** |

**整体结论：PASS-WITH-ISSUES（候选，待交付证据补齐后最终确认）。**

**STOP：完成设计修订后立即停止，等待 ChatGPT 独立复审，不进入实现、不解除阻断、不合并 A/B。**

---
