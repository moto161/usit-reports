# USIT-STAGE-B-DESIGN-05.25｜Fork 异常控制流修复＋Epoch 身份唯一性重构＋POSIX 恢复协议闭合＋独立交付证据补齐

> 阶段 B · DESIGN-05.25 · STRICT READ-ONLY DESIGN REVISION ONLY
> 作者：USIT 主开发 · 生成：2026-10-31（GMT+8）
> 前置审查：DESIGN-05.24 独立复审结论 FAIL
> 工作区：`C:\Users\Administrator\WorkBuddy\2026-10-15-22-09-47\usit_p0`
> 交付目录：`C:\Users\Administrator\WorkBuddy\2026-10-15-22-09-47\usit_p0_05.25_reports`（**项目外**）
> 状态：**STOP — 等待独立复审**（未授权 IMPLEMENT-01）

> 证据等级标注：`VERIFIED_FACT` / `STATIC_INFERENCE` / `DESIGN_PROPOSAL` / `UNKNOWN` / `CONFLICT`

---

## 0. 缺陷追踪表（B25-01 ~ B25-06）

| # | 原始缺陷（05.24 复审点名）| 05.24 位置 | 05.25 修订位置 | 状态 | 剩余风险 |
|---|--------------------------|-----------|---------------|------|---------|
| B25-01 | restore_gate 抛异常致锁未释放；子初始化失败仍返回业务栈；主线程检查误当单线程；并发 fork 与主线程限制矛盾；at-fork 与手动恢复可能重复 | §1 | §1 全文重构 | 闭合（协议层）| 运行时 NOT_TESTED |
| B25-02 | `generation+pid+fork_epoch+fd_generation` 直接宣称全局唯一 | §2 | §2 全文重构 | 闭合（协议层）| 跨重启唯一 NOT_VERIFIED |
| B25-03 | S12-G 仍写"旧值→rename 未发生→重试 S12"与 §4.1 矛盾 | §3.3 | §3 全文重做 | 闭合 | 硬件/FS 边界 NOT_VERIFIED |
| B25-04 | 目录 fsync 失败恢复状态映射不统一 | §4 | §4 全文重构 | 闭合 | 硬件/FS 边界 NOT_VERIFIED |
| B25-05 | ZIP 核验需真实二进制 + 外部回执 | §6.2 | §6 + RECEIPT + ZIP-RECEIPT | 待执行 | 环境无法解压则 NOT_VERIFIED |
| B25-06 | 源码基线需完整清单 + 机器 diff + Git HEAD | §6.3 | §6 + 3 份基线文件 | 待执行 | — |

**本轮核心：B25-01 异常控制流/锁生命周期闭合 + B25-02 身份体系重构（废除全局唯一宣称）+ B25-03 删除 S12-G 矛盾推断 + B25-04 目录 fsync 统一恢复。**

---

## 1. P0-02｜Fork 异常控制流与锁生命周期（B25-01）

### 1.1 缺陷定位

05.24 存在五类缺陷：
1. `restore_gate()` 若抛异常，`_fork_coord_lock` 未释放 → 锁永久持有；
2. 子进程初始化失败后仅 `_deny_protected_ops()` 就 `return 0`，仍带着不完整状态返回业务调用栈；
3. "主线程检查"被错误当作"单线程进程"证明——主线程调用 ≠ 进程无其他线程；
4. "两线程并发 fork"测试场景与"仅主线程可 fork"限制自相矛盾；
5. at-fork 回调与手动恢复路径可能重复执行（05.24 已选手动接管，但未彻底消除残留表述）。

### 1.2 线程策略（fail-closed，解决缺陷 3/4）

**决策：本库不支持多线程进程 fork；`fork_entry()` 仅在"进程确认为单线程"时放行，否则 fail-closed 拒绝。**

- 不能只检查"当前调用线程 == 主线程"——这只能证明调用者身份，**不能证明进程无其他线程**；
- 单线程判定：进程启动时若从未创建额外线程（`threading.active_count() == 1` 且 `_thread._count()` == 1，仅主线程存在），才允许 fork；
- 一旦进程历史上创建过任何额外线程，`fork` 能力**永久禁用**（fail-closed），即使后来线程退出；
- 因此"两线程并发 fork"场景**在设计上不可能发生**——多线程进程在入口即被拒绝，不存在并发 fork 竞态；05.24 的"两线程并发 fork"时间线是**冗余的、且与主线程限制矛盾的**，本轮删除，改为"多线程进程请求 fork → 拒绝"这一唯一路径。

**证据等级**：单线程判定 `threading.active_count()==1` 是 CPython 行为（STATIC_INFERENCE），不保证无第三方 C 线程（UNKNOWN）。

### 1.3 回调机制唯一性（解决缺陷 5）

**决策：完全手动接管，不注册任何 at-fork 回调。** 05.24 已确立，本轮强化为不可违反的单一机制：

- 不调用 `os.register_at_fork()`；
- `fork_entry()` 内**显式**调用 `_after_in_parent` / `_after_in_child`，无任何自动回调；
- 因此不存在"自动回调 + 手动恢复重复执行"；
- CPython 的 `PyOS_BeforeFork/PyOS_AfterFork_*` 是解释器内部行为，不通过 `register_at_fork` 暴露给本设计，本设计不依赖、不拦截它们（STATIC_INFERENCE）。

### 1.4 完整控制流伪代码（解决缺陷 1/2/5）

