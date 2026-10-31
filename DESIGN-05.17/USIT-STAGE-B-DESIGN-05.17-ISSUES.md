# USIT-STAGE-B-DESIGN-05.17-ISSUES｜问题清单及剩余风险

> 依据：ChatGPT 对 DESIGN-05.16 的独立复审（FAIL）
> 性质：READ-ONLY DESIGN REVISION ONLY · 日期：2026-10-31

## 一、四个缺陷修复清单

### P0-02（阻断项）｜Fork 冻结窗口

- **原缺陷**：05.16 §1.3 释放 `_fd_lock` 后、fork 前存在未冻结窗口，快照过期导致 child 误关。
- **修复**：持锁到 fork（`os._fork()` 避重入），冻结覆盖真正 fork 窗口。见主文档 §1.2。
- **剩余风险**：受控 fork 运行时行为 NOT_TESTED；第三方 C fork 无法拦截（DENY + 文档化）。

### P0-05（阻断项）｜Sealed 退役竞态

- **原缺陷**：05.16 §3.4 "chattr -i 临时解除 → rename"，解除后旧 FD 可写，竞态重现。
- **修复**：全程 +i 保持，退役仅 rename 移除命名空间引用（+i 不限制 rename）。见主文档 §3.2/§3.4。
- **剩余风险**：有 CAP_LINUX_IMMUTABLE 的 root/同 UID 特权进程可 -i，NOT_VERIFIED；macOS/Windows/无特权/网络 FS 默认 DENY。

### P0-04（补齐）｜Candidate 崩溃持久化顺序

- **原缺陷**：05.16 缺文件内容与目录项持久化的分步崩溃矩阵。
- **修复**：fsync 文件 → rename → fsync 目录严格顺序 + 每崩溃点恢复规则。见主文档 §2.2/§2.4。
- **剩余风险**：Windows 持久性 NOT_VERIFIED。

### P0-06（补齐）｜Windows Tombstone 占位崩溃恢复

- **原缺陷**：05.16 Windows CREATE_NEW 占位 + ReplaceFileW 的崩溃恢复未定义。
- **修复**：占位/有效内容区分 + 恢复判定（可继续/可重试/保留现场/INDETERMINATE）。见主文档 §4.3/§4.4。
- **剩余风险**：Windows ReplaceFileW/FlushFileBuffers 崩溃语义 NOT_TESTED。

## 二、未解决问题清单（P0/P1/P2 分级）

| 编号 | 问题 | 级别 | 影响范围 | 触发条件 | 现有证据 | 缺失证明 | 阻断状态 |
|------|------|------|---------|---------|---------|---------|---------|
| U-1 | 恶意同 UID 进程绕过协议 | P0 | 锁/源/tombstone | 同 UID 恶意进程 | 身份检测 + 边界 | 强制防御 | 不阻断设计，NOT_VERIFIED |
| U-2 | 特权进程撤销 +i | P0 | Source Retire | CAP_LINUX_IMMUTABLE | 威胁模型排除 | 强制防御 | 不阻断设计，NOT_VERIFIED |
| U-3 | 第三方 C fork 绕过 atfork | P1 | fork 生命周期 | C 直接 fork | DENY + 文档化 | 拦截 | 不阻断（DENY）|
| U-4 | Windows 无 renameat2/flock 等价原语 | P1 | P0-04/P0-06 | Windows | 独立协议 | 原子性证明 | 不阻断，NOT_VERIFIED |
| U-5 | macOS/Windows 无强制不可变 | P1 | Source Retire | 非 Linux | 默认 DENY | 强制原语 | 不阻断（DENY）|
| U-6 | 数据真正销毁（安全擦除）| P2 | PURGED | 需覆写 | NOT_IMPLEMENTED | 擦除协议 | 不阻断 |

## 三、结论

两个阻断项（P0-02/P0-05）已在设计级闭合；P0-04/P0-06 已补齐。所有 NOT_TESTED/NOT_VERIFIED 项保留。整体 = **PASS-WITH-ISSUES（设计级）**。
