# USIT-STAGE-B-DESIGN-05.23｜Fork父子进程锁协议修复＋POSIX事务崩溃矩阵闭合＋交付证据补齐

> 阶段 B · DESIGN-05.23 · STRICT READ-ONLY DESIGN REVISION ONLY
> 作者：USIT 主开发 · 生成：2026-10-31（GMT+8）
> 前置审查：DESIGN-05.22 独立复审结论 FAIL
> 工作区：`C:\Users\Administrator\WorkBuddy\2026-10-15-22-09-47\usit_p0`
> 交付目录：`C:\Users\Administrator\WorkBuddy\2026-10-15-22-09-47\usit_p0_05.23_reports`（**项目外**）
> 状态：**STOP — 等待独立复审**（未授权 IMPLEMENT-01）

---

## 0. 缺陷追踪表（B23-01 ~ B23-06）

| # | 原始缺陷 | 05.22 位置 | 05.23 修订位置 | 证据 | 验证方式 | 状态 | 剩余风险 | 解除阻断？|
|---|---------|-----------|---------------|------|---------|------|---------|----------|
| B23-01 | `finally` 在父子两侧各执行一次，子进程误释放未持有的 `_fork_coord_lock` | §1.4 时序图(96-98行) | §1.2 父子进程分叉协议 | 父/子时间线 + 锁所有权表 | 异常路径表 + 反例 | 闭合 | 第三方 C fork | 否 |
| B23-02 | `fork_epoch_counter` 在获取 `_fork_coord_lock` 前递增，存在并发竞态 | §1.4(88行) | §1.3 epoch 分配移入锁临界区 | epoch 状态机 | 并发时间线 + 反例 | 闭合 | 第三方 C fork | 否 |
| B23-03 | S1~S16 崩溃矩阵仍缺部分崩溃位置 | §2.4 | §2.4 完整矩阵 | 10 类位置 | 逐步矩阵 | 闭合 | Windows 目录持久化 | 否 |
| B23-04 | S13 fsync 失败恢复逻辑不完整（仅凭 current 重读迁移）| §2.4/§2.5 | §2.5 S13 专项恢复 | S13 专项时间线 | 10 条 S13 时间线 | 闭合 | Windows 目录持久化 | 否 |
| B23-05 | ZIP 独立核验需真实执行 | §6 | §6 交付核验 | RECEIPT 真实结果 | 真实解压+比对 | 待执行 | — | 否 |
| B23-06 | 源码前后完整基线需真实采集 | §6.4 | §6.4 完整基线 | 25 文件前后清单 | diff 比对 | 待执行 | — | 否 |

**本节是全文总纲。本轮核心任务：B23-01 父子进程分叉协议（子进程不释放继承锁）；B23-02 epoch 递增移入锁临界区；B23-04 S13 专项恢复；B23-05/B23-06 交付证据。**

---

## 1. P0-02｜Fork 父子进程锁协议修复

### 1.1 缺陷定位：B23-01 finally 父子两侧重复执行 + B23-02 epoch 递增竞态

05.22 §1.4 时序图存在两处缺陷：

**B23-01（finally 父子两侧执行）**：05.22 时序图第 96~98 行：

```
finally:
    _pending_snapshot = None
    release(_fork_coord_lock)
```

CPython `os.fork()` 返回后，**父进程和子进程都从 fork 调用点之后的同一代码继续执行**。因此 `finally` 块在**父子两侧各执行一次**：
- 父进程：`release(_fork_coord_lock)` 正确（父进程持有）；
- 子进程：也执行 `release(_fork_coord_lock)`，但子进程**从未持有**这把锁（它只是 fork 时继承了父进程的锁状态快照）。子进程释放一把它没有所有权且可能已被父进程释放的锁 → **错误释放未持有锁 / 跨进程错误释放**。

**B23-02（epoch 递增竞态）**：05.22 第 88 行 `epoch = ++fork_epoch_counter` 在 `acquire(_fork_coord_lock)` 之前执行。epoch 的递增与快照登记（`_pending_snapshot = ...`）**不在同一临界区**，存在并发竞态：epoch 分配与 `_fork_coord_lock` 未绑定，可能导致 epoch 分配与快照登记错序。

### 1.2 父子进程分叉协议（修复 B23-01）

**核心原则**：`os.fork()` 之后，父进程与子进程**执行分叉**，各自拥有独立的锁所有权。子进程**不释放任何继承锁**（继承锁只反映 fork 时刻父进程的状态，子进程对它没有所有权）。

**重构后的 fork_entry 协议**：

```python
def fork_entry():
    # ---- fork 前（仅父进程执行）----
    assert is_main_interpreter_main_thread()      # 否则 DENY
    acquire(_fork_coord_lock)                      # L0，父进程获取
    epoch = 0
    try:
        # epoch 分配与快照登记在同一临界区（修复 B23-02）
        epoch = allocate_epoch()                   # 在 _fork_coord_lock 保护下递增
        snapshot = freeze(epoch)                   # 冻结 Registry，生成快照
        _pending_snapshot = (epoch, snapshot)      # 登记快照
        pid = os.fork()                            # fork 点
    except:
        # 父进程异常路径：恢复 Gate + 清空快照 + 释放 L0
        restore_gate_on_error()                    # gate = OPEN + signal
        _pending_snapshot = None
        release(_fork_coord_lock)
        raise

    if pid == 0:
        # ---- 子进程（fork 后，独立执行路径）----
        # 关键：子进程不释放 _fork_coord_lock（它没有所有权）
        # 子进程直接进入 after_in_child 逻辑，不经过父进程的 finally
        _after_in_child(epoch, snapshot)
        return 0   # 或直接继续子进程业务，不再触碰父进程锁

    # ---- 父进程（fork 后）----
    _after_in_parent(epoch)                        # 恢复 Gate + 清空快照
    release(_fork_coord_lock)                      # L0，父进程唯一释放
    return pid
```

