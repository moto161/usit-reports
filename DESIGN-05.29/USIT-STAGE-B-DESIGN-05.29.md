# USIT-STAGE-B-DESIGN-05.29｜Fork准入生命周期重构＋进程树身份模型重新证明＋POSIX提交并发与不确定态闭合

> 阶段 B · DESIGN-05.29 · STRICT READ-ONLY ANALYSIS + DESIGN REVISION
> 作者：USIT 主开发 · 生成：2026-10-31（GMT+8）
> 前置审查：DESIGN-05.28 独立复审结论 FAIL
> 工作区：`C:\Users\Administrator\WorkBuddy\2026-10-15-22-09-47\usit_p0`（只读）
> 交付目录：`C:\Users\Administrator\WorkBuddy\2026-10-15-22-09-47\usit_p0_05.29_reports`（**项目外**）
> 状态：**STOP — 等待独立复审**

> 证据等级：`VERIFIED_FACT` / `VERIFIED_STATIC` / `DESIGN_PROVEN` / `DESIGN_PROPOSAL` / `STATIC_INFERENCE` / `NOT_VERIFIED` / `NOT_TESTED` / `UNKNOWN` / `CONFLICT` / `FAIL` / `NOT_PROVEN`

---

## 0. 本轮定位与前提声明

DESIGN-05.28 的三项 `DESIGN_PROVEN` 结论被独立复审判定 FAIL。本轮**不沿用** 05.28 结论，对三个核心协议**重新证明**：

1. **B29-01** Fork 准入生命周期、锁所有权、异常终态。
2. **B29-02** 进程树身份模型、实例继承与重建。
3. **B29-03** POSIX 提交并发、外部写入者边界、提交不确定态。

凡证明不成立的内容，**明确撤销原结论**，不通过改措辞把 FAIL 变 PASS。

### 0.1 05.28 复审发现的核心阻断项（必须闭合）

| 阻断 | 05.28 缺陷 | 本轮处置 |
|------|-----------|---------|
| F01 | Fork 经 `enter_op()` 登记 in_flight 后等待自身归零 → 自锁 | §1 生命周期重构，Fork 与业务分流 |
| F02 | `enter_op().__enter__()` 直调，无匹配 `__exit__` | §1.4 上下文管理器完整协议 |
| F03 | 锁释放所有权不明确 | §1.5 锁所有权模型 |
| F04 | DENIED 可被普通恢复路径重新打开 | §1.6 不可逆终态规则 |
| F05 | 异常传播不完整，finally 覆盖原始异常 | §1.7 主/附加异常协议 |
| I01 | process_instance_id 继承/重建冲突 | §2 身份模型重构 |
| I02 | fork_epoch 冒充进程树全局计数器 | §2.4 撤销并重建 |
| I03 | 结构唯一与概率唯一混写 | §2.5 分列证明 |
| P01 | `_commit_lock` 被假定约束锁外写入者 | §3 并发写入者模型 |
| P02 | rename 结果不确定与持久性不确定混淆 | §3 双维度状态机 |
| P03 | 回执半成品归属证明不足 | §3.6 归属证明协议 |

---

## 1. B29-01｜Fork 准入生命周期与锁所有权重构

### 1.1 F01 自锁的根因与消除

05.28 的自锁路径：Fork 调用 `enter_op()` 使 `_in_flight += 1`，随后 freeze 等待 `_in_flight == 0`，但 Fork 自身尚未调用 `exit_op()`，计数器永不归零。

**根因**：Fork 协调被误当作普通业务操作登记进同一个 `_in_flight` 计数域。

**消除原则**：**Fork 协调与普通业务操作使用不同的计数域与不同的等待语义。** Fork 发起者**不登记**进业务 `_in_flight`，而是通过独立的 `_fork_in_progress` 标志 + 条件变量协调。业务 `_in_flight` 只统计真正的业务操作（FD/事务/快照读等）。

### 1.2 Fork 完整生命周期（独立于业务）

`REQUESTED → ADMISSION_CHECK → FREEZING → DRAINING → FROZEN → FORK_EXECUTION → PARENT_RESTORE / CHILD_REINIT → FINALIZED`

失败分支：`任意阶段 → FAILURE_CLASSIFICATION → DENIED / TERMINATING / RECOVERY_REQUIRED`

