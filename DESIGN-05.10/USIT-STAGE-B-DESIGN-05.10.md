# USIT-STAGE-B-DESIGN-05.10｜锁协议基础修正与四项 P0 缺陷闭环

> 阶段 B · DESIGN-05.10 · READ-ONLY ANALYSIS + DESIGN REVISION
> 作者：USIT 主开发 · 生成：2026-10-31（GMT+8）
> 前置：DESIGN-05.9 独立审查结论 **FAIL**
> 工作区：`C:\Users\Administrator\WorkBuddy\2026-10-15-22-09-47\usit_p0`
> 交付目录：`C:\Users\Administrator\WorkBuddy\2026-10-15-22-09-47\usit_p0_05.10_reports`（**项目外**）
> 状态：**STOP — 等待独立复审**（未授权 IMPLEMENT-01）

---

## 0. 范围与执行顺序

本轮**不推倒重做 05.9，不扩大范围**，仅修复四项 P0。执行顺序严格为 **P0-01 → P0-02 → P0-03 → P0-04 → 交叉审查**，后序章节引用前序已确定的锁接口契约，不得各自设计互相矛盾的锁/存储协议。

05.9 的核心错误与 05.10 修正：

| 05.9 错误 | 05.10 修正 |
|-----------|-----------|
| 用 `F_GETLK` 返回 PID 证明"当前进程持有自身记录锁" | 该假设不成立；改用 **OFD 锁 + 进程内线程互斥** 组合，`assert_lock_owned` 不依赖"查询自身锁归属" |
| 传统 fcntl 进程关联记录锁语义混乱 | 明确区分进程关联锁 vs 开放文件描述关联（OFD）锁的生命周期 |
| 无进程内线程互斥层 | 新增 `ProcessMutex`，OS 文件锁不替代线程互斥 |

> 文档设计通过 ≠ 代码实现通过。不得把伪代码推演写成实测结果。未验证平台行为标记 NOT_VERIFIED，未执行测试标记 NOT_TESTED。

---

## 1. P0-01｜锁协议基础（先修复）

### 1.1 POSIX 锁机制重新评估

**两类锁的明确区分：**

| 维度 | 进程关联记录锁（传统 fcntl F_SETLK）| 开放文件描述关联锁（OFD，`F_OFD_SETLK`）|
|------|--------------------------------------|------------------------------------------|
| 锁归属 | 进程（跨线程共享）| 单个打开文件描述（open file description）|
| 同一进程多线程 | 不互斥（同进程锁不冲突）| **不同 FD 互斥**（每个 OFD 独立）|
| dup | 共享锁（dup 同一进程不冲突）| dup 共享同一 OFD 锁 |
| fork | 子进程**不继承**（锁随进程）| 子进程继承 fd，但 OFD 锁随打开文件描述，**子进程共享同一 OFD 锁** |
| close | 任一 fd close 释放该进程全部 fcntl 锁 | 仅当**所有**引用该 OFD 的 fd 都 close 才释放 |
| exec | fcntl 锁保留（除非 close-on-exec）| OFD 锁保留（除非 close-on-exec）|
| F_GETLK 查询 | 可查询，但**不能证明"本进程持有"**（返回的是阻塞者 pid，非自身）| 同样不能直接证明自身持有 |

**结论**：05.9 用 `F_GETLK` 返回 PID 证明"当前进程持有自身记录锁"是**错误假设**——`F_GETLK` 查询的是"谁阻塞了我"，不是"我是否持有锁"。此假设不成立，必须放弃。

### 1.2 重新设计的 POSIX 锁方案（OFD 锁 + 线程互斥）

**方案：OFD 锁（平台支持时）+ 独立进程内线程互斥锁（ProcessMutex）组合。**