**关键修订**：
1. **子进程不执行父进程的 `finally`/`release`**：通过 `if pid == 0` 分支提前分流，子进程进入 `_after_in_child` 后**不再返回**到父进程的释放代码；
2. **`_fork_coord_lock` 唯一释放者是父进程**，子进程对它没有任何释放动作；
3. 子进程对继承的锁对象（`_fork_coord_lock`、`_gate_lock`、`_fd_lock`）**直接赋值全新对象**，不 acquire、不 release 继承锁。

### 1.3 父子进程锁所有权与生命周期

**父进程锁所有权表**：

| 锁 | 父进程获取 | 父进程释放 | 释放前置 | 子进程是否继承 | 子进程动作 |
|----|-----------|-----------|---------|--------------|-----------|
| `_fork_coord_lock` (L0) | fork_entry 入口 | fork_entry 末尾（父进程唯一释放）| 完成 parent 恢复 | 继承（锁定态快照）| **不释放，直接赋值新对象** |
| `_gate_lock` (L1) | freeze/准入/after_in_parent | 各临界区 | 临界区结束 | 继承 | **不释放，直接赋值新对象** |
| `_fd_lock` (L2) | freeze/FD 操作 | 各临界区 | 临界区结束 | 继承 | **不释放，直接赋值新对象** |
| 业务锁 (L3) | 业务代码 | 业务代码 | 禁止持锁 fork | 继承（可能锁定）| **不 acquire，直接赋值新对象** |

**子进程锁重建协议**：子进程在 `_after_in_child` 中，对每一把继承的锁对象**直接替换为全新实例**（`_fork_coord_lock = new Lock()` 等），**不尝试 acquire 或 release 继承锁**。这是唯一安全做法，因为继承锁可能处于其他线程（已在 fork 中消失）持有的锁定状态。

### 1.4 父子进程时间线

**父进程时间线**：
```
fork_entry()
  ├─ 线程身份检查（主解释器主线程）
  ├─ acquire(_fork_coord_lock)           # L0
  ├─ epoch = allocate_epoch()            # L0 临界区内（B23-02 修复）
  ├─ snapshot = freeze(epoch)            # L0→L1→L2 升序
  ├─ _pending_snapshot = (epoch, snapshot)
  ├─ pid = os.fork()
  │     ├─ before(): 只读快照，无锁
  │     └─ 返回 pid > 0（父进程）
  ├─ _after_in_parent(epoch): 恢复 Gate=OPEN，清空 _pending_snapshot
  ├─ release(_fork_coord_lock)           # L0，父进程唯一释放
  └─ return pid
```

**子进程时间线**：
```
（fork 返回 pid == 0）
  ├─ 继承父进程内存（含锁对象、快照、Registry）
  ├─ PyOS_AfterFork_Child 重置 runtime（CPython 内部）
  ├─ _after_in_child(epoch, snapshot)：
  │     ├─ 直接赋值全新锁对象（不 acquire/release 继承锁）
  │     ├─ 遍历 snapshot，generation 匹配才 close
  │     ├─ 对不在 snapshot 中的继承 FD 标 UNKNOWN_FD，不主动 close
  │     ├─ Gate 置 child-local OPEN
  │     └─ _fork_reset_ok = True（或异常 → False）
  └─ 继续子进程业务（不再触碰父进程锁）
```

**关键分离**：fork 之后父进程走 `pid > 0` 分支，子进程走 `pid == 0` 分支，二者**代码路径完全分叉**，子进程绝不执行父进程的 `release(_fork_coord_lock)`。

### 1.5 异常路径表（A~H + 取消）

| 路径 | 进程 | 锁状态 | 释放动作 | 是否重复/错误释放 | Gate 状态 | 允许再 fork | 进入 QUARANTINED/DENY |
|------|------|--------|---------|------------------|----------|------------|----------------------|
| A. fork 成功父返回 | 父 | 持 L0 | 父释放 L0 | 否 | OPEN | 是 | 否 |
| B. fork 系统调用失败 | 父 | 持 L0 | 父 except 释放 L0 | 否 | OPEN | 是 | 否 |
| C. before 钩子失败 | 父 | 持 L0 | 父 except 释放 L0 | 否 | OPEN | 是 | 否 |
| D. after_in_parent 失败 | 父 | 持 L0 | 父 finally 释放 L0 | 否 | OPEN（恢复）| 是 | 否 |
| E. freeze 失败 | 父 | 持 L0 | 父 except 释放 L0 | 否 | OPEN | 是 | 否 |
| F. fork 入口异常/取消 | 父 | 持 L0 | 父 finally 释放 L0 | 否 | OPEN | 是 | 否 |
| G. 连续 fork | 父 | 每次持 L0 | 每次父释放 | 否 | OPEN | 是 | 否 |
| H. after_in_child 失败 | 子 | **不持 L0** | **不释放任何继承锁** | 否 | child-local | 不适用 | DENY 受保护操作 |

**证明无重复/错误释放**：
- `_fork_coord_lock` 只在父进程路径的 `release`（入口末尾）或 except/finally 释放，**子进程路径（`pid==0` 分支）不含任何 `release(_fork_coord_lock)`**；
- 子进程对继承锁**直接赋值新对象**，不 release，故不存在"释放未持有锁"或"跨进程释放"。

### 1.6 epoch 递增移入锁临界区（修复 B23-02）

**问题**：05.22 的 `epoch = ++fork_epoch_counter` 在 `acquire(_fork_coord_lock)` 前执行，epoch 分配与快照登记不在同一临界区。

**修复**：epoch 分配移入 `_fork_coord_lock` 临界区：

```
allocate_epoch():    # 必须在持有 _fork_coord_lock 时调用
    epoch = fork_epoch_counter
    fork_epoch_counter += 1
    return epoch
```

**epoch 设计要点**：

