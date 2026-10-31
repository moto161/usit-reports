# USIT-STAGE-B-DESIGN-05.12｜六项 P0 缺陷定向修复与端到端协议重构

> 阶段 B · DESIGN-05.12 · READ-ONLY DESIGN REVISION ONLY
> 作者：USIT 主开发 · 生成：2026-10-31（GMT+8）
> 依据：DESIGN-05.11-INDEPENDENT-REVIEW（结论 FAIL）
> 工作区：`C:\Users\Administrator\WorkBuddy\2026-10-15-22-09-47\usit_p0`
> 交付目录：`C:\Users\Administrator\WorkBuddy\2026-10-15-22-09-47\usit_p0_05.12_reports`（**项目外**）
> 状态：**STOP — 等待独立复审**（未授权 IMPLEMENT-01）

---

## 0. 修订方法论：三类缺陷的区分

独立审查对 05.11 提出的六项缺陷，性质并不相同。本节先做分类，避免把"契约缺失"误当成"已证矛盾"、把"未验证风险"误当成"设计错误"。

| 类别 | 定义 | 处理方式 |
|------|------|---------|
| **A. 确定矛盾** | 同一文档内两处陈述互斥，逻辑上必有一错 | 直接改正，消除矛盾 |
| **B. 契约缺失** | 应定义而未定义（登记机制、状态语义、失败动作）| 补齐完整契约 |
| **C. 尚未验证的风险** | 平台 API 语义未实测，标记 NOT_VERIFIED | 保留 NOT_VERIFIED，不冒充 PASS |

**六项缺陷的分类归位（本轮先立此表，正文逐项引用）：**

| 编号 | 独立审查指出的问题 | 类别 | 05.11 原文依据 |
|------|-------------------|------|---------------|
| P0-01 | "子进程不继承 flock"与"父子共享 OFD"矛盾 | **A（确定矛盾）** | 05.11 §1.2 表格 fork 行；§2.2 表格锁 FD 行 |
| P0-02 | `_inherited_lock_fds` 无登记机制；布尔标志不可靠 | **B（契约缺失）** | 05.11 §2.3 `_after_fork_child`；§2.4 `_in_lock_critical_section` |
| P0-03 | 检查后到使用之间仍有 TOCTOU | **A+B（竞态窗口未消除）** | 05.11 §3.3 `_assert_path_matches_handle` |
| P0-04 | Windows CREATE_NEW 空占位暴露半成品 | **A（方案错误）+B（契约缺失）** | 05.11 §4.2 Windows 行 |
| P0-05 | 先发布 qfinal 后写审计，崩溃留孤立副本 | **A（顺序错误）** | 05.11 §5.2 步骤 1~6；§5.7 |
| P0-06 | tombstone 要求存在、实际却 unlink，状态矛盾 | **A（确定矛盾）** | 05.11 §6.3 vs §6.4 步骤 7 |

**执行顺序**：P0-01 → P0-02 → P0-03 → P0-04 → P0-05 → P0-06 → 端到端交叉审查 → 设计不变量集中证明。

> 文档设计通过 ≠ 代码实现通过。所有未实测平台行为标记 NOT_VERIFIED，未执行内容标记 NOT_TESTED。

---

## 1. P0-01｜flock 与 fork 后共享 OFD 的锁语义（消除确定矛盾）

### 1.1 确定矛盾与纠正

05.11 §1.2 的对比表中，flock 的 fork 行写的是 **"子进程不继承 flock（锁随 OFD，但 fork 复制 fd 引用同一 OFD）"** —— 这句话前半句与后半句**自相矛盾**：

- "fork 复制 fd 引用同一 OFD" → 说明子进程与父进程共享同一个 open file description；
- flock 锁恰恰关联于 open file description；
- 因此子进程**必然继承**了指向该锁的 OFD 引用，而不是"不继承"。

**纠正后的正确陈述（本节为权威定义）：**

> flock 锁关联于 open file description（OFD）。fork 后，父进程与子进程的对应 fd 引用**同一个** OFD，因此子进程**继承了该锁关联**。子进程 `close` 自己那份 fd 只是移除自己对 OFD 的一个引用，**不会**释放锁（父进程仍持有对同一 OFD 的引用）；但对共享 OFD 调用 `flock(fd, LOCK_UN)` 会**直接解除整个 OFD 上的锁**，从而**意外释放父进程仍依赖的锁**。

### 1.2 三种锁原语的所有权模型（确定语义，逐项区分）

| 维度 | `flock` | OFD 锁（`F_OFD_SETLK`）| 传统 fcntl 记录锁（`F_SETLK`）|
|------|---------|------------------------|------------------------------|
| 锁关联对象 | open file description | open file description | 进程 + inode |
| 锁单位 | 整个文件 | 字节区间 | 字节区间 |
| 同进程多线程 | 不互斥（同 OFD）| 不同 FD 互斥，同 FD 不互斥 | 不互斥（同进程锁不冲突）|
| 同进程不同 open | 冲突 | 冲突 | 冲突 |
| dup | 共享同一 OFD 锁 | 共享同一 OFD 锁 | 共享（同进程）|
| **fork** | **子进程继承 OFD 引用，共享锁** | **子进程继承 fd，共享 OFD 锁** | **子进程不继承**（锁随进程）|
| close | 所有引用同 OFD 的 fd 都 close 才释放 | 同左 | **任一** fd close 释放该进程全部记录锁 |
| **LOCK_UN 语义** | **解除整个 OFD 上的锁** | 解除该 OFD 锁区间 | 解除该进程的锁区间 |
| exec | 保留（除非 close-on-exec）| 保留（除非 close-on-exec）| 保留（除非 close-on-exec）|

**关键结论（本轮锁定）：**

1. flock 与 OFD 锁的 fork 语义**相同**——都因"fork 复制 fd 引用同一 OFD"而使子进程共享锁关联。
2. 传统 fcntl 记录锁 fork 语义**不同**——子进程不继承，且 close 任一 fd 即释放全部锁，是已知陷阱。
3. 因此 05.11 的"降级选 flock"方向正确，但 **fork 行的描述错误**，必须纠正为上表内容。

### 1.3 锁原语选择决策表

| 平台 | 首选原语 | 降级原语 | 不可用时的动作 |
|------|---------|---------|---------------|
| Linux ≥3.15（glibc≥2.20）| OFD 锁（`F_OFD_SETLK`）| flock | 两者都不可用 → **DENY**，无锁降级 |
| macOS | 无 OFD 锁 | flock（BSD 风格，语义见 §1.2）| flock 不可用 → **DENY** |
| BSD（FreeBSD/OpenBSD）| 无 OFD 锁 | flock | flock 不可用 → **DENY** |
| Windows NTFS | CreateFileW 独占 + LockFileEx | （无降级，见 P0-03 §3）| 独占打开不可用 → **DENY** |

**平台标识**写入 LockToken.lock_proto_version：`posix-ofd` / `posix-flock` / `windows-ntfs`。