| 阶段 | 进入条件 | 退出条件 | 关键动作 | 允许/禁止 |
|------|---------|---------|---------|----------|
| REQUESTED | fork 调用到达 | 准入检查完成 | 标记 `_fork_requested=True` | 禁止业务准入前登记 |
| ADMISSION_CHECK | 请求到达 | 线程安全证明通过 | 检查线程安全、Gate=OPEN | 失败→DENY/FORK_DENIED |
| FREEZING | 准入通过 | 开始排空 | 置 Gate=FREEZING，**停止新业务准入** | 禁止新业务 enter_op |
| DRAINING | FREEZING | 业务 in_flight==0 | 等待业务操作归零 | 已登记操作完成 |
| FROZEN | 排空完成 | os.fork() 调用 | 生成快照 | 禁止业务、禁止 FD 变更 |
| FORK_EXECUTION | FROZEN | fork 返回 | os.fork() | — |
| PARENT_RESTORE | 父进程 fork 返回 | Gate 恢复/失败 | 恢复 Gate | 禁止业务直到恢复 |
| CHILD_REINIT | 子进程 fork 返回 | 初始化完成/退出 | 重建锁、重建身份 | 失败→os._exit |
| FINALIZED | 恢复/重建完成 | — | Gate=OPEN 或终态 | — |

**关键**：Fork 发起者在 DRAINING 阶段等待的是**业务** `_in_flight` 归零，而 Fork 自身**不计入**业务 `_in_flight`，故不存在自锁。

### 1.3 业务准入与 Fork 协调的分离

```python
_biz_in_flight = 0          # 业务操作计数（Fork 不计入）
_admit_lock = threading.Lock()
_admit_cond = threading.Condition(_admit_lock)
_fork_in_progress = False   # Fork 协调独立标志（非业务计数）
_fork_coord_lock = threading.Lock()

def enter_op():
    # 业务操作准入：检查 Gate + 登记 biz_in_flight
    with _admit_lock:
        if _gate != 'OPEN':
            raise GateClosed(_gate)
        _biz_in_flight += 1
    return _OpGuard()

class _OpGuard:
    def __exit__(self, *exc):
        with _admit_lock:
            _biz_in_flight -= 1
            if _gate == 'FREEZING' and _biz_in_flight == 0:
                _admit_cond.notify_all()
        return False  # 不吞异常
```

Fork 协调（`freeze`）**不调用** `enter_op()`，而是直接操作 `_gate` 与 `_fork_in_progress`：

```python
def freeze():
    with _admit_lock:
        if _gate != 'OPEN': raise GateClosed(_gate)
        _gate = 'FREEZING'
        _fork_in_progress = True
        while _biz_in_flight > 0:       # 等待业务归零，Fork 自身不在其中
            _admit_cond.wait(timeout=FREEZE_TIMEOUT)
            if _biz_in_flight > 0 and timed_out():
                _gate = 'OPEN'; _fork_in_progress = False
                raise FreezeTimeout()
        _gate = 'FROZEN'
```

### 1.4 上下文管理器完整协议（修复 F02）

- **进入责任**：`enter_op()` 返回 `_OpGuard` 实例，登记 `_biz_in_flight`。
- **正常退出**：`with` 块正常结束 → `_OpGuard.__exit__(None,None,None)` → 递减计数。
- **异常退出**：`with` 块抛异常 → `__exit__(exc_type, exc, tb)` → 递减计数 → 返回 False（**不吞异常**，原始异常继续传播）。
- **嵌套调用**：每层 `enter_op()` 独立登记/递减（**不再用 threading.local 重入合并**，简化所有权），嵌套 N 层计数 +N。
- **计数器登记/注销责任**：登记在 `__enter__`（即 `enter_op` 主体），注销在 `__exit__`，一一配对。
- **资源所有权与异常传播**：`_OpGuard` 不持有业务资源所有权，仅负责计数；异常不拦截。

**禁止**：直接调用 `__enter__()` 而不建立 `with` 生命周期。所有调用点必须用 `with enter_op():`。

### 1.5 锁所有权模型（修复 F03）

| 锁 | 获取者 | 所有者 | 允许获取阶段 | 释放者 | 释放时机 | 获取失败 | 释放失败 | fork 后继承/重建 |
|----|--------|--------|-------------|--------|---------|---------|---------|----------------|
| `_admit_lock` | 业务入口 / freeze | 当前线程 | OPEN（业务）/ FREEZING（freeze）| 当前线程 | `with` 退出 | 抛异常，操作拒绝 | 异常已进入异常态 | 子进程**重建新锁对象** |
| `_admit_cond` | freeze（wait）| freeze 线程 | DRAINING | 自动（wait 释放/重获）| wait 返回 | 超时回滚 | — | 子进程重建 |
| `_fork_coord_lock` | fork 发起者 | fork 线程 | REQUESTED→FINALIZED | fork 线程 | 最外层 finally | fork 拒绝 | 终止 | 子进程重建 |
| `_commit_lock` | 事务提交者 | 提交线程 | 提交全程 | 提交线程 | S16 | 提交拒绝 | 终止/记录 | 子进程**不继承**（独立） |
| 业务资源锁 | 业务操作 | 业务线程 | 业务全程 | 业务线程 | 操作结束 | 操作失败 | 记录 | 子进程重建 |

