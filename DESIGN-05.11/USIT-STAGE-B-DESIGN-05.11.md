# USIT-STAGE-B-DESIGN-05.11｜锁协议与事务恢复六项缺口定向修订

> 阶段 B · DESIGN-05.11 · READ-ONLY ANALYSIS + DESIGN REVISION
> 作者：USIT 主开发 · 生成：2026-10-31（GMT+8）
> 前置：DESIGN-05.10 独立审查结论 **FAIL**
> 工作区：`C:\Users\Administrator\WorkBuddy\2026-10-15-22-09-47\usit_p0`
> 交付目录：`C:\Users\Administrator\WorkBuddy\2026-10-15-22-09-47\usit_p0_05.11_reports`（**项目外**）
> 状态：**STOP — 等待独立复审**（未授权 IMPLEMENT-01）

---

## 0. 范围与执行顺序

本轮**不推倒重做 05.10，不增加无必要抽象层**，仅修复独立审查确认的六项缺口。执行顺序严格为：

**P0-01 POSIX 锁降级 → P0-02 fork 生命周期 → P0-03 锁文件身份 → P0-04 Candidate 恢复 → P0-05 Journal 隔离 → P0-06 终态快照与清理归因 → 端到端交叉审查。**

后序章节引用前序已确定的契约，不得各自设计互相矛盾的锁/存储协议。

> 文档设计通过 ≠ 代码实现通过。不得把伪代码推演写成实测结果。未验证平台行为标记 NOT_VERIFIED，未执行内容标记 NOT_TESTED。

**05.10 六项缺口 → 05.11 修正对照（总览）：**

| # | 05.10 缺口 | 05.11 修正 |
|---|-----------|-----------|
| P0-01 | `posix-nofd` 降级仅说"独立锁文件"，未定义真实跨进程锁原语 | 明确区分锁文件载体 vs OS 锁原语，降级选 `flock`，无锁则 DENY |
| P0-02 | 错误假设"fork 后子进程 LeaseRegistry 为空" | fork 复制 Registry/FD/锁，需 `register_at_fork` 清理 + 重建 |
| P0-03 | 未明确锁文件身份、路径替换、TOCTOU | 定义路径/句柄/对象身份三层，身份不一致即 DENY |
| P0-04 | `O_EXCL` 被误当后续 rename 保护；恢复仅凭 SHA-256 推断 | 无覆盖提交原语 + transaction_id/revision 全量绑定校验 |
| P0-05 | 隔离"幂等跳过"未重新校验；rename 后未 fsync 父目录 | 每次 rename 后 fsync 父目录；隔离副本重新校验 |
| P0-06 | 终态快照只存摘要字段；删除归因不分 ALREADY_ABSENT | 完整规范化终态内容 + tombstone 删除归因三态 |

---

## 1. P0-01｜POSIX 无 OFD 支持时的真实跨进程锁原语

### 1.1 核心区分：锁文件载体 ≠ OS 锁原语

05.10 的 `posix-nofd` 降级只说"创建独立锁文件"，这是缺陷——**创建锁文件不等于获得锁**。必须区分两个概念：

| 概念 | 定义 |
|------|------|
| **锁文件载体**（lock file carrier）| 文件系统上用于标识"这把锁代表哪个资源"的对象 |
| **OS 锁原语**（OS lock primitive）| 操作系统真正提供互斥语义的机制（OFD 锁 / flock / fcntl 记录锁）|

无 OS 锁原语时，"有锁文件"只是"有文件"，**不代表任何互斥**。因此降级必须在锁文件之上叠加一个真实 OS 锁原语。

### 1.2 非 OFD 跨进程锁机制比较

| 维度 | `flock`（BSD 风格）| 传统 `fcntl` 记录锁（F_SETLK）|
|------|-------------------|------------------------------|
| 锁单位 | 打开文件描述（open file description）| 进程（+ inode）|
| 同进程多线程 | **不互斥**（同 OFD 共享锁）| **不互斥**（同进程锁不冲突）|
| 同进程不同 open | 冲突（不同 OFD）| 冲突 |
| dup | 共享同一锁 | 共享（同一进程）|
| fork | 子进程**不继承** flock（锁随 OFD，但 fork 复制 fd 引用同一 OFD）| 子进程**不继承**记录锁 |
| close | 任一 fd close 该 OFD 则释放（所有引用同 OFD 的 fd close 才释放）| **任一** fd close 释放该进程对该 inode 的**全部**记录锁 |
| exec | 保留（除非 close-on-exec）| 保留（除非 close-on-exec）|
| 关键陷阱 | close 语义相对可预测 | **close 任一 fd 释放全部锁**，极危险 |

**结论**：在非 OFD 降级场景，选择 **`flock`**。理由：
1. flock 的锁单位是打开文件描述（open file description），与 OFD 锁语义最接近，FD 生命周期规则一致（所有引用同一 OFD 的 fd 都 close 才释放，而非"任一 close 释放全部"）。
2. 传统 fcntl 记录锁"close 任一 fd 释放该进程全部锁"是已知陷阱，若混用会制造"锁被意外释放"的窗口，与 05.10 已确定的"FD 唯一所有权 + release 必关闭"原则冲突。

### 1.3 降级实现契约（flock）

