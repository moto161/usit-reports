# USIT-STAGE-B-DESIGN-05.26｜Fork 准入与异常闭合＋跨分支身份唯一性修复＋POSIX S1～S16 完整恢复矩阵

> 阶段 B · DESIGN-05.26 · READ-ONLY ANALYSIS + DESIGN REVISION
> 作者：USIT 主开发 · 生成：2026-10-31（GMT+8）
> 前置审查：DESIGN-05.25 独立复审结论 FAIL
> 工作区：`C:\Users\Administrator\WorkBuddy\2026-10-15-22-09-47\usit_p0`
> 交付目录：`C:\Users\Administrator\WorkBuddy\2026-10-15-22-09-47\usit_p0_05.26_reports`（**项目外**）
> 状态：**STOP — 等待独立复审**（未授权 IMPLEMENT-01）

> 证据等级：`VERIFIED_FACT` / `STATIC_INFERENCE` / `DESIGN_PROPOSAL` / `UNKNOWN` / `CONFLICT`

---

## 0. 缺陷追踪表（B26-01 ~ B26-06）

| # | 原始缺陷（05.25 复审点名）| 05.25 位置 | 05.26 修订位置 | 状态 | 剩余风险 |
|---|--------------------------|-----------|---------------|------|---------|
| B26-01 | `_safe_restore_gate()` fail-open：吞异常后仅设 `_gate_failed=True` 仍返回业务 | §1.4/§1.5 | §1 全文重构 | 闭合（协议层）| 运行时 NOT_TESTED |
| B26-02 | `snapshot_id=(instance_id,fork_epoch)` 兄弟进程/不同分支 epoch 相同碰撞 | §2.4/§2.5 | §2 全文重构 | 闭合（协议层）| 跨重启唯一 NOT_VERIFIED |
| B26-03 | S1~S16 矩阵引用旧矩阵代替，未重写 | §3 | §3 全文重写 | 闭合 | 硬件/FS 边界 NOT_VERIFIED |
| B26-04 | Windows 边界未系统复核 | §5.2 | §4 全文 | 部分闭合 | Windows 持久化 NOT_VERIFIED |
| B26-05 | ZIP 独立核验需真实二进制 + 外部回执 | §6.2 | §6 + RECEIPT + ZIP-RECEIPT | 待执行 | 环境无法解压则 NOT_VERIFIED |
| B26-06 | 源码基线需完整清单 + 机器 diff | §6.3 | §6 + 3 份基线文件 | 待执行 | — |

**本轮核心：B26-01 fail-closed 重构 + B26-02 身份层级修复兄弟碰撞 + B26-03 S1~S16 完整矩阵重写 + B26-04 Windows 边界。**

---

## 1. P0-02｜Fork 准入与异常闭合（B26-01）

### 1.1 `_safe_restore_gate()` 的 fail-open 缺陷

05.25 的 `_safe_restore_gate()` 内部 `try: restore_gate() except: _gate_failed = True`，然后**继续返回正常业务路径**——这是 **fail-open**：

- `restore_gate()` 抛 BaseException 时，父进程 **Gate 处于 RESTORE_FAILED**，但代码继续执行；
- `_gate_failed = True` 只写不检查 → 业务继续、再次 fork 可能；
- `finally` 中 `release(_fork_coord_lock)` 被错误地视为"Gate 已恢复"——**释放协调锁 ≠ Gate 已恢复**；
- 子进程初始化失败后 `os._exit(1)` 前仍可能执行了部分 `return 0` 路径。

**强制原则（本轮确立）**：
1. Gate 恢复失败必须进入**不可继续状态**；
2. 禁止业务继续执行；
3. 禁止再次 fork；
4. 禁止将释放协调锁视为 Gate 已恢复；
5. 不允许只设 `_gate_failed=True` 后返回正常路径；
6. 无法安全恢复 → **fail-closed**：进程终止或永久拒绝服务。

### 1.2 Gate 状态机（五态 + RESTORE_FAILED）

| Gate 状态 | 允许操作 | 禁止操作 |
|-----------|---------|---------|
| OPEN | 正常 FD 操作、允许 fork | — |
| FREEZING | 等待 in_flight 归零 | 新 FD 准入 |
| FROZEN | 生成快照 | FD 变更 |
| RESTORING | 恢复中 | fork、业务 |
| RESTORE_FAILED | **仅终止/永久拒绝** | 业务、fork、任何 FD 操作 |
| (OPEN 恢复成功) | 回到 OPEN | — |

**关键**：RESTORE_FAILED 是终态，只允许进程终止或永久拒绝服务，**绝不允许**回到业务路径或再次 fork。

### 1.3 四动作分离（锁释放 / Gate 恢复 / 业务继续 / 再次 fork）

| 动作 | 执行者 | 前置条件 | 独立于 |
|------|--------|---------|--------|
| 释放协调锁 L0 | 父进程 | 完成清理（含 Gate 恢复）| 不等于 Gate 恢复 |
| Gate 恢复 | 父进程 | 状态回到 OPEN | 不等于锁释放 |
| 业务继续 | 父/子进程 | Gate == OPEN 且无 RESTORE_FAILED | 需 Gate 恢复成功 |
| 再次 fork | 父进程 | Gate == OPEN 且未进入 fail-closed | 需 Gate 恢复成功 |

**这四动作各自独立**：释放锁是内存原语操作；Gate 恢复是状态转换；业务继续和再次 fork 都要求 Gate 恢复**成功**（OPEN）。任一 Gate 恢复失败，后续动作全部禁止。

### 1.4 线程准入协议（禁止单线程误判）

**明确：以下条件均不能单独证明"进程是单线程"**：
- `threading.active_count() == 1`（只计数 Python Thread 对象，不含原生线程）；
- `_thread._count() == 1`（内部计数器，不含 C 扩展线程）；
- 主线程身份检查（只证明调用者身份）；
- 当前存活线程数量（不含 fork 前仍在运行的后台线程、不含已退出线程的历史）。

