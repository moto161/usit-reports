# USIT-STAGE-B-DESIGN-05.23 与 05.22 差异（DIFF）

> 本文件仅列 05.22 → 05.23 的实质性差异，不重述未改动内容。

## 1. P0-02｜Fork 协议（核心差异）

### 1.1 B23-01：父子进程分叉协议（新增）

**05.22**：`fork_entry()` 末尾的 `finally` 块在父子两侧各执行一次，子进程执行 `release(_fork_coord_lock)` 但从未持有 → 错误释放未持有锁 / 跨进程释放。

**05.23**：
- 通过 `if pid == 0` 分支提前分流，子进程进入 `_after_in_child` 后**不再返回**父进程的释放代码；
- `_fork_coord_lock` 唯一释放者是**父进程**；
- 子进程对继承锁（coord/gate/fd/业务锁）**直接赋值全新对象**，不 acquire、不 release。

### 1.2 B23-02：epoch 递增移入锁临界区

**05.22**：`epoch = ++fork_epoch_counter` 在 `acquire(_fork_coord_lock)` 前执行，epoch 分配与快照登记不在同一临界区。

**05.23**：epoch 分配移入 `_fork_coord_lock`（L0）临界区，`allocate_epoch()` 在持锁时调用，epoch 分配 + 快照登记 + `os.fork()` 同临界区，一一对应；失败 fork 也消耗 epoch（防 ABA）。

## 2. P0-04｜POSIX 事务（S13 专项强化）

### 2.1 B23-04：S13 fsync 失败恢复

**05.22**：S13 fsync 失败后，仅凭 current 重读成功即迁移 COMMIT_DURABLE_AUDIT_PENDING，语义错误。

**05.23**：新增 §2.5 S13 专项恢复协议——current 重读成功只能证明目录项**内存已更新**，不能证明**断电持久化**；必须重新核验文件身份/大小/SHA-256/txn_id/revision/父版本/Candidate 六字段/五谓词/PREPARED 后，才允许重试 fsync(dirfd)；重试成功才进 AUDIT_PENDING，失败进 INDETERMINATE，身份冲突进 QUARANTINED。

### 2.2 B23-03：S1~S16 崩溃矩阵补齐

**05.22**：矩阵部分崩溃位置缺失或用"同上/类似"省略。

**05.23**：§2.2 逐行覆盖 ①~⑩ 十类崩溃位置，不适用位置写 N/A 并说明理由，无省略。

## 3. 交付（B23-05 / B23-06）

- **B23-05**：ZIP 必须真实下载/解压，记录真实字节数 + SHA-256 + CRC32 + 逐文件逐字节比对，写入 HASH-RECEIPT；无法执行则 NOT_VERIFIED。
- **B23-06**：采集 25 个 `.py` 文件前后完整 SHA-256 + 字节数，diff 比对，不只给三个冻结文件。

## 4. 术语/编号统一

- 状态名：七态 `COMMIT_IN_PROGRESS / NOT_DURABLE / COMMIT_INDETERMINATE / COMMIT_DURABLE_AUDIT_PENDING / DURABLE / RECOVERY_REQUIRED / QUARANTINED`；
- 步骤编号：S1~S16 固定，不重排；
- 锁级别：L0=_fork_coord_lock → L1=_gate_lock → L2=_fd_lock → L3=业务锁；
- 快照标识：`fork_epoch` 单调 64 位，失败也消耗。