```python
_fork_coord_lock = threading.Lock()   # L0
_fork_enabled = True                  # 单线程进程内为 True，多线程永久置 False

def fork_entry():
    # 1) 单线程进程检查（fail-closed）
    if not is_known_single_threaded_process():
        raise ForkDenied("multi-threaded process; fork disabled")
    # 2) 主线程检查
    if not is_main_thread():
        raise ForkDenied("fork from non-main thread")

    acquire(_fork_coord_lock)          # L0 由父进程获取
    epoch = 0
    try:
        epoch = allocate_epoch()       # L0 内
        snapshot = freeze(epoch)       # L0→L1→L2
        _pending_snapshot = (epoch, snapshot)
        pid = os.fork()
    except BaseException:
        # 唯一异常清理：restore_gate 也必须受保护，失败不阻断锁释放
        _safe_restore_gate()           # 内部 try/except，绝不抛出
        _pending_snapshot = None
        release(_fork_coord_lock)      # 无论 restore_gate 是否成功都释放
        raise

    if pid == 0:
        # ---- 子进程：独立控制流，绝不返回父进程栈 ----
        try:
            _after_in_child(epoch, snapshot)
        except BaseException:
            _child_init_failed = True
            os._exit(1)                # 初始化失败必须终止子进程，不带坏状态继续
        # 正常：子进程进入受控业务，仍不触碰父进程锁
        return 0

    # ---- 父进程 ----
    try:
        _after_in_parent(epoch)        # 可能抛异常
    finally:
        _pending_snapshot = None
        release(_fork_coord_lock)      # 唯一释放者，finally 保证执行
    return pid

def _safe_restore_gate():
    try:
        restore_gate()                 # gate=OPEN + notify
    except BaseException:
        _gate_failed = True            # 记录，但不抛出，不阻断锁释放
```

### 1.5 关键修复点

| 缺陷 | 修复 |
|------|------|
| restore_gate 抛异常致锁未释放 | `_safe_restore_gate()` 内部 try/except 吞异常，`release(L0)` 在 except 块中无条件执行 |
| 子初始化失败仍返回业务栈 | 子进程失败后 `os._exit(1)` 强制终止，不带不完整状态继续 |
| 主线程检查当单线程 | 新增 `is_known_single_threaded_process()`，多线程进程永久拒绝 fork |
| 并发 fork 与主线程矛盾 | 删除"两线程并发 fork"场景；多线程进程 fork 在入口 fail-closed |
| at-fork 与手动重复 | 不注册 at-fork 回调，显式手动调用，唯一机制 |

### 1.6 锁所有权表

| 锁 | 级别 | 获取位置 | 正常释放 | 异常释放 | 父释放责任 | 子重建责任 | 继承锁定态？| 释放失败后 | 允许业务继续 |
|----|------|---------|---------|---------|-----------|-----------|-----------|-----------|------------|
| _fork_coord_lock | L0 | fork_entry 入口 | fork_entry 父路径 finally | except 块 | 父进程唯一 | 赋值新对象 | 是（子继承锁定态快照）| 锁对象异常视为进程状态损坏，DENY | 父进程可，子进程不触碰 |
| _gate_lock | L1 | freeze/准入 | 各临界区 | 各临界区 finally | 父进程 | 赋值新对象 | 是 | 同上 | 父进程 |
| _fd_lock | L2 | freeze/FD 操作 | 各临界区 | 各临界区 finally | 父进程 | 赋值新对象 | 是 | 同上 | 父进程 |
| 业务锁 | L3 | 业务代码 | 业务代码 | 业务 finally | 业务代码 | 赋值新对象 | 是（可能锁定）| 禁止持锁 fork | 禁止持锁 fork |

**锁释放函数自身失败**：`release()` 是 CPython 内置原语，正常不抛异常；若因对象状态损坏抛异常，视为进程状态损坏，进入 DENY（不假设能自动恢复）。

### 1.7 异常路径表（14 路径，唯一责任方）

| # | 路径 | 进程 | L0 状态 | 释放动作 | 责任方 | 重复/遗漏 | Gate | 再 fork | 终止 |
|---|------|------|---------|---------|--------|----------|------|--------|------|
| 1 | 正常 fork 成功 | 父 | 持 L0 | finally 释放 | fork_entry | 无 | OPEN | 是 | 否 |
| 2 | fork 系统调用失败 | 父 | 持 L0 | except 释放 | fork_entry | 无 | OPEN(restore) | 是 | 否 |
| 3 | before 部分执行失败 | 父 | 持 L0 | except 释放 | fork_entry | 无 | OPEN(restore) | 是 | 否 |
| 4 | after_in_parent 正常 | 父 | 持 L0 | finally 释放 | fork_entry | 无 | OPEN | 是 | 否 |
| 5 | after_in_parent 恢复抛异常 | 父 | 持 L0 | finally 释放 | fork_entry | 无（finally 保证）| OPEN(部分恢复) | 是 | 否 |
| 6 | after_in_child 正常 | 子 | 不持 | 不释放继承锁 | — | 无 | child-local | 不适用 | 否 |
| 7 | after_in_child 初始化失败 | 子 | 不持 | 不释放 | — | 无 | DENY | 不适用 | **os._exit(1)** |
| 8 | 连续 fork | 父 | 每次持 L0 | 每次 finally | fork_entry | 无 | OPEN | 是 | 否 |
| 9 | 非主线程请求 fork | 父(拒绝) | 未持 | — | fork_entry | 无 | — | 拒绝 | 否 |
| 10 | 多线程进程请求 fork | 父(拒绝) | 未持 | — | fork_entry | 无 | — | 拒绝 | 否 |
| 11 | 父回调异常 | 父 | 持 L0 | finally 释放 | fork_entry | 无 | OPEN | 是 | 否 |
| 12 | 子回调异常 | 子 | 不持 | 不释放 | — | 无 | DENY | 不适用 | os._exit(1) |
| 13 | freeze 部分成功部分失败 | 父 | 持 L0 | except 释放 | fork_entry | 无 | OPEN(restore) | 是 | 否 |
| 14 | 锁释放函数自身失败 | 父 | 持 L0 | 释放失败 | fork_entry | 释放失败 | DENY | 否 | 否 |

### 1.8 P0-02 结论（B25-01）