选择理由：
- OFD 锁以**打开文件描述**为锁单位，与 FD 生命周期绑定，语义清晰（不像传统 fcntl 锁"进程级、close 任一 fd 即释放全部锁"的陷阱）。
- OFD 锁能区分同一进程内不同 FD 的锁，配合进程内 `ProcessMutex`，才能实现"同一进程内多线程互斥"（传统 fcntl 锁做不到，同进程锁不冲突）。
- **兼容性边界**：OFD 锁（`F_OFD_SETLK`）在 Linux ≥3.15、glibc ≥2.20 支持；**并非所有 POSIX 系统支持**（macOS/BSD 不支持）。因此：

```
检测：尝试 F_OFD_SETLK 探测，成功 → 用 OFD 锁
不支持 → 降级为独立 OS 锁文件（第二把锁文件）+ ProcessMutex，
        并明确标记 platform="posix-nofd", 能力降级 NOT_VERIFIED
```

**禁止**：在不支持 OFD 锁的系统上静默降级为无锁执行——降级必须显式，且保证互斥仍由"独立锁文件 + 线程互斥"提供，只是不享受 OFD 的 FD 生命周期语义。

### 1.3 进程内线程互斥层（ProcessMutex）

OS 文件锁**不能自动替代**同一进程内多线程互斥（传统 fcntl 锁同进程不互斥；OFD 锁只对**不同 FD** 互斥，同一 FD 被多线程共享时仍需额外互斥）。

```python
class ProcessMutex:
    # 每个 authorization_id 一个进程内互斥锁（threading.Lock / RLock）
    _locks: dict[str, threading.Lock] = {}
    def __init__(self, auth_id):
        self.auth_id = auth_id
    def __enter__(self):
        self._mutex = ProcessMutex._locks.setdefault(self.auth_id, threading.Lock())
        self._mutex.acquire()
        return self
    def __exit__(self, *a):
        self._mutex.release()
```

**规则**：同一进程内对同一 auth_id 的所有临界区，先获取 `ProcessMutex`（线程互斥），再获取 OS 锁（跨进程互斥）。两者缺一不可。

### 1.4 FD 所有权与关闭规则

- 每个授权锁 FD 有**唯一所有者**：`LockHandle` 对象独占该 FD，`LockLease` 持有 LockHandle。
- 关闭规则：只有 `release_lock` 能关闭 FD；任何其他代码路径**禁止** close 该 FD。
- `dup` 产生的 FD 副本：**禁止**（LockHandle 不暴露 dup 能力；若系统层 dup，OS 层 OFD 锁仍随原打开文件描述，但设计层面禁止使用 dup 副本）。
- 防止"无意关闭其他 FD 导致锁释放"：锁 FD 与业务 FD 隔离，锁 FD 只在 LockHandle 内持有，不进入通用 fd 表。

### 1.5 fork 处理

- fork 后：子进程继承父进程的 fd 与 OFD 锁（OFD 锁随打开文件描述）。
- **禁止子进程未经重新 acquire 使用父进程 Lease**：子进程的 LeaseRegistry 为空，`assert_lock_owned` 会失败（registry lookup 无此 lease）。
- 子进程必须**重新 acquire_lock** 才能操作。
- 若 fork 后父进程退出，子进程持有的 OFD 锁仍有效（OFD 锁不随进程），但设计上要求子进程重新 acquire，避免共享父进程 Lease 的歧义。

### 1.6 异常 / 线程退出 / 进程退出 / FD 泄漏

| 场景 | 行为 |
|------|------|
| 异常 | acquire 失败不残留 FD；release 的 finally 保证关闭 |
| 线程退出 | ProcessMutex 由 threading.Lock 自动释放；Lease 绑定线程，线程退出后 registry sweep 清理 |
| 进程退出 | OS 释放 OFD 锁 + 关闭 FD |
| FD 泄漏 | LockHandle 持有 FD，release 必关闭；进程退出兜底 |
| 重复 acquire | 同 auth_id 同进程返回已有 lease 或报错（二选一，本设计：**报错**，避免嵌套）|
| 重复 release | 幂等（state 已 RELEASED 则忽略）|

