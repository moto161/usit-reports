# USIT-STAGE-B-DESIGN-05.15-ISSUES｜四项问题清单及剩余风险

> 依据：ChatGPT 对 DESIGN-05.14 的独立复审（四项核心缺陷）
> 性质：READ-ONLY DESIGN REVISION ONLY · 日期：2026-10-31

## 一、四项问题清单

### P0-02｜Fork 生命周期与死锁证明

- **原缺陷**：05.14 §1.4 未建立全局锁顺序，未处理"线程 A 持 Registry 锁、线程 B 持 ForkGate 锁并触发 fork"的死锁反例。
- **修复**：全局锁升序获取 + 禁止持锁 fork + before 回调无锁快照。见主文档 §2.3/§2.4。
- **剩余风险**：多线程 fork 的 OS 级回调行为未实测（NOT_VERIFIED）；外部代码绕过 API 用裸 fd 属"不支持"。

### P0-04｜废弃假 CAS

- **原缺陷**：05.14 §3.4/§3.5 的 `link(current, current.lock.<txn_id>)` 是假 CAS——每事务独立锁文件名，T1/T2 可同时 link 成功再分别 rename，丢失更新。
- **修复**：废弃 link 方案，改用单一固定锁文件 `.commit-lock` + flock(LOCK_EX) 临界区，revision 检查与发布串行化。见主文档 §1。
- **剩余风险**：Windows 的 LockFileEx + ReplaceFileW 原子性 NOT_VERIFIED；目录 fsync 在 macOS/Windows NOT_VERIFIED。

### P0-05｜Sealed 源对象模型重构

- **原缺陷**：05.14 把 sealed/append-only/O_EXCL 当不可变，但 seal 后旧可写 FD 仍可写，rename 不撤销旧 FD。
- **修复**：源对象六状态机 + SEALED 持久化证据 + seal 后重新验证；quarantine 只操作副本、Source Retire 独立事务。见主文档 §3。
- **剩余风险**：恶意同 UID / 旧 FD 强写无法强制阻止（除非 chattr +i 需特权），只能检测+拒绝，NOT_VERIFIED；mmap 写入需 fsync 后重验。

### P0-06｜Tombstone 对象模型

- **原缺陷**：05.14 把"源文件 rename 到 tombstone 路径"与"tombstone 内容含元数据"混为同一对象。
- **修复**：分离源对象 / 隔离对象 / Tombstone 元数据 / Journal 四个对象；Tombstone 元数据是独立文件，不承载源内容。见主文档 §4。
- **剩余风险**：rename/fsync 崩溃恢复语义 NOT_TESTED；归档运行时行为 NOT_TESTED。

## 二、跨章节剩余风险

1. **恶意同 UID 进程**（P0-05 旧 FD 强写、P0-03 继承的 inode 分裂）：本地 FS 无强制锁，无法防御，只能检测+拒绝。
2. **平台语义差异**：Windows LockFileEx/ReplaceFileW、macOS chflags、目录 fsync 均 NOT_VERIFIED。
3. **崩溃恢复依赖 POSIX 定义**：rename/fsync/flock 崩溃语义按标准推演，未实测。
4. **无运行时验证**：READ-ONLY，全部为设计级推演，NOT_TESTED。

## 三、结论

四项缺陷均在设计层面闭合（P0-02/P0-04/P0-06 达 PASS，P0-05 达 PASS-WITH-ISSUES），但均带明确 NOT_TESTED/NOT_VERIFIED 项，不伪造确定性。整体 = **PASS-WITH-ISSUES（设计级）**。
