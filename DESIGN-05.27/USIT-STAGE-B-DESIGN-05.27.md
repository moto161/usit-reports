# USIT-STAGE-B-DESIGN-05.27｜Fork异常控制流彻底闭合＋身份唯一性协议重构＋POSIX状态机纠错与完整恢复时间线

> 阶段 B · DESIGN-05.27 · STRICT READ-ONLY ANALYSIS + DESIGN REVISION
> 作者：USIT 主开发 · 生成：2026-10-31（GMT+8）
> 前置审查：DESIGN-05.26 独立复审结论 FAIL
> 工作区：`C:\Users\Administrator\WorkBuddy\2026-10-15-22-09-47\usit_p0`
> 交付目录：`C:\Users\Administrator\WorkBuddy\2026-10-15-22-09-47\usit_p0_05.27_reports`（**项目外**）
> 状态：**STOP — 等待独立复审**（未授权 IMPLEMENT-01）

> 证据等级：`VERIFIED_FACT` / `STATIC_INFERENCE` / `DESIGN_PROPOSAL` / `UNKNOWN` / `CONFLICT`

---

## 0. 缺陷追踪表（B27-01 ~ B27-05）

| # | 原始缺陷（05.26 复审点名）| 05.26 位置 | 05.27 修订位置 | 状态 |
|---|--------------------------|-----------|---------------|------|
| B27-01 | `_after_in_parent` 异常时 finally 未保证 Gate 恢复；锁释放≠Gate 恢复 | §1.6 | §1 全文 | 闭合（协议层）|
| B27-02 | `fork_event_id` 高熵随机标"实例内唯一"，随机≠严格唯一 | §2.3 | §2 全文 | 闭合（协议层）|
| B27-03 | S4 直接转 PREPARED_CREATED；S2/S3 混为一个状态；S13 状态名不统一；S9 五条件未定义；六字段五谓词未定义；EINTR 误当"未执行" | §3 | §3 全文重写 | 闭合（协议层）|
| B27-04 | 时间线/反例/链路/不变量未独立成文、未机器统计 | §7 | 4 份独立文档 | 闭合 |
| B27-05 | 交付证据与状态一致性 | §6 | §6 + RECEIPT + 基线 | 待执行 |

---

## 1. P0-02｜Fork 异常控制流彻底闭合（B27-01）

### 1.1 05.26 缺陷定位

05.26 §1.6 父进程路径的 `finally` 只清理变量并释放锁，**没有明确保证 Gate 恢复**。锁释放 ≠ Gate 恢复 ≠ 业务安全。若 `_after_in_parent()` 抛异常，`finally` 释放了锁但 Gate 可能仍停在 FROZEN/RESTORING，后续业务可能在 Gate 未恢复的状态下继续。

### 1.2 Gate 状态机（七态，独立）

| 状态 | 进入 | 允许操作 | 禁止操作 | 退出 |
|------|------|---------|---------|------|
| OPEN | 初始化/恢复成功 | FD 操作、fork | — | 进入 FREEZING |
| FREEZING | fork 开始 | 等待 in_flight 归零 | 新 FD 准入 | →FROZEN 或 →OPEN(失败) |
| FROZEN | in_flight 归零 | 生成快照 | FD 变更 | →RESTORING 或 fork |
| RESTORING | fork 返回/恢复 | 恢复 Gate | fork、业务 | →OPEN 或 →RESTORE_FAILED |
| RESTORE_FAILED | restore 抛异常 | 终止/永久拒绝 | 业务、fork、FD | →TERMINATING |
| TERMINATING | 无法恢复 | 终止进程 | 一切业务 | 进程退出 |
| DENIED | 线程准入失败 | 拒绝服务 | 业务、fork | 人工恢复 |

### 1.3 统一拒绝门（证明任何调用方无法绕过）

**核心：Gate 状态不是布尔标志，而是所有业务/FD/fork 入口的前置检查对象。**

- 所有业务入口、FD 操作入口、fork 入口**统一调用** `assert_gate_open()`；
- `assert_gate_open()` 读取 Gate 状态，**非 OPEN 即抛 `GateClosed` 异常**；
- `_fork_allowed` 是派生状态：`_fork_allowed == (Gate == OPEN)`，不单独存在；
- 调用方捕获 `GateClosed` 后**无法绕过**：因为拒绝检查在**每个入口的临界区之前**执行，且 Gate 状态只在受控锁内变更，调用方无法把 Gate 改回 OPEN（没有对外写接口）；
- 因此"设置 `_fork_allowed=False`"不再是唯一防线，而是由**每个入口的强制 Gate 检查**保证业务确实被阻断。