```python
# 锁文件载体 + OS 锁原语
def _acquire_posix_nofd(auth_id, lock_path, timeout) -> LockLease | None:
    # 1. open 锁文件（O_CREAT|O_RDWR，不截断，不 O_TRUNC）
    fd = os.open(lock_path, os.O_CREAT | os.O_RDWR, 0o600)
    # 2. 记录 file_identity（fstat）
    ident = _fstat_identity(fd)
    # 3. flock 非阻塞 + 超时重试
    deadline = now + timeout
    while True:
        try:
            fcntl.flock(fd, fcntl.LOCK_EX | fcntl.LOCK_NB)   # 排他 + 非阻塞
            break
        except BlockingIOError:
            if now >= deadline:
                os.close(fd); return None                    # 超时 DENY
            sleep(短间隔)
    # 4. 重新 fstat，确认锁文件身份未在 open 与 flock 之间变化
    if _fstat_identity(fd) != ident:
        fcntl.flock(fd, fcntl.LOCK_UN); os.close(fd); return None  # TOCTOU → DENY
    # 5. 构造 Token/Handle/Lease，登记 registry
    return _build_lease(auth_id, fd, "posix-nofd-flock", ident)

def _release_posix_nofd(lease):
    if lease.state != "ACTIVE": return                       # 幂等
    fcntl.flock(lease.handle.os_resource, fcntl.LOCK_UN)     # 解锁
    os.close(lease.handle.os_resource)                       # 关闭 FD
    lease.state = "RELEASED"; LeaseRegistry.deregister(lease.token.acquisition_id)
```

**行为定义：**

| 项 | 规定 |
|----|------|
| 锁区间 | flock 锁整个文件，无区间概念 |
| 阻塞/非阻塞 | 非阻塞（LOCK_NB）+ 超时重试 |
| 超时 | 超时返回 None（DENY）|
| 错误码 | BlockingIOError → 重试；其他异常 → 关闭 FD + DENY |
| 释放 | release 的 finally 保证 LOCK_UN + close |
| 异常退出 | 进程退出 OS 自动释放 flock + 关闭 FD；线程退出不影响 flock（锁随 OFD 不随线程）|
| 同进程多线程 | **仍由 ProcessMutex 提供**，flock 不替代线程互斥 |

### 1.4 平台支持标记（一律 NOT_VERIFIED，未经实测）

| 平台 | OFD 锁 | 非 OFD 降级（flock）| 说明 |
|------|--------|---------------------|------|
| Linux ≥3.15 | 可用 | flock 亦可用 | NOT_VERIFIED（未实测）|
| macOS | **不可用** | flock 可用 | 降级 flock，NOT_VERIFIED |
| BSD（FreeBSD/OpenBSD）| **不可用** | flock 可用 | 降级 flock，NOT_VERIFIED |

**平台标识**：`posix-ofd`（OFD 锁）/ `posix-nofd-flock`（flock 降级）/ `windows-ntfs`。写入 LockToken.lock_proto_version。

### 1.5 验收对照

| 验收项 | 结论 |
|--------|------|
| 区分锁文件载体 vs OS 锁原语 | ✅ §1.1 |
| 无 OFD 时选明确降级（flock）| ✅ §1.2/§1.3 |
| 平台无锁原语则 DENY，不允许无锁降级 | ✅（若 flock 也不可用 → DENY，不进入无锁状态）|
| 线程互斥仍由 ProcessMutex | ✅ §1.3 |
| Linux/macOS/BSD 分别标记 NOT_VERIFIED | ✅ §1.4 |

**P0-01 结论：PASS（设计级）**，flock 降级语义 NOT_VERIFIED（未实测）。

---

## 2. P0-02｜fork 后 Registry、FD、ProcessMutex 生命周期

### 2.1 纠正错误假设

05.10 §1.5 假设"fork 后子进程 LeaseRegistry 为空"是**错误**的：`fork()` 复制进程地址空间，子进程**继承**父进程的 LeaseRegistry 字典、LockLease 对象、LockToken、以及指向同一 OFD 的 FD。子进程不会自动获得空 Registry。

同样错误的还有：**ProcessMutex 中的 `threading.Lock` 不会因持有线程退出而自动释放**。若 fork 时父进程某线程正持有 ProcessMutex 且处于 locked 状态，子进程继承的是一把**永久锁死的锁**。

### 2.2 fork 继承资源所有权表

| 资源 | fork 后子进程状态 | 危险 | 唯一处理动作 |
|------|------------------|------|-------------|
| LeaseRegistry 字典 | 复制（含父进程 lease 引用）| 子进程误用父租约 | 清空 + 作废 |
| LockLease 对象 | 复制（浅）| 与父进程共享逻辑租约 | 标记 INVALIDATED |
| LockToken | 复制 | 复制后 acquisition_id 相同 | 作废，不参与 assert |
| 锁 FD（同一 OFD）| 继承（fd 表复制）| 子进程持有 FD 延长父进程 OFD 锁生命周期 | close（且**不调 flock LOCK_UN**，见下）|
| ProcessMutex（threading.Lock）| 复制（可能 locked）| 永久锁死 | 重建为全新未锁定 Lock |
| 业务 FD | 继承 | 资源泄漏 | 按需 close |

**关键（锁 FD 处理）**：子进程 close 继承的锁 FD 时，**不得先调 `flock(fd, LOCK_UN)`**。因为 OFD/flock 锁是"所有引用同一 OFD 的 fd 都 close 才释放"，子进程只 close 自己这份 fd 引用即可，调用 LOCK_UN 反而可能释放父进程仍持有的锁（flock 语义：LOCK_UN 释放整个 OFD 上的锁）。正确动作是**直接 `os.close(fd)`**，不碰锁。