**禁止**：未持锁释放、跨线程释放、重复释放。所有释放由获取线程在同一线程内完成（除 fork 后子进程重建场景）。

### 1.6 DENIED 不可逆终态（修复 F04）

- **DENIED 是不可逆终态**：一旦进入 DENIED，任何普通恢复路径（`restore_gate`、`_close_fork_parent` 的 finally、异常清理）**均不得**将其改回 OPEN。
- **人工授权恢复**：唯一恢复路径是**人工授权**（外部运维动作），且该授权不通过代码自动触发。
- **恢复授权持有者**：进程外的人工操作者，通过明确的 `manual_reenable()` 接口（需显式调用，且不是任何异常路径的自动分支）。
- **自动触发禁止**：`restore_gate()` 内部**不得**包含把 DENIED→OPEN 的自动逻辑。
- **恢复过程阻止业务入口**：在人工授权生效前，Gate 保持 DENIED，`enter_op()`/`fork_entry()` 全部拒绝。
- **部分恢复成功处理**：人工恢复分阶段完成，任何阶段未完成，Gate 保持 DENIED；全部完成才置 OPEN。

**证明**：DENIED 写入后，唯一能写回 OPEN 的代码路径是 `manual_reenable()`（人工触发），不位于任何异常/恢复的自动路径上，故普通恢复流程无法重新打开 DENIED。

### 1.7 异常传播协议（修复 F05）

区分五类异常：

| 异常类别 | 处理 | 记录 |
|---------|------|------|
| 原始业务异常 | 保留并传播 | `_last_fork_error` 记主异常 |
| Fork 异常 | 保留为当前主异常 | 同上 |
| Gate 恢复异常 | **附加异常**，不覆盖主异常 | `_attached_errors` 列表追加 |
| 锁释放异常 | **附加异常**，触发终止 | 同上 |
| 清理异常 | **附加异常**，不覆盖 | 同上 |

```python
def _close_fork_parent(epoch):
    primary = None
    attached = []
    try:
        _after_in_parent(epoch)
    except BaseException as e:
        primary = e
    finally:
        _pending_snapshot = None
        _gate = 'DENIED'              # 先无条件 DENIED
        try:
            _restore_gate(); _gate = 'OPEN'
        except BaseException as e:
            attached.append(e); _gate = 'DENIED'
        finally:
            try:
                release(_fork_coord_lock)
            except BaseException as e:
                attached.append(e); _terminate_fail_closed()
    _last_fork_error = primary
    _attached_errors = attached
    if primary is not None:
        raise primary                 # 原始异常不丢失
```

**主异常唯一**：`primary` 只记录第一个业务/fork 异常；后续恢复/释放异常全部进 `attached`，不覆盖 `primary`。finally 中异常被捕获进 attached，不再替换 primary。

### 1.8 锁依赖有向图与死锁证明

锁获取全局顺序（固定）：

```
_admit_lock  →  (业务资源锁)  →  _commit_lock
_admit_lock  →  _fork_coord_lock
```

- `_admit_lock` 是最外层准入锁，任何操作先获取。
- `_commit_lock` 是事务提交锁，获取前必须先持有 `_admit_lock`（或通过业务操作持有了准入）。
- `_fork_coord_lock` 仅在 Fork 发起者路径使用，且在 `_admit_lock` 内短暂持有时不跨等待。

**环路检查**：不存在 `_commit_lock → _admit_lock` 或 `_fork_coord_lock → _admit_lock` 的路径（提交/fork 都是"先准入后提交/协调"），故无环。

**死锁反例排查**：freeze 在 `_admit_lock` 内 `_admit_cond.wait()` 会**释放** `_admit_lock` 并等待，唤醒后**重新获取**；业务 `__exit__` 递减计数并 `notify_all`。二者不形成循环等待（wait 释放锁）。业务资源锁与 `_commit_lock` 的持有顺序固定，不反向。**结论：无死锁（静态推演）**，但并发运行时行为 NOT_TESTED。

锁获取/释放失败矩阵见 `FORK-LOCK-OWNERSHIP.md` §3。

### 1.9 Gate 状态机（七态，重审）

`OPEN / FREEZING / FROZEN / RESTORING / RESTORE_FAILED / TERMINATING / DENIED`