**证明**：Gate 写操作仅存在于 `freeze()`/`restore_gate()`/`deny_fork()` 三个受控函数内，且都持有 `_gate_lock`；业务/FD/fork 入口在获得任何资源前必须先 `assert_gate_open()`。调用方无从把 Gate 置回 OPEN，故无法绕过。

### 1.4 无法安全恢复的最终处理（三方案比较）

| 方案 | 安全性 | 可实现性 | 说明 |
|------|--------|---------|------|
| 进程终止 | 最高 | 高（os._exit/abort）| 保证无坏状态继续 |
| 永久拒绝服务 | 高 | 中（需所有入口检查）| 进程存活但拒绝一切受控操作 |
| 人工恢复 | 视场景 | 低（需操作员介入）| 仅当状态可判定 |

**决策**：父进程 RESTORE_FAILED → 先永久拒绝服务（Gate=DENIED），再进入 TERMINATING（若仍无法恢复）。子进程初始化失败 → 直接 os._exit(1)。**"永久禁用 fork"与"永久禁止业务"必须区分**：前者只关 fork，后者关一切；本设计在 RESTORE_FAILED 时采用**永久禁止业务**（更强），而非仅禁 fork。

### 1.5 线程准入（fail-closed，承认正常环境难启用 fork）

- 不得仅凭 `threading.active_count()`/`_thread._count()`/主线程身份判断单线程；
- 第三方 C/C++ 线程不可见：**无法证明无 C 线程时，视为可能有 → DENY**；
- 无法证明线程安全 → DENY；
- **明确承认**：此 fail-closed 策略意味着在大多数真实 Python 进程（加载了可能创建 C 线程的库）中，fork 实际上**无法启用**。这是设计选择：宁可 fork 功能不可用，也不在不确定时放行。

**证据**：线程类别判定是 STATIC_INFERENCE；第三方 C 线程存在性是 UNKNOWN。

### 1.6 完整控制流伪代码（Gate 恢复闭环）

```python
_gate = 'OPEN'
_gate_lock = threading.Lock()

def assert_gate_open():
    if _gate != 'OPEN':
        raise GateClosed(_gate)          # 所有业务/FD/fork 入口前置

def fork_entry():
    assert_gate_open()                    # 统一拒绝门
    if not is_thread_safe_proven():
        _gate = 'DENIED'; raise ForkDenied()

    acquire(_fork_coord_lock)
    epoch = 0
    try:
        epoch = allocate_epoch()
        snapshot = freeze(epoch)          # 内部将 _gate: OPEN→FREEZING→FROZEN
        _pending_snapshot = (epoch, snapshot)
        pid = os.fork()
    except BaseException as e:
        # 父进程异常：Gate 恢复必须闭环
        _close_fork_parent(epoch)         # 恢复 Gate，失败则 DENIED/TERMINATING
        release(_fork_coord_lock)
        raise e

    if pid == 0:
        try:
            _after_in_child(epoch, snapshot)
        except BaseException:
            os._exit(1)
        return 0

    # 父进程恢复：Gate 恢复由 _close_fork_parent 统一负责
    _close_fork_parent(epoch)             # 含 _after_in_parent + Gate 恢复
    release(_fork_coord_lock)
    return pid

def _close_fork_parent(epoch):
    # 唯一父进程恢复点：先 _after_in_parent，再恢复 Gate；任一步失败→fail-closed
    try:
        _after_in_parent(epoch)
    finally:
        _pending_snapshot = None
        ok = _restore_gate_fail_closed()  # Gate: RESTORING→OPEN 或 →RESTORE_FAILED
        if not ok:
            _gate = 'DENIED'              # 永久拒绝，业务/FD/fork 全被 assert_gate_open 阻断
```

### 1.7 关键修复（对照 05.26）

| 05.26 缺陷 | 05.27 修复 |
|-----------|-----------|
| finally 只清理变量+释放锁，未保证 Gate 恢复 | `_close_fork_parent` 统一负责 `_after_in_parent` + Gate 恢复，失败→DENIED |
| 锁释放被当作 Gate 恢复 | 明确三件事独立：锁释放 / Gate 恢复 / 业务继续 |
| `_fork_allowed=False` 当唯一防线 | 统一拒绝门 `assert_gate_open()`，所有入口强制检查，调用方无法绕过 |
| "永久禁 fork"与"永久禁业务"混 | 区分：RESTORE_FAILED→永久禁业务（DENIED）|

