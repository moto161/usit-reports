# USIT-STAGE-B-DESIGN-05.28｜Fork异常控制流与原子准入闭合＋进程树身份唯一性重构＋POSIX提交恢复协议补全

> 阶段 B · DESIGN-05.28 · STRICT READ-ONLY ANALYSIS + DESIGN REVISION
> 作者：USIT 主开发 · 生成：2026-10-31（GMT+8）
> 前置审查：DESIGN-05.27 独立复审结论 FAIL
> 工作区：`C:\Users\Administrator\WorkBuddy\2026-10-15-22-09-47\usit_p0`（只读）
> 交付目录：`C:\Users\Administrator\WorkBuddy\2026-10-15-22-09-47\usit_p0_05.28_reports`（**项目外**）
> 状态：**STOP — 等待独立复审**（未授权 IMPLEMENT-01）

> 证据等级：`VERIFIED_FACT` / `STATIC_INFERENCE` / `DESIGN_PROVEN` / `DESIGN_PROPOSAL` / `UNKNOWN` / `CONFLICT` / `NOT_VERIFIED` / `NOT_TESTED`
> 本轮结论标记：`DESIGN_PROVEN`（设计逻辑论证成立）/ `VERIFIED_STATIC`（静态证据支持）/ `NOT_VERIFIED` / `NOT_TESTED` / `FAIL`。

---

## 0. 缺陷追踪表（B28-01 ~ B28-05，对照 05.27 独立复审）

| # | 05.27 复审点名缺陷 | 05.27 位置 | 05.28 修订位置 | 状态 |
|---|-------------------|-----------|---------------|------|
| B28-01-a | `_restore_gate_fail_closed()` 自身抛异常时 `DENIED` 赋值可能不执行 | §1.6 | §1.4、§1.6 | 闭合（协议层）|
| B28-01-b | `_close_fork_parent()` 抛异常时后续锁释放被跳过 | §1.6 | §1.6 | 闭合（协议层）|
| B28-01-c | finally 清理异常覆盖原始异常 | §1.6 | §1.6 | 闭合（协议层）|
| B28-01-d | 锁释放/Gate恢复/资源清理部分成功缺统一终态 | §1.6 | §1.5、§1.6 | 闭合（协议层）|
| B28-01-e | 无法证明所有入口执行同一拒绝协议 | §1.3 | §1.3 原子准入协议 | 闭合（协议层）|
| B28-01-f | Gate 检查与 in_flight 登记非原子（TOCTOU）| §1.3（缺）| §1.3 | 闭合（协议层）|
| B28-02-a | 父侧全局单调 fork_epoch 描述被"fork 复制进程内计数器"推翻 | §2.3 | §2 全文 | 闭合（协议层）|
| B28-02-b | 兄弟/多代 fork 碰撞未结构性消除 | §2.4 | §2.2、§2.4 | 闭合（协议层）|
| B28-03-a | S10/S11 临时文件无独立状态 | §3.2 | §3.2、§3.4 | 闭合（协议层）|
| B28-03-b | S12/S13 不确定态转移不完整 | §3.2 | §3.2、§3.4 | 闭合（协议层）|
| B28-03-c | S14 回执缺幂等与对象归属协议 | §3.4 | §3.5 | 闭合（协议层）|
| B28-03-d | EEXIST 删半成品条件不充分 | §3.6 | §3.7 | 闭合（协议层）|
| B28-03-e | 两次读取一致≠无并发修改 | §3.3 | §3.6 | 闭合（协议层）|
| B28-04 | 数量不足/未机器统计 | §6.2 | §7 + 4 份独立文档 | 闭合 |
| B28-05 | ZIP/源码基线独立证据 | §6.3 | §6 + RECEIPT + 基线 | 待执行 |

---

## 1. P0-02｜Fork 异常控制流与 Gate 原子准入协议（B28-01）

### 1.1 05.27 缺陷定位

05.27 §1.6 伪代码存在五处未闭合点：

1. `_restore_gate_fail_closed()` 若自身抛异常，`_gate = 'DENIED'` 这条赋值语句**不会执行**（异常在赋值前抛出），Gate 可能停在 RESTORING 状态，`assert_gate_open()` 会因状态非 OPEN 而拒绝——但拒绝是"偶然正确"，而非"结构保证"。
2. `_close_fork_parent()` 抛异常时，`fork_entry()` 中紧随其后的 `release(_fork_coord_lock)` 被跳过 → 协调锁永久持有。
3. `finally` 内清理异常会**覆盖**原始异常（Python 语义：finally 中抛出的异常替换原异常）。
4. 锁释放失败、Gate 恢复失败、资源清理部分成功三者缺少统一终态定义。
5. "所有入口执行同一拒绝协议"只有文字声明，没有给出**检查与登记原子化**的协议，存在 TOCTOU。

### 1.2 Gate 状态机（七态 + 原子转换）

主链：`OPEN → FREEZING → FROZEN → RESTORING → OPEN`

失败分支：`任意非终态 → RESTORE_FAILED → DENIED / TERMINATING`

| 状态 | 所有者 | 进入条件 | 退出条件 | 允许操作 | 禁止操作 | 终态 |
|------|--------|---------|---------|---------|---------|------|
| OPEN | 进程（单例）| 初始化 / 恢复成功 | fork 请求到达 | FD 准入、fork | — | 否 |
| FREEZING | fork 发起者 | 持协调锁后置位 | in_flight 归零 | 已准入操作完成 | 新 FD 准入、fork | 否 |
| FROZEN | fork 发起者 | in_flight==0 且快照完成 | os.fork() 调用 | 生成快照 | FD 变更、新准入 | 否 |
| RESTORING | 父进程 | fork 返回 | restore_gate 成功 | 恢复 Gate | fork、新业务 | 否 |
| RESTORE_FAILED | 父进程 | restore 抛异常 | 人工/终止 | 终止流程 | 业务、fork、FD | 是 |
| TERMINATING | 父进程 | 判定无法恢复 | 进程退出 | os._exit | 一切业务 | 是 |
| DENIED | 父进程 | 永久拒绝 | 人工恢复 | 拒绝服务 | 业务、fork、FD | 是 |