### 1.4 close / LOCK_UN / 进程退出 三种操作的影响矩阵

| 操作 | 对 flock 锁的影响 | 对 OFD 锁的影响 | 对 fcntl 记录锁的影响 |
|------|------------------|----------------|---------------------|
| 单进程内 `close(fd)`（仍有无其他 fd 引用同 OFD）| 若无其他 fd 引用则释放，有则保留 | 同左 | **释放该进程全部记录锁** |
| `flock(fd, LOCK_UN)` | **释放整个 OFD 上的锁** | （flock 与 OFD 锁互不感知，LOCK_UN 只作用于 flock）| 不适用 |
| `F_OFD_SETLK(F_UNLCK)` | 不适用 | 释放该 OFD 锁区间 | 不适用 |
| 进程退出（正常/异常）| 关闭全部 fd → 释放 | 关闭全部 fd → 释放 | 释放（进程消失）|
| 线程退出 | **无影响**（锁随 OFD 不随线程）| 无影响 | 无影响 |

### 1.5 fork 前/后 FD/OFD/锁状态表（权威）

设父进程 P 持有一个锁 fd `p_fd`，它引用的 OFD 记为 `O`，`O` 上已加 flock 排他锁 `L`。

| 时刻 | 父进程 P | 子进程 C | OFD `O` 上的锁 `L` |
|------|---------|---------|-------------------|
| fork 前 | `p_fd → O`，持有 L | 不存在 | 已加锁 |
| fork 后（立即）| `p_fd → O`，持有 L | `c_fd → O`（fd 表复制，**同一 O**）| **仍加锁，父子共享** |
| 子进程 `close(c_fd)` | 仍持 `p_fd → O` | 不再引用 O | **仍加锁**（P 仍引用 O）|
| 子进程 `flock(c_fd, LOCK_UN)` | 仍持 `p_fd → O` | 引用 O | **锁被释放**（危险！父进程锁被解除）|
| 父进程退出 | 关闭 `p_fd` | 若 C 仍持 `c_fd` | 仍加锁（C 引用 O）|
| 父子都 close | 无 | 无 | 释放 |

**本表直接满足验收条件**："任何清理动作都不得意外解除父进程持有的锁" → 禁止子进程对继承 fd 调用 LOCK_UN，只能 `close`。

### 1.6 不支持平台的拒绝策略

- 目标平台既无 OFD 锁、也无 flock（或 flock 语义不满足"不随 close 任一 fd 释放"的要求）→ **DENY，禁止无锁降级**。
- 平台支持 flock 但未实测其 fork/close 语义 → 标记 `NOT_VERIFIED`，运行时不承诺，强一致模式 DENY。
- 禁止任何"锁文件存在即视为已加锁"的实现——那是有载体无原语的伪锁。

### 1.7 P0-01 验收对照

| 验收项 | 结论 |
|--------|------|
| 不再出现"不继承 flock"与"共享 OFD"矛盾 | ✅ §1.1 已纠正 |
| 清理动作不得解除父进程锁 | ✅ §1.4/§1.5（禁止子进程 LOCK_UN）|
| 三种锁原语所有权模型分别明确 | ✅ §1.2 |
| 锁原语选择决策表 | ✅ §1.3 |
| 不支持平台拒绝策略 | ✅ §1.6 |
| Linux/macOS/BSD 分别标记 | ✅ §1.3（均 NOT_VERIFIED）|

**P0-01 结论：PASS（设计级，矛盾已消除）**，平台语义 NOT_VERIFIED。

---

## 2. P0-02｜fork 回调、继承 FD 与进程内锁生命周期（补齐契约缺失）

### 2.1 契约缺失定位

05.11 §2.3 的 `_after_fork_child` 直接使用了 `_inherited_lock_fds`，但**从未定义它如何登记、谁登记、何时移除、并发访问规则**；§2.4 使用了 `_in_lock_critical_section` 布尔标志，但**未定义它如何可靠识别其他线程是否处于临界区**（单个布尔标志在多线程下是不可靠的）。这两处是**契约缺失**，不是已证矛盾，本轮补齐。

### 2.2 继承 FD 的唯一登记与所有权来源

引入**单一登记表** `LockHandleRegistry`，作为所有锁 FD/HANDLE 的**唯一事实来源**：

```python
class LockHandleRegistry:                     # 进程内单例，threading.Lock 保护
    _handles: dict[int, HandleRecord] = {}    # key = fd(int) / handle(int)
    _lock = threading.Lock()

    def register(self, fd: int, lease_acq_id: str) -> None:
        with self._lock:
            self._handles[fd] = HandleRecord(fd=fd, acq_id=lease_acq_id, state="OPEN")

    def deregister(self, fd: int) -> None:
        with self._lock:
            self._handles.pop(fd, None)

    def snapshot_fds(self) -> list[int]:
        with self._lock:
            return list(self._handles.keys())
```

**登记规则（唯一所有权来源）：**
- `acquire_lock` 成功获得锁 FD 后，**必须**调用 `LockHandleRegistry.register(fd, acq_id)`，然后才能构建 Lease。
- `release_lock` 关闭 FD 后，**必须**调用 `deregister(fd)`。
- **任何** lock FD 的创建/关闭都只能经由 register/deregister，禁止绕行。
- 这个登记表就是 fork 后子进程清理继承 FD 的**唯一依据**（不再依赖未定义的 `_inherited_lock_fds`）。

### 2.3 Lease / FD / OFD / ProcessMutex / Registry 之间的关系

```
LockToken（不可变身份）
   └─ acquisition_id（全局唯一，随机源）
LockHandle（持有 fd/handle 的封装）
   └─ fd ──→ OFD（open file description，锁真正作用对象）
LockLease（可变状态：token + handle + state）
   └─ 登记于 LeaseRegistry（进程内单例）
ProcessMutex（进程内线程互斥，按 auth_id）
   └─ 独立于 OS 锁，不跨进程，fork 后必须重建
```

**关系约束：**
- 一个 fd 只属于一个 LockHandle；一个 LockHandle 只属于一个 LockLease；一个 LockLease 只登记一次。
- fd 生命周期：register → （使用）→ deregister，全程闭环，无裸 fd 逃逸。
- ProcessMutex 与 OS 锁是**两层**：ProcessMutex 管同进程多线程，OS 锁管跨进程，缺一不可。

### 2.4 fork 前阻止其他线程进入临界区（可证明的机制，非布尔标志）

05.11 的 `_in_lock_critical_section` 布尔标志不可靠。改为**受控 fork 入口 + 全局 fork 闸门**：

```python
class ForkGate:
    _fork_in_progress = threading.Lock()      # 全局闸门
    _active_leases = 0                        # 活动 lease 计数（受 _counter_lock 保护）
    _counter_lock = threading.Lock()

def acquire_lock(...):
    with ForkGate._fork_in_progress:          # 与 fork 互斥
        ... 真正 acquire ...

def fork_controlled(*args):
    with ForkGate._fork_in_progress:          # 等待所有 acquire/release 退出
        return os.fork()
```