### 2.3 fork 后处理机制

```python
def _after_fork_child():
    # 1. 作废继承的 Lease / Token
    for acq_id, lease in list(LeaseRegistry._leases.items()):
        lease.state = "INVALIDATED"
    LeaseRegistry._leases.clear()          # 清空
    # 2. 关闭继承的锁 FD（不调 LOCK_UN）
    for fd in _inherited_lock_fds:
        try: os.close(fd)
        except OSError: pass
    _inherited_lock_fds.clear()
    # 3. 重建 ProcessMutex 为全新未锁定状态
    ProcessMutex._locks.clear()            # 丢弃可能 locked 的锁
    # 4. 重置 token 计数器/随机源（避免与父进程生成相同 acquisition_id）

# 注册
os.register_at_fork(after_in_child=_after_fork_child)
```

`os.register_at_fork`（Python 3.7+）保证每次 fork 后子进程自动执行清理。若平台不支持（无法注册），则**禁止在持锁期间 fork**，fork 前必须 release 所有锁（见 §2.4）。

### 2.4 持锁期间 fork 的策略

| 场景 | 策略 |
|------|------|
| acquire/release 临界区外 fork | 允许；after_in_child 清理，子进程须重新 acquire |
| acquire/release 临界区内 fork | **禁止**；在 acquire/release 代码路径设置 `_in_lock_critical_section` 标志，fork 前检查，若为 True 则 DENY（抛异常）|
| 平台不支持 register_at_fork | 禁止持锁 fork；要求 fork 前 release 全部锁 |

### 2.5 生命周期场景表

| 场景 | 父进程 | 子进程 | 唯一动作 |
|------|--------|--------|----------|
| fork 后父进程仍运行 | 持锁继续 | 清理继承 + 重新 acquire | 各自独立 acquire |
| fork 后父进程退出 | 退出释放锁 | 子进程 OFD 锁因父 close 仍持（FD 引用）| 子进程重新 acquire（若需）|
| 子进程先退出 | 不受影响 | close FD 释放其引用 | 无额外动作 |
| 子进程重新 acquire | — | 全新 lease 登记 | 独立 lease |

### 2.6 线程/异常/退出真实清理责任

| 场景 | 真实责任 |
|------|---------|
| 线程退出 | **ProcessMutex 不自动释放**（若线程异常退出未在 finally 释放，锁永久锁死）；需在 acquire 处用 try/finally 保证释放 |
| 异常 | release 的 finally 保证 LOCK_UN + close + deregister |
| 进程退出 | OS 自动释放 OS 锁 + 关闭 FD |
| 重复 release | 幂等（state != ACTIVE 则忽略）|
| FD 泄漏 | LockHandle 独占 FD，release 必关闭；进程退出兜底 |

**不变量（不变量 INV-2.1）**：任何时刻，一个锁 FD 只有一个所有者（LockHandle），释放只经 release_lock 单一路径；fork 后子进程不继承任何"有效"lease 或 locked 的 ProcessMutex。

### 2.7 验收对照

| 验收项 | 结论 |
|--------|------|
| fork 复制 Registry/FD/锁的事实被承认 | ✅ §2.1 |
| after_in_child 清理 + 作废 | ✅ §2.3 |
| 子进程不复用父租约 | ✅ §2.2 |
| 锁 FD 处理不延长父锁生命周期 | ✅ §2.2（close 不调 LOCK_UN）|
| ProcessMutex 重建 | ✅ §2.3 |
| 持锁 fork 策略 | ✅ §2.4 |
| 不声称 threading.Lock 自动释放 | ✅ §2.6 |

**P0-02 结论：PASS（设计级）**，register_at_fork 行为 NOT_TESTED。

---

## 3. P0-03｜锁文件身份、路径替换与 TOCTOU

### 3.1 三层身份区分

| 层 | 对象 | 识别方法 |
|----|------|---------|
| 路径身份（path）| 文件系统路径字符串 | `open()` 前记录路径 |
| 句柄锁对象身份（open object）| 打开后实际文件对象 | POSIX `fstat`（dev+ino）；Windows `GetFileInformationByHandle`（volume serial + file index）|
| 锁原语作用对象（lock target）| OS 锁实际锁住的东西 | OFD 锁→open file description；flock→OFD；Windows→HANDLE |

**关键**：锁原语作用对象绑定在**打开句柄/OFD**上，而非路径字符串上。路径被替换不影响已打开句柄仍锁住原对象，但会影响**后续通过路径重新打开**得到的对象。

### 3.2 锁文件生命周期操作规则

| 操作 | 规则 |
|------|------|
| 创建 | O_CREAT（不 O_TRUNC），Windows CREATE_NEW |
| 打开 | 记录 fstat/file info 身份 |
| 替换（rename 覆盖）| **持锁期间禁止**；锁文件身份变化即 DENY |
| 重命名 | 同替换，身份可能变 |
| 删除 | **持锁期间禁止**；删除后路径可被重建为新对象 |
| 符号链接/重解析点 | 打开用 O_NOFOLLOW（POSIX）拒绝 symlink；Windows 拒绝 reparse point |

### 3.3 持锁期间身份变化检测