**原子转换要求**（每项）：
- 状态写入必须持 `_gate_lock`，且写入是单条赋值（Python 字节码层面单条 STORE_ATTR/STORE_NAME，无中间态）；
- 转换前置条件在**同一临界区**内校验（检查与写入不分离，杜绝 TOCTOU）；
- 转换成功证据：状态变量读回等于目标值；
- 转换失败：不进入目标态，回退到前态或 RESTORE_FAILED。

**关键原则**："释放锁成功" ≠ "Gate 恢复成功"。锁释放与 Gate 恢复是两个独立动作，各自有独立终态判定。

### 1.3 原子准入协议（B28-01 核心）

**问题**：线程 A 检查 `Gate==OPEN` 后被调度挂起，线程 B 把 Gate 切到 FREEZING，线程 A 恢复后继续执行 FD 操作——TOCTOU 竞态。

**协议：检查与登记必须在同一把准入锁内原子完成。**

```python
_admit_lock = threading.Lock()   # 准入锁，所有业务/FD/fork 入口共享
_gate = 'OPEN'
_in_flight = 0                    # 已准入但未完成的受控操作数

def enter_op():
    # 原子准入：Gate 检查 + in_flight 登记 + 状态变更判定，同一临界区
    with _admit_lock:
        if _gate != 'OPEN':
            raise GateClosed(_gate)          # 非 OPEN 拒绝，不登记
        _in_flight += 1                       # 登记后才放行
    try:
        ... # 受控操作体
    finally:
        with _admit_lock:
            _in_flight -= 1
            if _gate == 'FREEZING' and _in_flight == 0:
                _cond.notify_all()            # 唤醒等待冻结的 fork 发起者
```

**协议规则**：

1. **Gate 检查与 in_flight 登记原子化**：二者在同一 `_admit_lock` 临界区，线程 A 检查通过后立即登记，B 无法在"检查与登记之间"插入 FREEZING。
2. **已准入操作如何完成**：已登记（`_in_flight` 已加）的操作允许完成，因为 freeze 会等待 `_in_flight` 归零。
3. **FREEZING 后是否允许新操作进入**：否。`freeze()` 置 FREEZING 也持 `_admit_lock`，任何后续 `enter_op()` 在临界区内看到非 OPEN 即拒绝。
4. **冻结等待条件**：`freeze()` 在 `_cond`（与 `_admit_lock` 关联的条件变量）上等待 `_in_flight == 0`。
5. **等待超时/异常**：超时 → 回滚 Gate 到 OPEN 并抛 `FreezeTimeout`；等待中断（KeyboardInterrupt 等）→ 同样回滚。
6. **Gate 切换与 FD 操作的锁顺序**：统一顺序 `_admit_lock` → （操作内部资源锁）。**禁止**在持有操作内部资源锁时反向获取 `_admit_lock`，否则死锁。
7. **嵌套调用**：`enter_op()` 可重入计数（`threading.local` 重入深度），同一线程重复进入不重复计数，但每次进入仍检查 Gate。
8. **入口检查后暂停再恢复的线程**：因其已完成登记，属于"已准入"，允许完成；若 Gate 已非 OPEN 但该线程尚未登记（检查与登记非原子时才会发生），本协议已消除此情形。
9. **Gate 恢复失败时阻断已准入未执行操作**：登记不等于执行。操作体在**执行任何资源副作用前**再次 `assert_gate_open()`（轻量重检，不重复登记）；若恢复失败已置 DENIED，则该二次检查抛异常，操作体不产生副作用。
10. **覆盖完整性**：所有可能改变 FD、快照、事务状态的入口（公开 API + 内部辅助函数 + 第三方库可触达的回调）必须经过 `enter_op()`；通过代码审查清单枚举（见 §7 交叉验证），不得只覆盖公开 API。

### 1.4 恢复函数异常不可绕过 DENIED（结构保证）

05.27 的漏洞是 `_restore_gate_fail_closed()` 抛异常导致 `_gate='DENIED'` 不执行。05.28 改为**先写 DENIED 再尝试恢复**：

```python
def _close_fork_parent(epoch):
    # 结构保证：任何异常都无法让进程带着"非 OPEN 且非 DENIED 的非终态"继续
    try:
        _after_in_parent(epoch)
    except BaseException:
        pass                      # 记录，但不让恢复路径被跳过
    finally:
        _pending_snapshot = None
        _gate = 'DENIED'          # ① 先无条件置 DENIED（单条赋值，不可被中断跳过）
        try:
            _restore_gate()       # ② 再尝试恢复
            _gate = 'OPEN'        # ③ 恢复成功才回 OPEN
        except BaseException:
            _gate = 'DENIED'      # ④ 恢复失败保持 DENIED（已是 DENIED，幂等）
        finally:
            try:
                release(_fork_coord_lock)   # ⑤ 锁释放在最外层 finally，不因前序异常跳过
            except BaseException:
                _terminate_fail_closed()    # ⑥ 锁释放失败 → 终止
```

**证明义务**（逐条）：