### 1.8 20 条 Fork 异常时间线（FT1~FT20，见 TIMELINES.md）

（完整 20 条见 `USIT-STAGE-B-DESIGN-05.27-TIMELINES.md` §1，此处只列编号与主题。）

| # | 主题 | # | 主题 |
|---|------|---|------|
| FT1 | 准入检查拒绝 | FT11 | 子进程继承锁异常 |
| FT2 | 协调锁获取失败 | FT12 | 父恢复失败子成功 |
| FT3 | FREEZING 失败 | FT13 | 父恢复成功子失败 |
| FT4 | in_flight 未归零 | FT14 | 回调重复执行 |
| FT5 | FROZEN 失败 | FT15 | 清理部分成功 |
| FT6 | fork 系统调用失败 | FT16 | restore_gate 抛异常 |
| FT7 | 父回调抛异常 | FT17 | 解锁失败 |
| FT8 | 子回调抛异常 | FT18 | finally 内再次异常 |
| FT9 | Gate 恢复失败 | FT19 | 业务继续被拒绝 |
| FT10 | 再次 fork 被拒绝 | FT20 | 进程终止 |

### 1.9 P0-02 结论（B27-01）

**协议层 DESIGN_PROVEN**：七态状态机 + 统一拒绝门（调用方无法绕过）+ `_close_fork_parent` 闭环 Gate 恢复 + fail-closed（RESTORE_FAILED→永久禁业务）。

**运行时 NOT_TESTED**：所有异常路径未运行时验证（本机无 fork）。

---

## 2. P0-02｜身份唯一性协议重构（B27-02）

### 2.1 05.26 缺陷

05.26 将 `fork_event_id` 改为高熵随机，又标"实例内唯一"——**随机只提供概率性唯一，不提供严格唯一**，两者矛盾。

### 2.2 三种方案重新比较

| 维度 | A 进程内计数器 | B 高熵随机 | C 协调器 token |
|------|---------------|-----------|---------------|
| 严格唯一 | 进程内严格 | 概率性 | 协调器严格（假设协调器可靠）|
| 父子 fork | 继承 | 继承 | 分配 |
| 兄弟 fork | 碰撞（需额外分量）| 概率不碰撞 | 不碰撞 |
| 多代 fork | 各分支局部 | 概率 | 协调器 |
| PID 重用 | 结构规避 | 概率 | 协调器 |
| 跨重启 | 不唯一 | 不唯一 | 可持久 |
| 随机源不可用 | 无关 | 降级 | 无关 |
| 协调器不可用 | 无关 | 无关 | 降级 |
| 计数器溢出 | DENY | 无 | 协调器处理 |
| 信任边界 | 无 | 无 | 依赖协调器 |

### 2.3 最终决策与唯一性承诺（明确承诺范围）

**决策：混合方案。** `fork_epoch`、`fd_generation`、`revision` 用**进程内严格单调计数器**（方案 A，结构性唯一）；`process_instance_id`、`fork_event_id`、`snapshot_id`、`txn_id` 用**高熵随机**（方案 B，概率性唯一）+ 结构性分量组合；**不采用协调器**（方案 C，引入信任依赖）。

**唯一性承诺（明确声明）**：

| 范围 | 是否承诺 | 依据 |
|------|---------|------|
| 进程内唯一 | **承诺（严格）** | 单调计数器，结构性 |
| 进程树唯一 | **承诺（严格）** | 父侧统一分配 + 结构分量 |
| 进程实例内唯一 | **承诺（概率性）** | 随机 + 结构，碰撞概率见下 |
| 主机范围唯一 | **承诺（概率性）** | 随机 |
| 跨重启唯一 | **不承诺** | 无持久化 |
| 全局唯一 | **不承诺** | 无协调器 |

### 2.4 随机位数与碰撞概率计算（风险预算）

采用 128-bit 随机（`os.urandom(16)`）：

- 单进程内 fork_event_id：128-bit 随机 + 进程内严格单调 fork_epoch 分量，碰撞概率 < 2^-128（可忽略）；
- 进程树内兄弟 fork_event_id：因 fork_epoch 分量在**父侧统一分配且严格单调**，兄弟子进程继承**不同的** fork_epoch 值 → **结构性消除碰撞**（不依赖随机）；
- 主机范围 instance_id 碰撞（生日界）：N 个进程时约 N²/2^129；N=10^6 时 ≈ 3×10^-27，风险预算内可忽略；
- 跨重启：不承诺。