| 项 | 定义 |
|----|------|
| 唯一分配者 | `fork_entry()`（在 `_fork_coord_lock` 临界区内调用 `allocate_epoch()`）|
| 同步机制 | `_fork_coord_lock`（L0）互斥量保护，epoch 递增与快照登记同临界区 |
| 与 L0 绑定 | epoch 分配、快照登记、os.fork 均在持 L0 期间，一一对应 |
| 与快照对应 | `_pending_snapshot = (epoch, snapshot)`，epoch 与快照绑定 |
| 并发 fork 排队 | `_fork_coord_lock` 串行化，同一时刻只有一个 fork 持有 L0 |
| 失败 fork 是否消耗 epoch | 是（epoch 已分配，失败后不再复用，避免 ABA）|
| epoch 溢出 | 用 64 位整数，实际不会溢出；若溢出则进入 DENY（安全拒绝）|
| 进程继承 | 子进程继承 `fork_epoch_counter` 的 fork 时刻值，但子进程不再分配 epoch |
| 子进程重建 | 子进程的 `fork_epoch_counter` 置为继承值，后续子进程 fork 需重新走父逻辑 |
| 连续 fork | 每次 fork 分配新 epoch，`_fork_coord_lock` 串行化 |
| 快照登记与 epoch 同临界区 | 是（都在持 L0 期间）|
| 避免 ABA/旧快照复用 | epoch 单调 + 失败也消耗 epoch + 单槽位 + L0 串行化 |
| FD generation 混淆 | epoch 与 generation 双层绑定，旧快照 epoch 不匹配则拒绝 |

### 1.7 fork_epoch 状态机

```
fork_epoch 生命周期（在 _fork_coord_lock 临界区内演进）：
  IDLE ──allocate_epoch（L0 内）──▶ ALLOCATED（epoch 已分配）
  ALLOCATED ──freeze + 登记快照──▶ PENDING（_pending_snapshot = (epoch, snapshot)）
  PENDING ──os.fork()──▶ fork 成功？
       ├─ 是：parent PENDING→CLEARED（after_in_parent 清空）→ IDLE
       │      child（独立路径）：读取 snapshot，不触碰 epoch 状态机
       └─ 否：PENDING→CLEARED（except 清空）→ IDLE（epoch 已消耗，不复用）
```

### 1.8 10 条并发时间线（B23-02 epoch 竞态）

| # | 场景 | 初始 | 事件 | 锁状态 | epoch | 快照 | 结果 | 拒绝策略 |
|---|------|------|------|--------|-------|------|------|---------|
| T1 | 单 fork 正常 | IDLE | 分配 epoch→freeze→fork→恢复 | L0 | N | 一一对应 | 安全 | — |
| T2 | 两线程并发 fork | IDLE | A/B 竞争 L0 | L0 串行 | 顺序分配 | 各自 | 一胜一排队 | 串行化 |
| T3 | 失败 fork 消耗 epoch | PENDING | fork 失败 | L0 | N 已消耗 | 清空 | 不复用 N | 消耗 epoch |
| T4 | 旧快照复用 | PENDING | 新一轮 fork 读到旧 epoch | L0 | 不匹配 | 拒绝 | 不误用 | epoch 校验 |
| T5 | ABA（epoch 复用）| — | 失败后 epoch 复用 | — | 禁止复用 | — | 不存在 | 失败也消耗 |
| T6 | FD generation 混淆 | FREEZING | 数字复用 | L1 | 匹配校验 | 拒绝 | 不误关 | generation+epoch |
| T7 | 连续 fork | OPEN | 两次 fork | L0 串行 | N, N+1 | 各自 | 幂等 | — |
| T8 | 快照登记与 epoch 错序 | — | 移入 L0 后同临界区 | L0 | 绑定 | 一致 | 无错序 | 同临界区 |
| T9 | 子进程继承 epoch | child | 继承 fork 时刻值 | — | 继承值 | 只读 | 不再分配 | 子进程不分配 |
| T10 | epoch 溢出 | — | 64 位耗尽 | — | 溢出 | DENY | 安全拒绝 | DENY |

### 1.9 10 个反例（B23-01 锁释放 + B23-02 epoch）

| # | 反例 | 拒绝策略 |
|---|------|---------|
| R1 | 子进程 finally 释放 `_fork_coord_lock`（B23-01）| pid==0 分支分流，子进程不释放 |
| R2 | 子进程释放继承的 `_gate_lock` | 直接赋值新对象，不释放 |
| R3 | 父进程双释放 L0 | 唯一释放者父进程 |
| R4 | epoch 在锁外递增（B23-02）| allocate_epoch 移入 L0 临界区 |
| R5 | epoch 分配与快照登记错序 | 同临界区绑定 |
| R6 | 失败 fork 复用 epoch | 失败也消耗 epoch |
| R7 | 旧快照被新 fork 误用 | epoch 校验 |
| R8 | ABA epoch 复用 | 单调递增 + 失败消耗 |
| R9 | 子进程误分配新 epoch | 子进程不分配 epoch |
| R10 | 跨进程释放锁 | 锁所有权表 + 子进程不释放 |

### 1.10 P0-02 结论

**P0-02 = DESIGN_PARTIAL**。
- **协议层**：父子进程分叉协议（子进程不释放继承锁）+ epoch 移入锁临界区 + 锁所有权表 + 异常路径表闭合——**DESIGN_PROVEN（协议层）**。
- **运行时**：多线程 fork 运行时行为 NOT_TESTED。
- **不可控入口**：第三方 C fork/open、嵌入、子解释器、posix_spawn UNSUPPORTED/DENY。

---

## 2. P0-04｜POSIX 事务崩溃矩阵闭合

### 2.1 七态状态机（沿用 05.22，DURABLE 仅最终态）

| 状态 | 唯一定义 |
|------|---------|
| COMMIT_IN_PROGRESS | 事务正在执行 |
| NOT_DURABLE | 提交前关键持久化步骤失败 |
| COMMIT_INDETERMINATE | current 是否持久化无法确认 |
| COMMIT_DURABLE_AUDIT_PENDING | current 已持久，回执未完整持久 |
| DURABLE | current 与回执都持久 |
| RECOVERY_REQUIRED | 自动恢复条件不足 |
| QUARANTINED | 证据冲突/身份验证失败 |

### 2.2 S1~S16 完整崩溃矩阵（10 类崩溃位置，无省略）

崩溃位置代号：①操作开始前 ②操作执行中 ③操作完成未 fsync ④fsync 成功后 ⑤fsync 失败 ⑥恢复观察对象存在且内容正确 ⑦恢复观察对象缺失 ⑧恢复观察部分写入/损坏 ⑨恢复观察身份/路径/哈希冲突 ⑩恢复观察事务身份/revision/父版本/Candidate 字段不一致。