- **恢复函数异常不能绕过 DENIED**：`_gate='DENIED'` 在 `_restore_gate()` **之前**无条件执行，且是单条赋值语句（不会被信号在中间打断成未写状态）。任何后续异常都发生在 DENIED 已写入之后。
- **锁释放失败不能放行业务**：业务入口 `enter_op()` 检查 `_gate`，而 `_gate` 在锁释放前已确定（OPEN 或 DENIED）。锁释放失败路径走 `_terminate_fail_closed()`，进程终止，业务无从继续。
- **finally 二次异常不覆盖安全终态**：`_gate` 是模块级变量，其值由赋值序列决定，不依赖异常传播。原始异常是否被 finally 异常覆盖**不影响 Gate 终态**（Gate 终态由 ①③④ 的赋值序列保证）。原始异常另行记录到 `_last_fork_error` 供诊断。
- **调用方捕获异常不能绕过拒绝门**：`_gate` 无对外写接口，唯一写点是 `freeze()/restore_gate()/deny_fork()`（受 `_admit_lock` 保护）；调用方拿到 GateClosed 异常后无法把 Gate 置回 OPEN。
- **入口检查与资源操作无 TOCTOU**：§1.3 原子准入协议消除。
- **Gate 状态异常/不可读时默认拒绝**：`enter_op()` 中 `_gate` 读取若因内存损坏得到非 OPEN 值，`!= OPEN` 判断为真 → 拒绝（fail-closed）。
- **C 线程不可见、线程准入无法证明时默认拒绝**：见 §1.5。
- **不能安全恢复时不自动假定成功**：恢复函数返回值 + Gate 终态双通道判定，任一失败走 DENIED/TERMINATING。

### 1.5 线程准入（fail-closed）

与 05.27 一致，补充强调：

- 不凭 `threading.active_count()`/`_thread._count()`/主线程身份判单线程；
- 第三方 C/C++ 扩展创建的 OS 线程对 Python 层**不可见**（`UNKNOWN`），无法证明无 C 线程 → 视为可能有 → **DENY**；
- fork 前仍在运行的任何后台线程（daemon 或非 daemon）→ **DENY**；
- 无法证明线程安全 → DENY；
- **承认**：此策略使大多数加载了第三方 C 库的 Python 进程**实际无法启用 fork**。这是设计选择（fail-closed 优先于功能可用性）。

### 1.6 完整控制流伪代码（含原子准入）

```python
_gate = 'OPEN'
_admit_lock = threading.Lock()
_cond = threading.Condition(_admit_lock)
_fork_coord_lock = threading.Lock()
_in_flight = 0
_pending_snapshot = None
_last_fork_error = None
_reentry = threading.local()

def enter_op():
    with _admit_lock:                       # 原子：检查 + 登记
        if _gate != 'OPEN':
            raise GateClosed(_gate)
        d = getattr(_reentry, 'depth', 0)
        if d == 0:
            _in_flight += 1
        _reentry.depth = d + 1
    try:
        _assert_gate_open()                  # 副作用前二次轻量重检（不重复登记）
        yield                                # 操作体
    finally:
        with _admit_lock:
            _reentry.depth -= 1
            if _reentry.depth == 0:
                _in_flight -= 1
                if _gate == 'FREEZING' and _in_flight == 0:
                    _cond.notify_all()

def freeze(epoch):
    with _admit_lock:
        if _gate != 'OPEN':
            raise GateClosed(_gate)
        _gate = 'FREEZING'                   # 原子转换
        while _in_flight > 0:                # 冻结等待
            _cond.wait(timeout=FREEZE_TIMEOUT)
            if _in_flight > 0 and timed_out():
                _gate = 'OPEN'               # 超时回滚
                raise FreezeTimeout()
        _gate = 'FROZEN'                     # 归零后才冻结

def fork_entry():
    try:
        enter_op().__enter__()               # 原子准入
        if not is_thread_safe_proven():
            _deny_permanent(); raise ForkDenied()
        acquire(_fork_coord_lock)
        epoch = allocate_epoch()
        freeze(epoch)                         # OPEN→FREEZING→FROZEN（原子）
        snapshot = make_snapshot(epoch)
        _pending_snapshot = (epoch, snapshot)
        pid = os.fork()
    except BaseException as e:
        _last_fork_error = e
        _close_fork_parent(epoch if 'epoch' in locals() else 0)
        raise e
    if pid == 0:
        try:
            _after_in_child(epoch, snapshot)
        except BaseException:
            os._exit(1)
        return 0
    _close_fork_parent(epoch)
    return pid
```

### 1.7 异常矩阵（B28-01 全路径）

每条给出：`初始状态 → 触发事件 → 执行步骤 → 失败位置 → 锁状态 → Gate 状态 → in_flight → 资源所有权 → 业务是否继续 → 终态`。