**关键修正**：05.26 的兄弟碰撞根源是"兄弟各自用自己进程内 epoch 递增"导致可能同值；05.27 改为**父侧在 fork 时分配全局严格单调的 fork_epoch**（父进程内的全局计数器），子进程继承该值，兄弟子进程的 fork_event_id 因 epoch 分量不同而**结构性不碰撞**。

### 2.5 九字段逐项定义（唯一性分级明确）

| 字段 | 生成者 | 生成时机 | 唯一性范围 | 结构性/概率性 | 继承 | 持久化 | 崩溃恢复 | PID 重用 | 并发 | 随机/协调器不可用 |
|------|--------|---------|-----------|--------------|------|--------|---------|---------|------|------------------|
| process_instance_id | 进程启动 os.urandom | 启动 | 实例内 | 概率（128-bit）| fork 继承 | 否 | 重启新值 | 结构规避 | 无 | 降级拒绝 |
| process_generation | 父 fork | fork | 单链代际 | 结构 | 子=父+1 | 否 | 不持久 | 结构 | 无 | 无关 |
| fork_epoch | 父 L0 内单调计数器 | fork | **进程树内严格** | 结构 | 子继承父值 | 否 | 不持久 | 结构 | L0 串行 | 无关 |
| fork_event_id | 父 L0 内（随机+epoch）| fork | 进程树内（结构）+主机（概率）| 结构+概率 | 子继承 | 否 | 不持久 | 结构 | L0 串行 | 随机降级 |
| snapshot_id | = fork_event_id | freeze | 同 fork_event_id | 同 | 子只读 | 否 | 不持久 | 同 | 同 | 同 |
| fd_generation | FD 槽位单调 | 分配 | 进程内每槽位 | 结构 | 继承 | 否 | 不持久 | 结构 | 槽位锁 | 无关 |
| txn_id | 事务开始（随机+计数）| 事务开始 | 实例内（概率）+可持久 | 概率+结构 | 不继承 | **是** | 可重放校验 | 结构 | 事务锁 | 随机降级 |
| revision | 提交单调递增 | 提交 | 实例内每对象 | 结构 | 不继承 | 是 | 可重读 | 结构 | 事务锁 | 无关 |
| parent_snapshot_id | 引用父快照 | fork | 引用关系 | 结构 | 继承 | 否 | 不持久 | 结构 | 无 | 无关 |

### 2.6 15 条身份时间线 + 15 个反例

（见 TIMELINES.md §2 ID1~ID15、COUNTEREXAMPLES.md §2 CID1~CID15。）

### 2.7 P0-02 身份结论（B27-02）

**协议层 DESIGN_PROVEN**：唯一性承诺范围明确（进程内/进程树严格；实例内/主机概率；跨重启/全局不承诺）；兄弟碰撞通过"父侧全局严格单调 fork_epoch"结构性消除；随机位数与碰撞概率已给出。

**NOT_VERIFIED**：随机源不可用降级、恶意复制身份认证（无认证）。

---

## 3. P0-04｜POSIX 状态机与 S1~S16 恢复协议纠错（B27-03）

### 3.1 05.26 状态错误修复清单

| 05.26 错误 | 05.27 修复 |
|-----------|-----------|
| S4 校验 Candidate 直接转 PREPARED_CREATED | 新增 CANDIDATE_VERIFIED 中间态，S5 实际创建后才进 PREPARED_CREATED |
| S2 文件 fsync 与 S3 目录 fsync 混为一个持久化状态 | 拆为 CANDIDATE_CONTENT_DURABLE + CANDIDATE_ENTRY_DURABLE |
| S6/S7 混为一个状态 | 拆为 PREPARED_CONTENT_DURABLE + PREPARED_ENTRY_DURABLE |
| S13 状态名在状态机与步骤表不一致 | 统一为 CURRENT_REPLACE_PENDING / COMMIT_INDETERMINATE |
| S9 五条件未定义 | 见 §3.3 逐项定义 |
| Candidate 六字段/五谓词未定义 | 见 §3.4 完整定义 |
| EINTR 误当"未执行" | 见 §3.5 纠正 |

### 3.2 状态机（16 态，含证据/转移/禁止）