```python
def _assert_lock_identity(lease) -> bool:
    # 用已打开句柄重新取对象身份，与打开时记录的身份比较
    cur = _handle_identity(lease.handle)      # POSIX fstat(fd)；Windows GetFileInformationByHandle
    return cur == lease.handle.file_identity

def _assert_path_matches_handle(lease) -> bool:
    # 路径当前指向的对象，是否仍是句柄锁住的对象
    try:
        path_stat = os.stat(lease.token.lock_path, follow_symlinks=False)
        return (path_stat.st_dev, path_stat.st_ino) == lease.handle.file_identity
    except FileNotFoundError:
        return False   # 路径已被删除/替换 → DENY
```

**验证规则**：`assert_lock_owned` 的"路径身份"层，不再只"重新 stat 相同"，而是**用句柄对象身份与路径当前对象身份比较**。两者不一致（路径被替换、原文件被删、指向新文件、硬链接/重解析点变化）→ **立即 DENY，禁止提交事务**。

### 3.4 锁文件是否允许清理

**不允许**在存在活动 Lease 或等待者时清理锁文件。若确需清理（如锁文件残留），必须同时满足：

1. 无任何活动 Lease（LeaseRegistry 中无该 auth_id 的 ACTIVE lease）。
2. 无等待者（无正在 acquire 的进程）。
3. 删除过程不制造第二把锁——删除前必须先取得该锁文件的 OS 锁（证明独占），删除后立即释放，窗口内无他进程能成功 acquire。

满足上述才允许删除；否则 DENY。**此能力标记 NOT_VERIFIED（未实测并发删除窗口）。**

### 3.5 Windows 职责区分（不混为一谈）

| 机制 | 职责 | 不是 |
|------|------|------|
| `CreateFileW(dwShareMode=0)` | **打开阶段**排他：其他进程无法同时打开该文件 | 不是锁区间机制 |
| `LockFileEx(LOCKFILE_EXCLUSIVE_LOCK)` | **锁区间**排他：锁定 byte range | 不是打开阶段机制 |

两者是**两个独立阶段**：先独占打开（挡住其他 opener），再锁区间（挡住其他 byte-range 锁）。独占打开已保证"能打开者即锁持有者"，区间锁是对同一 HANDLE 的字节范围排他。不得把独占打开语义等同于区间锁语义。

### 3.6 验收对照

| 验收项 | 结论 |
|--------|------|
| 三层身份区分 | ✅ §3.1 |
| POSIX fstat / Windows 句柄信息识别 | ✅ §3.1 |
| 创建/打开/替换/删除/symlink 规则 | ✅ §3.2 |
| 身份变化检测 + DENY | ✅ §3.3 |
| 不靠"路径重读相同"证明锁有效 | ✅ §3.3（句柄 vs 路径对象比较）|
| Windows 职责区分 | ✅ §3.5 |

**P0-03 结论：PASS（设计级）**，Windows 句柄身份、并发删除窗口 NOT_VERIFIED。

---

## 4. P0-04｜Candidate 无覆盖提交与事务身份恢复

### 4.1 Candidate 临时文件 vs 正式 Candidate 身份

| 对象 | 身份 | 说明 |
|------|------|------|
| Candidate 临时文件 | `candidate/<txn_id>.tmp.<random>` | 写入中，未发布 |
| 正式 Candidate | `candidate/<txn_id>.candidate` | 已发布，被 PREPARED 引用 |

两者是**不同身份**，不能混淆。`O_EXCL/CREATE_NEW` 只保护**临时文件创建**这一步，**不保护后续 rename 到正式路径的覆盖**——这是 05.10 的缺陷。

### 4.2 无覆盖提交原语

正式 Candidate 最终路径提交必须具备**无覆盖语义**（rename 到已存在路径会覆盖，需防止）。

| 平台 | 无覆盖提交原语 | 失败行为 |
|------|---------------|---------|
| POSIX | `renameat2(RENAME_NOREPLACE)`（Linux ≥3.15）；不支持则用 `link(tmp, final)` + `unlink(tmp)` 两步（link 已存在路径会 EEXIST，天然无覆盖）| EEXIST → DENY（该 txn_id 已有 candidate）|
| Windows | `MoveFileEx(..., MOVEFILE_REPLACE_EXISTING)` 会覆盖，**不可用**；用 `CreateFileW(CREATE_NEW)` 创建空占位 + 写入，或 `MoveFileEx` 前先 `CreateFileW(final, CREATE_NEW)` 探针占位 | 占位已存在（ERROR_FILE_EXISTS）→ DENY |

**POSIX 兜底**（无 renameat2 时）：`os.link(tmp, final)` 对已存在 final 会抛 `FileExistsError`，天然提供无覆盖语义；成功后 `os.unlink(tmp)`。这是可移植的无覆盖发布。

### 4.3 PREPARED 必须绑定的字段

PREPARED journal 必须绑定以下字段，缺一即 DENY：

| 字段 | 说明 |
|------|------|
| transaction_id | 唯一事务 ID |
| authorization_id | 授权 ID |
| candidate_identity | 正式 Candidate 的 fstat 身份（dev+ino）|
| candidate_sha256 | Candidate 内容 SHA-256 |
| old_revision | 旧 revision |
| target_revision | 目标 revision |
| formal_doc_target_sha256 | 正式文档替换后目标摘要 |
| schema_version | schema 版本 |

### 4.4 恢复时的事务身份验证（不能仅凭 SHA-256）

正式文档为**新版**时，恢复必须验证文档**内部**的 `transaction_id`、`revision`、目标摘要与 PREPARED **全部匹配**，不能仅凭 SHA-256 或 revision 推断归属。因为两个不同 transaction_id 的事务可能产生**内容摘要相同**的文档。