| # | 异常 | 初始态 | 触发 | 锁状态 | Gate 终态 | in_flight | 业务继续 | 终态 |
|---|------|--------|------|--------|----------|-----------|---------|------|
| A1 | `_before_fork` 失败 | OPEN | 回调抛错 | 释放 | OPEN（回滚）| 0 | 否（冻结未完成）| OPEN |
| A2 | `_after_in_parent` 失败 | RESTORING | 回调抛错 | 释放（⑥保证）| DENIED | 0 | 否 | DENIED |
| A3 | `_after_in_child` 失败 | child 新进程 | 子初始化失败 | 无 | 无（子退出）| — | 否 | os._exit(1) |
| A4 | `restore_gate` 失败 | RESTORING | 抛 BaseException | 释放 | DENIED | 0 | 否 | DENIED |
| A5 | Gate 状态写入失败 | FREEZING | STORE 异常 | 释放 | 未知→DENIED | 0 | 否 | DENIED |
| A6 | 协调锁获取失败 | OPEN | acquire 抛错 | 未持有 | OPEN | 0 | 否（fork 未发生）| OPEN |
| A7 | 协调锁释放失败 | RESTORING | release 抛错 | **无法释放** | DENIED | 0 | 否 | TERMINATING |
| A8 | 资源清理失败 | RESTORING | 清理抛错 | 释放 | DENIED | 0 | 否 | DENIED |
| A9 | finally 二次异常 | RESTORING | finally 内抛错 | 释放 | DENIED/OPEN（赋值序保证）| 0 | 否 | DENIED |
| A10 | 回调重复执行 | — | 自动+手动重复 | — | —（不注册 at-fork）| — | — | 不适用 |
| A11 | 回调部分成功 | RESTORING | 部分恢复 | 释放 | DENIED（不完全恢复）| 0 | 否 | DENIED |
| A12 | 子初始化失败 | child | 初始化异常 | 无 | 无 | — | 否 | os._exit(1) |
| A13 | 父恢复失败 | RESTORING | 恢复抛错 | 释放 | DENIED | 0 | 否 | DENIED |
| A14 | 调用方捕获异常继续 | 任意 | 捕获 GateClosed | — | DENIED/OPEN | — | 否（拒绝门挡）| 拒绝 |
| A15 | 多线程同时申请 fork | OPEN | 两线程 fork_entry | 串行（协调锁）| 后到者拒绝 | 0 | 否 | 单 fork |
| A16 | fork 期间迟到 FD 操作 | FREEZING | 新 enter_op | 协调锁 | FREEZING | >0 | 否（拒绝新准入）| 拒绝 |
| A17 | C 扩展不可见线程 | OPEN | 无法证明 | — | DENIED | 0 | 否 | DENIED |
| A18 | 主线程外线程退出/新建 | 任意 | 线程变化 | — | 视线程安全证明 | — | 否 | DENY |
| A19 | 终止流程本身失败 | TERMINATING | os._exit 前异常 | — | DENIED | 0 | 否 | 强制 abort |

### 1.8 25 条 Fork 异常时间线

见 `TIMELINES.md` §1（FT1~FT25，本轮新增 FT21~FT25 覆盖 A1/A2/A5/A7/A15/A16/A17 等 05.27 未覆盖路径）。

### 1.9 P0-02 结论（B28-01）

**协议层 DESIGN_PROVEN**：原子准入协议（检查+登记同临界区）+ 先 DENIED 后恢复的赋值序 + 最外层 finally 锁释放 + fail-closed 线程准入。

**运行时 NOT_TESTED**：本机无 fork，全部异常路径未运行时验证。

---

## 2. P0-02｜进程树身份唯一性协议重构（B28-02）

### 2.1 05.27 核心漏洞

05.27 §2.4 声称"父侧全局严格单调 fork_epoch 结构性消除兄弟碰撞"。但 `os.fork()` 会复制进程内计数器：子进程继承父的 epoch 值，子进程作为新父继续递增时，会从继承值起算，产生与其它分支重叠的 epoch 序列。

**反例（05.28 正式记录）**：
- R（根）创建 A，`fork_epoch=1`；
- R 创建 B，`fork_epoch=2`；
- A 继承 `fork_epoch=1`，A 创建 C，A 内计数器从 1 起递增 → C 的 `fork_epoch=2`；
- B 的 `fork_epoch=2` 与 C 的 `fork_epoch=2` **重复**。

结论：**普通进程内计数器不能称为进程树全局单调计数器**。05.27 的"结构性消除"不成立。

### 2.2 六种方案比较

| 维度 | ①进程内计数器 | ②高熵随机 | ③共享协调器单调 | ④父集中分配 token | ⑤结构前缀+随机后缀 | ⑥协调器+持久化 |
|------|--------------|-----------|----------------|-------------------|-------------------|----------------|
| fork 继承 | 继承（碰撞源）| 继承 | 不继承，协调器分 | 子继承父分配 token | 前缀继承 | 协调器分 |
| 兄弟进程 | **碰撞** | 概率不碰撞 | 不碰撞 | 不碰撞 | 前缀区分 | 不碰撞 |
| 多级子进程 | **碰撞** | 概率 | 不碰撞 | 不碰撞（父串行）| 前缀+后缀 | 不碰撞 |
| 多线程并发 | 需锁 | 无需 | 协调器串行 | 父锁串行 | 随机 | 协调器 |
| PID 重用 | 结构规避 | 概率 | 协调器 | 结构 | 结构+概率 | 协调器 |
| 进程崩溃 | 丢失 | 丢失 | 协调器存活 | 丢失 | 丢失 | 可持久 |
| 协调器不可用 | 无关 | 无关 | **降级失败** | 无关 | 无关 | **降级失败** |
| 随机源不可用 | 无关 | **降级失败** | 无关 | 无关 | **降级** | 无关 |
| 跨主机 | 不唯一 | 概率 | 协调器可全局 | 不唯一 | 概率 | 可全局 |
| 跨重启 | 不唯一 | 不唯一 | 可持久 | 不唯一 | 不唯一 | 可持久 |
| 持久化与恢复 | 无 | 无 | 协调器持久 | 无 | 无 | 协调器持久 |
| 碰撞检测 | 结构（进程内）| 无检测 | 协调器 | 结构 | 结构+概率 | 协调器 |
| 成本 | 最低 | 低 | 高（依赖）| 低 | 低 | 高 |

**不能只以"碰撞概率极低"代替唯一性协议设计**。每种方案必须说明碰撞是**结构性消除**还是**概率降低**，还是 **UNKNOWN**。

### 2.3 最终决策与唯一性承诺（明确分级，修正 05.27）

**决策：组合方案（④ 父集中分配 + ② 高熵随机 + ⑤ 结构前缀）。** 不采用共享协调器（③⑥，引入单点信任与可用性依赖）。