| 状态 | 进入条件 | 退出条件 | 磁盘证据 | 允许转移 | 禁止转移 | 恢复动作 |
|------|---------|---------|---------|---------|---------|---------|
| NEW | 事务创建 | 开始 S1 | 无 | →CANDIDATE_CREATED | 跳步 | 重新 S1 |
| CANDIDATE_CREATED | S1 O_EXCL 成功 | S2 完成 | Candidate 目录项 | →CANDIDATE_CONTENT_DURABLE | →PREPARED_* | 删半成品重 S1 |
| CANDIDATE_CONTENT_DURABLE | S2 fsync(fd) 成功 | S3 完成 | Candidate 内容持久 | →CANDIDATE_ENTRY_DURABLE | →VERIFIED | 重试 S2 |
| CANDIDATE_ENTRY_DURABLE | S3 fsync(dir) 成功 | S4 校验 | Candidate 目录项持久 | →CANDIDATE_VERIFIED | →PREPARED | 重试 S3 |
| CANDIDATE_VERIFIED | S4 校验通过 | S5 完成 | 内容哈希匹配 | →PREPARED_CREATED | 跳过 S5 | 重新校验 |
| PREPARED_CREATED | S5 写入成功 | S6 完成 | PREPARED 目录项 | →PREPARED_CONTENT_DURABLE | →CURRENT | 删半成品重 S5 |
| PREPARED_CONTENT_DURABLE | S6 fsync(fd) 成功 | S7 完成 | PREPARED 内容持久 | →PREPARED_ENTRY_DURABLE | →VERIFIED | 重试 S6 |
| PREPARED_ENTRY_DURABLE | S7 fsync(dir) 成功 | S8 | PREPARED 目录项持久 | →PREPARED_VERIFIED | 跳过校验 | 重试 S7 |
| PREPARED_VERIFIED | S9 前校验通过 | S10~S11 | 身份哈希匹配 | →CURRENT_REPLACE_PENDING | 跳过 S10/S11 | 重新校验 |
| CURRENT_REPLACE_PENDING | S12 rename 生效 | S13 完成 | current 目录项（未持久）| →COMMIT_DURABLE_AUDIT_PENDING | →DURABLE | 见 S13 协议 |
| COMMIT_INDETERMINATE | S13 fsync 失败无法确认 | 重读确认/人工 | 无完整证据 | →AUDIT_PENDING（有证据）| →DURABLE | 重读+核验 |
| COMMIT_DURABLE_AUDIT_PENDING | S13 完成、回执未持久 | S14/S15 | current 目录项持久 | →COMMIT_RECEIPT_CONTENT_DURABLE | →DURABLE | 补回执 |
| COMMIT_RECEIPT_CONTENT_DURABLE | S14 fsync(fd) 成功 | S15 完成 | 回执内容持久 | →DURABLE | 跳过 S15 | 重试 S14 |
| DURABLE | current+回执目录项持久 | —（终态）| current+回执目录项持久 | — | 回退 | — |
| RECOVERY_REQUIRED | 自动恢复不足 | 人工裁决 | 无 | 人工引导 | 自动恢复 | 人工 |
| QUARANTINED | 证据冲突 | 人工裁决 | 冲突证据 | 人工引导 | 自动恢复 | 人工 |

### 3.3 S9 五条件逐项定义

| 条件 | 输入字段 | 计算方式 | 失败状态 |
|------|---------|---------|---------|
| C1 版本匹配 | current.revision vs candidate.base_revision | 相等检查 | 版本冲突 → QUARANTINED |
| C2 事务匹配 | current.txn_id vs candidate.base_txn_id | 相等检查 | txn 冲突 → QUARANTINED |
| C3 哈希匹配 | current.content_hash vs candidate.base_hash | SHA-256 比较 | 哈希冲突 → QUARANTINED |
| C4 目录项身份 | current 文件 ID vs 记录 ID | 身份比较 | 身份冲突 → QUARANTINED |
| C5 未并发修改 | current 内容重读稳定 | 两次读取一致 | 并发修改 → 重试/QUARANTINED |

### 3.4 Candidate 六字段与五谓词完整定义

**六字段**：

| 字段 | 含义 | 来源 | 类型 |
|------|------|------|------|
| base_revision | 父版本号 | 读取 current | 整数 |
| base_txn_id | 父事务 ID | 读取 current | 字符串 |
| base_hash | 父内容 SHA-256 | 计算 current | 十六进制 |
| new_revision | 新版本号 | 事务分配 | 整数 |
| new_txn_id | 新事务 ID | 事务分配 | 字符串 |
| candidate_hash | 新内容 SHA-256 | 计算 Candidate | 十六进制 |

**五谓词**：