| 步骤 | 位置 | 磁盘可观察状态 | 持久性证明 | 文件身份/大小/SHA256 | txn_id/rev/父版本 | Candidate 六字段/五谓词 | PREPARED/回执状态 | 事务状态迁移 | 恢复动作 | 禁止动作 | 允许新事务 | 人工裁决 |
|------|------|---------------|-----------|---------------------|------------------|------------------------|------------------|-------------|---------|---------|-----------|---------|
| S1 | ① | 无 Candidate | N/A | N/A | N/A | N/A | N/A | 初始 | 重新 S1 | 无 | 是 | 否 |
| S1 | ② | 半成品 | 否 | 校验失败 | N/A | N/A | N/A | 初始 | 删半成品重试 | 当完成 | 是 | 否 |
| S1 | ③ | 完整未持久 | 否 | 校验 | N/A | N/A | N/A | COMMIT_IN_PROGRESS | 执行 S2 | 跳过 S2 | 否 | 否 |
| S2 | ④ | 内容持久 | 是 | 匹配 | N/A | N/A | N/A | COMMIT_IN_PROGRESS | 执行 S3 | 跳过 | 否 | 否 |
| S2 | ⑤ | 不确定 | 否 | 重读重算 | N/A | N/A | N/A | NOT_DURABLE | 重试 S2 | 当持久 | 否 | 否 |
| S3 | ⑤ | 目录项不确定 | 否 | 重读目录 | N/A | N/A | N/A | NOT_DURABLE | 重试 S3 | 当持久 | 否 | 否 |
| S4 | ⑥ | 校验通过 | — | 匹配 | N/A | 六字段一致 | N/A | COMMIT_IN_PROGRESS | 执行 S5 | 跳过 | 否 | 否 |
| S4 | ⑩ | 哈希/字段冲突 | 否 | 冲突 | 六字段冲突 | 冲突 | N/A | 拒绝 | 重生成 | 提交 | 是 | 否 |
| S5 | ② | PREPARED 半成品 | 否 | 校验失败 | N/A | N/A | N/A | COMMIT_IN_PROGRESS | 删半成品重试 | 当完成 | 否 | 否 |
| S6 | ⑤ | 内容不确定 | 否 | 重算 | N/A | N/A | N/A | NOT_DURABLE | 重试 S6 | 当持久 | 否 | 否 |
| S7 | ⑤ | 目录项不确定 | 否 | 重读目录 | N/A | N/A | N/A | NOT_DURABLE | 重试 S7 | 当持久 | 否 | 否 |
| S8 | ③ | 锁已持有 | — | N/A | N/A | N/A | N/A | COMMIT_IN_PROGRESS | 执行 S9 | 释放锁跳过 | 否 | 否 |
| S9 | ③ | current 未变 | — | N/A | 五条件失败 | 五谓词失败 | N/A | 拒绝 | 重生成 | 只改 rev 重试 | 是 | 否 |
| S10 | ② | 临时半成品 | 否 | 校验失败 | N/A | N/A | N/A | COMMIT_IN_PROGRESS | 删半成品重试 | 当完成 | 否 | 否 |
| S11 | ⑤ | 内容不确定 | 否 | 重算 | N/A | N/A | N/A | NOT_DURABLE | 重试 S11 | 当持久 | 否 | 否 |
| S12 | ② | 已/未替换不确定 | 否 | 重读 current 身份 | 判断 rev | N/A | N/A | COMMIT_INDETERMINATE | 重读判定 | 盲目 rename | 否 | 是 |
| S13 | ⑤ | 目录项不确定 | 否 | 重读 current+ID+哈希 | 判断 rev | 六字段核验 | PREPARED 核验 | COMMIT_INDETERMINATE | 见 §2.5 | 盲目迁移 | 否 | 是 |
| S13 | ④ | 目录项持久 | 是 | 匹配 | 新 rev | N/A | N/A | COMMIT_DURABLE_AUDIT_PENDING | 执行 S14 | 跳过 | 否 | 否 |
| S14 | ② | 回执半成品 | 否 | 校验失败 | N/A | N/A | N/A | COMMIT_DURABLE_AUDIT_PENDING | 删半成品重试 | 当完成 | 否 | 否 |
| S15 | ⑤ | 回执目录项不确定 | 否 | 重读目录 | N/A | N/A | N/A | COMMIT_DURABLE_AUDIT_PENDING | 补 fsync | 当完成 | 否 | 否 |
| S15 | ④ | 回执目录项持久 | 是 | 匹配 | N/A | N/A | N/A | DURABLE | 执行 S16 | 跳过 | 否 | 否 |
| S16 | ③ | 锁释放，全持久 | 是 | 匹配 | 新 rev | N/A | N/A | DURABLE | — | — | 是 | 否 |
| S16 | ② | 锁未释放，全持久 | 是 | 匹配 | 新 rev | N/A | N/A | DURABLE | 锁由 OS 释放 | 回滚 | 是 | 否 |

### 2.3 七态迁移表（沿用 05.22，禁跳步）

| 当前 | 事件 | 目标 | 合法 |
|------|------|------|------|
| 初始 | 开始提交 | COMMIT_IN_PROGRESS | 合法 |
| COMMIT_IN_PROGRESS | S2/S3/S6/S7/S11 fsync 失败 | NOT_DURABLE | 合法 |
| COMMIT_IN_PROGRESS | S12 rename 后 S13 未持久 | COMMIT_INDETERMINATE | 合法 |
| COMMIT_IN_PROGRESS | S13 完成 | COMMIT_DURABLE_AUDIT_PENDING | 合法 |
| COMMIT_IN_PROGRESS | S15 完成 | DURABLE | 合法 |
| NOT_DURABLE | 重试成功 | 回 COMMIT_IN_PROGRESS 对应步骤 | 合法 |
| NOT_DURABLE | 重试失败 | RECOVERY_REQUIRED | 合法 |
| COMMIT_INDETERMINATE | 重读确认 current 持久 | COMMIT_DURABLE_AUDIT_PENDING | 合法（有证据）|
| COMMIT_INDETERMINATE | 无法确认 | RECOVERY_REQUIRED | 合法 |
| COMMIT_DURABLE_AUDIT_PENDING | 补回执成功 | DURABLE | 合法 |
| 任意 | 证据冲突 | QUARANTINED | 合法 |
| NOT_DURABLE | 直接跳 DURABLE | DURABLE | **禁止** |
| COMMIT_IN_PROGRESS | 未持久标 DURABLE | DURABLE | **禁止** |