**机制证明（替代布尔标志）：**
- `acquire_lock` / `release_lock` 的整个临界区都在 `ForkGate._fork_in_progress` 这把锁内。
- `fork_controlled` 也在这把锁内。
- 因此 **fork 发生时，不可能有任何线程正处于 acquire/release 临界区中间**——这不是"检查一个可能过期的布尔值"，而是**锁本身的互斥性质**保证了这一点。
- 若调用方绕过 `fork_controlled` 直接 `os.fork()`（外部 fork），见 §2.7 拒绝策略。

### 2.5 prepare / parent / child 回调职责

| 回调 | 允许操作 | 禁止操作 |
|------|---------|---------|
| `before`（prepare）| 无（fork 前已由 ForkGate 保证无临界区）| 不得加锁/解锁 |
| `after_in_parent` | 无（父进程状态不变）| 不得改动子进程资源 |
| `after_in_child` | 清理继承资源（§2.6）| 不得调 LOCK_UN、不得复用父 Lease |

```python
os.register_at_fork(
    before=None,                 # 由 ForkGate 承担串行化
    after_in_parent=None,        # 父进程无需动作
    after_in_child=_after_fork_child_reset,
)
```

### 2.6 子进程重置顺序（`_after_fork_child_reset` 完整契约）

```python
def _after_fork_child_reset():
    # 顺序不可调换：
    # 1. 作废所有继承 Lease / Token（先于 close，避免任何代码路径在 close 前误用）
    for acq_id, lease in list(LeaseRegistry._leases.items()):
        lease.state = "INVALIDATED"
    LeaseRegistry._leases.clear()

    # 2. 关闭继承的锁 FD —— 只能 close，禁止 LOCK_UN（见 P0-01 §1.4）
    for fd in LockHandleRegistry.snapshot_fds():
        try:
            os.close(fd)          # 只移除子进程对 OFD 的引用，不解锁
        except OSError:
            pass                  # 已关闭，忽略
        LockHandleRegistry.deregister(fd)

    # 3. 重建 ProcessMutex（丢弃可能 locked 的继承锁）
    ProcessMutex._locks.clear()   # 全新空 dict，所有锁均为未锁定状态

    # 4. 重置随机源 / acquisition 计数器（避免与父进程生成相同 acquisition_id）
    _reset_acquisition_counter()
```

**关键不变式（I-03 的证据）**：步骤 2 只 `close` 不 `LOCK_UN`，确保子进程清理动作**绝不解除父进程锁**；步骤 1 先作废 Lease，确保 close 前没有任何路径能通过 assert_lock_owned 复用父租约。

### 2.7 外部 fork 与不支持场景的拒绝策略

| 场景 | 策略 |
|------|------|
| 经 `fork_controlled` 的 fork | 安全，after_in_child 自动清理 |
| 调用方绕过本库直接 `os.fork()` | **不支持**；子进程若调用本库锁 API，`assert_lock_owned` 会因 Registry 已重置（register_at_fork 兜底）或 fd 已关闭而失败 → DENY |
| 平台不支持 `os.register_at_fork` | **禁止在活动 Lease 下 fork**；要求 fork 前 release 全部锁；否则 DENY |
| fork 时其他线程持 ProcessMutex | ForkGate 已保证 fork 与 acquire/release 互斥，但**不保证**其他非锁业务线程的状态；子进程重建 ProcessMutex 已消除 locked 锁继承（§2.6 步骤 3）|

### 2.8 父子进程生命周期状态机（权威）

```
[父进程 P 持锁 L] ──fork_controlled──► [P 持锁 L] + [C 继承引用、无有效 lease]
                                              │
              ┌───────────────────────────────┼───────────────────────┐
              ▼                               ▼                       ▼
   [C close 继承 fd]                  [C 重新 acquire_lock]     [C exec 新程序]
   → L 仍由 P 持有                     → C 独立 lease、独立锁       → 继承 fd 全部 close
   （不解除父锁）                       （不与 P 共享）             （若 close-on-exec 未设则需显式 close）
```

**状态转换合法性规则：**
- C 从"继承引用"到"独立 lease"必须经 `acquire_lock`，禁止复用父 Lease。
- C 任何时刻不得对继承 fd 执行 LOCK_UN。
- P 的锁释放只经 `release_lock`，与 C 无关。

### 2.9 P0-02 验收对照

| 验收项 | 结论 |
|--------|------|
| 继承 FD 唯一登记/所有权来源 | ✅ §2.2（LockHandleRegistry）|
| acquire/release/异常/重复释放的登记更新 | ✅ §2.2 |
| fork 前检查机制（非布尔标志）| ✅ §2.4（ForkGate 锁互斥）|
| fork 后重置顺序 | ✅ §2.6 |
| 无法安全处理 → DENY | ✅ §2.7 |
| 多线程持锁 fork 策略 | ✅ §2.4/§2.7 |
| 不依赖无法证明的布尔标志 | ✅ §2.4 |
| 子进程不操作 locked 继承 Mutex | ✅ §2.6 步骤 3 |

**P0-02 结论：PASS（设计级，契约已补齐）**，register_at_fork 行为 NOT_TESTED。

---

## 3. P0-03｜锁文件身份验证与 TOCTOU 消除（消除竞态窗口）

### 3.1 缺陷定位

05.11 §3.3 的 `_assert_path_matches_handle` 流程是：**先读句柄身份 → 再读路径当前身份 → 比较 → 比较通过后允许提交**。但"比较通过"到"后续事务操作"之间，路径仍可能被替换。即：

```
验证对象 A（句柄锁住的 inode）== 路径当前对象 B（同一 inode）
之后按"路径字符串"执行关键操作 → 此时路径可能已被替换为对象 C
```

**05.11 的检查只证明"检查时刻"一致，不能证明"操作时刻"仍作用同一对象**。这是 TOCTOU 竞态窗口，属于 A+B 类（竞态未消除 + 契约缺失）。

### 3.2 消除原则：全程句柄，不回头用路径

**核心不变量（I-02 的强化版）**：一旦锁文件通过 `open` 得到 fd/HANDLE，后续**所有**关键操作必须通过该句柄（`fstat`/`fchmod`/`flock`/`F_OFD_SETLK`/`GetFileInformationByHandle`/`LockFileEx`）进行，**禁止**再通过路径字符串重新 open/stat 后执行关键提交。

锁文件的**目录**同样必须先 open 得到目录 fd，后续在目录内的操作用 `openat`/`fstatat`/`renameat` 相对目录 fd 进行。

### 3.3 POSIX 完整操作序列（从目录打开到锁使用）