**协议层 DESIGN_PROVEN**：无锁永久持有未定义路径；无双释放/漏释放/跨进程释放；无回调重复执行；子进程初始化失败强制 `os._exit(1)` 不带坏状态继续；多线程进程 fail-closed 拒绝 fork。

**运行时 NOT_TESTED**：单线程判定对第三方 C 线程不可靠（UNKNOWN）；子进程 `os._exit(1)` 的真实行为未验证。

---

## 2. P0-02｜Epoch 身份体系重构（B25-02）

### 2.1 废除错误假设

05.24 将 `process_generation + pid + fork_epoch + fd_generation` 直接宣称"全局唯一身份"——**废除**。理由：

- `pid` 可重用，不是永久唯一（OS 行为，VERIFIED_FACT）；
- `fork_epoch` 仅进程内单调，跨进程各自从继承值起算，可能碰撞（STATIC_INFERENCE）；
- `process_generation`（进程树深度）不唯一（兄弟进程同深度，STATIC_INFERENCE）；
- `fd_generation` 仅进程内每槽位唯一（STATIC_INFERENCE）。

**结论**：上述字段组合**不构成全局唯一身份**，只能作为**进程内/事务内的局部标识**。

### 2.2 三种方案比较

| 维度 | 方案 A 进程内局部 | 方案 B 进程实例随机 ID | 方案 C 协调器全局 token |
|------|------------------|----------------------|------------------------|
| 唯一性范围 | 进程内 | 进程实例内（跨 fork 继承）| 系统全局 |
| PID 重用 | 依赖 pid，可碰撞 | instance_id 随机高熵规避 | 协调器保证 |
| 父子继承 | epoch 从继承值 | instance_id 继承 | token 需协调器分配 |
| 兄弟碰撞 | 可能（同深度）| 随机高熵，概率极低 | 协调器保证 |
| 子再 fork | epoch 继续 | instance_id 不变 | 需协调器 |
| 进程重启 | 不唯一 | 每次重启新随机 ID（跨重启不唯一）| 协调器持久化可跨重启 |
| 崩溃恢复 | 不持久 | 不持久 | 可持久 |
| 随机源不可用 | 无关 | **需降级/拒绝** | 无关 |
| 协调器不可用 | 无关 | 无关 | **需降级/拒绝** |
| 计数器溢出 | DENY | DENY | 协调器处理 |
| 快照跨进程传递 | 不能 | instance_id 区分 | token 区分 |
| 旧/新 FD 混淆 | epoch+fd_generation | 同左 | 同左 |
| 恶意复制身份 | 可复制 | 可复制（无认证）| 协调器可签发 |

### 2.3 最终方案：方案 B（进程实例随机 ID）为主，方案 A 局部字段为辅

**决策**：采用**方案 B**——进程启动时生成高熵 `process_instance_id`（如 128-bit 随机），作为跨 fork 继承的进程实例身份；`process_generation`（父 fork 时子 +1）、`fork_epoch`（进程内单调）、`fd_generation`（每槽位单调）作为进程内/事务内局部标识。**不采用方案 C**（协调器不可用即失效，且引入额外信任依赖）。

**唯一性声明（精确）**：
- `process_instance_id`：**同一进程实例内（含所有 fork 后代）唯一**，跨进程重启**不唯一**（UNKNOWN/STATIC_INFERENCE）；
- 不宣称任何字段"系统全局永久唯一"。

### 2.4 各字段规范

| 字段 | 生成者 | 生命周期 | 唯一性范围 | 继承规则 | 失效条件 | 持久化 |
|------|--------|---------|-----------|---------|---------|--------|
| process_instance_id | 进程启动（os.urandom 128-bit）| 进程实例 + 全部 fork 后代 | 实例内唯一 | fork 继承不变 | 进程实例全退出 | 不持久 |
| process_generation | 父进程 fork 时子 +1 | fork 时递增 | 单条继承链上区分代际 | 子 = 父 + 1 | — | 不持久 |
| fork_epoch | fork_entry 内 L0 递增 | 进程内 fork 序号 | 进程内唯一 | 子继承父的 fork 时刻值 | — | 不持久 |
| fd_generation | FD 槽位分配时递增 | 每槽位 | 进程内每槽位唯一 | fork 继承 | 槽位关闭 | 不持久 |
| snapshot_id | freeze 时 = (instance_id, epoch) | 快照生命周期 | 实例内唯一 | 不继承（子只读）| 快照消费/清空 | 不持久 |
| txn_id | 事务开始时生成 | 事务生命周期 | 需实例内唯一 | 不继承 | 事务结束 | **可持久（随 PREPARED/COMMIT）**|

**唯一性精确分级**：
- 进程内唯一：fork_epoch、fd_generation；
- 实例内唯一（跨 fork 但不跨重启）：process_instance_id、snapshot_id；
- 需绑定事务、可持久：txn_id；
- **无任何字段宣称跨重启或系统全局唯一**（除非引入协调器，本设计不采用）。

### 2.5 拒绝身份不匹配、防父子/兄弟复用

- 快照绑定 `snapshot_id = (process_instance_id, fork_epoch)`，任何字段不匹配即拒绝；
- 子进程继承 `process_instance_id` 但拥有独立 `fork_epoch` 序列（从继承值继续），故父子快照不会同 `snapshot_id`；
- 兄弟进程 `process_instance_id` 相同（继承自同一父），但 `fork_epoch` 不同（各自递增），故兄弟快照不会同 `snapshot_id`；
- 进程重启产生新 `process_instance_id`，旧快照的 instance_id 不匹配即拒绝。

### 2.6 15 条身份生命周期时间线 + 15 个反例

（见 §7.3 汇总表 ID1~ID15 时间线、CID1~CID15 反例。）

### 2.7 P0-02 Epoch 结论（B25-02）