### 1.7 Windows 锁协议（保留平台区分，进一步明确）

| 职责 | 机制 |
|------|------|
| 打开阶段互斥 | `CreateFileW(dwShareMode=0)` 独占打开，冲突返回 ERROR_SHARING_VIOLATION |
| 锁区间竞争 | `LockFileEx(LOCKFILE_EXCLUSIVE_LOCK)`，区间 offset=0, length=MAXDWORD |
| HANDLE 唯一所有者 | LockHandle 独占 HANDLE，只有 release 关闭 |
| 异常释放 | release 的 finally 调 CloseHandle |
| 超时/重试 | 打开冲突/锁竞争均非阻塞重试，超时 DENY |
| 自身锁归属查询 | **Windows 无 F_GETLK 等价物，不得虚构** |

**Windows 的三重约束**（05.10 保留并明确）：
1. `dwShareMode=0` 独占打开 → 打开阶段即排他，无他进程能打开同一锁文件。
2. `LockFileEx` 排他锁 → 锁区间排他。
3. 进程内 LeaseRegistry + ProcessMutex → 本进程内线程互斥 + 防伪造。

**保证边界**：Windows 的锁归属**不依赖"查询自身锁"**——因为 `dwShareMode=0` 已在打开阶段排他，能打开该锁文件的进程必然就是锁持有者。此边界标记 **NOT_VERIFIED**（未实测）。

### 1.8 统一锁接口契约

**对象定义：**

```python
@dataclass(frozen=True)
class LockToken:
    authorization_id: str
    acquisition_id: str      # 签发方生成，随机不可猜
    process_id: int
    thread_id: int
    lock_path: str
    lock_proto_version: str  # "posix-ofd" / "posix-nofd" / "windows-ntfs"
    issued_at: str

@dataclass(frozen=True)
class LockHandle:
    os_resource: object      # POSIX: fd(int)；Windows: HANDLE(int)
    file_identity: str
    platform: str            # 同 lock_proto_version

@dataclass
class LockLease:
    token: LockToken
    handle: LockHandle
    state: str               # ACTIVE / RELEASED / INVALIDATED
    acquired_at: str
    released_at: str | None

class LeaseRegistry:         # 进程内单例，私有
    _leases: dict[str, LockLease]
    def register(self, lease): ...
    def lookup(self, acquisition_id): ...
    def deregister(self, acquisition_id): ...
    def sweep(self, pid, thread_id): ...
```

**接口：**

```python
def acquire_lock(auth_id, timeout) -> LockLease | None:
    # 1. ProcessMutex 线程互斥
    # 2. open 锁文件（POSIX O_CREAT|O_RDWR 不截断；Windows CREATE_NEW/OPEN_EXISTING）
    # 3. 记录 file_identity
    # 4. OS 锁（POSIX OFD 锁 F_OFD_SETLK；Windows LockFileEx）
    # 5. 构造 LockToken + LockHandle + LockLease，登记 registry
    # 失败：超时 None；平台不支持 OFD → 降级 posix-nofd 或 DENY

def assert_lock_owned(lease, auth_id) -> None:
    # 验证的是"进程内租约有效性"+"句柄生命周期"+"OS 互斥资源持有"+"路径身份"
    # 1. registry lookup（进程内租约有效，防伪造/复制）
    # 2. state == ACTIVE
    # 3. token.authorization_id == auth_id；pid/tid 匹配
    # 4. _handle_still_open(handle)  —— 句柄生命周期
    # 5. _lock_still_held(handle)     —— OS 互斥资源持有（见下）
    # 6. _file_identity(lock_path) == handle.file_identity —— 路径身份

def release_lock(lease) -> None:
    # 幂等；state != ACTIVE 则忽略
    # finally: unlock(OS) + close(FD/HANDLE) + state=RELEASED + deregister
```