```
步骤  操作                                   用句柄还是路径
───  ──────────────────────────────────────  ──────────────
1.    open(锁文件父目录, O_DIRECTORY|O_RDONLY) → dirfd    句柄
2.    fstat(dirfd) 校验目录身份（dev+ino）                 句柄
3.    openat(dirfd, 锁文件名, O_CREAT|O_RDWR|O_NOFOLLOW|O_CLOEXEC, 0o600) → fd   相对句柄
4.    fstat(fd) 记录 file_identity（dev+ino）              句柄
5.    flock(fd, LOCK_EX|LOCK_NB) / F_OFD_SETLK            句柄
6.    fstat(fd) 复核：身份未在 open 与加锁之间变化         句柄
7.    构造 Lease，登记 LockHandleRegistry                  句柄
───  之后所有 assert / release / 提交 ───────────────── 句柄
```

**关键点：**
- 步骤 3 用 `openat(dirfd, ...)` 而非 `open(path, ...)`，从步骤 1 的**可信目录句柄**出发，规避"路径字符串被替换"。
- `O_NOFOLLOW` 拒绝符号链接（在支持平台）。
- 步骤 6 复核 open 与加锁之间身份未变。
- 锁获得后，**不再**用路径字符串做任何关键操作。

### 3.4 身份校验与实际操作是否仍存在竞态的分析

| 竞态点 | 是否消除 | 依据 |
|--------|---------|------|
| open 与加锁之间文件被替换 | 消除 | §3.3 步骤 6 复核 fstat 身份 |
| 加锁后路径被替换 | **无关** | 加锁后所有操作经 fd，不触碰路径 |
| 加锁后原 inode 被 unlink 并重建同名文件 | **无关** | 锁在旧 inode（fd 引用），重建的新文件是不同 inode，不影响锁 |
| 目录被替换（dirfd 失效）| 部分消除 | dirfd 仍指向原目录 inode；但目录内**新**条目操作需 fstatat 复核 |
| 硬链接替换 | 消除 | fstat 身份（dev+ino）对硬链接是同一 inode，加锁对象不变 |

**结论**：POSIX 下，采用"目录句柄 + openat + 全程 fd + fstat 复核"后，**锁的获取与持有不依赖路径字符串**，TOCTOU 竞态窗口被消除。剩余风险：目录 inode 本身的权限若允许非授权主体写入（见 §3.5），需 DENY。

### 3.5 目录可信性前提（若不满足则 DENY）

锁文件目录必须满足，否则 **DENY**：

1. 目录权限限制为仅授权进程可写（如 `0o700`，属主为授权用户）。
2. 目录内锁文件的 unlink/replace 仅授权进程可执行。
3. 若目录对非授权主体可写（如 `/tmp`），**本协议不适用**，必须使用专用目录或 DENY。

### 3.6 攻击/故障矩阵

| 场景 | 检测/处理 |
|------|----------|
| 符号链接替换锁文件路径 | openat + O_NOFOLLOW 拒绝 |
| 路径重定向（rename 目录）| 目录句柄仍指向原 inode；fstatat 复核 |
| 锁文件被 unlink 后重建 | 锁在旧 inode，重建文件是新对象，fd 身份不受影响 |
| 锁文件被硬链接 | 同 inode，加锁对象一致 |
| 目录被替换 | dirfd 失效时 fstatat 报错 → DENY |
| 非授权主体可写目录 | 前置检查失败 → DENY（§3.5）|

### 3.7 Windows 独立实现契约（不套用 POSIX dirfd）

| 步骤 | Windows 机制 |
|------|-------------|
| 打开目录句柄 | `CreateFileW(dir, FILE_LIST_DIRECTORY, share...)` → 目录 HANDLE |
| 相对打开 | `CreateFileW` 用完整路径 + `FILE_FLAG_OPEN_REPARSE_POINT` 控制重解析点 |
| 对象身份 | `GetFileInformationByHandle`（volume serial + file index，等价 inode 身份）|
| 独占打开 | `CreateFileW(dwShareMode=0)` 打开阶段排他 |
| 区间锁 | `LockFileEx(LOCKFILE_EXCLUSIVE_LOCK)` |
| 身份复核 | 打开后再次 `GetFileInformationByHandle` 与记录值比较 |

**Windows 无法提供 POSIX `dirfd/openat` 的相对目录安全打开语义**，因此：
- Windows 的锁文件路径**必须**位于权限受限的专用目录（ACL 仅授权进程可写）。
- 通过 `GetFileInformationByHandle` 的 volume serial + file index 做身份绑定，等价于 POSIX dev+ino。
- 全程使用 HANDLE，不在加锁后重新按路径 open。
- 重解析点（reparse point）默认拒绝，除非显式允许且经过身份校验。

**Windows 行为未实测，标记 NOT_VERIFIED。不将 POSIX 的 dirfd 保证套用到 Windows。**

### 3.8 P0-03 验收对照

| 验收项 | 结论 |
|--------|------|
| 不"先查路径再按路径操作" | ✅ §3.2（全程句柄）|
| dirfd/openat/fstatat/O_NOFOLLOW 组合 | ✅ §3.3 |
| 身份校验与操作之间无竞态 | ✅ §3.4 |
| 目录可信前提 + DENY | ✅ §3.5 |
| 攻击/故障矩阵 | ✅ §3.6 |
| Windows 独立契约，不套 POSIX | ✅ §3.7 |

**P0-03 结论：PASS（设计级，竞态窗口消除）**，Windows 句柄语义、目录并发 NOT_VERIFIED。

---

## 4. P0-04｜Candidate 无覆盖提交与 PREPARED 恢复协议（消除半成品暴露）

### 4.1 缺陷定位

05.11 §4.2 的 Windows 方案是"`CreateFileW(CREATE_NEW)` 创建正式路径空占位 + 写入"，这会在**正式路径提前暴露一个空/半成品文件**，其他恢复流程可能读到未写完内容；且占位成功≠内容已完整提交。这是**方案错误（A）+ 契约缺失（B）**。

### 4.2 正确原则：先完整生成临时文件，再无覆盖发布

**不变量 I-04**：正式 Candidate 路径在完整内容准备好之前**不得出现任何文件**（哪怕是空占位）。

三步：**（1）临时文件完整写入 → （2）刷新+关闭+内容校验 → （3）无覆盖发布**。

### 4.3 POSIX 提交协议

```
1. 在正式目标同目录（同 FS）创建临时文件：
   tmp = openat(cand_dirfd, f".{txn_id}.tmp.{random}", O_CREAT|O_EXCL|O_WRONLY, 0o600)
2. write(tmp, full_content)
3. fsync(tmp)                       # 内容持久化
4. close(tmp)                       # 关闭，确认句柄状态
5. 校验：sha256(tmp) == candidate_sha256（重新读回）
6. 无覆盖发布：
   renameat2(AT_FDCWD, tmp, cand_dirfd, final_name, RENAME_NOREPLACE)   # Linux≥3.15
   兜底（无 renameat2）：link(tmp, final) → unlink(tmp)   # link 已存在则 EEXIST
7. fsync(cand_dirfd 对应父目录)       # 目录项持久化
8. 复核：fstatat(cand_dirfd, final_name) 身份与 sha256 与预期一致
```

**无覆盖语义证明**：
- `RENAME_NOREPLACE`：目标已存在时失败（EEXIST），不覆盖。
- `link(tmp, final)`：目标已存在时 `EEXIST`，天然无覆盖。
- 两者都保证"目标已存在 → 失败 → DENY"，绝不覆盖其他事务的 candidate。