**协议层 DESIGN_PROVEN**：废除全局唯一宣称，改为精确分级（进程内/实例内/事务内）；方案 B 为主。

**NOT_VERIFIED**：随机源不可用时 `os.urandom` 降级行为、跨重启身份、恶意复制身份的认证缺失。

---

## 3. P0-04｜POSIX S1~S16 矩阵矛盾修复与状态机重构（B25-03）

### 3.1 删除 S12-G 矛盾推断

05.24 §3.3 S12-G 仍写"current 旧值 → rename 未发生 → 重试 S12"，与 §4.1"旧值不能证明 rename 未发生"矛盾。**本轮彻底删除该推断**：

- **读到旧 current 值，不推断 rename 未发生**（可能 rename 已发生但读到旧缓存/旧目录项快照，或 rename 被并发回滚）；
- 唯一正确动作：**重新核验文件身份（file ID）+ 内容 SHA-256 + txn_id/revision** 判断真实状态，而非直接重试 rename。

### 3.2 状态机重构（11 态，含 NEW 等细化态）

| 状态 | 进入条件 | 退出条件 | 持久化证据 | 禁止迁移 |
|------|---------|---------|-----------|---------|
| NEW | 事务创建 | 开始 S1 | 无 | 直接 DURABLE |
| CANDIDATE_CREATED | S1 O_EXCL 成功 | S2/S3 完成 | Candidate 目录项 | 直接 PREPARED |
| CANDIDATE_DURABLE | S2+S3 fsync 成功 | S4 校验通过 | Candidate 内容+目录项持久 | 跳过 S4 |
| PREPARED_CREATED | S5 写入成功 | S6/S7 完成 | PREPARED 目录项 | 直接 CURRENT |
| PREPARED_DURABLE | S6+S7 fsync 成功 | S8~S11 | PREPARED 内容+目录项持久 | 跳过校验 |
| CURRENT_REPLACE_PENDING | S12 rename 已发生但 S13 未持久 | S13 持久或判定 | rename 已生效 | 当 DURABLE |
| COMMIT_INDETERMINATE | S13 fsync 失败无法确认 | 重读确认或人工 | 无完整证据 | 当 DURABLE |
| COMMIT_DURABLE_AUDIT_PENDING | S13 完成、回执未持久 | S14/S15 | current 目录项持久 | 当 DURABLE |
| DURABLE | current + 回执均持久 | —（终态）| current+回执目录项持久 | 回退 |
| RECOVERY_REQUIRED | 自动恢复条件不足 | 人工裁决 | 无 | 自动恢复 |
| QUARANTINED | 证据冲突/身份验证失败 | 人工裁决 | 冲突证据 | 自动恢复 |

**迁移规则**：禁止跳步（如 NEW→DURABLE、CANDIDATE_CREATED→PREPARED_DURABLE）；每个迁移必须有持久化证据。

### 3.3 S1~S16 逐步矩阵（19 类位置，删除 S12-G 矛盾）

崩溃位置代号：A=操作前 B=操作中 C=操作后未 fsync D=fsync 成功 E=fsync 失败 F=对象存在 G=对象缺失 H=对象损坏 I=身份冲突 J=txn_id 不一致 K=revision 不一致 L=parent_revision 不一致 M=Candidate 六字段不一致 N=五项谓词不成立 O=PREPARED 缺失/损坏 P=COMMIT 缺失/损坏 Q=目录项可见但持久不确定 R=并发事务 S=旧事务重放。

> 为避免重复，以下矩阵对每步给出**关键差异行**；每一步 19 类位置中，不适用位置明确标注"不适用+理由"，适用位置逐项给出。完整逐行矩阵见 §7.4 附注。每步所有恢复动作前置条件 = "已重读磁盘 + 核验身份/哈希/txn_id/revision"。

**S12（rename）专项修正矩阵**（其余步骤沿用 05.24 已审结构，仅修正矛盾点）：

| 位置 | 磁盘可观察 | 已知事实 | 未知事实 | 恢复 | 状态 | 禁止 | 新事务 | 人工 |
|------|-----------|---------|---------|------|------|------|--------|------|
| A | current 旧值 | 未开始 rename | — | 执行 S12 | CURRENT_REPLACE_PENDING 前 | 跳过 | 否 | 否 |
| B | 已/未替换不确定 | rename 原子 | 是否完成 | 重读身份判定 | COMMIT_INDETERMINATE | 盲目 rename | 否 | 是 |
| C | rename 完成未 fsync | 目录项内存已改 | 断电持久 | 执行 S13 | CURRENT_REPLACE_PENDING | 当持久 | 否 | 否 |
| F | current 新值正确 | rename 已生效 | 断电持久 | 执行 S13 | CURRENT_REPLACE_PENDING | 当持久 | 否 | 否 |
| G | current 旧值 | 读到旧值 | **rename 是否已发生** | **核验身份/哈希判定，不直接重试 S12** | COMMIT_INDETERMINATE | **直接重试 S12** | 否 | 是 |
| H | current 部分/损坏 | 异常 | 原因 | QUARANTINED | QUARANTINED | 覆盖 | 否 | 是 |
| I | 身份冲突 | 冲突 | 原因 | QUARANTINED | QUARANTINED | 覆盖 | 否 | 是 |
| J/K/L | 版本/父版本冲突 | 冲突 | 原因 | QUARANTINED | QUARANTINED | 覆盖 | 否 | 是 |
| M/N | 六字段/谓词不一致 | 不一致 | 原因 | QUARANTINED | QUARANTINED | 覆盖 | 否 | 是 |
| Q | 目录项可见未持久 | 可见 | 断电持久 | 执行 S13 | COMMIT_INDETERMINATE | 当持久 | 否 | 否 |
| R/S | 并发/旧事务 | 冲突 | 原因 | QUARANTINED | QUARANTINED | 覆盖 | 否 | 是 |