**`assert_lock_owned` 到底验证什么（不混淆四个概念）：**

| 验证层 | 方法 | 证明什么 |
|--------|------|----------|
| 进程内租约有效性 | LeaseRegistry.lookup | 本进程确实登记过该 lease（防伪造/复制）|
| 句柄生命周期 | `_handle_still_open`（POSIX `fcntl(F_GETFD)`；Windows `GetHandleInformation`）| FD/HANDLE 仍打开 |
| OS 互斥资源持有 | POSIX：**不查询自身锁归属**，而是依赖"能打开锁文件 + 已成功 OFD 锁"这一获取事实；Windows：依赖 dwShareMode=0 独占打开事实 | 锁互斥仍由本进程维持 |
| 路径身份 | 重新 `fstat`/`GetFileInformationByHandle` | 锁文件未被替换 |

**关键（不再依赖错误假设）**：`assert_lock_owned` **不使用 `F_GETLK` 查询自身锁归属**。POSIX 的"OS 互斥资源持有"由"本进程曾成功获取 OFD 锁、且锁 FD 仍打开、且锁文件身份未变"三者共同证明——因为 OFD 锁在 FD 仍打开时不会被其他进程夺走，身份未变说明锁文件未被替换，故锁仍归属本进程。这是**逻辑推导**而非不存在的"自身锁查询"。

### 1.9 P0-01 验收对照

| 验收项 | 结论 |
|--------|------|
| 锁验证有真实 OS 语义 | ✅（不再用 F_GETLK 查自身）|
| 进程内多线程互斥 | ✅（ProcessMutex）|
| FD/HANDLE 生命周期闭环 | ✅（唯一所有者 + release 必关闭）|
| 无错误的自身锁查询 | ✅ |
| 平台不支持不静默降级 | ✅（posix-nofd 显式降级 + NOT_VERIFIED）|

---

## 2. P0-02｜Candidate 提交与恢复协议

（以 P0-01 锁接口为前提）

### 2.1 Candidate 排他创建

- POSIX：`open(candidate_tmp, O_CREAT | O_EXCL | O_WRONLY, 0o600)`，随机命名，冲突重试。
- Windows：`CreateFileW(..., CREATE_NEW, ...)`，随机命名，ERROR_FILE_EXISTS 重试。
- 符号链接防护：POSIX `O_EXCL` + `O_NOFOLLOW`。

### 2.2 Candidate 创建→提交完整顺序

```
1. 写 candidate 临时文件（O_EXCL）→ 2. flush → 3. fsync 文件 → 4. rename 为正式 candidate → 5. fsync 父目录
```

### 2.3 正式文档提交协议（完整）

```python
def _commit_formal_doc(lease, auth_id, txn):
    assert_lock_owned(lease, auth_id)          # 1. 验证有效 Lease
    _verify_prepared_and_candidate(txn)        # 2. 校验 PREPARED + Candidate
    tmp = _unique_tmp(f"authorizations/{auth_id}/.authorization.json.tmp")  # 3. 同目录唯一临时
    _copy(txn.candidate_path, tmp)             # 4. 复制 Candidate
    assert sha256_file(tmp) == txn.new_document_sha256   # 5. 校验事务 ID/revision/哈希
    doc = _read(tmp)
    assert doc["authorization"]["record_revision"] == txn.new_revision
    _fsync_file(tmp)                           # 6. 刷新临时文件
    _atomic_replace(f"authorizations/{auth_id}/authorization.json", tmp)  # 7. 原子替换
    _fsync_parent_dir("authorizations/" + auth_id)   # 8. POSIX 父目录 fsync（Windows 见 2.4）
    assert sha256_file(f"authorizations/{auth_id}/authorization.json") == txn.new_document_sha256  # 9. 重新读取核验
    _journal_write(auth_id, txn.transaction_id, state="COMMITTED")       # 10. 更新 COMMITTED
```