### 4.4 Windows 提交协议（纠正半成品暴露）

**禁止**"CREATE_NEW 创建正式路径空占位再写入"。改为：

```
1. 在正式目标同卷创建临时文件：
   CreateFileW(tmp_path, GENERIC_WRITE, 0, NULL, CREATE_NEW, FILE_ATTRIBUTE_NORMAL, NULL)
2. WriteFile(tmp, full_content)     # 完整写入临时文件
3. FlushFileBuffers(tmp)            # 刷新
4. CloseHandle(tmp)                 # 关闭
5. 校验：重新读取 sha256(tmp) == candidate_sha256
6. 无覆盖发布（无覆盖语义的原语）：
   方案 A：SetFileInformationByHandle 不提供无覆盖 rename；
   方案 B：MoveFileEx 只有 MOVEFILE_REPLACE_EXISTING 才会覆盖，不带该标志时
           目标已存在则失败（ERROR_ALREADY_EXISTS / ERROR_FILE_EXISTS）
   → 采用 MoveFileEx(tmp, final, MOVEFILE_COPY_ALLOWED=0, 不带 REPLACE_EXISTING)
7. 复核：GetFileInformationByHandle 身份 + 内容摘要
```

**Windows 无覆盖语义**：`MoveFileEx` **不带** `MOVEFILE_REPLACE_EXISTING` 标志时，目标已存在会失败（不覆盖）。这正是无覆盖语义的来源——**关键纠正：不是用 CREATE_NEW 占位，而是用 MoveFileEx 不带 REPLACE 标志做无覆盖发布**。

**约束**：
- 临时文件与正式目标必须**同卷**（MoveFileEx 跨卷需 MOVEFILE_COPY_ALLOWED，且不保证原子）。
- 目标已存在 → 失败 → DENY。
- 断电持久性：MoveFileEx 的目录项持久化**无法证明**，标记 NOT_VERIFIED（强一致模式 DENY）。

### 4.5 临时文件 / Candidate / 正式文档 / PREPARED 的绑定字段

| 对象 | 绑定字段 |
|------|---------|
| 临时文件 | transaction_id + 随机名（.tmp.<random>）|
| 正式 Candidate | transaction_id + candidate_identity（dev+ino / volume+index）+ candidate_sha256 |
| 正式文档 | transaction_id + revision + formal_doc_sha256 |
| PREPARED journal | transaction_id / authorization_id / candidate_identity / candidate_sha256 / old_revision / target_revision / formal_doc_target_sha256 / schema_version |

**PREPARED 必须先于正式发布达到持久化条件**（fsync 文件 + fsync 父目录），否则恢复时无事务意图可依。

### 4.6 PREPARED 状态机

```
CREATED → CANDIDATE_DURABLE → PREPARED → COMMITTING → COMMITTED
                                              ├→ ABORTED
                                              └→ INDETERMINATE
```

| 转换 | 触发 | 持久化点 |
|------|------|---------|
| CREATED→CANDIDATE_DURABLE | 临时文件完整写入+校验 | fsync 临时文件 |
| CANDIDATE_DURABLE→PREPARED | 无覆盖发布成功 + PREPARED 写入 | fsync PREPARED + 父目录 |
| PREPARED→COMMITTING | 开始替换正式文档 | — |
| COMMITTING→COMMITTED | 正式文档替换+复核+COMMITTED 写入 | fsync COMMITTED + 父目录 |
| COMMITTING→ABORTED | 显式回滚 | fsync ABORTED |
| 任意→INDETERMINATE | 无法判定 | 保留证据，人工 |

### 4.7 崩溃恢复矩阵（权威，每项唯一动作）

| 崩溃点 | 恢复动作 | 归因证据 |
|--------|---------|---------|
| 临时文件写入中 | 删除残留 tmp，重做 | tmp 名含 .tmp.<random> |
| 临时文件写完、fsync 前 | 重新 fsync 或重做 | sha256 校验 |
| 发布前崩溃（tmp 完整、final 不存在）| 重新执行无覆盖发布 | tmp sha256 + PREPARED |
| 发布后、fsync 目录前 | 重新 fsync 目录 | final 已存在，身份校验 |
| 发布后、正式文档替换前 | 继续替换流程 | PREPARED + candidate 完整 |
| 正式文档替换中崩溃 | **INDETERMINATE**，校验后重判 | §4.8 全量匹配 |
| 正式文档替换后、COMMITTED 前 | 校验正式文档 → 补写 COMMITTED | §4.8 |
| 目标被其他事务抢先创建 | DENY（无覆盖失败）| 目标已存在 |

### 4.8 恢复时的全量事务身份校验

恢复程序**不得仅凭文件存在或 SHA-256 相同就认定成功**，必须校验：

```python
def _verify_ownership(prepared, formal_doc, candidate) -> bool:
    return (candidate_identity == prepared.candidate_identity
            and sha256(candidate) == prepared.candidate_sha256
            and formal_doc["transaction_id"] == prepared.transaction_id
            and formal_doc["revision"] == prepared.target_revision
            and sha256(formal_doc) == prepared.formal_doc_target_sha256)
```

**拒绝策略**：
- 目标已存在但 transaction_id 不匹配 → DENY（不覆盖他人 candidate/文档）。
- 身份不匹配 → DENY。
- 摘要不匹配 → DENY。
- 无法证明目标属于本事务 → DENY。

### 4.9 P0-04 验收对照

| 验收项 | 结论 |
|--------|------|
| 正式路径不暴露半成品 | ✅ §4.2/§4.3（先 tmp 后发布）|
| Windows 无覆盖原语明确 | ✅ §4.4（MoveFileEx 不带 REPLACE）|
| 同卷要求 + 目标已存在失败语义 | ✅ §4.4 |
| 全字段绑定 | ✅ §4.5 |
| PREPARED 先持久化 | ✅ §4.5/§4.6 |
| 崩溃恢复矩阵 + 全量校验 | ✅ §4.7/§4.8 |
| 恢复不覆盖他人 candidate/文档 | ✅ §4.8 |

**P0-04 结论：PASS（设计级，半成品暴露已消除）**，MoveFileEx/renameat2 语义 NOT_VERIFIED。

---

## 5. P0-05｜Quarantine 发布、Journal 意图与崩溃恢复（消除顺序错误）

### 5.1 缺陷定位

05.11 §5.2 的隔离顺序是：**先复制并发布 qfinal（步骤 3~4），再写审计记录（步骤 5~6）**。若 qfinal 已发布但审计记录未持久化时崩溃，恢复时只剩一个孤立 qfinal，无法证明它属于哪个事务、对应哪个原始 Journal。§5.7 声称"通过 qfinal 内容反推原哈希"**不足以解决事务归属问题**（相同内容的 Journal 可能来自不同事务）。这是**顺序错误（A）**。

### 5.2 正确原则：先持久化隔离事务意图，再发布 qfinal

