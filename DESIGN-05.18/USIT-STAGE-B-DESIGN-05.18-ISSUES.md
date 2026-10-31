# USIT-STAGE-B-DESIGN-05.18-ISSUES｜问题清单及剩余风险

> 依据：ChatGPT 对 DESIGN-05.17 的独立复审（FAIL）
> 性质：READ-ONLY DESIGN REVISION ONLY · 日期：2026-10-31

## 一、四个缺陷修复清单

### P0-02（阻断）｜Fork 协议

- **原缺陷**：05.17 用 `os._fork()` 绕过 CPython 生命周期后执行 Python，未证明运行时安全。
- **修复**：改回 `os.fork()`（保留 CPython 生命周期），before 钩子无锁消除重入死锁；主线程约束 + 子解释器 DENY。见主文档 §1。
- **结论**：DESIGN_PARTIAL——协议层可证明，CPython 多线程 fork 运行时 NOT_TESTED。

### P0-05（阻断）｜Source Retire

- **原缺陷**：05.17 声称 `+i` 保持下可 rename，被 chattr(1) 权威文档双重推翻（+i 阻止 rename 且不阻旧 FD 写）。
- **修复**：废弃 +i 方案，Source Retire 在所有本地平台 DENY。见主文档 §3。
- **结论**：**DENY**——本地 FS 无法证明源内容稳定，必须 DENY，不能 PASS。

### P0-04（补齐）｜Candidate 持久化

- **原缺陷**：05.17 缺目录项 fsync。
- **修复**：补 fsync(versions_dirfd)/fsync(prepared_dirfd)/fsync(parent_dirfd) + 分步崩溃矩阵。见主文档 §2。
- **结论**：DESIGN_PROVEN（POSIX）/ NOT_VERIFIED（Windows）。

### P0-06（补齐）｜Windows Tombstone

- **原缺陷**：05.17 未分析 ReplaceFileW 部分失败。
- **修复**：错误状态矩阵 + 按内容判定 + 恢复状态机。见主文档 §4。
- **结论**：DESIGN_PARTIAL。

## 二、未解决问题清单（P0/P1/P2 分级）

| 编号 | 问题 | 级别 | 影响 | 触发 | 证据 | 缺失证明 | 阻断 |
|------|------|------|------|------|------|---------|------|
| U-1 | Source Retire 无法证明安全（+i 被推翻）| **P0** | 源退役 | 任何本地平台 | chattr(1) man page | 强制不可变原语 | **阻断** |
| U-2 | os._fork 后 CPython 运行时未重置 | **P0** | fork | 绕过生命周期 | CPython 语义 | 运行时验证 | 阻断（已改 os.fork）|
| U-3 | 恶意同 UID 进程 | P0 | 锁/源/tombstone | 同 UID | 身份检测 | 强制防御 | 不阻断（NOT_VERIFIED）|
| U-4 | 特权进程撤销 +i | P1 | Source Retire | CAP_LINUX_IMMUTABLE | 威胁排除 | 强制防御 | 不阻断 |
| U-5 | 第三方 C fork | P1 | fork | C 直接 fork | DENY | 拦截 | 不阻断（DENY）|
| U-6 | Windows 目录持久化（FlushFileBuffers）| P1 | P0-04/P0-06 | Windows | NOT_VERIFIED | 持久化证明 | 不阻断 |
| U-7 | ReplaceFileW 部分失败中间态 | P1 | tombstone | Windows | 错误矩阵 | 具体行为 | 不阻断 |
| U-8 | 数据真正销毁（安全擦除）| P2 | PURGED | 需覆写 | NOT_IMPLEMENTED | 擦除协议 | 不阻断 |

## 三、结论

两个阻断项（P0-02/P0-05）中：P0-02 降级为 DESIGN_PARTIAL（改回 os.fork 后协议层可证明，但运行时 NOT_TESTED）；**P0-05 诚实结论为 DENY（+i 方案被权威文档推翻，Source Retire 无法证明安全）**。P0-04/P0-06 补强到位但 Windows 持久性 NOT_VERIFIED。

**由于 P0-05 阻断项本轮无法闭合（DENY），整体 = FAIL。**