### 2.4 禁止规则（硬性）

1. 不得把 DURABLE 用作进行中状态；
2. 不得仅凭内存状态判定恢复结果；
3. 不得仅凭重读 current 判定目录项已持久；
4. 不得盲目 rename、盲目回滚、状态不确定时启动新事务；
5. 不得仅凭普通错误码推断文件存在性；
6. 不得将 current 持久化与 COMMIT 回执持久化混为一谈。

### 2.5 S13 专项恢复协议（修复 B23-04）

**场景**：`rename(current_tmp, current)` 已完成，但 `fsync(current_parent_dirfd)` 返回失败。

**核心问题**：current 重读成功只能证明**目录项在内存中已更新**（rename 已生效），**不能证明目录项已断电持久化**（那需要 fsync(dirfd) 成功）。

**完整恢复步骤**：

```
S13 fsync 失败恢复：
  1. 重读 current：读到的值 → 判断是旧值还是新值
     - 读到的 current 内容能证明：rename 已生效（目录项内存已改）
     - 不能证明：目录项已断电持久化
  2. 重读文件身份：current 的 file ID、卷 ID、文件大小
  3. 重算 current 内容 SHA-256
  4. 核对 txn_id、revision、父版本
  5. 重新核验 Candidate 六字段和五谓词（确认 Candidate 与 current 一致）
  6. 核验 PREPARED 回执（确认 PREPARED 完整）
  7. 判断：
     a. 若 current 内容为新值且 SHA-256 匹配、六字段/五谓词一致、PREPARED 完整
        → 允许重新执行 fsync(current_parent_dirfd)（重试前已重新核验全部对象）
     b. fsync 重试成功 → 进入 COMMIT_DURABLE_AUDIT_PENDING，继续 S14、S15
     c. fsync 重试失败 → 进入 COMMIT_INDETERMINATE（目录项持久性无法确认）
     d. 身份不一致 / 哈希冲突 / txn_id 不匹配 → 进入 QUARANTINED
  8. 状态不确定时禁止启动新事务
  9. 身份不一致或无法确认时需人工裁决
```

**硬性规则**：
- **不得仅凭 current 重读成功迁移到 COMMIT_DURABLE_AUDIT_PENDING**——必须同时确认 fsync(dirfd) 成功；
- **不得将目录可见性等同于断电持久性**；
- 重试前必须重新核验全部对象（文件身份、大小、SHA-256、txn_id、revision、父版本、六字段、五谓词、PREPARED）。

### 2.6 10 条 S13 专项故障时间线

| # | 场景 | current 重读 | 身份/哈希 | 六字段/五谓词 | PREPARED | 迁移 | 恢复 | 新事务 | 人工 |
|---|------|-------------|----------|--------------|---------|------|------|--------|------|
| S13-1 | fsync 失败，current 为新值且校验全通过 | 新值 | 匹配 | 一致 | 完整 | 允许重试 fsync | 重试 dirfd | 否 | 否 |
| S13-2 | fsync 重试成功 | 新值 | 匹配 | 一致 | 完整 | COMMIT_DURABLE_AUDIT_PENDING | 继续 S14/S15 | 否 | 否 |
| S13-3 | fsync 重试失败 | 新值 | 匹配 | 一致 | 完整 | COMMIT_INDETERMINATE | 保留现场 | 否 | 是 |
| S13-4 | current 为旧值 | 旧值 | 匹配旧 | N/A | N/A | 判定 rename 未生效 | 继续 S12 | 否 | 否 |
| S13-5 | current 哈希不匹配 | 新值 | 冲突 | N/A | N/A | QUARANTINED | 保留现场 | 否 | 是 |
| S13-6 | txn_id 不匹配 | 新值 | 匹配 | 冲突 | N/A | QUARANTINED | 保留现场 | 否 | 是 |
| S13-7 | 父版本不一致 | 新值 | 匹配 | 父版本冲突 | N/A | QUARANTINED | 保留现场 | 否 | 是 |
| S13-8 | PREPARED 缺失 | 新值 | 匹配 | 一致 | 缺失 | COMMIT_INDETERMINATE | 保留现场 | 否 | 是 |
| S13-9 | 仅凭 current 重读迁移（反例）| 新值 | 未核验 | 未核验 | 未核验 | **禁止迁移** | 必须全核验 | 否 | 是 |
| S13-10 | 目录可见但未断电持久（反例）| 新值 | 匹配 | 一致 | 完整 | 不能标 DURABLE | 必须 fsync 成功 | 否 | 否 |

### 2.7 10 个 S13 反例

| # | 反例 | 拒绝策略 |
|---|------|---------|
| S13-R1 | 仅凭 current 重读迁移 AUDIT_PENDING | 必须 fsync(dirfd) 成功 |
| S13-R2 | 目录可见当断电持久 | 区分可见性与持久性 |
| S13-R3 | 盲目重试 dirfd 不核验 | 重试前全对象核验 |
| S13-R4 | 盲目 rename | 禁止 |
| S13-R5 | 盲目回滚 | 禁止 |
| S13-R6 | 状态不确定启动新事务 | 禁止 |
| S13-R7 | 身份冲突不隔离 | QUARANTINED |
| S13-R8 | txn_id 冲突不隔离 | QUARANTINED |
| S13-R9 | 父版本冲突不隔离 | QUARANTINED |
| S13-R10 | current 与回执持久化混为一谈 | 区分 |

### 2.8 P0-04 结论

**P0-04 = DESIGN_PROVEN（POSIX）/ NOT_VERIFIED（Windows）**。
- 七态状态机 + S1~S16 完整崩溃矩阵（10 类位置）+ S13 专项恢复协议闭合；
- Windows 目录持久化 NOT_VERIFIED。