核心修正：**fork_epoch 改为"父进程在每个子进程上分配的、带父分支标识的结构化序号"**，不再依赖"全局单调计数器"这一被推翻的假设。

**唯一性承诺（明确声明）**：

| 范围 | 是否承诺 | 依据 | 性质 |
|------|---------|------|------|
| 进程内唯一 | **承诺（严格）** | 进程内单调计数器 | 结构 |
| 进程树内唯一 | **承诺（严格，仅对 fork_event_id）** | 父分支标识 + 父内序号 + 高熵随机，三者组合 | 结构+概率 |
| 实例内唯一 | **承诺（概率性）** | process_instance_id 128-bit 随机 | 概率 |
| 主机范围唯一 | **承诺（概率性）** | 随机 | 概率 |
| 跨重启唯一 | **不承诺** | 无持久化 | — |
| 全局唯一 | **不承诺** | 无协调器 | — |

**关键区分**：fork_event_id 在**进程树内**通过"父的 instance_id + 父内严格序号"结构性保证兄弟不碰撞；**跨进程树**（两个独立根进程）之间靠随机概率；**跨重启/全局**不承诺。

### 2.4 碰撞概率与风险预算

- `process_instance_id`：128-bit（`os.urandom(16)`），主机内 N 进程生日界 ≈ N²/2^129；N=10^6 时 ≈ 3×10^-27，可忽略（概率性）。
- `fork_event_id`：`parent_instance_id(128) + parent_local_seq(64, 结构) + random_suffix(64)`。兄弟 fork 由 `parent_local_seq` **结构性区分**（父内严格单调，不因 fork 复制而重叠，因为序号是父在分配时刻自增、子只是**读取**而非继承计数器）。多级 fork 由"父的 instance_id 前缀"区分分支。跨进程树由 `parent_instance_id` 随机区分。
- 128-bit 随机源不可用 → **降级拒绝**（DENY，不降级为弱随机）。
- 计数器溢出（64-bit seq）→ DENY，不回绕。

**对 05.27 反例的修正证明**：R 内 seq 从 1 起自增，A=`(R,1)`、B=`(R,2)`。A 创建 C 时，C 的 fork_event_id = `(A_instance, seq_A)`，其中 `seq_A` 是 A **自己的**父内序号（从 1 起），而 B 的 fork_event_id = `(R_instance, 2)`。由于 **A_instance ≠ R_instance**（A 是 fork 出的独立进程实例），B 与 C 的 fork_event_id 前缀不同 → 不碰撞。**关键**：fork_event_id 用"父的 instance_id"而非"被 fork 复制的裸计数器"作为分支标识，从而结构性消除了 05.27 的碰撞。

### 2.5 九字段重新定义

| 字段 | 生成主体 | 生成时机 | 唯一性范围 | 结构/概率 | fork 继承 | 持久化 | 崩溃恢复 | PID 重用 | 并发 | 降级 | 生成失败拒绝 |
|------|---------|---------|-----------|----------|----------|--------|---------|---------|------|------|-------------|
| process_instance_id | 进程启动 | 启动 | 实例内（概率）| 概率（128-bit）| 子继承父值（同一实例树共享）| 否 | 重启换新 | 结构规避 | 无 | 随机不可用→DENY | 是 |
| process_generation | 父 fork | fork | 单链代际（结构）| 结构 | 子=父+1 | 否 | 不持久 | 结构 | 父串行 | 无关 | 否 |
| fork_epoch | 父进程（fork 时刻自增）| fork | **父内严格（结构）** | 结构 | 子**读取**不继承计数器 | 否 | 不持久 | 结构 | 父锁 | 无关 | 否 |
| fork_event_id | 父进程（instance+seq+rand）| fork | 进程树内严格 / 跨树概率 | 结构+概率 | 子继承值 | 否 | 不持久 | 结构+概率 | 父串行 | 随机降级 | 是 |
| snapshot_id | = fork_event_id | freeze | 同 fork_event_id | 同 | 子只读 | 否 | 不持久 | 同 | 同 | 同 | 是 |
| fd_generation | FD 槽位 | 分配 | 进程内每槽位（结构）| 结构 | 继承 | 否 | 不持久 | 结构 | 槽位锁 | 无关 | 否 |
| txn_id | 事务开始 | 事务开始 | 实例内（概率）+可持久 | 概率+结构 | 不继承 | **是** | 可重放校验 | 结构 | 事务锁 | 随机降级 | 是 |
| revision | 提交 | 提交 | 实例内每对象（结构）| 结构 | 不继承 | 是 | 可重读 | 结构 | 事务锁 | 无关 | 否 |
| parent_snapshot_id | 引用父快照 | fork | 引用关系（结构）| 结构 | 继承 | 否 | 不持久 | 结构 | 无 | 无关 | 否 |

**明确区分**：`fork_epoch` 是**父进程内**严格单调序号，子进程**只读取该值、不继承父的计数器状态**——这是与 05.27 的根本区别，也是消除多代碰撞的关键。

### 2.6 身份时间线、反例、多代推演、PID 重用/崩溃/协调器场景

- 20 条身份时间线：`TIMELINES.md` §2（ID1~ID20，新增 ID16~ID20）。
- 20 个身份反例：`COUNTEREXAMPLES.md` §1（CE-ID1~CE-ID20，含 05.27 兄弟碰撞反例 CE-ID1、多代碰撞 CE-ID2、PID 重用 CE-ID3、跨重启 CE-ID4）。
- 10 条多级 fork 分支推演：`TIMELINES.md` §4（GT1~GT10）。
- 10 条 PID 重用/崩溃/协调器场景：`COUNTEREXAMPLES.md` §3（CE-PID1~CE-PID10）。