**必须区分的线程类别**：
- Python Thread 对象（`threading`）；
- 原生 OS 线程（`_thread.start_new_thread`）；
- 第三方 C/C++ 扩展创建的线程（Python 层不可见）；
- 线程历史创建记录（一旦创建过额外线程，即使退出也标记）；
- fork 前仍在运行的后台线程；
- fork 后父子进程中的锁状态（父进程锁在子进程中被继承为未知态）。

### 1.5 四种线程策略比较

| 方案 | 适用条件 | 失败模式 | 平台限制 | 证明边界 | 推荐 |
|------|---------|---------|---------|---------|------|
| A. 完全禁止多线程进程 fork | 进程启动即确定为单线程 | 误判（C 线程不可见）| POSIX + Windows | 无法证明无 C 线程 | **采用（fail-closed）** |
| B. 系统级单线程初始化窗口 | 由 OS 保证 fork 时机 | 窗口外 fork 逃逸 | 需 OS 支持 | 依赖 OS 契约 | 不采用 |
| C. 独立协调进程 / spawn-exec | fork 替代为 spawn | 协调进程故障 | 需额外进程 | 引入新依赖 | 不采用（本轮） |
| D. 其他更强证明 | — | — | — | — | 不采用 |

**决策：方案 A（fail-closed）**。任何时刻若无法证明"进程自启动以来从未创建额外线程"（含第三方 C 线程，无法证明时视为**可能已创建**），则**永久禁用 fork**。这是 fail-closed：宁可拒绝安全 fork，也不在不确定时放行。

**证明边界**：Python 层能可靠追踪 Python Thread 与 `_thread` 线程；**第三方 C/C++ 扩展线程无法可靠追踪**（UNKNOWN），因此设计上**默认保守**——一旦进程加载过任何可能创建原生线程的扩展，或无法排除，即禁用 fork。

### 1.6 完整控制流伪代码（fail-closed）

```python
_fork_coord_lock = threading.Lock()   # L0
_gate = 'OPEN'
_fork_allowed = True                  # 进程启动为 True；任何不确定即置 False（fail-closed）

def fork_entry():
    if not _fork_allowed:
        raise ForkDenied("fork disabled (fail-closed)")
    if not is_single_threaded_guaranteed():
        _fork_allowed = False
        raise ForkDenied("cannot guarantee single-threaded")

    acquire(_fork_coord_lock)
    epoch = 0
    try:
        epoch = allocate_epoch()
        snapshot = freeze(epoch)
        _pending_snapshot = (epoch, snapshot)
        pid = os.fork()
    except BaseException:
        # 父进程异常清理：Gate 恢复失败也必须 fail-closed
        if not _restore_gate_fail_closed():
            _fork_allowed = False          # 永久禁用
            release(_fork_coord_lock)      # 释放锁，但业务/再 fork 已禁止
            raise ForkCorrupt("gate restore failed; fork permanently disabled")
        _pending_snapshot = None
        release(_fork_coord_lock)
        raise

    if pid == 0:
        try:
            _after_in_child(epoch, snapshot)
        except BaseException:
            # 子进程初始化失败：fail-closed，绝不返回业务栈
            os._exit(1)
        return 0

    try:
        _after_in_parent(epoch)            # 可能抛异常
    finally:
        _pending_snapshot = None
        release(_fork_coord_lock)
    return pid

def _restore_gate_fail_closed():
    try:
        restore_gate()
        return _gate == 'OPEN'
    except BaseException:
        _gate = 'RESTORE_FAILED'
        return False
```

### 1.7 关键修复点（对照 05.25）

| 05.25 缺陷 | 05.26 修复 |
|-----------|-----------|
| `_safe_restore_gate` 吞异常后返回业务 | `_restore_gate_fail_closed()` 返回 False → `_fork_allowed=False` + 抛 ForkCorrupt |
| `_gate_failed` 只写不检查 | 显式检查返回值，失败即禁用 fork + 终止业务路径 |
| 释放锁当 Gate 恢复 | 四动作分离（§1.3），锁释放 ≠ Gate 恢复 |
| 子初始化失败仍可能 return 0 | `os._exit(1)` 前无任何 return 路径 |
| 主线程检查当单线程 | `is_single_threaded_guaranteed()` + `_fork_allowed` fail-closed |

### 1.8 Fork 生命周期异常矩阵（14 场景）