```python
def _verify_formal_doc_txn(prepared, formal_doc) -> bool:
    doc = _read_json(formal_doc)
    return (doc.get("transaction_id") == prepared.transaction_id
            and doc.get("revision") == prepared.target_revision
            and sha256_file(formal_doc) == prepared.formal_doc_target_sha256)
```

### 4.5 完整恢复组合表（唯一动作）

| 组合 | 唯一动作 | 所需证据 |
|------|----------|----------|
| PREPARED 有效 + Candidate 缺失 | **DENY** | 无候选，无法提交 |
| 正式文档新版 + Candidate 缺失 | **INDETERMINATE** | 无法确认提交完整性 |
| COMMITTING + 正式文档新版 + Journal 未提交 | **DENY**（隔离 + 人工）| 提交中断，状态不明 |
| 多个 PREPARED 指向同一正式文档 | **QUARANTINE + DENY** | revision/事务 ID 冲突 |
| Candidate 已替换但目录同步失败 | **INDETERMINATE** | 无法确认持久化 |
| 正式文档摘要相同但 transaction_id 不同 | **DENY** | 事务归属不明 |
| PREPARED 有效 + 正式文档旧版 + Candidate 完整 | **REPLAY** | 三者身份/摘要全匹配 |
| PREPARED 有效 + 正式文档新版 + 匹配 | **确认 COMMITTED** | §4.4 全量匹配 |
| PREPARED 缺失 + Candidate 存在 | **QUARANTINE**（孤儿 candidate）| 无事务意图 |

### 4.6 幂等规则

| 场景 | 规则 |
|------|------|
| 重复请求 | 同 transaction_id 的重复恢复 → 读现状，幂等重判，不二次提交 |
| 重复 Candidate | 以 PREPARED 引用的 candidate_identity 为 canonical，其余隔离 |
| 事务 ID 冲突 | DENY（不同事务共用 txn_id 不可接受）|
| revision 冲突 | DENY |

### 4.7 验收对照

| 验收项 | 结论 |
|--------|------|
| 临时 vs 正式 Candidate 身份区分 | ✅ §4.1 |
| 无覆盖提交原语（O_EXCL 不保护 rename）| ✅ §4.2 |
| PREPARED 全字段绑定 | ✅ §4.3 |
| 不凭 SHA-256/revision 推断归属 | ✅ §4.4 |
| 六个恢复组合补齐 | ✅ §4.5 |
| 幂等规则 | ✅ §4.6 |

**P0-04 结论：PASS（设计级）**，renameat2/link 兜底、MoveFileEx 语义 NOT_VERIFIED。

---

## 5. P0-05｜Journal Quarantine 持久化与崩溃恢复

### 5.1 四对象身份与哈希绑定

| 对象 | 身份 | 哈希绑定 |
|------|------|---------|
| 原 Journal | `journal/<txn_id>.txn`（dev+ino）| journal_sha256 |
| 隔离临时副本 | `quarantine/<txn_id>.txn.qtmp.<random>` | 复制后哈希 = 原哈希 |
| 正式隔离副本 | `quarantine/<txn_id>.txn.qfinal.<random>` | 哈希 = 原哈希 |
| 审计记录 | `audit/quarantine_<txn_id>.json` | 记录三者哈希绑定 |

**绑定规则**：qtmp 与 qfinal 的哈希必须等于原 Journal 哈希；审计记录记录 `{txn_id, original_sha256, qfinal_path, qfinal_sha256, quarantined_at}`。

### 5.2 POSIX 每次 rename 后 fsync 父目录

05.10 §3.6 只 fsync 了隔离副本文件，**未 fsync 父目录**，这是缺陷。修正后每个 rename 都必须对应父目录 fsync：

```
1. copy src → qtmp（写内容）
2. fsync(qtmp)                         # 文件内容持久化
3. rename(qtmp → qfinal)               # 进入 quarantine
4. fsync(quarantine 父目录)            # ★ 目录项持久化
5. write audit 记录
6. fsync(audit 文件) + fsync(audit 父目录)
7. rename(src → src.corrupt)           # 原文件改名（保留，不删）
8. fsync(journal 父目录)               # ★ 目录项持久化
```

### 5.3 Windows 边界

| 步骤 | Windows 机制 | 断电持久性 |
|------|-------------|-----------|
| 文件刷新 | FlushFileBuffers | 保证进程崩溃 |
| 重命名/替换 | MoveFileEx / SetFileInformationByHandle | 原子性保证，断电后目录项可能丢失 |
| 目录持久化 | **无等价 fsync 目录** | **NOT_VERIFIED** |
| 结论 | 仅承诺进程崩溃恢复；断电持久性 DENY/降级 | NOT_VERIFIED |

### 5.4 隔离副本存在时的重新校验（禁止盲目幂等跳过）

05.10 的"qfinal 已存在 → 幂等跳过"是缺陷：副本可能是**半写**或**内容已被替换**。修正：

```python
def _quarantine_resume(txn_id):
    qfinal = _find_qfinal(txn_id)
    if qfinal is None: return "NOT_STARTED"
    # 重新校验，不得直接跳过
    if sha256_file(qfinal) != _original_journal_sha256(txn_id):  # 需审计记录中的原哈希
        return "QUARANTINE_CORRUPT"        # 副本损坏 → DENY + 人工
    if _file_identity(qfinal) 未在审计记录中登记:
        return "INDETERMINATE"             # 身份不明 → DENY
    # 校验原文件当前状态
    if src 仍存在且哈希未变: return "REQUARANTINE"   # 上次改名未完成，重做
    if src 已不存在或已改名为 .corrupt: return "COMPLETE"   # 隔离已完成
    return "DENY"
```