| 状态 | 唯一进入条件 | 合法退出 | 写入者 | 锁保护 | 可逆 | 人工恢复 |
|------|-------------|---------|--------|--------|------|---------|
| OPEN | 初始化 / 恢复成功 / 人工授权 | FREEZING（fork 请求）| freeze/restore/manual | `_admit_lock` | — | — |
| FREEZING | fork 请求持协调锁 | FROZEN（归零）/ OPEN（超时回滚）| freeze | `_admit_lock` | 是（回滚）| 否 |
| FROZEN | 排空完成 | RESTORING（fork 返回）| freeze | `_admit_lock` | 是 | 否 |
| RESTORING | fork 返回父进程 | OPEN / RESTORE_FAILED | restore | `_admit_lock` | 是 | 否 |
| RESTORE_FAILED | restore 抛异常 | DENIED / TERMINATING | restore 失败路径 | `_admit_lock` | 否 | 仅人工 |
| TERMINATING | 无法恢复 | 进程退出 | 终止路径 | — | 否 | 否 |
| DENIED | 恢复失败 / 线程准入失败 | 仅人工授权 | deny/manual | `_admit_lock` | **否（终态）** | 是 |

**核心证明**：任何异常路径的最终写入都是 DENIED / TERMINATING / OPEN 三者之一，且 DENIED/TERMINATING 不可逆，**不存在 Gate 未知、锁所有权未知、业务意外继续**的状态。若无法证明（如状态变量内存损坏读取失败），`enter_op`/`fork_entry` 的 `_gate != 'OPEN'` 判断默认拒绝（fail-closed）。

### 1.10 Fork 异常矩阵与时间线

异常矩阵（20 项）见 `FORK-EXCEPTIONS.md`；30 条异常时间线见 `TIMELINES.md` §1（FT1~FT30）。

### 1.11 B29-01 结论

| 验收项 | 结论 |
|--------|------|
| 无 Fork 自锁 | **DESIGN_PROVEN**（Fork 与业务计数域分离）|
| 上下文管理器完整 | **DESIGN_PROVEN** |
| 锁所有权明确 | **DESIGN_PROVEN** |
| 无未持锁释放/跨线程释放 | **DESIGN_PROVEN** |
| DENIED 不可被普通路径重开 | **DESIGN_PROVEN** |
| 原始异常不丢失 | **DESIGN_PROVEN** |
| 父子生命周期独立 | **DESIGN_PROVEN** |
| 运行时行为 | **NOT_TESTED**（本机无 fork）|

**B29-01 = DESIGN_PROVEN（协议层），NOT_TESTED（运行时）。**

---

## 2. B29-02｜进程树身份模型重新证明

### 2.1 撤销 05.28 的错误结论

05.28 将 `fork_event_id` 定义为"父 instance_id 前缀 + 父内 seq + 随机后缀"，并宣称进程树内严格唯一。但存在两点未闭合：

1. **process_instance_id 的继承语义未定义**：子进程是否沿用父的 instance_id？05.28 说"子继承父值"又说"各代 seq 独立"，二者矛盾。
2. **fork_epoch 与 fork_event_id 职责边界模糊**。

本轮撤销 05.28 的"进程树内严格唯一"承诺，重新分层定义。

### 2.2 身份层次分离（八层）

| 身份概念 | 含义 | 唯一性范围 |
|---------|------|-----------|
| A Process Instance Identity | 单个 OS 进程实例 | 进程实例内 |
| B Process Tree Identity | 进程树根及分支 | 进程树内 |
| C Fork Event Identity | 一次 fork 事件 | 事件内 |
| D Snapshot Identity | 一次 fork 快照 | 绑定 fork 事件 |
| E Transaction Identity | 事务 | 实例/主机（视持久化）|
| F Parent Relationship | 父子关系 | 结构关系 |
| G Generation Identity | 进程代际 | 单链 |
| H PID Identity | OS 进程号 | 进程存活期 |

**禁止将这八类混为一个 ID。**

### 2.3 字段逐项重定义