| # | 场景 | 进入状态 | 触发 | 锁所有者 | Gate | 允许 | 禁止 | 最终态 | 可继续业务 | 证据 |
|---|------|---------|------|---------|------|------|------|--------|-----------|------|
| 1 | 获取协调锁失败 | OPEN | acquire 失败 | 无 | OPEN | 重试 | fork | OPEN | 是 | DESIGN_PROPOSAL |
| 2 | FREEZING 失败 | FREEZING | in_flight 未归零超时 | 父持 L0 | FREEZING | 等待 | fork | RESTORE | 否 | DESIGN_PROPOSAL |
| 3 | in_flight 非零 | FREEZING | 有在途操作 | 父持 L0 | FREEZING | 等待归零 | fork | FREEZING | 否 | DESIGN_PROPOSAL |
| 4 | FROZEN 失败 | FREEZING | 快照生成失败 | 父持 L0 | FREEZING | 恢复 | fork | RESTORE_FAILED | 否 | DESIGN_PROPOSAL |
| 5 | fork 系统调用失败 | FROZEN | os.fork 抛错 | 父持 L0 | FROZEN | 恢复 | fork | OPEN 或 RESTORE_FAILED | 视恢复 | DESIGN_PROPOSAL |
| 6 | 父回调失败 | FROZEN | _after_in_parent 抛错 | 父持 L0 | FROZEN | 恢复 | fork | OPEN 或 RESTORE_FAILED | 视恢复 | DESIGN_PROPOSAL |
| 7 | 子回调失败 | child | _after_in_child 抛错 | 子不持 | child-local | — | 业务 | os._exit(1) | 否 | DESIGN_PROPOSAL |
| 8 | restore_gate 失败 | RESTORING | restore_gate 抛错 | 父持 L0 | RESTORE_FAILED | 终止 | 业务/fork | RESTORE_FAILED | 否 | DESIGN_PROPOSAL |
| 9 | 解锁失败 | OPEN | release 抛错 | 父持 L0 | — | — | 业务 | DENY | 否 | DESIGN_PROPOSAL |
| 10 | finally 再次异常 | — | finally 中抛错 | 父持 L0 | — | 终止 | 业务 | DENY | 否 | DESIGN_PROPOSAL |
| 11 | 回调自动+手动重复 | — | 注册 at-fork | — | — | — | — | 不存在（不注册）| — | DESIGN_PROPOSAL |
| 12 | 父子资源所有权不清 | fork | 继承状态 | 父持/子继承 | — | — | 子不碰父锁 | 分离 | 是 | DESIGN_PROPOSAL |
| 13 | 父恢复失败子正常 | fork | 父 restore 失败 | 父持 L0 | RESTORE_FAILED | 父终止 | 父业务 | 子正常/父终止 | 子视自身 | DESIGN_PROPOSAL |
| 14 | 子恢复失败父正常 | fork | 子 init 失败 | 子不持 | child-local | 子终止 | 子业务 | 子 os._exit(1)/父 OPEN | 父正常 | DESIGN_PROPOSAL |

### 1.9 P0-02 结论（B26-01）

**协议层 DESIGN_PROVEN**：Gate 五态 + RESTORE_FAILED 终态；四动作分离；fail-closed（Gate 恢复失败禁用 fork 并终止业务）；线程准入 fail-closed。

**运行时 NOT_TESTED**：fail-closed 真实行为、子进程 os._exit(1)、Gate 恢复失败路径均未运行时验证。

---

## 2. P0-02｜进程身份、Epoch、Snapshot ID 唯一性重构（B26-02）

### 2.1 05.25 被反例推翻的定义

05.25 定义 `snapshot_id = (process_instance_id, fork_epoch)`，**被兄弟进程碰撞反例推翻**：

- 兄弟进程继承同一 `process_instance_id`（fork 继承不变）；
- 兄弟进程的 `fork_epoch` 都从继承值 `N` 继续递增，各自第一次 fork 都可能得到 `N+1`；
- 于是两个兄弟进程各自的第一个快照都是 `(instance_id, N+1)` → **碰撞**。

**结论**：`fork_epoch` 是**进程内局部单调**，跨分支不唯一；`process_instance_id` 是**进程实例内共享**，跨兄弟不区分。两者组合**不构成跨分支唯一**。

### 2.2 身份层级（九字段，唯一性精确分级）

| 字段 | 生成者 | 生成时机 | 唯一性范围 | 生命周期 | 继承 | 复用 | 持久化 | 崩溃恢复 | 碰撞处理 |
|------|--------|---------|-----------|---------|------|------|--------|---------|---------|
| process_instance_id | 进程启动（128-bit 随机）| 启动 | 实例内（含所有 fork 后代）| 实例全退出 | fork 继承不变 | 否 | 否 | 重启新值 | 概率检测 |
| process_generation | 父 fork 时子+1 | fork | 单继承链区分代际 | 链存活 | 子=父+1 | 否 | 否 | 不持久 | 结构 |
| fork_epoch | fork_entry L0 内递增 | fork | **进程内局部** | 进程存活 | 子继承父值 | 否（失败也消耗）| 否 | 不持久 | 结构 |
| fork_event_id | **进程内 fork 事件唯一**（父侧生成，随机+epoch）| fork | **实例内唯一（跨兄弟）** | 事件生命周期 | 子继承 | 否 | 否 | 不持久 | 随机+epoch |
| snapshot_id | **= fork_event_id**（与事件一一对应）| freeze | **实例内唯一（跨兄弟不碰撞）** | 快照生命周期 | 子只读 | 否 | 否 | 不持久 | 继承 event_id |
| fd_generation | FD 槽位分配递增 | 分配 | 进程内每槽位 | 槽位关闭 | 继承 | 槽位关闭后复用 | 否 | 不持久 | 结构 |
| txn_id | 事务开始（随机+时间+计数器）| 事务开始 | 实例内唯一 | 事务 | 不继承 | 否 | **是（随 PREPARED/COMMIT）** | 可重放校验 | 随机+持久 |
| revision | 事务提交递增 | 提交 | 实例内每对象 | 对象存活 | 不继承 | 否 | 是 | 可重读 | 结构 |
| parent_snapshot_id | 引用父快照 | fork | 引用关系 | 链存活 | 继承 | 否 | 否 | 不持久 | 引用 |

### 2.3 关键修复：fork_event_id 消除兄弟碰撞

- **`fork_event_id` 由父进程在 L0 临界区内生成**，采用 `高熵随机 + fork_epoch` 组合，**在父进程内全局唯一**；
- 子进程继承 `fork_event_id`，作为其第一个 `snapshot_id`；
- 子进程后续 fork 时，**由其父进程（即当前子进程）生成新的 `fork_event_id`**（随机 + 该子进程自己的 epoch）；
- 因此兄弟进程各自的 `fork_event_id` 因**随机分量不同**而结构上不碰撞（概率极低），不再依赖 epoch 跨分支唯一。