### 2.4 POSIX 父目录 fsync / Windows 边界

- POSIX：步骤 8 **必须** `fsync(父目录 fd)`。
- Windows：区分文件刷新（`FlushFileBuffers`）、替换原子性（`MoveFileEx`）、断电持久性（**无法证明**）。Windows 下步骤 8 降级为 `FlushFileBuffers`，断电持久性标记 **NOT_VERIFIED**。

### 2.5 Candidate 原件保留

Candidate 原件保留到事务终态 + 清理授权完成，提交是复制（非移动）。

### 2.6 唯一处理动作表

| 情形 | 唯一动作 |
|------|----------|
| 临时文件已存在（O_EXCL/CREATE_NEW 失败）| 换随机名重试 |
| 重复 Candidate | 以 PREPARED candidate_path 为准，其余隔离 |
| 哈希冲突（candidate ≠ new_document_sha256）| 隔离 + DENY |
| revision 冲突 | DENY |
| 替换失败 | 保留旧文档 + 临时文件隔离 + DENY |
| 目录同步失败 | 标记 INDETERMINATE + DENY（无法确认持久化）|

### 2.7 恢复矩阵（状态 × 文件组合）

| PREPARED | 正式文档 | Candidate | 唯一动作 |
|----------|----------|-----------|----------|
| 缺失 | 旧版 | 存在 | 孤儿 candidate，隔离 + DENY |
| 缺失 | 旧版 | 缺失 | 无事务，无副作用 |
| 损坏 | 任意 | 任意 | 隔离 journal + DENY |
| 有效 | 旧版 | 完整 | 自动重放提交 |
| 有效 | 旧版 | 缺失 | DENY（无候选）|
| 有效 | 新版 | 任意 | 补 COMMITTED |
| 有效 | 未知版本 | 任意 | 隔离 + DENY |
| 有效(COMMITTING) | 旧/新 | 完整 | 按摘要唯一判定 |
| 有效(ABORTED) | 旧版 | 完整 | 清理 candidate，不提交 |
| 有效(INDETERMINATE) | 任意 | 任意 | DENY + 人工 |

每个崩溃点唯一动作，证据不足隔离 + DENY。

---

## 3. P0-03｜CAS 原子性与 Journal 隔离恢复

（以 P0-01 锁接口为基础）

### 3.1 CAS 绑定有效授权锁

所有 Journal 写入入口**必须持有同一授权锁**（acquire + assert_lock_owned 后调用）。

### 3.2 CAS 完整原子性前置条件

```python
def _journal_update_cas(lease, auth_id, txn_id, expected_revision, new_state):
    assert_lock_owned(lease, auth_id)          # 1. 锁获取（已持）
    cur = _journal_read(auth_id, txn_id)       # 2. 读取 revision
    if cur is None:
        if expected_revision != 0: return CAS_FAILED
    elif cur.revision != expected_revision:     # 3. 比较
        return CAS_FAILED
    next_rev = expected_revision + 1           # 4. 生成 next
    tmp = _unique_tmp_journal(txn_id)          # 5. 唯一临时文件（O_EXCL）
    _write(tmp, {**new_state, "revision": next_rev})
    _fsync(tmp)                                # 6. 刷新
    _atomic_replace(f"journal/{txn_id}.txn", tmp)  # 7. 原子替换
    _fsync_parent_dir("journal")               # 8. 目录持久化
    _verify(f"journal/{txn_id}.txn")           # 9. 最终校验
    return CAS_OK
```

### 3.3 并发证明（无两个写入者同时通过同一版本检查）

- 同一进程多线程：ProcessMutex 串行。
- 不同进程：同一授权锁（OFD 锁 + 独占打开）串行。
- 恢复程序并发：恢复也需 acquire 同一授权锁。
- **结论**：CAS 的读-比较-写在授权锁内完成，任何时刻只有一个写入者持有该锁，故不存在两个写入者同时通过同一 expected_revision 检查。