### 5.5 原文件复制中变化 → DENY

复制 src→qtmp 后、rename 前，必须**重新校验 src 哈希**。若 src 在复制期间被修改（哈希变化），则 DENY，不得把旧副本与新原文件混为一体：

```
copy 前：src_hash_before = sha256(src)
copy src → qtmp
src_hash_after = sha256(src)
if src_hash_after != src_hash_before: DENY  # 复制期间原文件变化
```

### 5.6 崩溃窗口恢复动作表

| 崩溃点 | 恢复动作 |
|--------|---------|
| copy 中途崩溃（qtmp 半写）| qtmp 哈希 ≠ 原哈希 → 删除 qtmp，重新开始 |
| fsync(qtmp) 前崩溃 | 同上，重做 |
| rename(qtmp→qfinal) 前崩溃 | qtmp 存在但无 qfinal → 重新 rename（qtmp 哈希已校验）|
| rename 后、fsync 目录前崩溃 | qfinal 存在 → 重新 fsync 目录 |
| audit 写前崩溃 | qfinal 存在 → 补写 audit（需能重建原哈希绑定）|
| audit 写后、rename(src→corrupt) 前崩溃 | src 仍存在 → 重新改名 |
| rename(src→corrupt) 后、fsync 目录前崩溃 | src.corrupt 存在 → 重新 fsync 目录 |

每个崩溃窗口唯一动作，证据不足保留原件并 DENY。

### 5.7 审计写入失败恢复

审计写入失败 → **保留原 Journal，标记 INDETERMINATE，DENY**。不得在无审计记录时继续 rename 原文件（否则隔离动作无归因）。恢复时若发现 qfinal 存在但审计记录缺失，需**能通过 qfinal 内容反推原哈希**才能补审计；否则 DENY + 人工。

### 5.8 伪代码函数契约（列输入/输出/失败语义/持久化责任）

| 函数 | 输入 | 输出 | 失败语义 | 持久化责任 |
|------|------|------|---------|-----------|
| `_copy_file(src, dst)` | 路径 | None | 抛 IOError | 无（调用方 fsync）|
| `_fsync_file(path)` | 路径 | None | 抛 OSError | fsync 文件内容 |
| `_fsync_dir(path)` | 目录路径 | None | 抛 OSError | fsync 目录项 |
| `_rename_noreplace(src, dst)` | 路径 | None | EEXIST → DENY | 无（调用方 fsync 目录）|
| `_quarantine_journal(txn_id)` | txn_id | quarantine 结果 | 任一步失败 → DENY/INDETERMINATE | 负责 §5.2 全部 fsync |
| `_write_audit(record)` | record | None | 抛 OSError → DENY | fsync 文件 + 父目录 |

### 5.9 验收对照

| 验收项 | 结论 |
|--------|------|
| 四对象身份哈希绑定 | ✅ §5.1 |
| 每次 rename 后 fsync 父目录 | ✅ §5.2 |
| Windows 刷新/替换/断电边界 | ✅ §5.3 |
| 隔离副本重新校验不盲目跳过 | ✅ §5.4 |
| 原文件复制中变化 DENY | ✅ §5.5 |
| 崩溃窗口唯一动作 | ✅ §5.6 |
| 审计失败恢复 | ✅ §5.7 |
| 伪代码函数契约完整 | ✅ §5.8 |

**P0-05 结论：PASS（设计级）**，Windows 断电持久性 NOT_VERIFIED。

---

## 6. P0-06｜完整终态快照与清理删除归因

### 6.1 完整终态快照（不能只有摘要字段）

`TerminalStateSnapshot` 必须保存**完整规范化终态 Journal 内容**（或等价的完整可验证证明材料），不能只有摘要字段：

```python
@dataclass(frozen=True)
class TerminalStateSnapshot:
    transaction_id: str
    authorization_id: str
    revision: int
    terminal_state: str                # COMMITTED / ABORTED
    formal_doc_sha256: str             # 正式文档摘要
    source_journal_identity: str       # 源 Journal fstat 身份
    journal_full_content: str          # ★ 完整规范化终态 Journal 内容（非摘要）
    journal_content_sha256: str        # 上述完整内容的 SHA-256（校验用）
    snapshot_version: str
    created_at: str
```

**验证规则**：`sha256(journal_full_content) == journal_content_sha256`，且 `journal_full_content` 中内嵌的 transaction_id/revision/state 与顶层字段一致。完整内容使清理后仍可独立验证"事务确已进入终态"。

### 6.2 快照先于 DELETE_INTENT 持久化

顺序强制：**先持久化 TerminalStateSnapshot（含 fsync 文件 + 父目录），再允许写 DELETE_INTENT**。若快照持久化失败，禁止进入任何删除流程（DENY）。

### 6.3 三种删除结果明确区分

| 结果 | 含义 | 判定 |
|------|------|------|
| `ALREADY_ABSENT` | 恢复时对象已不存在，**不能归因于本次清理** | 无 DELETE_INTENT 或 intent 未达该对象 |
| `REMOVED_BY_THIS_CLEANUP` | 有证据证明本次清理完成删除 | tombstone 存在 + 身份/哈希匹配 + REMOVED 已持久化 |
| `INDETERMINATE` | 无法确认对象消失原因 | 证据不足 |