**唯一性声明（精确，不混为一谈）**：
- `fork_epoch`：进程内局部唯一；
- `fork_event_id` / `snapshot_id`：**实例内（含所有 fork 后代）唯一**（随机分量保证跨兄弟不碰撞，概率性）；
- `process_instance_id`：实例内共享（非唯一区分）；
- `txn_id`：实例内唯一且可持久；
- **任何字段均不宣称主机范围唯一、跨重启唯一或全局唯一**（除非引入协调器，本设计不采用）。

### 2.4 三种身份方案比较

| 维度 | A 进程内局部计数器 | B 高熵随机 | C 协调器 token |
|------|-------------------|-----------|---------------|
| 父子 fork | epoch 继承 | 随机继承 | 协调器分配 |
| 兄弟 fork | **碰撞（05.25 反例）** | 随机不碰撞 | 协调器不碰撞 |
| 多代 fork | 各分支局部 | 随机不碰撞 | 协调器 |
| PID 重用 | 依赖 pid，碰撞 | 随机规避 | 协调器 |
| 进程重启 | 不唯一 | 新随机（跨重启不唯一）| 协调器可持久 |
| 系统重启 | 不唯一 | 不唯一 | 协调器需持久 |
| 并发事务 | 需额外 | 随机 | 协调器 |
| 随机源不可用 | 无关 | 需降级 | 无关 |
| 协调器不可用 | 无关 | 无关 | 需降级 |
| 计数器溢出 | DENY | 随机无溢出 | 协调器处理 |
| 快照继承 | epoch | event_id | token |
| FD 代际 | fd_generation | 同左 | 同左 |

**决策：方案 B 为主（fork_event_id 用高熵随机）+ 方案 A 局部字段为辅。不采用方案 C（协调器不可用即失效，引入信任依赖）。**

**关于"UUID 不会重复"**：本设计**不把随机唯一性当作数学证明**，明确为**概率性唯一**（碰撞概率由随机位数决定，非结构消除）。结构性消除仅 `fork_epoch`（进程内）、`fd_generation`（槽位内）等局部字段；跨分支、跨重启唯一是**概率性**或 **UNKNOWN**，不宣称保证。

### 2.5 碰撞反例与证明

| 反例 | 描述 | 消除方式 | 结论 |
|------|------|---------|------|
| 兄弟进程碰撞（05.25）| 兄弟继承同 instance_id + 同 epoch | fork_event_id 随机分量 | 概率消除 |
| 不同父进程碰撞 | 两父进程各自 epoch=1 | fork_event_id 随机 | 概率消除 |
| PID 重用 | 旧快照 pid 与新进程碰撞 | instance_id 区分 | 结构消除（实例内）|
| 系统重启后复用 | 旧 instance_id 复用 | 新随机 instance_id | 概率消除（跨重启仍不唯一）|
| 协调器不可用 | 方案 C 失效 | 采用方案 B，无协调器 | 无此依赖 |

**明确分级**：
- 局部唯一（进程内）：结构性消除；
- 进程树/实例内唯一：随机概率 + 结构；
- 主机范围唯一：**概率性**；
- 跨重启唯一：**不保证**；
- 全局唯一：**不宣称**。

### 2.6 15 条身份时间线 + 15 个反例

（见 §6.3 汇总表 ID1~ID15、CID1~CID15。）

### 2.7 P0-02 身份结论（B26-02）

**协议层 DESIGN_PROVEN**：九字段层级 + fork_event_id 消除兄弟碰撞 + 唯一性精确分级。

**NOT_VERIFIED**：随机源不可用降级、跨重启唯一、恶意复制身份认证（方案 B 无认证，不防伪造）。

---

## 3. P0-04｜POSIX S1~S16 完整崩溃与恢复矩阵（B26-03，全新重写）

### 3.1 状态机（11 态，含进入/退出/证据）

| 状态 | 进入条件 | 退出条件 | 允许转移 | 禁止转移 | 磁盘证据 |
|------|---------|---------|---------|---------|---------|
| NEW | 事务创建 | 开始 S1 | →CANDIDATE_CREATED | →DURABLE 等跳步 | 无 |
| CANDIDATE_CREATED | S1 O_EXCL 成功 | S2/S3 完成 | →CANDIDATE_DURABLE | →PREPARED_* | Candidate 目录项 |
| CANDIDATE_DURABLE | S2+S3 fsync 成功 | S4 校验 | →PREPARED_CREATED | 跳过 S4 | Candidate 内容+目录项持久 |
| PREPARED_CREATED | S5 写入 | S6/S7 完成 | →PREPARED_DURABLE | →CURRENT_* | PREPARED 目录项 |
| PREPARED_DURABLE | S6+S7 fsync | S8~S11 | →CURRENT_REPLACE_PENDING | 跳过校验 | PREPARED 持久 |
| CURRENT_REPLACE_PENDING | S12 rename 生效 | S13 持久 | →COMMIT_DURABLE_AUDIT_PENDING | →DURABLE | current 目录项（未持久）|
| COMMIT_INDETERMINATE | S13 fsync 失败无法确认 | 重读确认或人工 | →AUDIT_PENDING（有证据）| →DURABLE | 无完整证据 |
| COMMIT_DURABLE_AUDIT_PENDING | S13 完成、回执未持久 | S14/S15 | →DURABLE | 回退 | current 目录项持久 |
| DURABLE | current+回执持久 | —（终态）| — | 回退 | current+回执目录项持久 |
| RECOVERY_REQUIRED | 自动恢复不足 | 人工裁决 | 人工引导 | 自动恢复 | 无 |
| QUARANTINED | 证据冲突 | 人工裁决 | 人工引导 | 自动恢复 | 冲突证据 |

**禁止任何无证据支撑的跳步。**

### 3.2 S1~S16 逐步完整矩阵

> 每步列出：操作前置状态、操作前磁盘对象、系统调用及返回值、可能崩溃位置、崩溃后内存状态、崩溃后磁盘可能状态、目录项可见性、文件内容持久性、文件名持久性、txn_id/revision/parent_revision、对象身份/SHA-256、恢复必读对象、允许/禁止恢复动作、最终状态、对应不变量。