### 3.4 CAS 失败唯一动作

CAS_FAILED → 返回错误，**不静默覆盖、不自动改写 expected_revision、不无条件重试**。上层按错误码决定：重试需重新 acquire + 重新读取最新 revision。

### 3.5 Journal 临时文件唯一命名与遗留 tmp

- 唯一命名：`journal/<txn_id>.txn.tmp.<random>`，O_EXCL/CREATE_NEW 排他。
- 遗留 tmp 归属识别：tmp 文件名含 txn_id + random，扫描时按 txn_id 归属。
- 隔离/恢复/清理：遗留 tmp 若无对应 PREPARED → 视为半写，隔离 + 清理；有对应 PREPARED → 保留待恢复。

### 3.6 Journal 损坏隔离（重新设计，消除伪接口）

05.9 用了未定义的 `_atomic_replace(src, None)`，必须消除。

```python
def _quarantine_journal(auth_id, txn_id):
    src = f"journal/{txn_id}.txn"
    # 1. 验证原文件身份
    src_identity = _file_identity(src)
    # 2. 复制到唯一隔离临时文件
    qtmp = f"quarantine/{txn_id}.txn.qtmp.{random}"
    _copy(src, qtmp)
    # 3. 校验隔离副本哈希
    assert sha256_file(qtmp) == sha256_file(src)
    # 4. 刷新隔离副本
    _fsync(qtmp)
    # 5. 原子进入 quarantine（rename 为正式隔离名）
    qfinal = f"quarantine/{txn_id}.txn.{random}"
    _atomic_replace(qfinal, qtmp)
    # 6. 持久化隔离审计记录
    _write_audit(qfinal, "journal quarantined")
    # 7. 按明确顺序处理原文件：rename src -> src.corrupt（保留，不直接删）
    _atomic_replace(src + ".corrupt", src)
```

每个崩溃窗口可恢复：qtmp 半写 → 下次发现 qtmp 无对应 qfinal → 重做；qfinal 已存在 → 幂等跳过。

### 3.7 审计失败 / 隔离目录不可用 / 原文件变化 / 重复副本

| 情形 | 唯一动作 |
|------|----------|
| 审计记录写入失败 | DENY（隔离不完整，人工）|
| 隔离目录不可用 | DENY |
| 原文件变化（隔离中 src 被改）| 身份/哈希不匹配 → DENY + 人工 |
| 重复隔离副本 | qfinal 已存在 → 幂等，不重复隔离 |

### 3.8 POSIX / Windows 真实写入顺序

- POSIX：写 tmp → fsync(tmp) → rename → fsync(父目录)。
- Windows：写 tmp → FlushFileBuffers → MoveFileEx（替换）→（无目录 fsync 等价）→ 断电持久性 NOT_VERIFIED。

---

## 4. P0-04｜清理归因与终态审计证据

### 4.1 审计存储独立性

| 维度 | 规定 |
|------|------|
| 文件系统 | 与授权文档/Candidate/Journal**同一文件系统**（无法实现跨 FS 原子性）|
| 故障域 | **同一故障域**——不宣称"独立可靠存储" |
| 谁有权修改 | 仅授权进程（权限 0o600）|
| 能否被普通清理删除 | **否**——审计记录存独立 `audit/` 目录，清理流程只删 candidate/journal，不删 audit |

**限制声明**：审计存储与事务文件同 FS 同故障域，**不具备独立故障保护能力**。因此本设计只承诺"审计记录不会被清理流程误删"，**不承诺**"审计存储自身故障时仍可靠"。若需独立故障保护，须额外授权引入外部存储，本工单不涉及。

### 4.2 终态证明快照（不只存路径/哈希/状态字符串）

清理审计记录保存**完整不可变终态快照**：