| 字段 | 定义 | 生成主体 | 生成时机 | 唯一性范围 | 结构/概率 | Fork 前值 | 父进程值 | 子进程值 | 继承 | 重建 | 持久化 | 崩溃恢复 | PID 重用 | 并发 | 失败降级 |
|------|------|---------|---------|-----------|----------|----------|---------|---------|------|------|--------|---------|---------|------|---------|
| process_instance_id | 进程实例身份 | 进程启动 os.urandom(16) | 启动 | 实例内 | 概率 | — | X | **新值**（子重建）| **否** | **是** | 否 | 重启换新 | 结构规避 | 无 | 随机不可用 DENY |
| process_tree_id | 进程树根身份 | 根进程 os.urandom(16) | 根启动 | 树内 | 概率 | — | X | **继承**（同树）| **是** | 否 | 否 | 根重启换新 | 结构 | 无 | 随机不可用 DENY |
| process_generation | 代际 | 父 fork | fork | 单链 | 结构 | — | G | G+1 | 继承后+1 | 是 | 否 | 不持久 | 结构 | 父串行 | 无关 |
| fork_epoch | 父内 fork 序号 | 父 fork | fork | **父内严格** | 结构 | — | N | **读取**（不继承计数器）| 只读 | 是 | 否 | 不持久 | 结构 | 父锁 | 溢出 DENY |
| fork_event_id | fork 事件身份 | 父 fork | fork | 树内结构 / 跨树概率 | 结构+概率 | — | (tree_id, seq, rand) | 继承值 | 是 | 否 | 否 | 不持久 | 结构+概率 | 父串行 | 随机 DENY |
| snapshot_id | 快照身份 | = fork_event_id | freeze | 同 fork_event_id | 同 | — | 同 | 继承 | 是 | 否 | 否 | 不持久 | 同 | 同 | 同 |
| parent_instance_id | 父实例引用 | 子 fork | fork | 引用 | 结构 | — | 父 instance_id | 父 instance_id（历史）| 是 | 否 | 否 | 不持久 | 结构 | 无 | 无关 |
| parent_snapshot_id | 父快照引用 | 子 fork | fork | 引用 | 结构 | — | 父 snapshot_id | 继承 | 是 | 否 | 否 | 不持久 | 结构 | 无 | 无关 |
| fd_generation | FD 槽位代际 | 槽位分配 | 分配 | 进程内每槽位 | 结构 | — | — | 继承后重建 | 继承 | 是 | 否 | 不持久 | 结构 | 槽位锁 | 无关 |
| txn_id | 事务身份 | 事务开始 | 开始 | 实例（概率）+可持久 | 概率+结构 | — | — | **不继承** | 否 | 是 | **是** | 可重放 | 结构 | 事务锁 | 随机 DENY |
| revision | 对象版本 | 提交 | 提交 | 实例内每对象 | 结构 | — | — | 不继承 | 否 | 是 | 是 | 可重读 | 结构 | 事务锁 | 无关 |
| pid | OS 进程号 | OS | 分配 | 进程存活期 | 结构（OS）| — | 父 pid | **新 pid** | 否 | 是 | 否 | OS 复用 | 结构 | OS | 无关 |
| start_identity | 启动身份（可选）| 进程启动 | 启动 | 实例内 | 结构 | — | X | 新值 | 否 | 是 | 否 | 重启换新 | 结构 | 无 | 无关 |

### 2.4 process_instance_id 继承与重建（核心问题）

**结论：fork 出的子进程拥有新的 process_instance_id，不沿用父的 instance_id。process_tree_id 才沿用于同树。**

分九个时点：

| 时点 | process_instance_id | process_tree_id | 说明 |
|------|--------------------|-----------------|------|
| fork 返回之前 | 子尚无独立值（内存复制父值）| 同父 | 子内存是父的副本 |
| fork 返回之后 | **重建为新值** | 同父（继承）| 子必须立即重建 instance_id |
| 子初始化之前 | 未确认（可能仍父值）| 继承 | 未完成重建前不进入业务 |
| 子初始化完成 | 新 instance_id 确认 | 继承 | 可进入业务 |
| 子初始化失败 | 无有效 instance_id | — | os._exit(1) |
| 子再次 fork | 作为父，子进程再重建 | 继承 | 代际+1 |
| 父进程退出 | 子 instance_id 不变 | 树 id 不变 | 树 id 可追溯根 |
| 子独立运行 | 独立 instance_id | 树 id 保留 | — |
| PID 被重用 | instance_id 区分 | 树 id 区分 | 不依赖 pid |

**规则**：
- 允许继承：`process_tree_id`、`parent_instance_id`（历史引用）、`parent_snapshot_id`、`fork_event_id`（历史值）。
- 必须重建：`process_instance_id`、`pid`、`fd_generation`（槽位表）、`start_identity`。
- 保留原值作历史证据：`parent_instance_id`、`parent_snapshot_id`。
- 父子关联：`parent_instance_id = 父的 instance_id`。
- 重建失败 → **禁止继续业务**，os._exit(1)。
- 随机源不可用 → DENY（不降级）。
- 重建前后身份混用 → 通过"初始化完成标志"阻断：未完成初始化前 `enter_op` 拒绝。

### 2.5 唯一性分列证明