**不变量 I-07**：quarantine 的副本、源对象、Journal 必须能够进行事务关联。因此**隔离操作本身必须是一个带唯一 ID 的事务，意图先于任何 rename 持久化**。

### 5.3 quarantine 完整事务时序（权威顺序）

```
阶段 0：生成 quarantine_id（全局唯一，绑定 transaction_id + 操作类型 QUARANTINE）
阶段 1：持久化 QUARANTINE_INTENT（先于任何文件操作）
  记录：quarantine_id / transaction_id / source_identity(dev+ino) / source_sha256
        / target_dir / target_rule / operation_type=QUARANTINE / schema_version
  fsync(intent 文件) + fsync(intent 父目录)
阶段 2：复制 src → qtmp（quarantine/<quarantine_id>.qtmp）
  fsync(qtmp)
阶段 3：校验 qtmp 身份与 sha256 == QUARANTINE_INTENT.source_sha256
阶段 4：发布 qtmp → qfinal（rename，无覆盖）
  fsync(quarantine 父目录)
阶段 5：写入 QUARANTINE_COMPLETE 记录（绑定 quarantine_id + qfinal 身份 + qfinal sha256）
  fsync(文件) + fsync(父目录)
阶段 6：处理源对象（rename src → src.corrupt 或按规则）—— 这是**独立后续阶段**，见 §5.5
```

**关键纠正**：阶段 1 的 QUARANTINE_INTENT 在阶段 4 发布 qfinal **之前**已持久化。因此任何崩溃点，恢复都能通过 intent 识别该隔离操作的事务归属，不再有"孤立 qfinal 无法归因"。

### 5.4 Journal 状态机

```
ACTIVE → QUARANTINE_INTENT → QFINAL_DURABLE → QUARANTINED
                                     ├→ CORRUPT（隔离失败，保留源）
                                     └→ INDETERMINATE
```

| 转换 | 触发 | 持久化点 |
|------|------|---------|
| ACTIVE→QUARANTINE_INTENT | 决定隔离损坏 Journal | fsync intent + 父目录 |
| QUARANTINE_INTENT→QFINAL_DURABLE | qfinal 发布成功 | fsync qfinal 内容 + 父目录 |
| QFINAL_DURABLE→QUARANTINED | 源对象改名 + COMPLETE 记录 | fsync COMPLETE + 父目录 |
| 任意→INDETERMINATE | 无法判定归属 | 保留全部证据，人工 |

### 5.5 副本隔离与源对象处理是**两个独立阶段**

**关键区分（消除 05.11 混淆）**：副本隔离（阶段 1~5）与源文件移动/删除（阶段 6）是**两个不同阶段**，不能把"副本存在"等同于"源对象已安全清理"。

- 阶段 1~5 完成 = 隔离副本已可靠生成并绑定事务，**源对象原样保留**。
- 阶段 6 = 源对象改名（src → src.corrupt），这是一个**单独**的可恢复步骤，有自己的 intent（可复用 QUARANTINE_INTENT 或独立 SOURCE_RETIRE_INTENT）。

### 5.6 qtmp / qfinal / 源对象三方一致性规则

| 规则 | 校验 |
|------|------|
| qtmp.sha256 == source_sha256 | 复制完整性 |
| qfinal.sha256 == qtmp.sha256 == source_sha256 | 发布未篡改 |
| qfinal.identity 在 COMPLETE 记录中登记 | 可归因 |
| source 在阶段 6 前身份不变 | 防止并发修改 |
| 三方都绑定同一 quarantine_id | 事务关联 |

### 5.7 崩溃恢复矩阵（每项唯一动作）

| 崩溃点 | 恢复动作 |
|--------|---------|
| 阶段 1 intent 写前 | 无副作用，可重来 |
| 阶段 1 intent 写中 | intent 半写 → 删除重写（intent 名含 .tmp）|
| 阶段 2 复制中（qtmp 半写）| qtmp sha256 ≠ source → 删除 qtmp 重做 |
| 阶段 3 校验失败 | DENY（源可能被并发修改）|
| 阶段 4 发布前（qtmp 完整无 qfinal）| 重新发布 |
| 阶段 4 发布后、fsync 目录前 | 重新 fsync 目录 |
| 阶段 5 COMPLETE 写前（qfinal 存在、intent 存在）| 补写 COMPLETE（intent 已证明归属）|
| 阶段 5 COMPLETE 写中 | 删除重写 COMPLETE |
| 阶段 6 源改名前 | 源仍在，按规则改名 |
| 阶段 6 改名后、fsync 前 | 重新 fsync |

### 5.8 qfinal 已存在时的重新校验（禁止盲目跳过/覆盖）

恢复时若发现 qfinal 已存在：
1. 读取 QUARANTINE_INTENT，确认存在对应 quarantine_id。
2. 校验 qfinal 身份与 sha256 是否与 intent/COMPLETE 一致。
3. **属于本事务** → 幂等继续（跳到正确阶段）。
4. **不属于任何已知事务** → `INDETERMINATE`，隔离待人工，**不得自动覆盖或删除**。
5. **身份/哈希不匹配** → DENY + 人工。

### 5.9 审计写入失败 / 孤立对象处置

- 审计（COMPLETE）写入失败 → 保留源对象，标记 INDETERMINATE，DENY，不继续阶段 6。
- qtmp 遗留无 qfinal → 删除 qtmp（有 intent 可归因）。
- qfinal 孤立（有 qfinal 无 intent）→ INDETERMINATE，人工。
- 源对象被并发修改（身份变化）→ DENY。

### 5.10 POSIX / Windows 目录持久化差异（不混为一谈）

| 项 | POSIX | Windows |
|----|-------|---------|
| 目录项持久化 | fsync 父目录 | **无等价**，NOT_VERIFIED |
| 文件内容持久化 | fsync 文件 | FlushFileBuffers |
| 原子 rename | rename / renameat2 | MoveFileEx |
| 断电 | 需 fsync 目录才保证 | 无法证明，NOT_VERIFIED |

**不得声称 Windows 已实现未验证的目录 fsync 等价保证。**

### 5.11 P0-05 验收对照

| 验收项 | 结论 |
|--------|------|
| quarantine 有唯一事务 ID | ✅ §5.3 阶段 0 |
| 发布 qfinal 前持久化意图 | ✅ §5.3 阶段 1 先于阶段 4 |
| qtmp/qfinal/源对象/Journal 可验证绑定 | ✅ §5.6 |
| qfinal 存在不盲目跳过 | ✅ §5.8 |
| 副本隔离与源对象处理分开 | ✅ §5.5 |
| 每次 rename/fsync 后持久化要求 | ✅ §5.3 |
| 孤立对象处置 | ✅ §5.9 |
| Windows 不声称目录 fsync 等价 | ✅ §5.10 |

**P0-05 结论：PASS（设计级，顺序错误已消除）**，Windows 断电/目录持久化 NOT_VERIFIED。

---