```python
@dataclass(frozen=True)
class TerminalStateSnapshot:
    transaction_id: str
    terminal_state: str          # COMMITTED / ABORTED
    terminal_journal_digest: str # 终态 journal 的完整内容摘要
    revision: int
    new_document_sha256: str
    snapshot_version: str
    source_identity: str         # 来源文件身份
```

**验证规则**：终态快照必须与 COMMITTED/ABORTED journal 的完整内容摘要一致，才能证明事务确已进入终态。**不能只存 journal 路径、文件哈希或一个状态字符串。**

### 4.3 Candidate 与 Journal 分别五步清理

每个对象独立执行：`DELETE_INTENT 持久化 → 校验身份/哈希/终态授权 → 删除 → 验证不存在 → REMOVED 持久化`。

### 4.4 对象不存在的证明规则

若对象不存在，**必须证明其满足本清理事务的删除目标**，不能仅凭"不存在"推断本次删除成功：
- 校验点：DELETE_INTENT 已持久化 + 该对象身份/哈希与 intent 记录一致 → 才能确认"对象已按意图删除"。
- 证据不足 → **DENY**。

### 4.5 清理审计记录自身持久化

用唯一临时文件 + 排他创建（O_EXCL/CREATE_NEW）+ 原子替换 + 平台持久化协议（同 P0-03 §3.8）。

### 4.6 唯一动作表

| 情形 | 唯一动作 |
|------|----------|
| 审计损坏/不可用 | DENY（失去审计依据）|
| 身份变化 | 隔离 + DENY |
| 哈希变化 | 隔离 + DENY |
| 重复请求 | cleanup_id 幂等 |
| 恢复接管 | 按 cleanup_state 重判，幂等 |
| 再次崩溃 | 重新读取审计记录，重判 |

### 4.7 统一清理状态机（消除两套表达）

05.9 同时保留 `CLAIMED/CANDIDATE_REMOVED/JOURNAL_REMOVED` 与 `DELETE_INTENT/REMOVED` 两套表达，矛盾。统一为**单一状态机**：

```
PENDING → DELETE_INTENT → REMOVED → CLEANED
              │                │
              ├→ FAILED（可重试）│
              └→ INDETERMINATE（接管重判）
```

- candidate 对象与 journal 对象**各走一遍**此状态机（`cleanup_object ∈ {candidate, journal}` 作为记录字段区分）。
- 字段：`cleanup_id / authorization_id / transaction_id / cleanup_object / cleanup_state / object_identity / object_sha256 / terminal_snapshot / claimed_at / completed_at / failure_reason / retry_count / cleanup_sequence`。

---

## 5. 端到端交叉审查

### 5.1 端到端时序图

```
acquire_lock(auth_id)                       [ProcessMutex + OS 锁]
  assert_lock_owned
  [C] Candidate 写(O_EXCL) → fsync → rename → fsync目录
  [P] PREPARED journal CAS → fsync → rename → fsync目录
  [D] 复制 candidate → 正式文档 tmp → fsync → 原子替换 → fsync目录 → 核验
  [M] COMMITTED journal CAS → fsync → rename → fsync目录
release_lock                                [释放 OS 锁 + ProcessMutex]

--- 清理（独立 acquire，同一锁协议）---
acquire_lock(auth_id)
  [清理candidate] DELETE_INTENT → 校验 → 删 → 验证 → REMOVED
  [清理journal]   DELETE_INTENT → 校验 → 删 → 验证 → REMOVED
  [终态快照已在 DELETE_INTENT 前持久化]
release_lock
```

### 5.2 四套状态转换图

- 锁状态：`(无) → ACTIVE → RELEASED / INVALIDATED`
- 事务状态：`(无) → PREPARED → COMMITTING → COMMITTED / ABORTED / INDETERMINATE`
- Journal 状态：`(无) → PREPARED → COMMITTED / ABORTED / INDETERMINATE`（revision 单调）
- 清理状态：`PENDING → DELETE_INTENT → REMOVED → CLEANED / FAILED / INDETERMINATE`