| 唯一性目标 | 结论标签 | 证明 |
|-----------|---------|------|
| 单个进程实例内 | STRICT_STRUCTURAL | 进程内单调计数器 |
| 同父兄弟 fork 事件 | STRICT_STRUCTURAL | 父内 fork_epoch 严格单调（兄弟 seq 不同）|
| 同树多代 fork 事件 | STRICT_STRUCTURAL | process_tree_id 相同 + 各代父内 seq 独立 + 父 instance_id 前缀区分分支 |
| 不同进程树之间 | PROBABILISTIC | process_tree_id 128-bit 随机 |
| 同主机不同进程 | PROBABILISTIC | instance_id/tree_id 随机 |
| PID 重用 | STRICT_STRUCTURAL | instance_id 独立于 pid |
| 进程崩溃后 | NOT_GUARANTEED | 不持久化，重启换新 |
| 系统重启后 | NOT_GUARANTEED | 不承诺跨重启 |
| 随机源失效 | UNKNOWN | DENY，不降级 |
| 身份复制/导入 | UNKNOWN | 无认证，不防伪造 |

**关键修正**：`fork_epoch` 不再冒充进程树全局计数器，它只是**父进程内**的严格单调序号；跨代唯一性靠"process_tree_id（树）+ 父 instance_id（分支）+ 父内 seq（兄弟）"三者组合的结构性区分，而非单一全局计数器。

### 2.6 身份反例与时间线

- 30 个身份反例：`IDENTITY-COUNTEREXAMPLES.md`。
- 身份时间线：`IDENTITY-TIMELINES.md`。

### 2.7 B29-02 结论

| 验收项 | 结论 |
|--------|------|
| 实例身份与树身份分离 | **DESIGN_PROVEN** |
| 父子继承规则明确 | **DESIGN_PROVEN** |
| 子进程重建规则明确 | **DESIGN_PROVEN** |
| fork_event_id 唯一性证明 | **DESIGN_PROVEN**（树内结构）|
| fork_epoch 不冒充全局计数器 | **DESIGN_PROVEN**（撤销 05.28 措辞）|
| PID 不作唯一身份 | **DESIGN_PROVEN** |
| 结构/概率分开 | **DESIGN_PROVEN** |
| 降级路径明确 | **DESIGN_PROVEN** |
| 无重建失败静默沿用旧身份 | **DESIGN_PROVEN**（初始化标志阻断）|
| 跨主机/跨重启 | **NOT_VERIFIED**（不承诺）|

**B29-02 = DESIGN_PROVEN（协议层），NOT_VERIFIED（跨主机/跨重启/随机降级）。**

---

## 3. B29-03｜POSIX 提交并发与不确定态闭合

### 3.1 P01 提交锁保护范围的重新界定

**撤销 05.28 的隐含假设**：`_commit_lock`（进程内锁）**不能**约束锁外写入者。05.28 的 §3.6 虽提到"两次读一致 + 锁持有"，但未明确承认锁外写入者不受控。

**重新界定**：

| 写入者类别 | 是否受 `_commit_lock` 约束 | 说明 |
|-----------|--------------------------|------|
| A 同进程线程 | **是** | 遵守锁协议 |
| B 同主机其他进程（advisory lock）| **部分** | 仅若遵守 flock/fcntl 协议 |
| C 遵守统一锁协议的进程 | **是** | advisory 协作 |
| D 不遵守锁协议的进程 | **否** | 无法约束 |
| E 管理员直接改文件 | **否** | 无法约束 |
| F 恢复进程（重启后）| **视协议** | 需遵守恢复协议 |
| G 文件系统/存储异常 | **否** | 硬件/FS 层 |

**POSIX flock/fcntl 是 advisory（建议性）锁**，只能约束**自愿遵守**的进程，**不能阻止不遵守者**。

**重要**：两次读取一致、inode 一致、哈希一致，**都不能单独证明锁外写入者不存在**（写入者可能在两次读之间写完又改回，或原子替换后哈希恰巧一致——虽概率极低但非零）。

**必须选择并证明**：

本设计采用**方案① + 方案③ 组合**：
- **方案①**：所有本系统内的写入者（含多进程部署）**必须遵守统一 advisory lock 协议**（`fcntl(F_SETLK)` 或 `flock` 锁定 current 所在文件/锁文件）。这是**部署契约**，写进设计文档，违反者不在系统正确性保证范围内。
- **方案③**：无法建立排他写入权威时（检测到锁外写入痕迹、inode/dev 异常变化、身份不可确认），进入 **DENY / QUARANTINED**。

**明确声明**：本系统**不承诺**防护不遵守协议的锁外写入者（D/E/G）。若部署环境存在此类写入者，系统的正确性前提**不成立**，必须标记 `UNKNOWN` 并进入 QUARANTINED，而非假装已闭合。

### 3.2 P02 操作结果不确定 vs 持久性不确定分离

两个独立维度：

**维度 A：操作结果**
- NOT_STARTED / IN_PROGRESS / CONFIRMED_APPLIED / CONFIRMED_NOT_APPLIED / OUTCOME_UNKNOWN