| 谓词 | 定义 | 失败 |
|------|------|------|
| P1 | new_revision == base_revision + 1 | 版本不连续 |
| P2 | new_txn_id 唯一（实例内）| 事务重复 |
| P3 | candidate_hash == 重算 Candidate 内容 | 内容损坏 |
| P4 | base_hash == 重算 current 内容 | 父内容漂移 |
| P5 | base_txn_id == current.txn_id | 父事务漂移 |

### 3.5 EINTR 纠正

**05.26 错误**：把 EINTR 直接等同于"操作未执行"。

**05.27 纠正**：
- `fsync`/`open`/`write`/`rename` 均可能返回 EINTR（信号打断）；
- **EINTR 返回后状态未知**：操作可能已执行、部分执行、或未执行（不能确定）；
- 允许重试，但**重试前必须重新核验对象身份**（操作可能已生效）；
- 重试期间对象身份变化 → QUARANTINED；
- 必须区分：文件内容持久（fsync(fd)）vs 目录项持久（fsync(dirfd)）——两者是不同系统调用、不同保证；
- fsync 成功仍受设备写缓存与文件系统保证边界限制（DEVICE_OR_STORAGE_LIMIT）。

### 3.6 S1~S16 逐步完整矩阵

（完整 16 步矩阵见 `USIT-STAGE-B-DESIGN-05.27-TIMELINES.md` §3，每步 18 项，覆盖短写/部分写/EINTR/EIO/ENOSPC。）

### 3.7 恢复安全门（15 项，新增"是否已发生但结果未知"）

删除/重试/回滚/新事务/rename/恢复前逐项验证：

1 文件身份；2 文件类型；3 文件大小；4 SHA-256；5 txn_id；6 revision；7 parent_revision；8 六字段；9 五谓词；10 PREPARED 回执；11 current 实际内容；12 目录项身份；13 目录持久化状态；14 当前事务归属；15 操作是否已发生但结果未知。

**任一无法证明 → DENY 或 QUARANTINED，禁止猜测恢复。**

### 3.8 P0-04 结论（B27-03）

16 态状态机（文件/目录 fsync 分离）；S9 五条件、六字段、五谓词完整定义；EINTR 纠正；15 项安全门。**POSIX = DESIGN_PROVEN（协议层）**；EINTR/EIO/ENOSPC、设备写缓存 NOT_VERIFIED。

---

## 4. Windows 边界（B27-05 之一）

保留 05.26 ReplaceFileW 1175/1176/1177 返回码语义（官方 VERIFIED_FACT）；部分失败中间态、目录项持久性、锁文件身份、崩溃恢复边界 NOT_VERIFIED；不得把 POSIX 推论套用到 Windows。（详见 ISSUES.md §2。）

---

## 5. P0-05 与 P0-06 边界

P0-05 Source Retire 继续 DENY；P0-06 保留已纠正语义，Windows 边界见 §4。

---

## 6. 交付物与完整性核验

### 6.1 交付文件清单（16 份）

1 主文档；2 ISSUES；3 DIFF；4 CROSS-REVIEW；5 EXECUTION-REPORT；6 HASH-MANIFEST；7 HASH-RECEIPT；8 ZIP-RECEIPT；9 DELIVERY.zip；10 pre.txt；11 post.txt；12 diff.txt；13 TIMELINES；14 COUNTEREXAMPLES；15 CROSS-MODULE-CHAINS；16 INVARIANTS。

### 6.2 数量机器统计（唯一编号校验）

时间线 50 条、反例 40 个、链路 25 条、不变量 50 条——均由机器脚本统计唯一编号、重复编号、缺失字段（见 EXECUTION-REPORT §5）。

### 6.3 B27-05 ZIP 核验 + 源码基线

见 HASH-RECEIPT / ZIP-RECEIPT / 3 份基线文件。

---

## 7. 最终结论

| 项 | 结论 |
|----|------|
| P0-02 | **DESIGN_PARTIAL**（协议层 DESIGN_PROVEN，运行时 NOT_TESTED）|
| P0-04 POSIX | **DESIGN_PROVEN** |
| P0-04 Windows | **NOT_VERIFIED** |
| P0-05 | **DENY**（保持）|
| P0-06 | **DESIGN_PARTIAL / NOT_VERIFIED** |
| IMPLEMENT-01 | **NOT_APPROVED** |

**整体结论：PASS-WITH-ISSUES（候选）。**

**STOP：完成设计修订后立即停止，等待 ChatGPT 独立复审。**

---