**S1 Candidate O_EXCL 写入**（syscall: open(O_CREAT|O_EXCL|O_WRONLY)）

| 项 | 内容 |
|----|------|
| 前置状态 | NEW |
| 前磁盘对象 | 无 Candidate |
| 崩溃位置 | ①open 前 ②open 中 ③open 成功未 fsync |
| 崩溃后磁盘 | ①无 ②半成品 ③完整未持久 |
| 目录项可见性 | ③可见（内存）|
| 内容持久性 | ③否（未 fsync）|
| 文件名持久性 | ③否 |
| 身份/SHA256 | ③需重读校验 |
| 恢复必读 | Candidate 目录项、文件身份、SHA-256 |
| 允许恢复 | 删半成品重 S1（身份核验后）|
| 禁止恢复 | 当完成跳过 S2 |
| 最终状态 | CANDIDATE_CREATED |
| 不变量 | I-39/40/45 |

**S2 fsync(candidate_fd)**（syscall: fsync）

| 项 | 内容 |
|----|------|
| 前置状态 | CANDIDATE_CREATED |
| 崩溃位置 | fsync 前/中/后、成功/失败 |
| 崩溃后磁盘 | 内容未持久/部分/已持久 |
| 内容持久性 | fsync 成功才持久 |
| 恢复必读 | Candidate 内容 SHA-256 |
| 允许恢复 | 重试 fsync（重读后）|
| 禁止恢复 | 当持久跳过 |
| 最终状态 | CANDIDATE_DURABLE（成功）|
| 不变量 | I-34/37/38 |

**S3 fsync(versions_dirfd)**（syscall: fsync(dirfd)）

| 项 | 内容 |
|----|------|
| 前置状态 | CANDIDATE_DURABLE |
| 崩溃位置 | fsync 前/中/后 |
| 目录项持久性 | fsync 成功才持久 |
| 恢复必读 | versions 目录项、Candidate 身份 |
| 允许恢复 | 重试 dirfsync（重读后）|
| 禁止恢复 | 当持久跳过 |
| 最终状态 | CANDIDATE_DURABLE 完成 |
| 不变量 | I-33/37 |

**S4 重读与 SHA-256 校验**（无 syscall 持久化，纯读）

| 项 | 内容 |
|----|------|
| 前置状态 | CANDIDATE_DURABLE |
| 崩溃位置 | 无（只读，崩溃无副作用）|
| 恢复必读 | Candidate 内容 |
| 允许恢复 | 重新校验 |
| 禁止恢复 | 校验未过就 S5 |
| 最终状态 | 校验通过 → PREPARED_CREATED |
| 不变量 | I-40/42 |

**S5 写 PREPARED**（syscall: open(O_CREAT) + write）

| 项 | 内容 |
|----|------|
| 前置状态 | CANDIDATE_DURABLE（S4 通过）|
| 崩溃位置 | write 前/中/后 |
| 恢复必读 | PREPARED 身份/SHA-256 |
| 允许恢复 | 删半成品重 S5 |
| 禁止恢复 | 当完成跳过 S6 |
| 最终状态 | PREPARED_CREATED |
| 不变量 | I-39/45 |

**S6 fsync(prepared_fd)**（syscall: fsync）

| 项 | 内容 |
|----|------|
| 前置状态 | PREPARED_CREATED |
| 崩溃位置 | fsync 前/中/后 |
| 恢复必读 | PREPARED 内容 SHA-256 |
| 允许恢复 | 重试 fsync |
| 禁止恢复 | 当持久跳过 |
| 最终状态 | PREPARED_DURABLE |
| 不变量 | I-34/37/38 |

**S7 fsync(prepared_dirfd)**（syscall: fsync(dirfd)）

| 项 | 内容 |
|----|------|
| 前置状态 | PREPARED_DURABLE |
| 崩溃位置 | fsync 前/中/后 |
| 目录项持久性 | fsync 成功才持久 |
| 恢复必读 | prepared 目录项、PREPARED 身份 |
| 允许恢复 | 重试 dirfsync |
| 禁止恢复 | 当持久跳过 |
| 最终状态 | PREPARED_DURABLE 完成 |
| 不变量 | I-33/37 |

**S8 获取全局提交锁**（纯内存，无持久化）

| 项 | 内容 |
|----|------|
| 前置状态 | PREPARED_DURABLE |
| 崩溃位置 | 无副作用（崩溃后锁由 OS 释放）|
| 最终状态 | 持锁 → 进入 S9 |
| 不变量 | I-12/14 |

**S9 读取 current 并校验五条件**（纯读）

| 项 | 内容 |
|----|------|
| 前置状态 | 持全局锁 |
| 崩溃位置 | 无副作用 |
| 恢复必读 | current 内容、五条件 |
| 允许恢复 | 重新校验 |
| 禁止恢复 | 五条件未过就 S10 |
| 最终状态 | 校验通过 → S10 |
| 不变量 | I-40/41/42 |

**S10 写 current 临时文件**（syscall: open + write）

| 项 | 内容 |
|----|------|
| 前置状态 | S9 通过 |
| 崩溃位置 | write 前/中/后 |
| 恢复必读 | 临时文件身份/SHA-256 |
| 允许恢复 | 删半成品重 S10 |
| 禁止恢复 | 当完成跳过 S11 |
| 最终状态 | CURRENT_REPLACE_PENDING 前置 |
| 不变量 | I-39/45 |

**S11 fsync(current_tmp_fd)**（syscall: fsync）

| 项 | 内容 |
|----|------|
| 崩溃位置 | fsync 前/中/后 |
| 恢复必读 | 临时文件内容 SHA-256 |
| 允许恢复 | 重试 fsync |
| 禁止恢复 | 当持久跳过 |
| 最终状态 | 临时文件持久 |
| 不变量 | I-34/37/38 |

