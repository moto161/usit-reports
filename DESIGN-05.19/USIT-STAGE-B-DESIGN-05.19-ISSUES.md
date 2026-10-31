# USIT-STAGE-B-DESIGN-05.19-ISSUES｜问题与剩余风险

> 阶段 B · DESIGN-05.19 · 未解决问题与剩余风险清单
> 生成：2026-10-31（GMT+8）

---

## 未解决问题（按优先级）

| 编号 | 优先级 | 问题 | 影响范围 | 触发条件 | 现有证据 | 缺失证明 | 阻断状态 |
|------|--------|------|---------|---------|---------|---------|---------|
| ISS-01 | P0 | 第三方 C 扩展直接 fork/open/close 无法被 Python 层拦截 | P0-02 Fork | 项目依赖直接调 libc fork/open 的 C 扩展 | 本库 FD 入口全覆盖矩阵（§1.4）| 无法拦截底层 syscall | **阻断**（该环境 UNSUPPORTED/DENY）|
| ISS-02 | P0 | CPython 多线程 fork 运行时行为未验证 | P0-02 Fork | 多线程环境下受控 fork | CPython 官方文档语义（§1.5）| 实际运行时验证 | 非阻断（已标 NOT_TESTED + DENY 边界）|
| ISS-03 | P1 | 恶意同 UID 进程锁替换 | P0-04 CAS | 同 UID 恶意进程 unlink/recreate 锁文件 | 锁身份校验设计 | 强制防同 UID 机制 | 非阻断（威胁模型排除，NOT_VERIFIED）|
| ISS-04 | P1 | Windows 目录项断电持久性 | P0-04/P0-06 | Windows 平台提交/tombstone 发布 | FlushFileBuffers 文档语义（§2.7/§3.5）| 官方保证目录项持久 | 非阻断（标 NOT_VERIFIED）|
| ISS-05 | P0 | Source Retire 无强制不可变原语 | P0-05 | 任何本地文件系统退役操作 | chattr(1) 官方文档（§4.1）| 同 UID+root 威胁下强制不可变 | **阻断**（保持 DENY）|
| ISS-06 | P1 | ReplaceFileW 1176（未指定备份）/1177 部分移动中间态 | P0-06 Tombstone | Windows ReplaceFileW 部分失败 | Microsoft Learn 官方契约（§3.2）| 运行时中间态验证 | 非阻断（INDETERMINATE 判定）|
| ISS-07 | P2 | 嵌入式 Python / 子解释器 fork | P0-02 | 嵌入式宿主或子解释器场景 | CPython 约束（§1.6）| 该场景安全证明 | 非阻断（DENY）|

## 剩余风险（设计级，非阻断）

1. **child 未知 FD 泄漏**：child 对第三方创建的、不在快照中的继承 FD 不主动 close，可能泄漏 FD（宁可泄漏，不误关）。属可接受的权衡，记录审计。
2. **NOT_DURABLE 态的恢复复杂度**：fsync 失败后的重试/人工确认流程依赖运维纪律，设计无法完全消除人为误操作。
3. **Windows ReplaceFileW 同卷约束**：备份/被替换/替换文件必须同卷，跨卷场景必须拒绝，需在实现时强制校验。

## 已闭合问题（本轮修复）

| 编号 | 问题 | 修复位置 |
|------|------|---------|
| FIX-01 | 快照后释放锁导致未门禁窗口 | §1.3（FREEZING 态覆盖 fork 窗口）|
| FIX-02 | 五条件未逐项列谓词 | §2.2（P1~P5 完整谓词）|
| FIX-03 | 缺持久化故障状态机 | §2.4（五态状态机）|
| FIX-04 | 1175/1176/1177 未逐码区分 | §3.2/§3.3（官方矩阵）|
| FIX-05 | P0-05 边界表述过度断言 | §4.2（区分"未证明"与"不存在"）|

---

## 结论

无伪造 PASS。ISS-01、ISS-05 为阻断项（分别对应 P0-02 的第三方 C fork 与 P0-05 的 Source Retire），均以 DENY/UNSUPPORTED 诚实拒绝，不宣称通用安全。

---