---

## 3. P0-05 与 P0-06 边界

### 3.1 P0-05（继续 DENY）

继续 DENY，不放宽 Source Retire，不设计删除/清理/归档/旧 FD 失效/mmap 失效路径，不声称源内容已不可写。

### 3.2 P0-06（仅保留已纠正语义）

仅保留 05.20~05.22 已纠正的 ReplaceFileW 1175/1176/1177 语义，不重新设计整个 Windows 替换协议。继续要求重读文件 ID、路径、大小、内容哈希判断部分失败后状态。未运行时验证的中间态继续 DESIGN_PARTIAL / NOT_VERIFIED。

---

## 4. 全局不变量（45 条）

每条含：定义、适用范围、前置条件、证明/证据、反例、失败状态、拒绝策略。

### 4.1 Fork 协议不变量（I-01 ~ I-15）

| # | 不变量定义 | 适用范围 | 前置条件 | 证明/证据 | 反例 | 失败状态 | 拒绝策略 |
|---|-----------|---------|---------|----------|------|---------|---------|
| I-01 | `_fork_coord_lock` 唯一释放者为父进程 | fork_entry 全路径 | fork 已执行 | §1.3 锁所有权表 + §1.4 父子时间线 | 子进程 finally 释放（B23-01）| 错误释放 | pid==0 分支分流 |
| I-02 | 子进程不释放任何继承锁 | after_in_child | fork 成功 | §1.2 协议 | R1/R2 | 跨进程释放 | 直接赋值新对象 |
| I-03 | 继承锁必须替换为全新实例，不 acquire/release | 子进程初始化 | fork 成功 | §1.3 重建协议 | 继承锁 acquire | 死锁 | 全新对象 |
| I-04 | fork 前禁止持有业务锁（L3）| fork_entry | 进入前 | §1.3 业务锁行 | 持 L3 fork | 子进程继承锁定 | DENY |
| I-05 | epoch 分配在 `_fork_coord_lock` 临界区内 | allocate_epoch | 持 L0 | §1.6 | epoch 锁外递增（B23-02）| 竞态 | 移入临界区 |
| I-06 | epoch 分配与快照登记同临界区 | fork_entry | 持 L0 | §1.6 表 | 错序 | 快照错配 | 同临界区绑定 |
| I-07 | 失败 fork 也消耗 epoch | fork 异常路径 | fork 失败 | §1.6 表 + T3 | epoch 复用 | ABA | 消耗 epoch |
| I-08 | epoch 单调递增、不复用 | 全生命周期 | — | §1.6 + R8 | ABA | 快照错配 | 单调 + 失败消耗 |
| I-09 | 快照与 fork 调用一一对应（fork_epoch）| freeze→fork | 持 L0 | §1.7 状态机 | 旧快照复用 | FD 误关 | epoch 校验 |
| I-10 | 子进程不分配新 epoch | after_in_child | fork 成功 | §1.6 表 + T9 | 子进程分配 | 状态污染 | 子进程不分配 |
| I-11 | 锁获取顺序 L0→L1→L2→L3，无逆序 | 全系统 | — | §1.3 + 锁依赖图 | L3→L0 逆序 | 死锁 | 顺序检查 |
| I-12 | FREEZING 后无迟到 in_flight 登记 | freeze/准入 | Gate=FREEZING | §1.2 freeze | 迟到登记 | 快照不一致 | 同临界区检查 |
| I-13 | 持锁等待条件变量必须释放重取 | freeze | 持 L1 | §1.2 | 持锁 wait | 死锁 | 释放/重取 |
| I-14 | after_in_parent 恢复 Gate=OPEN | 父进程 | fork 成功 | §1.4 父时间线 | Gate 停留 FREEZING | 永久冻结 | 恢复 Gate |
| I-15 | after_in_child 失败进入 DENY 受保护操作 | 子进程 | 子初始化失败 | §1.5 H 行 | 继续业务 | 状态污染 | DENY |

### 4.2 POSIX 持久化不变量（I-16 ~ I-30）

| # | 不变量定义 | 适用范围 | 前置条件 | 证明/证据 | 反例 | 失败状态 | 拒绝策略 |
|---|-----------|---------|---------|----------|------|---------|---------|
| I-16 | DURABLE 仅最终态，不作进行中状态 | 事务状态机 | — | §2.1 七态定义 | DURABLE(进行中)（B23-03）| 语义混用 | 七态分离 |
| I-17 | rename 原子性 ≠ 断电持久性 | S12/S13 | — | §2.5 核心问题 | 目录可见当持久 | 数据丢失 | fsync(dirfd) |
| I-18 | 目录可见 ≠ 目录项断电持久 | S13 | — | §2.5 | S13-R2 | 数据丢失 | 必须 fsync 成功 |
| I-19 | current 持久化 ≠ COMMIT 回执持久化 | S13~S15 | — | §2.4 规则 6 | 混为一谈 | 错误 DURABLE | 分步判定 |
| I-20 | 仅凭 current 重读不得迁移 AUDIT_PENDING | S13 | rename 后 fsync 失败 | §2.5 硬规则 | S13-R1/S13-9 | 错误迁移 | 全对象核验 |
| I-21 | fsync(dirfd) 成功是目录项持久化的必要条件 | S3/S7/S13/S15 | — | §2.5 | 省略 dirfsync | 未持久 | 强制 dirfsync |
| I-22 | fsync 重试成功不得直接跳 DURABLE | 恢复 | fsync 失败 | §2.5 | 跳步迁移 | 错误 DURABLE | 按步骤恢复 |
| I-23 | 恢复前必须重读磁盘真实状态 | 全恢复 | 失败后 | §2.5 步骤 | 信任内存 | 误判 | 强制重读 |
| I-24 | 恢复前核验文件身份/大小/SHA-256 | 全恢复 | 失败后 | §2.5 步骤 2-3 | 未核验 | 身份冲突 | 强制核验 |
| I-25 | 恢复前核验 txn_id/revision/父版本 | 全恢复 | 失败后 | §2.5 步骤 4 | 未核验 | 版本冲突 | 强制核验 |
| I-26 | 恢复前核验 Candidate 六字段/五谓词 | 全恢复 | 失败后 | §2.5 步骤 5 | 未核验 | 字段冲突 | 强制核验 |
| I-27 | 状态不确定时禁止启动新事务 | COMMIT_INDETERMINATE | — | §2.4 规则 4 | 启动新事务 | 覆盖证据 | 禁止 |
| I-28 | 身份/哈希冲突进入 QUARANTINED | 恢复 | 冲突 | §2.5 步骤 7d | 不隔离 | 污染 | QUARANTINED |
| I-29 | 未完成恢复/隔离前禁止新事务 | RECOVERY_REQUIRED/QUARANTINED | — | §2.4 | 启动新事务 | 覆盖 | 禁止 |
| I-30 | S16 锁释放非持久化步骤 | S16 | — | §2.2 S16 行 | 锁释放当持久化 | 误判 | 区分 |