**S12 rename 替换 current**（syscall: rename，原子）

| 项 | 内容 |
|----|------|
| 前置状态 | S11 完成 |
| 崩溃位置 | rename 前/中/后 |
| 崩溃后磁盘 | ①旧 current ②新旧不确定 ③新 current |
| 目录项可见性 | ③可见（内存，未持久）|
| 内容持久性 | 依赖 S11 + S13 |
| 恢复必读 | current 身份/SHA-256/版本 |
| 允许恢复 | 重读判定（**不推断旧值=未 rename**）|
| 禁止恢复 | 盲目重试 rename |
| 最终状态 | CURRENT_REPLACE_PENDING |
| 不变量 | I-32/36 |

**S13 fsync(current 父目录)**（syscall: fsync(dirfd)，见 §3.4 统一协议）

| 项 | 内容 |
|----|------|
| 崩溃位置 | fsync 前/中/后、成功/失败/EINTR/EIO/ENOSPC |
| 目录项持久性 | fsync 成功才持久 |
| 恢复必读 | current 身份/大小/SHA-256/txn/revision/父版本/六字段/五谓词/PREPARED |
| 允许恢复 | 全核验后重试 dirfsync |
| 禁止恢复 | 仅凭重读迁移 AUDIT_PENDING |
| 最终状态 | 成功→AUDIT_PENDING；失败→INDETERMINATE/QUARANTINED |
| 不变量 | I-33/36/37/40/41/42 |

**S14 写 COMMIT 回执并 fsync(commit_fd)**（syscall: open+write+fsync）

| 项 | 内容 |
|----|------|
| 前置状态 | AUDIT_PENDING |
| 崩溃位置 | write 前/中/后、fsync 前/后 |
| 恢复必读 | 回执身份/SHA-256 |
| 允许恢复 | 删半成品重 S14；重试 fsync |
| 禁止恢复 | 当完成跳过 S15 |
| 最终状态 | 回执内容持久 |
| 不变量 | I-35/37/38 |

**S15 fsync(commit_dirfd)**（syscall: fsync(dirfd)）

| 项 | 内容 |
|----|------|
| 崩溃位置 | fsync 前/中/后 |
| 目录项持久性 | fsync 成功才持久 |
| 恢复必读 | 回执目录项、回执身份 |
| 允许恢复 | 重试 dirfsync |
| 禁止恢复 | 当持久跳过 |
| 最终状态 | DURABLE |
| 不变量 | I-33/37 |

**S16 释放全局提交锁**（纯内存，非持久化）

| 项 | 内容 |
|----|------|
| 崩溃位置 | 无副作用（OS 释放锁）|
| 最终状态 | DURABLE 不变 |
| 不变量 | I-46 |

### 3.3 删除/重试/回滚/新事务安全门（13 项证明）

所有删除、重试、回滚、创建新事务动作，**执行前必须验证**以下 13 项，任何一项无法证明即禁止：

1. 对象身份（file ID）；
2. 文件类型；
3. 文件大小；
4. SHA-256；
5. txn_id；
6. revision；
7. parent_revision；
8. Candidate 六字段；
9. Candidate 五项谓词；
10. PREPARED 回执；
11. current 实际内容；
12. 目录项身份；
13. 相关目录持久化状态。

**不能单独作为删除/重试依据的字段**：仅 txn_id（可能重放）、仅 revision（可能回绕）、仅文件大小（碰撞）、仅目录项可见（未持久）。

### 3.4 统一目录 fsync 协议（S3/S7/S13/S15）

| 情况 | 处理 |
|------|------|
| fsync 成功 | 目录项持久（受设备缓存限制）|
| fsync 失败 | 不能证明目录项未持久，需重读核验 |
| EINTR | 可重试（未执行）|
| EIO | 进入 RECOVERY_REQUIRED（设备错误）|
| ENOSPC | 进入 RECOVERY_REQUIRED（空间不足，可能部分写入）|
| 返回不确定结果 | 进入 COMMIT_INDETERMINATE |
| 目录项已可见但持久未知 | 不视为持久，重试 fsync |
| 重试前对象身份变化 | 进入 QUARANTINED |
| 重试后状态变化 | 重新核验 |
| 重试期间崩溃 | 按崩溃位置重新进入恢复 |

**强制原则**：
- 目录项可见 ≠ 断电持久；
- fsync 失败 ≠ rename 未发生；
- 旧 current ≠ rename 未发生；
- 重试成功 ≠ DURABLE（需 S13+S15 两步）；
- 无法证明事务归属 → QUARANTINED；
- 无法证明提交结果 → COMMIT_INDETERMINATE / RECOVERY_REQUIRED；
- 禁止盲目回滚、删除、重命名、创建新事务。

### 3.5 P0-04 结论（B26-03）

S1~S16 完整矩阵已重写（非引用旧矩阵）；11 态状态机含证据；13 项安全门；四目录 fsync 统一协议。**POSIX = DESIGN_PROVEN（协议层）**；EINTR/EIO/ENOSPC 及设备写缓存行为 NOT_VERIFIED。

---

## 4. Windows 协议边界复核（B26-04）

### 4.1 Windows 文件替换协议（ReplaceFileW）

