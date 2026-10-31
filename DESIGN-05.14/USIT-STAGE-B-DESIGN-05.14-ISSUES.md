# USIT-STAGE-B-DESIGN-05.14-ISSUES｜五项问题清单及剩余风险

> 依据：DESIGN-05.13 独立审查结论
> 性质：READ-ONLY DESIGN REVISION ONLY
> 日期：2026-10-31

## 一、五项问题清单

### P0-02｜Fork 生命周期与同步对象安全

- **原缺陷**：05.13 以 `_fork_fd_snapshot` 是否为空识别外部 fork，但 `os.register_at_fork` 的回调同样被外部 `os.fork()` 触发，标志失效。
- **修复方案**：改为"无论何种 fork，child 回调无条件安全重置；外部 fork 的代价由子进程后续操作 self-check 承担"。见主文档 §1.2/§1.7。
- **剩余风险**：register_at_fork 的 before/child 回调执行时序、异常行为，未实测（NOT_TESTED）。外部代码绕过 API 直接使用裸 fd 属"调用方不遵守 API"，本库不支持。

### P0-03｜锁对象稳定性与跨进程互斥

- **原缺陷**："永久锁文件 + 禁 unlink"无法对抗同 UID 恶意进程；`fstat(fd)` 只证明 FD 指向对象，不证明路径仍指向同一对象。
- **修复方案**：句柄绑定（dirfd/openat/fd）全程不用路径重开，消除"验证 A、操作 B"；诚实承认本地 FS 无法防御恶意同 UID 进程，靠身份异常检测拒绝。见主文档 §2.5/§2.6/§2.9。
- **剩余风险**：同 UID 恶意进程的 inode 分裂（§2.2）是文件系统锁模型的固有能力边界，**NOT_VERIFIED**，无法在本设计内消除。彻底解决需跨信任边界锁服务（§2.4 路径一，NOT_IMPLEMENTED）。macOS/BSD O_NOFOLLOW 与目录 fsync 语义 NOT_VERIFIED。

### P0-04｜正式文档版本化发布与真正 CAS

- **原缺陷**："目标存在时无覆盖发布失败"不能更新已存在文档；"读 revision→比较→写入"非原子，会丢更新。
- **修复方案**：不可变版本文件 + 原子指针；POSIX 用 `link()` 占有标记实现真正 CAS（EEXIST 原子失败）。见主文档 §3.2/§3.4/§3.5。
- **剩余风险**：Windows 的 `ReplaceFileW` + `dwShareMode=0` 组合语义 NOT_VERIFIED；目录 fsync 在 macOS/Windows 上 NOT_VERIFIED；数据库 CAS（方案 B）NOT_IMPLEMENTED。

### P0-05｜源对象并发写入与 Source Retire

- **原缺陷**：最后一次 sha256 与 rename 之间有窗口；rename 不能阻止已持可写 FD 的进程继续写同一 inode。
- **修复方案**：三层结论（可证明/只能检测/无法证明）；采用"源不可变（sealed）+ 复制后只操作副本"；quarantine 与 Source Retire 独立事务。见主文档 §4.3/§4.4。
- **剩余风险**：纯哈希比对（等级 B）对"rename 后旧 FD 写入"（§4.5 W5）无能为力，必须源不可变或排他才能消除。文件锁协作、sealed 语义 NOT_TESTED。

### P0-06｜Tombstone 事务归属与删除恢复

- **原缺陷**：tombstone 唯一性/事务绑定/不可替换/磁盘增长未定义；DELETE_INTENT/UNLINK_INTENT 被误当删除完成证据。
- **修复方案**：唯一命名（cleanup_id + O_EXCL）+ 内容元数据绑定事务 + 不可替换边界 + 归档协议；REMOVED 持久化前不删 tombstone；无法归因 → INDETERMINATE。见主文档 §5.3/§5.4/§5.6。
- **剩余风险**：rename/fsync 的崩溃恢复语义（§5.7 D2/D3/D7）依赖 POSIX 定义行为，NOT_TESTED；归档运行时行为 NOT_TESTED。

## 二、跨章节剩余风险汇总

1. **恶意同 UID 进程**（P0-03/P0-05 共有）：本地 FS 无强制锁，无法防御，只能检测+拒绝。
2. **平台语义差异**：macOS/BSD/Windows 的 O_NOFOLLOW、目录 fsync、ReplaceFileW、共享模式语义均 NOT_VERIFIED。
3. **崩溃恢复依赖 POSIX 定义**：rename/fsync/flock 的崩溃语义按 POSIX 标准推演，未实测。
4. **无运行时验证**：本工单 READ-ONLY，所有行为均为设计级推演，NOT_TESTED。

## 三、结论

五项缺陷均在设计层面给出了闭合的协议与推演证据，但均带有明确标注的 NOT_TESTED/NOT_VERIFIED 项，不伪造确定性。整体 = **PASS-WITH-ISSUES（设计级）**。