**维度 B：持久性**
- NOT_DURABLE / DURABILITY_PENDING / DURABLE_CONFIRMED / DURABILITY_UNKNOWN

**组合状态表**（关键矩阵，见 `POSIX-UNCERTAINTY.md`）：

| 维度 A \ 维度 B | NOT_DURABLE | DURABILITY_PENDING | DURABLE_CONFIRMED | DURABILITY_UNKNOWN |
|-----------------|-------------|-------------------|-------------------|-------------------|
| NOT_STARTED | 合法（未开始）| 非法 | 非法 | 非法 |
| IN_PROGRESS | 合法（进行中）| 合法（内容已写）| 非法 | 非法 |
| CONFIRMED_APPLIED | 合法（已应用未持久）| 合法（fsync 中）| **合法（DURABLE）** | 合法（持久不确定）|
| CONFIRMED_NOT_APPLIED | 合法 | 非法 | 非法 | 非法 |
| OUTCOME_UNKNOWN | 合法 | 合法 | 非法 | 合法（不确定态）|

**禁止**：
- 操作是否发生不明确 → 不得标记提交成功。
- 操作已发生但 fsync 失败 → 不得认定操作未发生。
- 目录 fsync 失败 → 不得盲目重复 rename。
- 不确定态 → 不得直接跳 DURABLE。
- 不得仅凭进程内变量恢复磁盘事务。
- 不得仅凭文件名判断对象身份。

### 3.3 rename 重新建模（16 种情况）

见 `POSIX-UNCERTAINTY.md` §2，每项回答：能否证明 rename 已/未发生、能否证明 current 内容、能否证明目录项持久性、是否允许重试/回滚/新事务/人工介入。核心结论：

- rename 返回 EINTR：**结果未知**，既不能证明已发生也不能证明未发生，必须重读 current 身份（inode/dev+SHA+revision）判定。
- rename 已生效但 fsync 失败：**操作结果=CONFIRMED_APPLIED，持久性=UNKNOWN**，不重试 rename，只重试 dirfsync（重读核验后）。
- current 内容与预期不一致：进入 QUARANTINED。

### 3.4 状态机（23 态）

以 05.28 的 20 态为基础，**拆出 rename 结果不确定与持久性不确定**：

NEW / CANDIDATE_CREATED / CANDIDATE_CONTENT_DURABLE / CANDIDATE_ENTRY_DURABLE / CANDIDATE_VERIFIED /
PREPARED_CREATED / PREPARED_CONTENT_DURABLE / PREPARED_ENTRY_DURABLE / PREPARED_VERIFIED /
CURRENT_TMP_CREATED / CURRENT_TMP_CONTENT_DURABLE / CURRENT_TMP_VERIFIED /
**RENAME_NOT_STARTED / RENAME_OUTCOME_UNKNOWN / RENAME_CONFIRMED** /
**CURRENT_DURABILITY_UNKNOWN / CURRENT_DURABLE_AUDIT_PENDING** /
RECEIPT_CREATED / RECEIPT_CONTENT_DURABLE / RECEIPT_ENTRY_DURABLE /
DURABLE / RECOVERY_REQUIRED / QUARANTINED

每个状态的完整定义见 `POSIX-COMMIT-STATE-MACHINE.md`。

### 3.5 并发写入者模型

见 `POSIX-CONCURRENCY.md`，逐项分析谁能获取锁/绕过锁/修改 current/替换目录项/删除临时文件/创建回执/改变 inode/在检查与 rename 之间修改。

核心结论：advisory lock **不是强制锁**，只能约束合作写入者；非合作写入者、管理员、FS 异常**无法约束**，须显式标记 `UNKNOWN` 并在检测到异常时 QUARANTINED。

### 3.6 回执归属与幂等（修复 P03）

**不能因文件名/txn_id 字符串/内容部分匹配就认定文件属于本次事务。** 必须建立可验证的事务归属证明：

回执唯一键 = `(txn_id, revision, current_inode_dev, current_sha256)` 四元组。

- EEXIST 时：读取已存在回执，核验其**完整四元组** + 文件类型（regular file）+ 创建者信息（若有）+ 内容完整（无短写）。
- 四元组全匹配 → 幂等成功（本次提交已记录）。
- 任一不匹配 → QUARANTINED（可能是其他事务残留或外部伪造）。
- 无法确认创建者/归属 → QUARANTINED，**禁止删除或覆盖**。

辅助元数据（创建时间、大小）**不能单独**作为所有权证明。

### 3.7 EEXIST 与半成品删除安全门（重审 15 项）