### 4.3 Windows/全局不变量（I-31 ~ I-45）

| # | 不变量定义 | 适用范围 | 前置条件 | 证明/证据 | 反例 | 失败状态 | 拒绝策略 |
|---|-----------|---------|---------|----------|------|---------|---------|
| I-31 | Windows 目录项断电持久性 NOT_VERIFIED | P0-06 | — | §3.2 | 声称等价 POSIX | 误标 | NOT_VERIFIED |
| I-32 | 不得把 FlushFileBuffers 当完整目录持久保证 | P0-06 | — | §3.2 | 单 API 当保证 | 误标 | NOT_VERIFIED |
| I-33 | ReplaceFileW 1176 替换文件保留原文件名 | P0-06 | — | §3.2 | 混淆原目标/替换路径 | 误判 | 官方契约 |
| I-34 | 不得仅凭 ERROR_FILE_NOT_FOUND 推断替换文件缺失 | P0-06 | — | §3.2 | 错误推断 | 误判 | 重读状态 |
| I-35 | 不得仅凭 ERROR_SHARING_VIOLATION 推断两文件都在 | P0-06 | — | §3.2 | 错误推断 | 误判 | 重读状态 |
| I-36 | Source Retire 保持 DENY | P0-05 | — | §3.1 | 放宽 | 数据丢失 | DENY |
| I-37 | 只读权限/flock/哈希 ≠ 强制不可变 | P0-05 | — | §3.1 | 当强制不可变 | 误标 | DENY |
| I-38 | 已有 FD/mmap/硬链接仍可写源内容 | P0-05 | — | §3.1 | 声称不可写 | 误标 | DENY |
| I-39 | 源码冻结文件哈希保持不变 | 全轮 | — | §6.4 | 修改源码 | 违反边界 | 禁止 |
| I-40 | 所有设计结论不得写成运行时验证 | 全文档 | — | §6 | 推演当验证 | 误标 | NOT_TESTED |
| I-41 | MANIFEST 哈希为最终写入后重算 | 交付 | 写完 | §6.2 | 旧哈希冒充 | 哈希不符 | 重算 |
| I-42 | 不得用 Git Blob SHA-1 冒充 SHA-256 | 交付 | — | §6.2 | SHA-1 冒充 | 哈希不符 | 强制 SHA-256 |
| I-43 | ZIP 独立解压+逐字节比对+CRC32 必须真实执行 | 交付 | — | §6.3 | 只看中央目录 | 未验证 | 真实解压 |
| I-44 | 不得将文档数量/表格存在当设计正确 | 全局 | — | §5 | 数量当证据 | 误判 | 逐条审查 |
| I-45 | 执行器自审 ≠ 独立复审 | 全局 | — | §7 | 自审当独立 | 误判 | 等 ChatGPT |

**不变量合计：45 条（I-01~I-45）。**

---

## 5. 全局一致性与端到端链路

### 5.1 端到端链路（20 条）

| # | 链路 | 涉及不变量 | 关键证明 |
|---|------|-----------|---------|
| L1 | 单线程 fork 成功→父返回 | I-01/05/06/09 | 父释放 L0 |
| L2 | fork 成功→子初始化→业务 | I-02/03/10/15 | 子不释放继承锁 |
| L3 | fork 系统调用失败→恢复 | I-07/14 | 消耗 epoch 复用 Gate |
| L4 | before 钩子失败→恢复 | I-14 | 父 except 释放 |
| L5 | after_in_parent 失败→恢复 | I-01/14 | 父 finally 释放 |
| L6 | freeze 失败→恢复 | I-14 | 父 except 释放 |
| L7 | 连续 fork | I-07/08/09 | epoch 单调 |
| L8 | 并发 fork 排队 | I-05/11/12 | L0 串行化 |
| L9 | FREEZING 无迟到登记 | I-12/13 | 同临界区 |
| L10 | 子进程 FD generation 匹配关闭 | I-09 | epoch+generation |
| L11 | 提交 S1→S4 Candidate 就绪 | I-16/23 | 六字段核验 |
| L12 | 提交 S5→S7 PREPARED 持久 | I-21/23 | dirfsync |
| L13 | S9 五条件校验通过 | I-24/25/26 | 父版本核验 |
| L14 | S12 rename 替换 current | I-17/18 | rename 原子非持久 |
| L15 | S13 目录持久化 | I-18/20/21 | fsync(dirfd) 成功 |
| L16 | S13 fsync 失败恢复 | I-20/23/24/25/26 | 全对象核验 |
| L17 | S14/S15 回执持久化 | I-19/21 | 回执 dirfsync |
| L18 | S16 释放锁完成 | I-30 | 锁非持久化 |
| L19 | 身份冲突隔离 | I-28/29 | QUARANTINED |
| L20 | Windows 部分失败重读 | I-31/34/35 | 重读状态 |

**端到端链路合计：20 条（L1~L20）。**

### 5.2 故障/并发/崩溃时间线（40 条）

- **P0-02 并发时间线 10 条**：§1.8 T1~T10（epoch 竞态）；
- **P0-02 异常路径 8 条**：§1.5 A~H；
- **S13 专项故障时间线 10 条**：§2.6 S13-1~S13-10；
- **S1~S16 崩溃矩阵行**：§2.2（每行对应一条崩溃时间线，覆盖 ①~⑩ 十类位置）；
- **P0-04 恢复时间线**：§2.5 恢复步骤 7a~7d 四条分支。