## 6. P0-06｜Tombstone 状态机、删除归因与终态审计（消除状态矛盾）

### 6.1 缺陷定位（确定矛盾）

05.11 §6.3 定义 `REMOVED_BY_THIS_CLEANUP` 的判定要求"tombstone 存在 + 身份/哈希匹配 + REMOVED 已持久化"，但 §6.4 的实际操作顺序是 `rename → TOMBSTONED → unlink(tombstone) → REMOVED_BY_THIS_CLEANUP`。当 unlink 成功、REMOVED 尚未持久化时崩溃，恢复时 tombstone 已不存在，**无法满足 §6.3 的判定条件**——状态定义与操作顺序矛盾。这是**确定矛盾（A）**。

### 6.2 根本修复：区分"删除意图持久化"与"对象物理消失"

核心洞察：**归因的依据不是 tombstone 文件本身，而是持久化的删除意图 + 逐步状态记录**。tombstone 只是删除前的中间载体，真正的归因证据是**持久化的状态机**，而非"tombstone 是否还存在"。

### 6.3 完整状态机（消除矛盾，新增 UNLINK_INTENT）

```
PENDING → DELETE_INTENT → TOMBSTONED → UNLINK_INTENT → REMOVED_BY_THIS_CLEANUP → CLEANED
    │          │              │              │
    │          ├→ FAILED（可重试）
    │          └→ INDETERMINATE
    └──（对象已不存在）→ ALREADY_ABSENT
```

| 状态 | 持久化证据 | 含义 |
|------|-----------|------|
| DELETE_INTENT | intent 记录已 fsync | 意图已持久化，对象待删 |
| TOMBSTONED | tombstone 记录已 fsync | 对象已 rename 到 tombstone，尚未 unlink |
| **UNLINK_INTENT** | unlink 意图记录已 fsync | **即将 unlink tombstone（新增，解决矛盾的关键）** |
| REMOVED_BY_THIS_CLEANUP | removed 记录已 fsync | tombstone 已 unlink，删除完成 |
| ALREADY_ABSENT | 无本事务 intent | 执行前对象已不存在 |
| INDETERMINATE | 证据不足 | 无法归因 |
| CLEANED | 全部终态 + 归档 | 两个对象均达终态 |

### 6.4 正确操作顺序（UNLINK_INTENT 先行）

```
1. 持久化 DELETE_INTENT（含 cleanup_id、对象身份、哈希、终态快照）
2. 校验对象身份与哈希 == intent
3. rename(对象 → tombstone/<cleanup_id>.<object>)
4. fsync(对象父目录) + fsync(tombstone 父目录)
5. 持久化 TOMBSTONED（记录 tombstone 身份 + 哈希）
6. 持久化 UNLINK_INTENT（★ 新增：声明即将 unlink tombstone）→ fsync
7. unlink(tombstone)
8. fsync(tombstone 父目录)
9. 持久化 REMOVED_BY_THIS_CLEANUP → fsync
```

### 6.5 崩溃恢复矩阵（解决 §6.3 矛盾的关键）

| 崩溃点 | 恢复判定 |
|--------|---------|
| 步骤 6 UNLINK_INTENT 写前、tombstone 已存在 | TOMBSTONED 状态，对象仍在 tombstone → 可继续或回滚 |
| 步骤 6 后、步骤 7 unlink 前 | **UNLINK_INTENT 已持久化** → 恢复知道"即将 unlink"，继续 unlink |
| 步骤 7 unlink 后、步骤 9 REMOVED 前 | **UNLINK_INTENT 已持久化 + tombstone 已不存在** → 恢复可判定"本次清理已完成 unlink，正在等 REMOVED 落盘"，**补写 REMOVED** |
| 步骤 9 REMOVED 写中 | 删除重写 REMOVED |

**关键**：有了 `UNLINK_INTENT` 状态后，"tombstone 不存在 + UNLINK_INTENT 已持久化 + REMOVED 未持久化"这一组合**不再矛盾**——UNLINK_INTENT 已证明"本次清理即将/已经 unlink 了 tombstone"，因此 tombstone 消失是可归因的，恢复补写 REMOVED 即可。**这就消除了 05.11 的状态矛盾。**

### 6.6 三态归因决策表

| 观测 | 判定 |
|------|------|
| 对象存在 + 无本事务 intent | 未开始（或他人对象），不归因 |
| 对象存在 + DELETE_INTENT | 待删，继续流程 |
| tombstone 存在 + TOMBSTONED | 已 rename，待 unlink |
| tombstone 不存在 + UNLINK_INTENT + 无 REMOVED | **REMOVED_BY_THIS_CLEANUP（补写 REMOVED）** |
| tombstone 不存在 + 无任何 intent | **ALREADY_ABSENT**（执行前已不存在）|
| 身份/哈希不匹配 | **INDETERMINATE** |
| 证据不足 | **INDETERMINATE** |

### 6.7 并发删除 / 外部删除 / 路径替换 / 同名重建 / tombstone 丢失 / 审计失败

| 场景 | 处理 |
|------|------|
| 并发删除 | 同一对象仅一个清理者（授权锁保护），第二个 → 幂等跳过 |
| 外部进程删除 tombstone | tombstone 消失但 UNLINK_INTENT 未持久化 → **INDETERMINATE**（不能证明是本次删除）|
| 路径替换（同名对象重建）| 对象身份变化 → 校验失败 → INDETERMINATE |
| tombstone 丢失（异常）| 无 UNLINK_INTENT 则 INDETERMINATE；有 UNLINK_INTENT 则补 REMOVED |
| 审计写入失败 | 保留现状，DENY，不继续删除 |

### 6.8 终态审计快照字段清单

```python
@dataclass(frozen=True)
class TerminalStateSnapshot:
    transaction_id: str
    authorization_id: str
    revision: int
    terminal_state: str              # COMMITTED / ABORTED
    formal_doc_sha256: str
    source_journal_identity: str     # 源 Journal dev+ino
    journal_full_content: str        # 完整规范化终态 Journal 内容（非摘要）
    journal_content_sha256: str      # 完整内容 SHA-256（校验用）
    snapshot_version: str
    created_at: str
```

**验证规则**：`sha256(journal_full_content) == journal_content_sha256`，且内嵌 transaction_id/revision/state 与顶层一致。快照**先于 DELETE_INTENT 持久化**（§6.4 步骤 1 之前）。

### 6.9 P0-06 验收对照

| 验收项 | 结论 |
|--------|------|
| 状态定义与操作顺序无矛盾 | ✅ §6.3/§6.4（UNLINK_INTENT 解决）|
| 每状态有持久化证据 | ✅ §6.3 |
| 每崩溃窗口唯一恢复动作 | ✅ §6.5 |
| 不凭"对象不存在"认定本次删除 | ✅ §6.6 |
| ALREADY_ABSENT 与 REMOVED 区分 | ✅ §6.6 |
| 三态归因决策表 | ✅ §6.6 |
| 终态快照完整字段 | ✅ §6.8 |