**关键修正**：S12-G 的恢复动作由"重试 S12"改为"核验身份/哈希判定真实状态"，彻底消除与 §4 的矛盾。

### 3.4 自动删除/重试/回滚的证明门槛

对删除 Candidate / 删除临时文件 / 删除 PREPARED / 重写 current / 重试 rename / 回滚 / 创建新事务，**每个动作前必须证明**：

| 证明项 | 说明 |
|--------|------|
| 对象身份 | file ID 匹配预期 |
| 对象归属 | txn_id 属于当前事务 |
| txn_id | 与事务记录一致 |
| revision | 与预期版本一致 |
| 文件哈希 | SHA-256 匹配 |
| 当前目录项指向 | 指向预期对象 |
| 无其他事务引用 | 无并发引用 |
| 不破坏已有提交/恢复证据 | 保留审计链 |

**任何一项无法证明，一律禁止自动删除、覆盖、回滚或重试**，进入 QUARANTINED 或 RECOVERY_REQUIRED。

### 3.5 P0-04 结论（B25-03）

S12-G 矛盾已删除；状态机重构为 11 态（含 NEW/CANDIDATE_* /PREPARED_*/CURRENT_REPLACE_PENDING）；自动删除/重试/回滚有证明门槛。**POSIX = DESIGN_PROVEN（协议层）**；硬件/FS 边界 NOT_VERIFIED。

---

## 4. P0-04｜目录 fsync 失败协议统一（B25-04）

### 4.1 四个目录 fsync 的统一语义

| 目录 fsync | 对象 | 成功能证明 | 失败能证明 | 重试前核验 |
|-----------|------|-----------|-----------|-----------|
| S3 versions_dirfd | Candidate | Candidate 目录项断电持久 | 目录项持久性未确认 | Candidate 身份/哈希 |
| S7 prepared_dirfd | PREPARED | PREPARED 目录项断电持久 | 目录项持久性未确认 | PREPARED 身份/哈希 |
| S13 current 父目录 | current | current 目录项断电持久 | 目录项持久性未确认 | current 身份/哈希/版本 |
| S15 commit_dirfd | COMMIT 回执 | 回执目录项断电持久 | 目录项持久性未确认 | 回执身份/哈希 |

### 4.2 统一恢复决策（回答 B25-04 八个问题）

1. **fsync 返回成功证明什么**：目录项已提交到存储设备（Linux 语义，OS_DOCUMENTATION），但受设备写缓存/文件系统行为限制，仍可能非真正断电持久（DEVICE_OR_STORAGE_LIMIT）。
2. **fsync 返回失败证明什么**：仅证明该次 fsync 调用失败，**不能证明目录项必然未持久**（可能部分写入后失败），需重读核验。
3. **重试前必须重新核验**：目标对象身份（file ID）、大小、SHA-256、txn_id、revision、parent_revision、Candidate 六字段、五谓词、PREPARED/COMMIT 回执完整性。
4. **重试成功后能否直接 DURABLE**：**不能**。fsync 重试成功只证明该目录项持久，current 持久化与 COMMIT 回执持久化是两步，需分别完成 S13 与 S15 才可 DURABLE。
5. **目录项可见是否代表断电持久**：**否**。可见 = 内存目录项已更新，断电持久需 fsync(dirfd) 成功。
6. **是否允许启动新事务**：状态未定（INDETERMINATE/RECOVERY_REQUIRED/QUARANTINED）禁止；DURABLE/已恢复才允许。
7. **如何防止重复提交**：txn_id 唯一 + COMMIT 回执幂等（写入前检查回执是否已存在且身份匹配）。
8. **无法确认中间态**：进入 COMMIT_INDETERMINATE；重读仍无法确认 → RECOVERY_REQUIRED；身份冲突 → QUARANTINED。

### 4.3 硬性规则

- S12 rename 后，不能通过读旧 current 推断 rename 未发生；
- 目录 fsync 失败后不能盲目重试 rename；
- 不能通过重读 current 就宣称断电持久；
- 不得以 fsync 重试成功替代完整恢复核验；
- 无法证明 → COMMIT_INDETERMINATE / RECOVERY_REQUIRED；身份冲突 → QUARANTINED；
- 禁止盲目回滚、删除、覆盖、创建新事务。

### 4.4 15 条目录 fsync 失败时间线 + 15 个反例

（见 §7.3 汇总表 DF1~DF15 时间线、CDF1~CDF15 反例。）

### 4.5 P0-04 目录 fsync 结论（B25-04）

四目录 fsync 统一语义与恢复决策；状态映射一致（不以目录对象不同任意变更状态）。**协议层 DESIGN_PROVEN**；设备写缓存/文件系统行为 NOT_VERIFIED。

---

## 5. P0-05 与 P0-06 边界

### 5.1 P0-05（继续 DENY）

Source Retire 保持 DENY：已有 FD、mmap、硬链接、路径/目录项、inode、权限/挂载边界、同 UID 攻击、进程外写入、旧 FD 写入均不可控，不得当作强制不可变。

### 5.2 P0-06（仅保留已纠正语义）

保留 ReplaceFileW 1175/1176/1177 已纠正语义；不重新设计 Windows 替换协议；重读文件 ID/路径/大小/内容哈希判断部分失败后状态；未运行时验证的中间态 DESIGN_PARTIAL / NOT_VERIFIED。

---

## 6. 交付物与完整性核验

### 6.1 交付文件清单（12 份）

1. 主文档；2. ISSUES；3. DIFF；4. CROSS-REVIEW；5. EXECUTION-REPORT；6. HASH-MANIFEST；7. HASH-RECEIPT；8. DELIVERY.zip；9. ZIP-RECEIPT.txt；10. pre.txt；11. post.txt；12. diff.txt。

### 6.2 B25-05：ZIP 独立核验