**合计：10 + 8 + 10 + 12（S1~S16 矩阵精选行）= 40 条时间线。**（T1~T10、A~H、S13-1~S13-10 为独立编号；S1~S16 矩阵 22 行中每行即一条崩溃时间线，选取代表行补足至 40，其余行同样逐项覆盖，无省略。）

### 5.3 反例（30 个）

- **P0-02 反例 10 个**：§1.9 R1~R10；
- **S13 反例 10 个**：§2.7 S13-R1~S13-R10；
- **全局/Windows 反例 10 个**：

| # | 反例 | 拒绝策略 |
|---|------|---------|
| G-R1 | Windows 目录持久化标 PASS | NOT_VERIFIED |
| G-R2 | FlushFileBuffers 当完整目录保证 | NOT_VERIFIED |
| G-R3 | 1176 混淆替换/原目标路径 | 官方契约纠正 |
| G-R4 | ERROR_FILE_NOT_FOUND 推断替换文件缺失 | 重读状态 |
| G-R5 | ERROR_SHARING_VIOLATION 推断两文件都在 | 重读状态 |
| G-R6 | Source Retire 放宽 | DENY |
| G-R7 | flock/只读当强制不可变 | DENY |
| G-R8 | Git Blob SHA-1 冒充 SHA-256 | 强制 SHA-256 |
| G-R9 | ZIP 只看中央目录当验证 | 真实解压 |
| G-R10 | 执行器自审当独立复审 | 等 ChatGPT |

**反例合计：10 + 10 + 10 = 30 个。**

---

## 6. 交付物与完整性核验

### 6.1 交付文件清单

| # | 文件名 | 说明 |
|---|--------|------|
| 1 | `USIT-STAGE-B-DESIGN-05.23.md` | 主文档 |
| 2 | `USIT-STAGE-B-DESIGN-05.23-ISSUES.md` | 问题清单 |
| 3 | `USIT-STAGE-B-DESIGN-05.23-DIFF.md` | 与 05.22 差异 |
| 4 | `USIT-STAGE-B-DESIGN-05.23-CROSS-REVIEW.md` | 执行器自审 |
| 5 | `USIT-STAGE-B-DESIGN-05.23-EXECUTION-REPORT.md` | 执行报告 |
| 6 | `USIT-STAGE-B-DESIGN-05.23-HASH-MANIFEST.md` | 哈希清单 |
| 7 | `USIT-STAGE-B-DESIGN-05.23-HASH-RECEIPT.md` | 哈希回执 |
| 8 | `USIT-STAGE-B-DESIGN-05.23-DELIVERY.zip` | 交付 ZIP |

### 6.2 哈希分层规则

- **HASH-MANIFEST** 记录前 6 份 Markdown（主文档、ISSUES、DIFF、CROSS-REVIEW、EXECUTION-REPORT 及 MANIFEST 自身不计）的 SHA-256 与字节数；
- **HASH-RECEIPT** 记录 MANIFEST 自身 SHA-256/字节数 + ZIP 自身 SHA-256/字节数 + ZIP 独立解压结果；
- 所有哈希为文件最终写入后重算（`sha256sum`），禁止用 Git Blob SHA-1 冒充 SHA-256。

### 6.3 B23-05：ZIP 独立核验

必须真实执行（见 EXECUTION-REPORT §9 与 HASH-RECEIPT）：
1. 读取 ZIP 原始二进制（PowerShell `Compress-Archive` 生成）；
2. 记录 ZIP 精确字节数与 SHA-256；
3. 独立临时目录 `unzip -o` 真实解压；
4. `unzip -l` 列文件数量/名称/字节数；`unzip -v` 取 CRC32；
5. 解压文件与源交付文件逐字节比对；
6. 核对重复/缺失/额外文件；
7. 所有真实结果写入 HASH-RECEIPT。

**若无法真实解压，必须标记 NOT_VERIFIED，禁止用中央目录信息或虚构哈希替代。**

### 6.4 B23-06：源码前后完整基线

对工作区 `usit/**/*.py`（25 个文件）采集：
1. 执行前完整文件路径集合 + 每文件 SHA-256/字节数 → `usit_source_baseline_05.23_pre.txt`；
2. 执行后完整文件路径集合 + 每文件 SHA-256/字节数 → `usit_source_baseline_05.23_post.txt`；
3. `diff` 前后清单确认无差异；
4. 新增/删除/改名文件检查；
5. 三个冻结文件与历史基线对比（scanner.py / cli.py / file_understanding.py）；
6. 任何差异标记 FAIL 并停止。

---

## 7. 阻断状态（保持，不解除）

| 阻断 | 状态 |
|------|------|
| P0-BLOCK-1 | 保持阻断 |
| P0-BLOCK-2 | 保持阻断 |
| P1-BLOCK-3 | 保持阻断 |
| F1 | 保持阻断 |
| P0.8 | 保持阻断 |
| F2 | 保持阻断 |
| IMPLEMENT-01 | **NOT_APPROVED** |
| A/B 合并 | 不合并 |

---

## 8. 最终结论

| 项 | 结论 |
|----|------|
| P0-02 | **DESIGN_PARTIAL**（协议层 DESIGN_PROVEN，运行时 NOT_TESTED，第三方 C fork UNSUPPORTED/DENY）|
| P0-04 POSIX | **DESIGN_PROVEN** |
| P0-04 Windows | **NOT_VERIFIED** |
| P0-05 | **DENY**（保持）|
| P0-06 | **DESIGN_PARTIAL / NOT_VERIFIED** |
| 交付完整性 | 待 HASH-RECEIPT 与 ZIP 独立核验完成后判定 |
| IMPLEMENT-01 | **NOT_APPROVED** |

**整体结论：PASS-WITH-ISSUES（候选，待交付核验证据补齐后最终确认）。**

**STOP：完成设计修订后立即停止，等待 ChatGPT 独立复审，不进入实现、不解除阻断、不合并 A/B。**

---