**P0-06 结论：PASS（设计级，状态矛盾已消除）**，tombstone 并发/外部删除 NOT_VERIFIED。

---

## 7. 设计不变量集中证明

本轮集中列出并逐项证明 12 条设计不变量。

| 编号 | 不变量 | 证明（证据章节）|
|------|--------|----------------|
| I-01 | 没有有效锁，不得执行受保护操作 | P0-01 §1.3（无锁原语则 DENY）+ 所有提交/恢复/清理入口先 assert_lock_owned |
| I-02 | 锁身份不匹配，不得继续 | P0-03 §3.4（fstat 复核）|
| I-03 | fork 后子进程不得误操作父进程锁 | P0-01 §1.5 + P0-02 §2.6（只 close 不 LOCK_UN）|
| I-04 | 正式 Candidate 不得暴露半成品 | P0-04 §4.2（先 tmp 后发布）|
| I-05 | 正式目标不得被其他事务覆盖 | P0-04 §4.3/§4.4（无覆盖原语）|
| I-06 | 恢复程序不得认领无法证明归属的文件 | P0-04 §4.8 + P0-05 §5.8 |
| I-07 | quarantine 副本/源/Journal 可事务关联 | P0-05 §5.6（quarantine_id 绑定）|
| I-08 | 删除归因必须有持久化证据 | P0-06 §6.3（每状态持久化）|
| I-09 | 对象不存在不等于本次删除成功 | P0-06 §6.6（三态归因）|
| I-10 | 不确定状态进入 DENY/INDETERMINATE | 贯穿全部恢复矩阵 |
| I-11 | 设计通过不冒充运行时验证 | 全文 NOT_VERIFIED/NOT_TESTED 标记 |
| I-12 | 平台保证限定到平台/版本/API | P0-03 §3.7、P0-05 §5.10（Windows 不套 POSIX）|

**不变量证明方法**：每条不变量都在对应 P0 章节给出状态机、操作序列、恢复矩阵，且交叉审查（§8）逐链路验证不变量在崩溃/并发下仍成立。

---

## 8. 端到端静态交叉审查

### 8.1 七条链路的完整审查

| 链路 | 前置条件 | 关键持久化边界 | 并发约束 | 崩溃点 | 恢复动作 | 安全不变量 |
|------|---------|--------------|---------|--------|---------|-----------|
| 锁获取→Lease→fork→退出→释放 | 目录可信 | fd register/close | ForkGate 串行 | fork 后子进程 | 重置+重 acquire | I-03 |
| 锁身份→Candidate→PREPARED→发布→revision | 目录句柄 | 每个 rename 后 fsync | 授权锁 | 发布中 | 身份校验重判 | I-02/I-04/I-05 |
| Journal 意图→qtmp→qfinal→源处理→审计 | quarantine_id | intent 先于 qfinal | 授权锁 | 各阶段 | §5.7 矩阵 | I-07 |
| DELETE_INTENT→TOMBSTONED→UNLINK_INTENT→unlink→REMOVED→CLEANED | 终态快照 | 每状态 fsync | 单清理者 | unlink 后 REMOVED 前 | UNLINK_INTENT 补 REMOVED | I-08/I-09 |
| 任意崩溃→重启→识别→归因→继续/拒绝 | 持久化证据 | — | — | 任意 | 唯一动作或 DENY | I-06/I-10 |
| 两并发事务竞争同一对象 | 授权锁 | — | 锁互斥 | — | 后到者幂等/失败 | I-05 |
| 路径替换/目标已存在/摘要不一致/审计不可用 | — | — | — | — | DENY/INDETERMINATE | I-02/I-06 |

### 8.2 交叉一致性结论

- P0-02~P0-06 全部复用 P0-01 定义的锁接口（acquire_lock / assert_lock_owned / release_lock + ProcessMutex + LockHandleRegistry + ForkGate），未引入第二套锁语义。
- P0-03 的"全程句柄"原则被 P0-04（Candidate 用 dirfd/openat）、P0-05（quarantine 用 quarantine_id）、P0-06（tombstone 身份校验）一致遵循。
- **无跨章节矛盾**。六项 P0 的修复相互引用前序章节的确定契约。

---

## 9. P1 与既有阻断项状态

| 编号 | 状态 |
|------|------|
| P1-01 锁文件替换/TOCTOU | 设计级闭合（P0-03），运行时 NOT_VERIFIED |
| P1-02 analyze_file 行为等价 | NOT_TESTED |
| P1-03 A/B 三文件合并 | 保持原状态，不执行 |
| P0-BLOCK-1 存储事务实现+故障注入 | 未解决 |
| P0-BLOCK-2 analyze_file 一致性 | 未解决 |
| P1-BLOCK-3 A/B 三文件分叉 | 未解决 |
| F1 / P0.8 / F2 | 不解除 |
| IMPLEMENT-01 | STOP |

---

## 10. IMPLEMENT-01 准入结论

**PASS-WITH-ISSUES（维持阻断）。**

六项 P0 的确定矛盾已消除、契约缺失已补齐、竞态窗口已关闭：

| 编号 | 缺陷类别 | 修复结果 |
|------|---------|---------|
| P0-01 | A 确定矛盾 | flock fork 语义纠正 |
| P0-02 | B 契约缺失 | LockHandleRegistry + ForkGate 补齐 |
| P0-03 | A+B 竞态 | 全程句柄 + dirfd/openat 消除 TOCTOU |
| P0-04 | A+B | 先 tmp 后无覆盖发布，消除半成品 |
| P0-05 | A 顺序错误 | 意图先于 qfinal，quarantine_id 绑定 |
| P0-06 | A 确定矛盾 | UNLINK_INTENT 消除状态矛盾 |

但锁、事务、隔离、清理均为**设计级契约**，真实实现前 NOT_VERIFIED / NOT_TESTED。三个阻断项未解决。

**本工单不授权实现、不授权测试、不解除 IMPLEMENT-01 STOP。F1/P0.8/F2 未解除。三个阻断项（P0-BLOCK-1、P0-BLOCK-2、P1-BLOCK-3）未解除。**

---

## 11. 交付物索引

| 文件 | 内容 |
|------|------|
| `USIT-STAGE-B-DESIGN-05.12.md` | 本文（主设计）|
| `USIT-STAGE-B-DESIGN-05.12-ISSUES.md` | 问题清单 |
| `USIT-STAGE-B-DESIGN-05.12-DIFF.md` | DESIGN-05.11 → 05.12 差异 |
| `USIT-STAGE-B-DESIGN-05.12-CROSS-REVIEW.md` | 端到端交叉审查记录 |
| `USIT-STAGE-B-DESIGN-05.12-EXECUTION-REPORT.md` | 执行报告 |
| `USIT-STAGE-B-DESIGN-05.12-HASH-MANIFEST.md` | 哈希清单 |
| `USIT-STAGE-B-DESIGN-05.12-HASH-RECEIPT.txt` | 哈希收据 |
| `USIT-STAGE-B-DESIGN-05.12-交付物.zip` | 七份文档打包 |