**关键**：`ALREADY_ABSENT` 不得记为 `REMOVED`。对象不存在但无法证明是本次删除 → `INDETERMINATE` 或 `ALREADY_ABSENT`，**不是** `REMOVED_BY_THIS_CLEANUP`。

### 6.4 删除归因机制：tombstone（先 rename 后记录）

避免直接 `unlink` 后无法归因。改为**先原子 rename 到 cleanup_id 专属 tombstone，再记录移除完成**：

```
1. 持久化 TerminalStateSnapshot（完整终态）
2. 持久化 DELETE_INTENT（含对象身份、哈希、cleanup_id、cleanup_object）
3. 校验对象身份与哈希（与 intent 一致）
4. rename(对象 → tombstone/<cleanup_id>.<object>)     # ★ 非 unlink
5. fsync(对象父目录) + fsync(tombstone 父目录)          # 目录项持久化
6. 持久化 TOMBSTONED（记录 tombstone 路径 + 身份 + 哈希）
7. unlink(tombstone)                                   # 真正删除（此时已有归因）
8. fsync(tombstone 父目录)
9. 持久化 REMOVED_BY_THIS_CLEANUP
```

tombstone 使"删除"这一动作有了中间可归因状态：`TOMBSTONED` 阶段对象仍以 tombstone 形式存在，删除与否可追溯。

### 6.5 Candidate 与 Journal 分别定义顺序

两个对象（`cleanup_object ∈ {candidate, journal}`）**各自独立**走 §6.4 流程，`cleanup_id` 区分对象。字段：`cleanup_id / authorization_id / transaction_id / cleanup_object / cleanup_state / object_identity / object_sha256 / terminal_snapshot / tombstone_path / claimed_at / completed_at / failure_reason / retry_count / cleanup_sequence`。

### 6.6 tombstone 要求

| 项 | 要求 |
|----|------|
| 文件系统 | 与正式对象**同一文件系统**（保证 rename 原子性）|
| 目录 fsync | rename 后 fsync 对象父目录 + tombstone 父目录 |
| 重复清理 | cleanup_id 幂等 |
| 并发清理 | 同一对象同一时间仅一个清理者（受授权锁保护）|
| 恢复接管 | 按 cleanup_state 重判，幂等 |

### 6.7 审计存储限制声明

审计存储仍与事务文件**同 FS、同故障域**，**不具备独立故障保护**。本设计只承诺"审计记录不会被清理流程误删、快照先于删除持久化"，不承诺审计存储自身故障时可靠。

### 6.8 清理状态机（区分归因状态）

```
PENDING → DELETE_INTENT → TOMBSTONED → REMOVED_BY_THIS_CLEANUP → CLEANED
    │            │            │
    │            ├→ FAILED（可重试）
    │            └→ INDETERMINATE（接管重判）
    └──（对象已不存在）→ ALREADY_ABSENT（终止，不记 REMOVED）
```

**状态定义：**

| 状态 | 含义 |
|------|------|
| DELETE_INTENT | 意图已持久化，对象待删 |
| TOMBSTONED | 对象已 rename 到 tombstone，尚未 unlink |
| REMOVED_BY_THIS_CLEANUP | tombstone 已 unlink，REMOVED 已持久化 |
| ALREADY_ABSENT | 恢复时对象已不存在，非本次删除 |
| INDETERMINATE | 无法归因 |
| CLEANED | 两个对象均达终态，审计归档 |

### 6.9 验收对照

| 验收项 | 结论 |
|--------|------|
| 快照保存完整终态内容（非摘要）| ✅ §6.1 |
| 快照绑定全字段 | ✅ §6.1 |
| 快照先于 DELETE_INTENT | ✅ §6.2 |
| 三种结果区分 | ✅ §6.3 |
| tombstone 归因机制 | ✅ §6.4 |
| Candidate/Journal 分别定义 | ✅ §6.5 |
| tombstone 同 FS + fsync + 幂等 | ✅ §6.6 |
| 审计存储限制声明 | ✅ §6.7 |
| 状态机区分归因状态 | ✅ §6.8 |

**P0-06 结论：PASS（设计级）**，tombstone rename/unlink 并发行为 NOT_VERIFIED。

---

## 7. 端到端交叉审查

### 7.1 锁协议三路径互斥原语

| 路径 | 跨进程互斥原语 | 进程内线程互斥 |
|------|---------------|---------------|
| `posix-ofd` | OFD 锁（F_OFD_SETLK）| ProcessMutex |
| `posix-nofd-flock` | flock | ProcessMutex |
| `windows-ntfs` | CreateFileW 独占打开 + LockFileEx | ProcessMutex |

三路径均具备真实、明确、可推演的跨进程互斥原语，无"有锁文件但无锁"状态。

### 7.2 线程与进程生命周期一致性

- ProcessMutex：三路径统一，fork 后重建（§2.3）。
- FD/HANDLE：唯一所有者，release 单路径关闭，fork 后子进程 close 不调 LOCK_UN（§2.2）。
- LeaseRegistry：fork 后清空 + 作废（§2.3）。

### 7.3 路径身份 TOCTOU

- 锁文件替换：句柄身份 vs 路径对象身份比较，不一致 DENY（§3.3）。
- Candidate 替换：无覆盖提交原语（§4.2），PREPARED 绑定 candidate_identity（§4.3）。
- Journal 隔离：四对象身份哈希绑定，复制中变化 DENY（§5.1/§5.5）。