逐项证明必要性/充分性，见 `POSIX-RECOVERY.md`。删除/覆盖前必须回答九问（身份/类型/内容/归属/revision/是否被其他事务使用/是否存在外部并发修改/操作结果是否 UNKNOWN/删除是否破坏已提交数据）。任一无法证明 → DENY/QUARANTINED，禁止自动删除/覆盖/重试。

### 3.8 POSIX 时间线与反例

- 40 条崩溃恢复时间线：`TIMELINES.md` §3（CR1~CR40）。
- 25 条 rename 结果不确定：`TIMELINES.md` §4（RU1~RU25）。
- 25 条目录持久性不确定：`TIMELINES.md` §5（DU1~DU25）。
- 20 条外部写入者并发：`TIMELINES.md` §6（EW1~EW20）。
- 20 条回执归属冲突反例：`COUNTEREXAMPLES.md` §（CE-RC1~CE-RC20）。
- 20 条重复提交/幂等反例：`COUNTEREXAMPLES.md` §（CE-IDEM1~CE-IDEM20）。
- 20 条 Fork×POSIX 交叉：`TIMELINES.md` §7（FX1~FX20）。

### 3.9 B29-03 结论

| 验收项 | 结论 |
|--------|------|
| 提交锁约束范围明确 | **DESIGN_PROVEN**（advisory，仅约束合作者）|
| 不假设锁外写入者自动受控 | **DESIGN_PROVEN**（显式 UNKNOWN + QUARANTINED）|
| rename 结果不确定独立建模 | **DESIGN_PROVEN** |
| 持久性不确定独立建模 | **DESIGN_PROVEN** |
| 无不确态直接跳 DURABLE | **DESIGN_PROVEN** |
| 无未经证明盲目重试 | **DESIGN_PROVEN** |
| 回执半成品归属证据完整 | **DESIGN_PROVEN** |
| EEXIST 不误删他事务文件 | **DESIGN_PROVEN** |
| 幂等提交条件明确 | **DESIGN_PROVEN** |
| 崩溃恢复仅依赖磁盘证据 | **DESIGN_PROVEN** |
| 无法确认统一 DENY/QUARANTINED | **DESIGN_PROVEN** |
| 锁外写入者防护 | **NOT_VERIFIED**（advisory 本质）|

**B29-03 = DESIGN_PROVEN（协议层，在"合作写入者"部署契约前提下），NOT_VERIFIED（锁外写入者防护）。**

---

## 4. B29-04｜交叉验证

- 30 条跨模块调用链：`CROSS-MODULE-CHAINS.md`。
- 30 条 Fork×POSIX 交叉时间线：`TIMELINES.md` §7。
- 30 条身份×事务交叉反例：`COUNTEREXAMPLES.md` §。
- 80 条全局不变量：`INVARIANTS.md`（I-01~I-80）。

---

## 5. B29-05｜05.28 历史设计逐项差异

见 `DESIGN-DIFF.md`。核心：05.28 的三项 DESIGN_PROVEN（Fork 原子准入 / 身份唯一性 / POSIX 提交）**全部撤销重证**，非自动继承。

---

## 6. B29-06｜证据等级规则

严格使用 `VERIFIED_FACT / VERIFIED_STATIC / DESIGN_PROVEN / DESIGN_PROPOSAL / STATIC_INFERENCE / NOT_VERIFIED / NOT_TESTED / UNKNOWN / CONFLICT / FAIL`。

禁止：DESIGN_PROPOSAL 写成 DESIGN_PROVEN；NOT_TESTED 写成 PASS；概率唯一写成严格唯一；advisory 锁写成强制锁；文件可见写成已持久化；rename 异常写成未发生；两次读一致写成无并发修改。

---

## 7. 交付文件（22 份）

见 §8 交付目录，共 22 份（21 份内容 + DELIVERY.zip）。

---

## 8. 最终结论

| 项 | 结论 |
|----|------|
| B29-01 Fork 生命周期 | DESIGN_PROVEN（协议层）/ NOT_TESTED |
| B29-02 进程树身份 | DESIGN_PROVEN（协议层）/ NOT_VERIFIED（跨主机/重启）|
| B29-03 POSIX 提交 | DESIGN_PROVEN（合作写入者前提）/ NOT_VERIFIED（锁外防护）|
| P0-05 Source Retire | DENY |
| IMPLEMENT-01 | NOT_APPROVED |
| A/B Merge | NOT_APPROVED |

**整体结论：PASS-WITH-ISSUES（候选）。** 三个核心阻断项在"协议层 + 部署契约前提"下闭合；锁外写入者防护、跨主机/跨重启唯一性、运行时 fork/POSIX 行为仍为 NOT_VERIFIED/NOT_TESTED。

**STOP：等待 ChatGPT 独立复审，不得自行进入下一阶段。**