### 5.3 Windows/POSIX 协议差异表

| 项 | POSIX | Windows |
|----|-------|---------|
| 锁 | OFD 锁（支持时）/ 独立锁文件（降级）| CreateFileW 独占 + LockFileEx |
| 线程互斥 | ProcessMutex | ProcessMutex |
| 锁归属验证 | 获取事实 + FD 打开 + 身份未变（不查自身锁）| 独占打开事实（无查询接口）|
| 目录持久化 | fsync 父目录 | 无等价，FlushFileBuffers，断电 NOT_VERIFIED |

### 5.4 进程崩溃 vs 断电故障模型

| 场景 | POSIX | Windows |
|------|-------|---------|
| 进程崩溃 | OFD 锁释放 + fd 关闭；journal 已 fsync 可恢复 | HANDLE 关闭；FlushFileBuffers 保证 |
| 断电 | 需父目录 fsync 才保证 | **无法证明**，NOT_VERIFIED |

### 5.5 竞争矩阵

| 场景 | 结果 |
|------|------|
| 同进程多线程 | ProcessMutex 串行 |
| 多进程 | OS 锁串行 |
| fork | 子进程须重新 acquire |
| 重复请求 | transaction_id / cleanup_id 幂等 |
| 恢复接管 | 重新 acquire + 重判，幂等 |

### 5.6 恢复分支唯一动作清单

汇总 §2.7、§3.4、§3.7、§4.6 的所有唯一动作，形成完整清单（见 DIFF/ISSUES 附录）。

### 5.7 一致性结论

**P0-02～P0-04 只使用 P0-01 定义的锁接口**（acquire_lock / assert_lock_owned / release_lock + ProcessMutex），未自行引入另一套锁语义。无跨章节矛盾。

---

## 6. P1 与既有阻断项状态

| 编号 | 状态 |
|------|------|
| P1-01 | NOT_VERIFIED |
| P1-02 | NOT_TESTED |
| P1-03 | 保持原状态，不执行 |
| P0-BLOCK-1 | 未解决 |
| P0-BLOCK-2 | 未解决 |
| P1-BLOCK-3 | 未解决 |
| F1 / P0.8 / F2 | 不解除 |
| IMPLEMENT-01 | STOP |

---

## 7. IMPLEMENT-01 准入结论

**PASS-WITH-ISSUES（维持阻断）。**

四项 P0 已闭环：锁协议基础修正（不再依赖错误的 F_GETLK 自身锁查询，改用 OFD 锁 + ProcessMutex）、Candidate 提交协议完整、CAS 绑定锁 + Journal 隔离恢复、清理归因 + 终态快照。

但存储事务、锁机制、崩溃恢复为**设计级契约**，真实实现前 NOT_VERIFIED / NOT_TESTED。三个阻断项未解决。

**本工单不授权实现、不授权测试、不解除 IMPLEMENT-01 STOP。F1/P0.8/F2 未解除。最终原则：锁协议先行、证据优先、失败即停止、未验证不标 PASS、未经授权不实现。**

---

## 8. 交付物索引

| 文件 | 内容 |
|------|------|
| `USIT-STAGE-B-DESIGN-05.10.md` | 本文 |
| `USIT-STAGE-B-DESIGN-05.10-ISSUES.md` | 问题清单 |
| `USIT-STAGE-B-DESIGN-05.10-DIFF.md` | DESIGN-05.9 → 05.10 差异 |
| `USIT-STAGE-B-DESIGN-05.10-EXECUTION-REPORT.md` | 执行报告 |
| `USIT-STAGE-B-DESIGN-05.10-HASH-MANIFEST.md` | 哈希清单 |
| `USIT-STAGE-B-DESIGN-05.10-HASH-RECEIPT.txt` | 哈希收据 |