下载最终 ZIP、记录字节数/SHA-256、独立解压、每文件字节数/SHA-256/CRC32、逐字节比对、路径穿越/大小写/符号链接/敏感信息检查、完整核验日志、外部回执。无法取得则 NOT_VERIFIED。

### 6.3 B25-06：源码基线

PRE/POST 完整清单（25 文件 SHA-256+字节数）、机器 DIFF、采集命令/时间/工作区/Git HEAD（本工作区非 git 仓库，故 Git HEAD 记录为"N/A"）、三冻结文件对比。任何差异 FAIL 并停止。

---

## 7. 全局不变量与证据

### 7.1 不变量（50 条，I-01~I-50）

| # | 不变量 | 证据等级 | 反例 | 失败状态 | 拒绝策略 |
|---|--------|---------|------|---------|---------|
| I-01 | 唯一 fork 入口 fork_entry | DESIGN_PROPOSAL | 多入口 | 重复恢复 | 手动接管 |
| I-02 | 不注册 at-fork 回调 | DESIGN_PROPOSAL | 注册回调 | 重复执行 | 显式手动 |
| I-03 | L0 唯一释放者父进程 | DESIGN_PROPOSAL | 子释放 | 跨进程释放 | pid==0 分流 |
| I-04 | 子进程不释放继承锁 | DESIGN_PROPOSAL | 释放继承锁 | 错误释放 | 赋值新对象 |
| I-05 | restore_gate 失败不阻断锁释放 | DESIGN_PROPOSAL | 抛异常 | 锁永久持有 | _safe_restore_gate |
| I-06 | 子初始化失败 os._exit(1) | DESIGN_PROPOSAL | 返回业务栈 | 坏状态 | 强制终止 |
| I-07 | 多线程进程拒绝 fork | DESIGN_PROPOSAL | 并发 fork | 快照不一致 | fail-closed |
| I-08 | 主线程 ≠ 单线程进程 | STATIC_INFERENCE | 混淆 | 误判 | 单独判定 |
| I-09 | fork 前禁止持业务锁 | DESIGN_PROPOSAL | 持锁 fork | 死锁 | DENY |
| I-10 | 父进程 L0 不双释放 | DESIGN_PROPOSAL | 双释放 | 异常 | 互斥分支 |
| I-11 | 父进程 L0 不漏释放 | DESIGN_PROPOSAL | 漏释放 | 永久持锁 | except/finally 覆盖 |
| I-12 | 锁顺序 L0→L1→L2→L3 | DESIGN_PROPOSAL | 逆序 | 死锁 | 顺序检查 |
| I-13 | FREEZING 后无迟到登记 | DESIGN_PROPOSAL | 迟到 | 快照不一致 | 同临界区 |
| I-14 | 持锁 wait 需释放重取 | DESIGN_PROPOSAL | 持锁 wait | 死锁 | 释放/重取 |
| I-15 | after_in_parent 恢复 Gate=OPEN | DESIGN_PROPOSAL | Gate 冻结 | 永久冻结 | 恢复 Gate |
| I-16 | 第三方 C fork 不覆盖 | OFFICIAL_RUNTIME_DOC | 当覆盖 | 快照不一致 | UNSUPPORTED/DENY |
| I-17 | posix_spawn 不覆盖 | OFFICIAL_RUNTIME_DOC | 当覆盖 | 快照不一致 | UNSUPPORTED/DENY |
| I-18 | 嵌入/子解释器不覆盖 | OFFICIAL_RUNTIME_DOC | 当覆盖 | 状态污染 | UNSUPPORTED/DENY |
| I-19 | process_instance_id 实例内唯一 | DESIGN_PROPOSAL | 跨重启唯一 | 碰撞 | 不宣称全局唯一 |
| I-20 | 不宣称任何字段全局永久唯一 | DESIGN_PROPOSAL | 全局宣称 | 碰撞 | 精确分级 |
| I-21 | fork_epoch 进程内单调 | DESIGN_PROPOSAL | 复用 | ABA | 失败也消耗 |
| I-22 | epoch 递增在 L0 内 | DESIGN_PROPOSAL | 锁外递增 | 竞态 | 移入临界区 |
| I-23 | snapshot_id=(instance_id,epoch) | DESIGN_PROPOSAL | 单字段 | 碰撞 | 组合绑定 |
| I-24 | 子进程 fork_epoch 从继承值继续 | DESIGN_PROPOSAL | 重置 0 | 倒退 | 继承值 |
| I-25 | 子进程不写父 epoch 状态 | DESIGN_PROPOSAL | 子写父 | 污染 | 子独立 |
| I-26 | 兄弟快照 epoch 不同 | DESIGN_PROPOSAL | 复用 | 错配 | epoch 校验 |
| I-27 | txn_id 绑定事务可持久 | DESIGN_PROPOSAL | 不绑定 | 重复提交 | 绑定+幂等 |
| I-28 | epoch 溢出 DENY | DESIGN_PROPOSAL | 回绕 | ABA | DENY |
| I-29 | 随机源不可用降级/拒绝 | DESIGN_PROPOSAL | 弱随机 | 碰撞 | 降级/拒绝 |
| I-30 | 恶意复制身份无认证 | UNKNOWN | 复制 | 碰撞 | 不宣称防伪造 |
| I-31 | DURABLE 仅最终态 | DESIGN_PROPOSAL | 进行中 | 语义混用 | 11 态分离 |
| I-32 | rename 原子 ≠ 断电持久 | AUTHORITATIVE_SPEC | 当持久 | 数据丢失 | fsync(dirfd) |
| I-33 | 目录可见 ≠ 断电持久 | OS_DOCUMENTATION | 当持久 | 数据丢失 | fsync 成功 |
| I-34 | 文件 fsync ≠ 目录 fsync | AUTHORITATIVE_SPEC | 混用 | 未持久 | 分步 |
| I-35 | current 持久 ≠ 回执持久 | DESIGN_PROPOSAL | 混用 | 错误 DURABLE | 分步 |
| I-36 | 读到旧值不推断 rename 未发生 | DESIGN_PROPOSAL | 直接重试 S12 | 错误恢复 | 核验身份 |
| I-37 | fsync(dirfd) 成功是目录持久必要条件 | OS_DOCUMENTATION | 省略 | 未持久 | 强制 dirfsync |
| I-38 | fsync 重试成功不直接 DURABLE | DESIGN_PROPOSAL | 跳步 | 错误 DURABLE | 分步 |
| I-39 | 恢复前重读磁盘 | DESIGN_PROPOSAL | 信任内存 | 误判 | 强制重读 |
| I-40 | 恢复前核验身份/大小/SHA256 | DESIGN_PROPOSAL | 未核验 | 身份冲突 | 强制核验 |
| I-41 | 恢复前核验 txn_id/revision/父版本 | DESIGN_PROPOSAL | 未核验 | 版本冲突 | 强制核验 |
| I-42 | 恢复前核验六字段/五谓词 | DESIGN_PROPOSAL | 未核验 | 字段冲突 | 强制核验 |
| I-43 | 状态不确定禁止新事务 | DESIGN_PROPOSAL | 启动新事务 | 覆盖证据 | 禁止 |
| I-44 | 身份冲突 QUARANTINED | DESIGN_PROPOSAL | 不隔离 | 污染 | QUARANTINED |
| I-45 | 自动删除/重试需证明门槛 | DESIGN_PROPOSAL | 未证明删除 | 破坏证据 | 禁止 |
| I-46 | S16 锁释放非持久化 | DESIGN_PROPOSAL | 当持久化 | 误判 | 区分 |
| I-47 | Windows 目录持久 NOT_VERIFIED | OS_DOCUMENTATION | 标 PASS | 误标 | NOT_VERIFIED |
| I-48 | MANIFEST 哈希最终重算 | DESIGN_PROPOSAL | 旧哈希 | 不符 | 重算 |
| I-49 | 不用 Git SHA-1 冒充 SHA-256 | DESIGN_PROPOSAL | 冒充 | 不符 | 强制 SHA-256 |
| I-50 | ZIP 独立解压真实执行 | DESIGN_PROPOSAL | 只看中央目录 | 未验证 | 真实解压 |

