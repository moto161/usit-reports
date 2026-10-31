# USIT-STAGE-B-DESIGN-05.20-ISSUES｜问题与剩余风险

> 阶段 B · DESIGN-05.20 · 未解决问题与剩余风险清单
> 生成：2026-10-31（GMT+8）

---

## 未解决问题（按优先级）

| 编号 | 优先级 | 问题 | 影响范围 | 触发条件 | 现有证据 | 缺失证明 | 阻断状态 |
|------|--------|------|---------|---------|---------|---------|---------|
| ISS-01 | P0 | 第三方 C 扩展直接 FD 操作 / fork 无法被 Python 层拦截 | P0-02 Fork | 依赖直接调 libc open/fork 的 C 扩展 | FD 入口覆盖矩阵（§1.6）| 底层 syscall 拦截 | **阻断**（该环境 UNSUPPORTED/DENY）|
| ISS-02 | P0 | CPython 多线程 fork 运行时行为未验证 | P0-02 Fork | 多线程环境受控 fork | CPython 官方文档语义 | 实际运行时验证 | 非阻断（NOT_TESTED + DENY 边界）|
| ISS-03 | P1 | 恶意同 UID 进程锁替换 | P0-04 CAS | 同 UID 恶意进程 unlink/recreate 锁 | 锁身份校验设计 | 强制防同 UID 机制 | 非阻断（NOT_VERIFIED）|
| ISS-04 | P1 | Windows 目录项断电持久性 | P0-04/P0-06 | Windows 平台提交/tombstone 发布 | FlushFileBuffers 文档语义 | 官方目录项持久保证 | 非阻断（NOT_VERIFIED）|
| ISS-05 | P0 | Source Retire 无强制不可变原语 | P0-05 | 本地文件系统退役 | chattr(1) 官方文档 | 同 UID+root 强制不可变 | **阻断**（保持 DENY）|
| ISS-06 | P1 | ReplaceFileW 1176/1177 部分移动中间态 | P0-06 Tombstone | ReplaceFileW 部分失败 | Microsoft Learn 官方契约 | 运行时中间态验证 | 非阻断（INDETERMINATE）|
| ISS-07 | P2 | 嵌入 Python / 子解释器 fork | P0-02 | 嵌入宿主或子解释器 | CPython 约束 | 该场景安全证明 | 非阻断（DENY）|

## 剩余风险（设计级，非阻断）

1. **child 未知 FD 泄漏**：child 对第三方创建的、不在快照中的继承 FD 不主动 close，可能泄漏 FD（宁可泄漏，不误关）。可接受权衡，记录审计。
2. **NOT_DURABLE / RECOVERY_REQUIRED 的恢复依赖运维纪律**：设计无法完全消除人为误操作。
3. **Windows ReplaceFileW 同卷约束**：备份/被替换/替换文件必须同卷，跨卷须拒绝，实现时强制校验。

## 已闭合问题（本轮修复）

| 编号 | 问题 | 修复位置 |
|------|------|---------|
| FIX-01 | 准入"检查 Gate 后、计数前"竞态窗口 | §1.2（原子准入，检查+计数同临界区）|
| FIX-02 | 状态机语义过粗（回执未持久误当不确定）| §2.3（拆出 COMMIT_DURABLE_AUDIT_PENDING）|
| FIX-03 | 步骤编号不一致（13 vs 16）| §2.5（统一 S1~S16）|
| FIX-04 | 1176 路径语义错误（"以原目标名"）| §3.3（纠正为"在其原始名"）|
| FIX-05 | 错误码推断文件状态 | §3.4（禁止推断，强制重读）|

---

## 结论

无伪造 PASS。ISS-01、ISS-05 为阻断项，均以 DENY/UNSUPPORTED 诚实拒绝。整体 PASS-WITH-ISSUES。

---