# USIT-STAGE-B-DESIGN-05.22-ISSUES｜问题与剩余风险

> 阶段 B · DESIGN-05.22 · 未解决问题与剩余风险清单
> 生成：2026-10-31（GMT+8）

---

## 未解决问题（按优先级）

| 编号 | 优先级 | 问题 | 影响范围 | 触发条件 | 现有证据 | 缺失证明 | 阻断状态 |
|------|--------|------|---------|---------|---------|---------|---------|
| ISS-01 | P0 | 第三方 C 扩展直接 FD/fork 无法拦截 | P0-02 | 依赖直接调 libc 的 C 扩展 | FD 入口矩阵 | 底层 syscall 拦截 | **阻断**（UNSUPPORTED/DENY）|
| ISS-02 | P0 | CPython 多线程 fork 运行时未验证 | P0-02 | 多线程受控 fork | CPython 官方文档 | 运行时验证 | 非阻断（NOT_TESTED）|
| ISS-03 | P1 | 恶意同 UID 锁替换 | P0-04 | 同 UID 恶意进程 | 锁身份校验设计 | 强制防同 UID | 非阻断（NOT_VERIFIED）|
| ISS-04 | P1 | Windows 目录项断电持久性 | P0-04/P0-06 | Windows 平台 | FlushFileBuffers 文档 | 官方目录持久保证 | 非阻断（NOT_VERIFIED）|
| ISS-05 | P0 | Source Retire 无强制不可变原语 | P0-05 | 本地 FS 退役 | chattr(1) 文档 | 强制不可变 | **阻断**（DENY）|
| ISS-06 | P1 | ReplaceFileW 1176/1177 中间态 | P0-06 | ReplaceFileW 部分失败 | Microsoft Learn | 运行时中间态验证 | 非阻断（NOT_VERIFIED）|
| ISS-07 | P2 | 嵌入/子解释器/posix_spawn fork | P0-02 | 嵌入宿主/子解释器 | CPython 约束 | 该场景安全证明 | 非阻断（UNSUPPORTED/DENY）|

## 剩余风险（设计级，非阻断）

1. **child 未知 FD 泄漏**：child 对第三方创建的、不在快照中的继承 FD 不主动 close，可能泄漏。可接受权衡。
2. **NOT_DURABLE/RECOVERY_REQUIRED 恢复依赖运维纪律**。
3. **Windows ReplaceFileW 同卷约束**：备份/被替换/替换文件必须同卷。

## 已闭合问题（本轮修复）

| 编号 | 问题 | 修复位置 |
|------|------|---------|
| B22-01 | `_fork_coord_lock` 双重释放 | §1.2（唯一释放者 fork_entry）|
| B22-02 | 快照存储/所有权/生命周期未定义 | §1.4（fork_epoch 单槽位协议）|
| B22-03 | DURABLE 与 DURABLE（进行中）混用 | §2.2（七态状态机）|
| B22-04 | ZIP 回执缺真实哈希/核验 | §6（真实解压+比对）|
| B22-05 | 源码未修改缺本轮前后基线 | §6.4（本轮前后基线采集）|

---

## 结论

无伪造 PASS。ISS-01、ISS-05 为阻断项，均以 DENY/UNSUPPORTED 诚实拒绝。整体 PASS-WITH-ISSUES。

---