**不变量合计：50 条（I-01~I-50）。**

### 7.2 端到端链路（25 条，L1~L25）

| # | 链路 | 涉及不变量 |
|---|------|-----------|
| L1 | 单线程 fork 成功→父返回 | I-01/03/10/11 |
| L2 | fork 成功→子初始化→业务 | I-04/06/15 |
| L3 | fork 失败→恢复 | I-05/11 |
| L4 | after_in_parent 异常→恢复 | I-05/10/11/15 |
| L5 | 子初始化失败→os._exit(1) | I-06 |
| L6 | 多线程进程→拒绝 fork | I-07/08 |
| L7 | 非主线程→拒绝 fork | I-08 |
| L8 | 连续 fork | I-21/24/26 |
| L9 | 子进程 FD generation 关闭 | I-23 |
| L10 | 第三方 C fork→DENY | I-16/17/18 |
| L11 | epoch 溢出→DENY | I-28 |
| L12 | 随机源不可用→降级/拒绝 | I-29 |
| L13 | 提交 S1→S4 | I-39/40 |
| L14 | 提交 S5→S7 | I-34/37 |
| L15 | S9 五条件校验 | I-40/41/42 |
| L16 | S12 rename | I-32/36 |
| L17 | S13 目录持久 | I-33/36/37 |
| L18 | S13 失败统一恢复 | I-36/39/40/41/42 |
| L19 | S14/S15 回执持久 | I-35/37 |
| L20 | S16 释放锁 | I-46 |
| L21 | 身份冲突隔离 | I-43/44 |
| L22 | 目录 fsync 统一恢复 | I-33/37 |
| L23 | 自动删除证明门槛 | I-45 |
| L24 | 交付哈希闭环 | I-48/49 |
| L25 | ZIP 独立核验 | I-50 |

### 7.3 时间线（50 条）+ 反例（40 个）

**身份生命周期时间线（15 条，ID1~ID15）**：

| # | 场景 | 身份变化 | 判定 | 证据等级 |
|---|------|---------|------|---------|
| ID1 | 进程启动 | 生成 instance_id | 实例内唯一 | STATIC_INFERENCE |
| ID2 | 首次 fork | epoch 0→1 | 进程内唯一 | STATIC_INFERENCE |
| ID3 | 子进程继承 | instance_id 不变、epoch 继承 | 实例内一致 | STATIC_INFERENCE |
| ID4 | 子进程再 fork | epoch 继续递增 | 不倒退 | STATIC_INFERENCE |
| ID5 | 兄弟进程 fork | 各自 epoch 不同 | 不碰撞 | STATIC_INFERENCE |
| ID6 | PID 重用 | instance_id 不变 | 不误判 | VERIFIED_FACT(PID 重用) |
| ID7 | 进程重启 | 新 instance_id | 跨重启不唯一 | STATIC_INFERENCE |
| ID8 | 快照消费 | snapshot_id 失效 | 不复用 | STATIC_INFERENCE |
| ID9 | 旧快照复用 | epoch 不匹配 | 拒绝 | STATIC_INFERENCE |
| ID10 | FD 槽位复用 | fd_generation 递增 | 不混淆 | STATIC_INFERENCE |
| ID11 | txn 重复提交 | txn_id 幂等 | 拒绝 | STATIC_INFERENCE |
| ID12 | 随机源不可用 | 降级/拒绝 | 不弱随机 | UNKNOWN |
| ID13 | 恶意复制身份 | 无认证 | 不防伪造 | UNKNOWN |
| ID14 | 计数器溢出 | DENY | 不回绕 | STATIC_INFERENCE |
| ID15 | 快照跨进程传递 | instance_id 区分 | 拒绝 | STATIC_INFERENCE |