### 7.4 持久化顺序完整性

| 操作 | POSIX 顺序 | Windows 顺序 |
|------|-----------|-------------|
| 文件写入 | write → fsync | write → FlushFileBuffers |
| 原子替换 | rename（+ 父目录 fsync）| MoveFileEx |
| 目录同步 | fsync 父目录 | 无等价，NOT_VERIFIED |
| 断电 | 需父目录 fsync | 无法证明，NOT_VERIFIED |

### 7.5 事务恢复唯一动作

§4.5 完整恢复组合表，每项唯一动作，验证 transaction_id + revision + 摘要全量匹配。

### 7.6 清理归因可证明

终态快照完整（§6.1），tombstone 删除归因（§6.4），三种结果区分（§6.3）。

### 7.7 崩溃边界（进程崩溃 vs 系统崩溃 vs 断电）

| 场景 | POSIX | Windows |
|------|-------|---------|
| 进程崩溃 | OFD/flock 释放 + fd 关闭；journal 已 fsync 可恢复 | HANDLE 关闭；FlushFileBuffers 保证 |
| 系统崩溃 | 需父目录 fsync 才保证 | 无目录 fsync，NOT_VERIFIED |
| 断电 | 需父目录 fsync 才保证 | 无法证明，NOT_VERIFIED |

### 7.8 PASS 项的设计不变量与反例检查

| PASS 项 | 设计不变量 | 前置条件 | 反例检查 |
|---------|-----------|---------|---------|
| P0-01 | 每平台要么有明确锁原语，要么 DENY | 平台支持 flock 或 OFD | 无锁降级（反例：仅有文件无锁）→ 已排除 |
| P0-02 | 子进程不继承有效 lease 或 locked 锁 | register_at_fork 可用 | 子进程复用父租约 → 已排除 |
| P0-03 | 锁对象身份 = 句柄身份 = 路径当前对象身份 | 打开后记录 fstat | 路径替换后仍提交 → 已排除 |
| P0-04 | 提交无覆盖 + 事务身份全量绑定 | renameat2/link 或 CREATE_NEW 可用 | 仅凭 SHA-256 推断归属 → 已排除 |
| P0-05 | 隔离每步可恢复 + rename 后 fsync 目录 | 父目录可 fsync | 副本盲目跳过 → 已排除 |
| P0-06 | 删除可归因 + 快照先于删除 | tombstone 同 FS | 无法归因记 REMOVED → 已排除 |

### 7.9 一致性结论

P0-02～P0-06 只使用 P0-01 定义的锁接口（acquire_lock / assert_lock_owned / release_lock + ProcessMutex），未自行引入另一套锁语义。无跨章节矛盾。

---

## 8. P1 与既有阻断项状态

| 编号 | 状态 |
|------|------|
| P1-01 锁文件替换/TOCTOU | 本轮 P0-03 直接处理，**设计级闭合**，运行时仍 NOT_VERIFIED |
| P1-02 analyze_file 行为等价 | NOT_TESTED |
| P1-03 A/B 三文件合并 | 保持原状态，不执行 |
| P0-BLOCK-1 存储事务实现+故障注入 | 未解决 |
| P0-BLOCK-2 analyze_file 一致性 | 未解决 |
| P1-BLOCK-3 A/B 三文件分叉 | 未解决 |
| F1 / P0.8 / F2 | 不解除 |
| IMPLEMENT-01 | STOP |

> 注：P1-01 的"锁文件替换与 TOCTOU"在本轮 P0-03 中被直接处理（因为独立审查认定它是 P0-01 的直接依赖）。但**运行时验证仍 NOT_VERIFIED**，不因设计级闭合而把 P1-01 整体标记为 PASS。

---

## 9. IMPLEMENT-01 准入结论

**PASS-WITH-ISSUES（维持阻断）。**

六项 P0 已闭环：POSIX 锁降级明确 flock 原语、fork 生命周期闭环、锁文件身份与 TOCTOU、Candidate 无覆盖提交 + 事务身份全量绑定、Journal 隔离持久化、终态快照 + tombstone 删除归因。

但锁、事务、隔离、清理均为**设计级契约**，真实实现前 NOT_VERIFIED / NOT_TESTED。三个阻断项未解决。

**本工单不授权实现、不授权测试、不解除 IMPLEMENT-01 STOP。F1/P0.8/F2 未解除。最终原则：锁基础先行、身份与事务绑定、恢复动作唯一、删除归因可证明、证据不足即 DENY、未经授权不实现。**

---

## 10. 交付物索引

| 文件 | 内容 |
|------|------|
| `USIT-STAGE-B-DESIGN-05.11.md` | 本文 |
| `USIT-STAGE-B-DESIGN-05.11-ISSUES.md` | 问题清单 |
| `USIT-STAGE-B-DESIGN-05.11-DIFF.md` | DESIGN-05.10 → 05.11 差异 |
| `USIT-STAGE-B-DESIGN-05.11-EXECUTION-REPORT.md` | 执行报告 |
| `USIT-STAGE-B-DESIGN-05.11-HASH-MANIFEST.md` | 哈希清单 |
| `USIT-STAGE-B-DESIGN-05.11-HASH-RECEIPT.txt` | 哈希收据 |
| `USIT-STAGE-B-DESIGN-05.11-交付物.zip` | 六份文档打包 |