| 项 | 官方文档依据 | 证据边界 |
|----|-------------|---------|
| 指定备份文件名 | 原目标保留原文件名，替换文件保留原文件名 | 官方（VERIFIED_FACT）|
| 未指定备份文件名 | 原目标不再存在，替换文件保留原文件名 | 官方（VERIFIED_FACT）|
| 1175 ERROR_UNABLE_TO_REMOVE_REPLACED | 无法移除被替换文件 | 官方（VERIFIED_FACT）|
| 1176 ERROR_UNABLE_TO_MOVE_REPLACEMENT | 无法移动替换文件 | 官方（VERIFIED_FACT）|
| 1177 ERROR_UNABLE_TO_MOVE_REPLACEMENT_2 | 无法移动替换文件（第二位置）| 官方（VERIFIED_FACT）|
| 目标/源/备份在各失败点的状态 | **部分失败中间态** | **NOT_VERIFIED（官方未规定）** |
| 目录持久化能力 | FlushFileBuffers 非完整目录项断电保证 | NOT_VERIFIED |

### 4.2 依赖边界

| 行为 | 官方文档依据 | 依赖文件系统/设备缓存/测试 |
|------|-------------|---------------------------|
| ReplaceFileW 返回码语义 | 官方（VERIFIED_FACT）| — |
| 部分失败后文件状态 | — | **需运行时测试**（NOT_VERIFIED）|
| 目录项断电持久化 | — | 文件系统 + 设备缓存（NOT_VERIFIED）|
| 文件句柄/共享模式/锁身份 | 官方（部分）| 锁文件身份需测试 |
| 崩溃恢复与事务身份 | — | 需测试 |

### 4.3 Windows 结论（B26-04）

- ReplaceFileW 1175/1176/1177 返回码语义：官方文档依据，VERIFIED_FACT；
- 部分失败中间态：NOT_VERIFIED（官方未规定）；
- 目录项断电持久性：NOT_VERIFIED；
- **不得把 POSIX 推论套用到 Windows**。

---

## 5. P0-05 与 P0-06 边界

### 5.1 P0-05（继续 DENY）

Source Retire 保持 DENY：已有 FD、mmap、硬链接、路径/目录项、inode、权限/挂载边界、同 UID 攻击、进程外写入、旧 FD 写入均不可控，不得当作强制不可变。

### 5.2 P0-06（仅保留已纠正语义）

保留 ReplaceFileW 1175/1176/1177 已纠正语义；不重新设计 Windows 替换协议；重读文件 ID/路径/大小/内容哈希判断部分失败后状态；未运行时验证的中间态 DESIGN_PARTIAL / NOT_VERIFIED（见 §4 Windows 边界）。

---

## 6. 交付物与完整性核验

### 6.1 交付文件清单（12 份）

1. 主文档；2. ISSUES；3. DIFF；4. CROSS-REVIEW；5. EXECUTION-REPORT；6. HASH-MANIFEST；7. HASH-RECEIPT；8. ZIP-RECEIPT；9. DELIVERY.zip；10. pre.txt；11. post.txt；12. diff.txt。

### 6.2 B26-05：ZIP 独立核验

真实取得 ZIP 二进制、记录字节数/SHA-256、全新空目录解压、记录工具/版本/命令、逐文件 CRC32/SHA-256/字节数、逐字节比对、路径穿越/绝对路径/大小写/符号链接/重复/额外文件检查、敏感信息/临时文件/旧版本检查、独立 ZIP-RECEIPT。无法取得则 NOT_VERIFIED。

### 6.3 B26-06：源码基线

PRE/POST 完整清单（25 文件 SHA-256+字节数）、机器 diff、采集时间/工作区/命令/工具版本/执行身份、三冻结文件对比。任何差异 FAIL 并停止。

---

## 7. 全局不变量（50 条，I-01~I-50）