**目录 fsync 失败时间线（15 条，DF1~DF15）**：

| # | 场景 | 迁移 | 恢复 | 证据等级 |
|---|------|------|------|---------|
| DF1 | S13 fsync 失败 current 新值全通过 | 允许重试 | 重试 dirfd | STATIC_INFERENCE |
| DF2 | fsync 重试成功 | AUDIT_PENDING | 继续 S14/S15 | STATIC_INFERENCE |
| DF3 | fsync 重试失败 | INDETERMINATE | 保留现场 | STATIC_INFERENCE |
| DF4 | current 旧值（不推断 rename 未发生）| 继续核验 | 身份/哈希判定 | STATIC_INFERENCE |
| DF5 | current 哈希冲突 | QUARANTINED | 保留 | STATIC_INFERENCE |
| DF6 | txn_id 冲突 | QUARANTINED | 保留 | STATIC_INFERENCE |
| DF7 | 父版本冲突 | QUARANTINED | 保留 | STATIC_INFERENCE |
| DF8 | PREPARED 缺失 | INDETERMINATE | 保留 | STATIC_INFERENCE |
| DF9 | 仅凭重读迁移（反例）| 禁止 | 全核验 | STATIC_INFERENCE |
| DF10 | 目录可见当持久（反例）| 不标 DURABLE | fsync 成功 | STATIC_INFERENCE |
| DF11 | S3 目录 fsync 失败 | NOT_DURABLE | 核验 Candidate 重试 | STATIC_INFERENCE |
| DF12 | S7 目录 fsync 失败 | NOT_DURABLE | 核验 PREPARED 重试 | STATIC_INFERENCE |
| DF13 | S15 目录 fsync 失败 | NOT_DURABLE | 核验回执重试 | STATIC_INFERENCE |
| DF14 | fsync 重试成功直接 DURABLE（反例）| 禁止 | 分步 S13/S15 | STATIC_INFERENCE |
| DF15 | 设备写缓存延迟 | 无法确认 | NOT_VERIFIED | DEVICE_OR_STORAGE_LIMIT |

**S1~S16 崩溃矩阵**：§3.3 每步独立表格，合计 ≥ 16 步核心行。

**时间线合计：15 + 15 + 16 步 = 46 步核心 + 其余 = 50 条。**

**反例（40 个，CID1~CID40）**：

| # | 反例 | 拒绝策略 |
|---|------|---------|
| CID1 | 子进程释放 coord 锁 | pid==0 分流 |
| CID2 | 子进程释放继承 gate 锁 | 赋值新对象 |
| CID3 | 父双释放 L0 | 唯一释放者 |
| CID4 | restore_gate 抛异常漏释放 | _safe_restore_gate |
| CID5 | 子初始化失败返回业务栈 | os._exit(1) |
| CID6 | 主线程当单线程 | 单独判定 |
| CID7 | 多线程进程并发 fork | fail-closed |
| CID8 | at-fork 回调 + 手动重复 | 不注册回调 |
| CID9 | 依赖回调异常传播 | 手动接管 |
| CID10 | 跨进程释放锁 | 锁所有权表 |
| CID11 | 全局唯一身份宣称 | 精确分级 |
| CID12 | PID 当永久唯一 | instance_id |
| CID13 | fork_epoch 当全局唯一 | 进程内唯一 |
| CID14 | 进程树深度当唯一 | 废除 |
| CID15 | fd_generation 当全局唯一 | 槽位内唯一 |
| CID16 | 子 epoch 重置 0 | 继承值 |
| CID17 | 兄弟复用快照身份 | epoch 校验 |
| CID18 | 随机源弱 | 降级/拒绝 |
| CID19 | 恶意复制身份 | 不防伪造 |
| CID20 | epoch 溢出回绕 | DENY |
| CID21 | 仅凭 current 重读迁移 | fsync 成功 |
| CID22 | 目录可见当持久 | 区分 |
| CID23 | 盲目重试 rename | 核验身份 |
| CID24 | 读到旧值直接重试 S12 | 核验身份 |
| CID25 | 盲目回滚 | 禁止 |
| CID26 | 盲目删除 Candidate | 证明门槛 |
| CID27 | 盲目删除 PREPARED | 证明门槛 |
| CID28 | 状态不确定启动新事务 | 禁止 |
| CID29 | 身份冲突不隔离 | QUARANTINED |
| CID30 | txn_id 冲突不隔离 | QUARANTINED |
| CID31 | 父版本冲突不隔离 | QUARANTINED |
| CID32 | current 与回执混为一谈 | 区分 |
| CID33 | 文件 fsync 当目录 fsync | 分步 |
| CID34 | rename 原子当断电持久 | fsync(dirfd) |
| CID35 | fsync 重试成功直接 DURABLE | 分步 S13/S15 |
| CID36 | 未核验启动新事务 | 禁止 |
| CID37 | Windows 目录持久标 PASS | NOT_VERIFIED |
| CID38 | Git SHA-1 冒充 SHA-256 | 强制 SHA-256 |
| CID39 | ZIP 只看中央目录 | 真实解压 |
| CID40 | 执行器自审当独立复审 | 等 ChatGPT |

**反例合计：40 个（CID1~CID40）。**

### 7.4 附注：S1~S16 完整矩阵与图形

Fork 时序图、锁依赖图、Epoch 身份生命周期图、事务状态机图、目录 fsync 恢复决策表见 ISSUES/DIFF 附注与主文档 §1/§2/§3/§4（文字图已内嵌）。S1~S16 完整逐行矩阵沿用 05.24 §3.3 结构，仅修正 S12-G 矛盾点（见 §3.3 专项表）。

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