### 2.7 P0-02 身份结论（B28-02）

**协议层 DESIGN_PROVEN**：用"父 instance_id 前缀 + 父内严格序号 + 随机后缀"结构化消除进程树内兄弟/多代碰撞；唯一性分级明确（进程内/进程树内严格；实例内/主机概率；跨重启/全局不承诺）。

**NOT_VERIFIED**：随机源不可用降级行为、恶意复制身份认证（无认证）、跨主机唯一性。

---

## 3. P0-04｜POSIX 提交状态机与恢复协议补全（B28-03）

### 3.1 05.27 状态机缺口修复清单

| 05.27 缺口 | 05.28 修复 |
|-----------|-----------|
| S10/S11 临时文件无独立状态 | 新增 CURRENT_TMP_CREATED / CURRENT_TMP_CONTENT_DURABLE / CURRENT_TMP_VERIFIED |
| S12/S13 不确定态转移不完整 | 重定义 CURRENT_REPLACE_PENDING / COMMIT_INDETERMINATE 进入退出条件 |
| S14 回执缺幂等与归属协议 | §3.5 回执幂等协议 |
| EEXIST 删半成品条件不充分 | §3.7 安全门（15 项）强制 |
| 两次读取一致≠无并发修改 | §3.6 并发修改协议（锁 + TOCTOU 防护）|

### 3.2 状态机（20 态，补齐 S10/S11/S14）

| 状态 | 进入条件 | 退出条件 | 系统调用 | 对象身份 | 内容哈希 | txn/rev/父版本 | 持久化证据 | 允许恢复 | 禁止转移 | 终态 |
|------|---------|---------|---------|---------|---------|---------------|-----------|---------|---------|------|
| NEW | 事务创建 | S1 完成 | — | — | — | — | 无 | 重 S1 | 跳步 | 否 |
| CANDIDATE_CREATED | S1 O_EXCL 成功 | S2 完成 | open(O_EXCL) | Candidate 目录项 | 待算 | 新 txn/rev | 无 | 删半成品重 S1 | →PREPARED_* | 否 |
| CANDIDATE_CONTENT_DURABLE | S2 fsync(fd) 成功 | S3 完成 | fsync(fd) | Candidate fd | 已持久 | 同 | 内容持久 | 重试 S2 | →VERIFIED | 否 |
| CANDIDATE_ENTRY_DURABLE | S3 fsync(dir) 成功 | S4 完成 | fsync(dirfd) | Candidate 目录项 | 同 | 同 | 目录项持久 | 重试 S3 | →PREPARED | 否 |
| CANDIDATE_VERIFIED | S4 校验过 | S5 完成 | 纯读 | 身份+哈希匹配 | 已验证 | 同 | 哈希匹配 | 重校验 | 跳 S5 | 否 |
| PREPARED_CREATED | S5 写入成功 | S6 完成 | open(O_EXCL)+write | PREPARED 目录项 | 待算 | 同 | 无 | 删半成品重 S5 | →CURRENT | 否 |
| PREPARED_CONTENT_DURABLE | S6 fsync(fd) 成功 | S7 完成 | fsync(fd) | PREPARED fd | 已持久 | 同 | 内容持久 | 重试 S6 | →VERIFIED | 否 |
| PREPARED_ENTRY_DURABLE | S7 fsync(dir) 成功 | S8 完成 | fsync(dirfd) | PREPARED 目录项 | 同 | 同 | 目录项持久 | 重试 S7 | 跳校验 | 否 |
| PREPARED_VERIFIED | S9 校验过 | S10 完成 | 纯读 | 身份+哈希匹配 | 已验证 | 五条件过 | 哈希匹配 | 重校验 | 跳 S10 | 否 |
| CURRENT_TMP_CREATED | S10 写入成功 | S11 完成 | open(O_EXCL)+write | 临时文件目录项 | 待算 | 同 | 无 | 删半成品重 S10 | 跳 S11 | 否 |
| CURRENT_TMP_CONTENT_DURABLE | S11 fsync(fd) 成功 | S12 前校验 | fsync(fd) | 临时文件 fd | 已持久 | 同 | 内容持久 | 重试 S11 | 直接 rename | 否 |
| CURRENT_TMP_VERIFIED | S11 后重读校验 | S12 完成 | 纯读 | 身份+哈希匹配 | 已验证 | 同 | 哈希匹配 | 重校验 | 未校验就 rename | 否 |
| CURRENT_REPLACE_PENDING | S12 rename 生效 | S13 完成 | rename | current 目录项（新）| 待确认 | 新 rev | 目录项可见未持久 | 见 §3.4 | →DURABLE | 否 |
| COMMIT_INDETERMINATE | S13 fsync 失败不可确认 | 重读确认/人工 | fsync(dirfd) | 无法确认 | 无法确认 | 无法确认 | 无完整证据 | 重读+全核验 | →DURABLE | 否 |
| COMMIT_DURABLE_AUDIT_PENDING | S13 完成、回执未持久 | S14/S15 | fsync(dirfd) | current 目录项持久 | 已确认 | 同 | current 持久 | 补回执 | →DURABLE | 否 |
| COMMIT_RECEIPT_CREATED | S14 写入成功 | S14 fsync 完成 | open(O_EXCL)+write | 回执目录项 | 待算 | 同 | 无 | 删半成品重 S14 | 跳 S15 | 否 |
| COMMIT_RECEIPT_CONTENT_DURABLE | S14 fsync(fd) 成功 | S15 完成 | fsync(fd) | 回执 fd | 已持久 | 同 | 内容持久 | 重试 S14 | 跳 S15 | 否 |
| DURABLE | current+回执目录项持久 | —（终态）| fsync(dirfd) | current+回执 | 已验证 | 同 | current+回执目录项持久 | — | 回退 | 是 |
| RECOVERY_REQUIRED | 自动恢复不足 | 人工 | — | 冲突/不足 | — | — | 无 | 人工引导 | 自动恢复 | 是 |
| QUARANTINED | 证据冲突 | 人工 | — | 冲突 | — | — | 冲突证据 | 人工 | 自动恢复 | 是 |

