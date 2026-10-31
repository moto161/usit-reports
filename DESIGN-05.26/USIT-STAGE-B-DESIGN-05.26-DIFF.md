# USIT-STAGE-B-DESIGN-05.26 与 05.25 差异（DIFF）

> 仅列 05.25 → 05.26 实质性差异。

## 1. P0-02｜Fork 异常闭合（B26-01）

**05.25**：`_safe_restore_gate()` 内部 try/except 吞异常，仅设 `_gate_failed=True` 后返回正常业务路径（fail-open）。

**05.26**：
- `_restore_gate_fail_closed()` 返回布尔值，失败 → `_fork_allowed=False` + 抛 `ForkCorrupt` + 终止业务路径；
- Gate 状态机新增 **RESTORE_FAILED 终态**（只允许终止/永久拒绝，禁止业务/fork/FD 操作）；
- 明确四动作分离：锁释放 ≠ Gate 恢复 ≠ 业务继续 ≠ 再次 fork；
- 线程准入 fail-closed：无法证明单线程即永久禁用 fork。

## 2. P0-02｜身份唯一性（B26-02）

**05.25**：`snapshot_id = (process_instance_id, fork_epoch)`，被兄弟进程碰撞反例推翻。

**05.26**：
- 新增 **`fork_event_id`**（父进程 L0 内生成，高熵随机 + fork_epoch），`snapshot_id = fork_event_id`；
- 兄弟进程因随机分量不同而结构上不碰撞（概率性）；
- 身份层级扩至九字段（增 fork_event_id、parent_snapshot_id）；
- 唯一性精确分级：局部唯一（结构性）/ 实例内唯一（随机+结构）/ 跨重启唯一（不保证）/ 全局唯一（不宣称）；
- 明确"UUID 不重复"非数学证明，是概率性唯一。

## 3. P0-04｜S1~S16 矩阵（B26-03）

**05.25**：S1~S16 矩阵引用 05.24 旧矩阵，未重写。

**05.26**：全新重写，每步列出 16 项（前置状态/磁盘对象/syscall/崩溃位置/内存状态/磁盘状态/可见性/内容持久/文件名持久/txn-revision/身份哈希/必读对象/允许恢复/禁止恢复/最终状态/不变量）；安全门扩至 13 项；四目录 fsync 协议覆盖 EINTR/EIO/ENOSPC。

## 4. Windows（B26-04）

**05.25**：Windows 仅一句话带过。

**05.26**：§4 系统复核——ReplaceFileW 返回码（官方 VERIFIED_FACT）、部分失败中间态（NOT_VERIFIED）、目录持久化（NOT_VERIFIED）、不得套用 POSIX 推论。

## 5. 交付（B26-05 / B26-06）

- ZIP-RECEIPT 文件名由 `.txt` 改为 `.md`（工单要求 `ZIP-RECEIPT.md`）；
- 交付文件清单 12 份（同 05.25）。

## 6. 术语统一

证据等级标注；Gate 五态 + RESTORE_FAILED；身份九字段；11 态状态机；13 项安全门；四目录 fsync 统一协议。
