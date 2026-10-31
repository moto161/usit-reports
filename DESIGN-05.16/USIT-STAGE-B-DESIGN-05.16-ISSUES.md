# USIT-STAGE-B-DESIGN-05.16-ISSUES｜四项问题清单及剩余风险

> 依据：ChatGPT 对 DESIGN-05.15 的独立复审（四项剩余缺陷）
> 性质：READ-ONLY DESIGN REVISION ONLY · 日期：2026-10-31

## 一、四项问题清单

### P0-02｜Fork 生命周期与 FD 快照一致性

- **原缺陷**：05.15 无锁枚举 FD Registry 有竞态（close/复用 FD 数字导致 child 误关）；第三方 C fork 不被 register_at_fork 覆盖。
- **修复**：FD generation/version 绑定 + fork 前持锁冻结协调；第三方 C fork 明确 DENY。见主文档 §1。
- **剩余风险**：fork 协调运行时行为 NOT_TESTED；第三方 C fork 无法拦截（调用方责任）。

### P0-04｜真正基线 CAS 与锁对象身份

- **原缺陷**：Candidate 未绑定父版本；固定锁文件 inode 替换导致双持。
- **修复**：Candidate 绑定 base_revision/base_txn_id/base_hash + 五条件校验；锁对象身份校验（fstat + fstatat + openat + O_NOFOLLOW）。见主文档 §2。
- **剩余风险**：恶意同 UID 锁替换 NOT_VERIFIED；Windows 原子原语 NOT_VERIFIED。

### P0-05｜Sealed 强制不可变与 Source Retire

- **原缺陷**：最终哈希校验与 rename 之间仍有竞态；哈希通过/rename 成功/SEALED 记录都不等于不可变。
- **修复**：明确只有文件系统不可变属性（Linux chattr +i）能强制阻止旧 FD 写；无强制原语平台 Source Retire 默认 DENY。见主文档 §3。
- **剩余风险**：macOS chflags uchg / Windows 只读属性非强制，均 DENY；同 UID 恶意进程可 chattr -i（若有特权）NOT_VERIFIED。

### P0-06｜Tombstone 原子发布与归档

- **原缺陷**：O_EXCL 创建后逐步写入当原子发布；归档混用 O_EXCL 与 rename。
- **修复**：临时文件 + 校验 + 无覆盖发布原语（renameat2 NOREPLACE / link EEXIST）；归档区分 O_EXCL 与 rename。见主文档 §4。
- **剩余风险**：Windows 无覆盖发布 NOT_VERIFIED；rename/fsync 崩溃语义 NOT_TESTED。

## 二、未解决问题清单（P0/P1/P2 分级）

| 编号 | 问题 | 级别 | 影响范围 | 触发条件 | 是否阻断 |
|------|------|------|---------|---------|---------|
| U-1 | 恶意同 UID 进程绕过协议（锁替换/旧 FD 写/chattr -i）| P0 | 锁、源、tombstone | 同 UID 恶意进程 | 不阻断设计，标 NOT_VERIFIED |
| U-2 | 第三方 C fork 绕过 atfork | P1 | fork 生命周期 | C 扩展直接 fork | 不阻断（DENY + 文档化）|
| U-3 | Windows 无 renameat2 NOREPLACE / flock 等价原语 | P1 | P0-04/P0-06 提交与发布 | Windows 平台 | 不阻断（NOT_VERIFIED）|
| U-4 | macOS/Windows 无强制不可变属性 | P1 | P0-05 Source Retire | 非 Linux 平台 | 不阻断（默认 DENY）|
| U-5 | 数据真正销毁（安全擦除）| P2 | P0-06 PURGED | 需覆写 | 不阻断（NOT_IMPLEMENTED）|
| U-6 | 崩溃恢复语义未实测 | P1 | 全协议 | 进程/系统崩溃 | 不阻断（NOT_TESTED）|

## 三、结论

四项缺陷均在设计层面闭合（P0-02/P0-06 达 PASS，P0-04/P0-05 达 PASS-WITH-ISSUES），但均带明确 NOT_TESTED/NOT_VERIFIED 项，不伪造确定性。整体 = **PASS-WITH-ISSUES（设计级）**。