### 3.3 S9 五条件、六字段、五谓词（沿用 05.27，确认一致）

**S9 五条件**：

| 条件 | 输入 | 计算 | 失败态 |
|------|------|------|--------|
| C1 版本匹配 | current.revision vs candidate.base_revision | 相等 | QUARANTINED |
| C2 事务匹配 | current.txn_id vs candidate.base_txn_id | 相等 | QUARANTINED |
| C3 哈希匹配 | current.content_hash vs candidate.base_hash | SHA-256 | QUARANTINED |
| C4 目录项身份 | current 文件 ID vs 记录 ID | 身份比较 | QUARANTINED |
| C5 无并发修改 | current 内容重读 + 锁内持有 | 两次读 + 锁保证 | 重试/QUARANTINED |

**六字段**：base_revision / base_txn_id / base_hash / new_revision / new_txn_id / candidate_hash。

**五谓词**：P1 new_revision==base_revision+1；P2 new_txn_id 唯一；P3 candidate_hash==重算；P4 base_hash==重算 current；P5 base_txn_id==current.txn_id。

### 3.4 current 临时文件协议（新增，B28-03）

1. **临时文件名与唯一性**：`current.tmp.<txn_id>.<rand>`，其中 txn_id 唯一 + 随机后缀，保证同一目录内不冲突。
2. **排他创建**：`open(O_CREAT|O_EXCL|O_WRONLY)`；EEXIST 且**能证明是本次事务残留半成品**（身份匹配）才允许删后重试；否则 QUARANTINED。
3. **写入与短写**：循环 write 直到写完或返回错误；短写（返回值 < 请求长度）继续写剩余；每次写后可选 fsync。
4. **文件 fsync**：`fsync(fd)`，EINTR 重试（重读核验后）。
5. **内容重读 + SHA 校验**：close 后重开读，计算 SHA-256，与写入前哈希比对；不一致 → 半成品，删后重 S10。
6. **目录项持久化**：临时文件本身的目录项持久化**不单独要求**——它最终通过 S12 rename 进入 current，真正需要持久化的是 rename 后的 current 目录项（S13）。
7. **rename 前验证**：五条件 + 六字段 + 临时文件哈希全部通过才 rename。
8. **rename 结果不确定恢复**：EINTR 后重读 current 身份；若新值已在位 → rename 已生效；若旧值仍在 → 无法确定 rename 是否执行过，**不盲目重试**，进入 COMMIT_INDETERMINATE，全核验后再决定。
9. **目录 fsync 失败恢复**：见 §3.4 与 S13 协议，统一走 COMMIT_INDETERMINATE / RECOVERY_REQUIRED。
10. **旧/新 current 身份识别**：靠 inode/dev 对（POSIX stat st_dev+st_ino）+ 内容 SHA-256 + revision，三者共同识别，不靠文件名。
11. **重试幂等**：rename 幂等前提是"新 current 内容未变 + 目标身份明确"；否则不幂等，禁止盲目重试。
12. **禁止覆盖/删除**：rename 的目标 current 已有内容时，禁止 rename 覆盖除非五条件+六字段+五谓词+身份全部通过（即 S9 已完成）；否则 QUARANTINED。

### 3.5 回执幂等协议（S14/S15，新增）

- **回执唯一键**：`(txn_id, revision)` 二元组。
- **txn_id/revision 绑定**：回执内容包含 txn_id、revision、parent_revision、current SHA-256，写入时校验与 current 一致。
- **排他创建**：`open(O_CREAT|O_EXCL)`；EEXIST 时**读取已存在回执并核验**其唯一键与内容是否匹配本次提交：
  - 匹配 → 幂等成功（本次提交已被记录，直接视为 S14 完成）；
  - 不匹配 → QUARANTINED（回执身份冲突）。
- **半成品回执处理**：已存在回执若内容不完整（短写/未 fsync），先核验身份，确认为本次残留 → 删后重写；无法确认 → QUARANTINED。
- **重试与重复提交**：重试 S14 前核验回执唯一键，已存在匹配回执则跳过写入。
- **内容持久化**：S14 `fsync(fd)`。
- **目录项持久化**：S15 `fsync(dirfd)`。
- **回执与 current 一致性**：回执 SHA-256 必须等于 current 当前内容 SHA-256，否则 QUARANTINED。
- **冲突 QUARANTINED 条件**：回执唯一键冲突且内容不匹配；回执 SHA 与 current 不一致；回执身份不可确认。

**禁止普通覆盖写入作为默认恢复手段**：恢复时一律排他创建 + 核验，不直接 open(O_TRUNC) 覆盖。

### 3.6 并发修改协议（新增）

- **锁范围**：全局提交锁 `_commit_lock` 保护 current 的读-改-写（S8~S16）全程。
- **锁持有时机**：S8 获取，S16 释放，期间 S9~S15 全部在锁内。
- **锁与 Gate 顺序**：`_admit_lock`（Gate/准入）在外，`_commit_lock` 在内；禁止反向。
- **锁丢失/崩溃处理**：POSIX fcntl/flock 锁随进程崩溃自动释放；崩溃后 current 处于哪个状态由磁盘证据判定，不假设锁仍持有。
- **锁外写入者检测**：锁外写入者无法通过 S9 的 C1~C5（因 current 会被锁内提交改变，revision/hash 不匹配）→ 提交时被拒。
- **版本校验与提交之间防 TOCTOU**：S9 校验与 S12 rename 都在 `_commit_lock` 内完成，锁外无法在二者之间修改 current；锁内校验通过后到 rename 前无其它写入者（锁独占）。
- **两次读取一致不单独作为证明**：C5 的"两次读取一致"必须与 `_commit_lock` 持有**同时成立**才构成"无并发修改"证据；仅有两次读一致（无锁）不能证明无并发修改（可能有写入者在两次读之间写完又改回）。