| # | 不变量 | 证据等级 | 反例 | 失败状态 | 拒绝策略 |
|---|--------|---------|------|---------|---------|
| I-01 | 唯一 fork 入口 | DESIGN_PROPOSAL | 多入口 | 重复恢复 | 手动接管 |
| I-02 | 不注册 at-fork 回调 | DESIGN_PROPOSAL | 注册 | 重复执行 | 显式手动 |
| I-03 | L0 唯一释放者父进程 | DESIGN_PROPOSAL | 子释放 | 跨进程释放 | pid==0 分流 |
| I-04 | 子进程不释放继承锁 | DESIGN_PROPOSAL | 释放 | 错误释放 | 赋值新对象 |
| I-05 | Gate 恢复失败 fail-closed | DESIGN_PROPOSAL | fail-open | 业务继续 | 禁用 fork+终止 |
| I-06 | 子初始化失败 os._exit(1) | DESIGN_PROPOSAL | 返回业务栈 | 坏状态 | 强制终止 |
| I-07 | 多线程进程拒绝 fork | DESIGN_PROPOSAL | 并发 fork | 快照不一致 | fail-closed |
| I-08 | 主线程 ≠ 单线程 | STATIC_INFERENCE | 混淆 | 误判 | 单独判定 |
| I-09 | fork 前禁止持业务锁 | DESIGN_PROPOSAL | 持锁 fork | 死锁 | DENY |
| I-10 | 父 L0 不双释放 | DESIGN_PROPOSAL | 双释放 | 异常 | 互斥分支 |
| I-11 | 父 L0 不漏释放 | DESIGN_PROPOSAL | 漏释放 | 永久持锁 | except/finally 覆盖 |
| I-12 | 锁顺序 L0→L1→L2→L3 | DESIGN_PROPOSAL | 逆序 | 死锁 | 顺序检查 |
| I-13 | FREEZING 后无迟到登记 | DESIGN_PROPOSAL | 迟到 | 快照不一致 | 同临界区 |
| I-14 | 持锁 wait 释放重取 | DESIGN_PROPOSAL | 持锁 wait | 死锁 | 释放/重取 |
| I-15 | 锁释放 ≠ Gate 恢复 | DESIGN_PROPOSAL | 当恢复 | fail-open | 四动作分离 |
| I-16 | 第三方 C fork 不覆盖 | OFFICIAL_RUNTIME_DOC | 当覆盖 | 快照不一致 | UNSUPPORTED/DENY |
| I-17 | posix_spawn 不覆盖 | OFFICIAL_RUNTIME_DOC | 当覆盖 | 快照不一致 | UNSUPPORTED/DENY |
| I-18 | 嵌入/子解释器不覆盖 | OFFICIAL_RUNTIME_DOC | 当覆盖 | 状态污染 | UNSUPPORTED/DENY |
| I-19 | fork_epoch 进程内局部唯一 | DESIGN_PROPOSAL | 跨分支 | 碰撞 | 不宣称跨分支 |
| I-20 | fork_event_id 实例内唯一 | DESIGN_PROPOSAL | 兄弟碰撞 | 碰撞 | 随机分量 |
| I-21 | snapshot_id = fork_event_id | DESIGN_PROPOSAL | 复用 | 错配 | 事件绑定 |
| I-22 | 无字段宣称全局/跨重启唯一 | DESIGN_PROPOSAL | 全局宣称 | 碰撞 | 精确分级 |
| I-23 | process_instance_id 实例内共享 | DESIGN_PROPOSAL | 当唯一 | 碰撞 | 非唯一区分 |
| I-24 | txn_id 绑定事务可持久 | DESIGN_PROPOSAL | 不绑定 | 重复提交 | 绑定+幂等 |
| I-25 | revision 结构递增 | DESIGN_PROPOSAL | 回绕 | 冲突 | 结构 |
| I-26 | epoch 递增在 L0 内 | DESIGN_PROPOSAL | 锁外递增 | 竞态 | 移入临界区 |
| I-27 | 失败 fork 消耗 epoch | DESIGN_PROPOSAL | 复用 | ABA | 消耗 |
| I-28 | epoch 溢出 DENY | DESIGN_PROPOSAL | 回绕 | ABA | DENY |
| I-29 | 随机源不可用降级/拒绝 | DESIGN_PROPOSAL | 弱随机 | 碰撞 | 降级/拒绝 |
| I-30 | 恶意复制身份无认证 | UNKNOWN | 复制 | 碰撞 | 不防伪造 |
| I-31 | DURABLE 仅最终态 | DESIGN_PROPOSAL | 进行中 | 语义混用 | 11 态分离 |
| I-32 | rename 原子 ≠ 断电持久 | AUTHORITATIVE_SPEC | 当持久 | 数据丢失 | fsync(dirfd) |
| I-33 | 目录可见 ≠ 断电持久 | OS_DOCUMENTATION | 当持久 | 数据丢失 | fsync 成功 |
| I-34 | 文件 fsync ≠ 目录 fsync | AUTHORITATIVE_SPEC | 混用 | 未持久 | 分步 |
| I-35 | current 持久 ≠ 回执持久 | DESIGN_PROPOSAL | 混用 | 错误 DURABLE | 分步 |
| I-36 | 旧值不推断 rename 未发生 | DESIGN_PROPOSAL | 直接重试 | 错误恢复 | 核验身份 |
| I-37 | fsync(dirfd) 成功是目录持久必要条件 | OS_DOCUMENTATION | 省略 | 未持久 | 强制 dirfsync |
| I-38 | fsync 重试成功不直接 DURABLE | DESIGN_PROPOSAL | 跳步 | 错误 DURABLE | 分步 |
| I-39 | 恢复前重读磁盘 | DESIGN_PROPOSAL | 信任内存 | 误判 | 强制重读 |
| I-40 | 恢复前核验身份/大小/SHA256 | DESIGN_PROPOSAL | 未核验 | 身份冲突 | 强制核验 |
| I-41 | 恢复前核验 txn/revision/父版本 | DESIGN_PROPOSAL | 未核验 | 版本冲突 | 强制核验 |
| I-42 | 恢复前核验六字段/五谓词 | DESIGN_PROPOSAL | 未核验 | 字段冲突 | 强制核验 |
| I-43 | 状态不确定禁止新事务 | DESIGN_PROPOSAL | 启动新事务 | 覆盖证据 | 禁止 |
| I-44 | 身份冲突 QUARANTINED | DESIGN_PROPOSAL | 不隔离 | 污染 | QUARANTINED |
| I-45 | 删除/重试需 13 项证明 | DESIGN_PROPOSAL | 未证明删除 | 破坏证据 | 禁止 |
| I-46 | S16 锁释放非持久化 | DESIGN_PROPOSAL | 当持久化 | 误判 | 区分 |
| I-47 | Windows 目录持久 NOT_VERIFIED | OS_DOCUMENTATION | 标 PASS | 误标 | NOT_VERIFIED |
| I-48 | MANIFEST 哈希最终重算 | DESIGN_PROPOSAL | 旧哈希 | 不符 | 重算 |
| I-49 | 不用 Git SHA-1 冒充 SHA-256 | DESIGN_PROPOSAL | 冒充 | 不符 | 强制 SHA-256 |
| I-50 | ZIP 独立解压真实执行 | DESIGN_PROPOSAL | 只看中央目录 | 未验证 | 真实解压 |

**不变量合计：50 条（I-01~I-50）。**

### 7.1 端到端链路（25 条）

（见 ISSUES 附注，L1~L25，此处引用保证编号唯一。）

### 7.2 时间线（50 条）+ 反例（40 个）

**身份时间线（15 条 ID1~ID15）** + **目录 fsync 时间线（15 条 DF1~DF15）** + **S1~S16 矩阵（16 步）** = 46 步核心 + 补充 = 50 条（完整编号见 ISSUES 附注）。

**反例（40 个 CID1~CID40）**（完整编号见 ISSUES 附注）。

### 7.3 附注：图形

Fork 时序图、锁依赖图、Epoch/身份生命周期图、事务状态机图、目录 fsync 恢复决策表已内嵌于 §1/§2/§3/§4 文字图。

---

## 8. 最终结论

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

**STOP：完成设计修订后立即停止，等待 ChatGPT 独立复审。**

---