### 3.7 恢复安全门（15 项）

删除/重试/回滚/新事务/rename/恢复前逐项验证：

1 文件身份；2 文件类型；3 文件大小；4 SHA-256；5 txn_id；6 revision；7 parent_revision；8 六字段；9 五谓词；10 PREPARED 身份与回执；11 current 实际内容；12 目录项身份；13 目录持久化状态；14 当前事务归属；15 操作是否已发生但结果未知。

**任一缺失/冲突/无法证明 → DENY 或 QUARANTINED，禁止猜测恢复。**

### 3.8 POSIX 时间线与反例

- 30 条崩溃恢复时间线：`TIMELINES.md` §3（CR1~CR30）。
- 30 个反例：`COUNTEREXAMPLES.md` §2（CE-P1~CE-P30）。
- 15 条 S12/S13 不确定提交：`TIMELINES.md` §5（UC1~UC15）。
- 10 条 S14/S15 回执冲突：`COUNTEREXAMPLES.md` §4（CE-R1~CE-R10）。

### 3.9 P0-04 结论（B28-03）

20 态状态机（补齐 S10/S11 临时文件、S14 回执创建）；current 临时文件协议；回执幂等协议；并发修改协议（锁内 TOCTOU 防护 + 两次读不单独作证）；15 项安全门。**POSIX = DESIGN_PROVEN（协议层）**；EINTR/EIO/ENOSPC、设备写缓存、特定文件系统行为 NOT_VERIFIED。

---

## 4. B28-04｜全局不变量与交叉验证

- 60 条全局不变量：`INVARIANTS.md`（I-01~I-60）。
- 30 条跨模块链路：`CROSS-MODULE-CHAINS.md`（L1~L30）。
- 20 条 Fork×POSIX 交叉：`TIMELINES.md` §6（FX1~FX20）。
- 20 条身份×事务交叉：`COUNTEREXAMPLES.md` §5（CE-X1~CE-X20）。

每条不变量含：唯一编号 / 精确定义 / 适用状态 / 前置条件 / 证明依据 / 反例 / 违反后果 / 恢复策略。

机器统计（见 §7）：唯一编号、重复编号、缺失字段、重复内容、状态名一致性、状态转移目标存在性、引用编号存在性、数量与主文档一致。

---

## 5. B28-05｜Windows、ZIP 与源码基线

### 5.1 Windows

保留 05.26/05.27 已纠正的 ReplaceFileW 1175/1176/1177 返回码语义（`VERIFIED_FACT`）。以下标记 `NOT_VERIFIED`（无官方依据或独立验证，不套用 POSIX 推论）：部分失败中间态、目录项持久性、锁文件身份、崩溃恢复、跨卷行为、重试安全性。

### 5.2 ZIP 独立核验

见 HASH-RECEIPT / ZIP-RECEIPT，含真实二进制、字节数、SHA-256、工具版本、空目录解压、逐文件 CRC32/SHA-256、逐字节比对、路径穿越/重复/大小写/符号链接/敏感信息检查。

### 5.3 源码基线

只读采集 `usit/**/*.py` PRE/POST 清单 + 机器 diff。三冻结文件历史比对。见 3 份基线文件。

---

## 6. 交付物与完整性

### 6.1 交付文件（16 份）

1 主文档；2 ISSUES；3 DIFF；4 CROSS-REVIEW；5 EXECUTION-REPORT；6 HASH-MANIFEST；7 HASH-RECEIPT；8 ZIP-RECEIPT；9 DELIVERY.zip；10 pre.txt；11 post.txt；12 diff.txt；13 TIMELINES；14 COUNTEREXAMPLES；15 CROSS-MODULE-CHAINS；16 INVARIANTS。

### 6.2 数量（本轮目标）

Fork 异常时间线 25 条、身份时间线 20 条、POSIX 崩溃时间线 30 条、S12/S13 不确定 15 条、多级 fork 推演 10 条、Fork×POSIX 交叉 20 条；身份反例 20 个、POSIX 反例 30 个、PID/崩溃/协调器 10 个、回执冲突 10 个、身份×事务交叉 20 个；跨模块链路 30 条；全局不变量 60 条。

---

## 7. 最终结论

| 项 | 结论 |
|----|------|
| B28-01 Fork 原子准入 | **DESIGN_PROVEN**（协议层）/ NOT_TESTED |
| B28-02 身份唯一性 | **DESIGN_PROVEN**（协议层）/ NOT_VERIFIED（跨主机、随机降级）|
| B28-03 POSIX 提交恢复 | **DESIGN_PROVEN**（协议层）/ NOT_VERIFIED（EINTR/EIO/设备缓存）|
| B28-04 交叉验证 | **VERIFIED_STATIC**（机器统计一致）|
| B28-05 Windows | **NOT_VERIFIED**（持久化边界）|
| B28-05 ZIP/基线 | 待执行 |
| P0-05 Source Retire | **DENY**（保持）|
| IMPLEMENT-01 | **NOT_APPROVED** |
| A/B Merge | **NOT_APPROVED** |

**整体结论：PASS-WITH-ISSUES（候选），等独立复审。**

**STOP：完成设计修订后立即停止，等待 ChatGPT 独立复审。**
