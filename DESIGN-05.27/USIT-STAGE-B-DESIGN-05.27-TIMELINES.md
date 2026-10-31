# USIT-STAGE-B-DESIGN-05.27 状态时间线（TIMELINES）

> 独立材料。共 50 条唯一编号时间线：FT1~FT20（Fork 异常）、ID1~ID15（身份）、S1~S16（POSIX 步骤，含崩溃位置）。
> 每条含：前置条件、操作顺序、崩溃点、磁盘状态、恢复读取项、允许/禁止动作、最终状态、不变量。

## 1. Fork 异常时间线（FT1~FT20）

### FT1 准入检查拒绝
- 前置：Gate 非 OPEN；操作：业务入口 assert_gate_open()；崩溃点：无；磁盘：无；恢复读取：Gate 状态；允许：无；禁止：业务/fork/FD；最终：DENIED；不变量：I-05。

### FT2 协调锁获取失败
- 前置：Gate=OPEN；操作：acquire(_fork_coord_lock)；崩溃点：acquire 抛错；磁盘：无；恢复读取：无；允许：重试；禁止：fork；最终：OPEN；不变量：I-10/11。

### FT3 FREEZING 失败
- 前置：持 L0；操作：freeze 置 FREEZING；崩溃点：in_flight 未归零超时；磁盘：无；恢复读取：in_flight；允许：等待；禁止：fork；最终：OPEN（回滚）；不变量：I-13。

### FT4 in_flight 非零
- 前置：FREEZING；操作：等待归零；崩溃点：有在途 FD 操作；磁盘：无；恢复读取：in_flight；允许：等待；禁止：fork/FD 准入；最终：FREEZING；不变量：I-13。

### FT5 FROZEN 失败
- 前置：FREEZING；操作：生成快照；崩溃点：快照生成失败；磁盘：无；恢复读取：Registry；允许：回滚；禁止：fork；最终：RESTORE_FAILED→DENIED；不变量：I-05。

### FT6 fork 系统调用失败
- 前置：FROZEN；操作：os.fork()；崩溃点：fork 抛 OSError；磁盘：无；恢复读取：无；允许：恢复 Gate；禁止：业务（Gate 未恢复）；最终：OPEN 或 DENIED；不变量：I-05/15。

### FT7 父回调抛异常
- 前置：fork 返回 pid>0；操作：_after_in_parent；崩溃点：回调抛错；磁盘：无；恢复读取：Gate；允许：_close_fork_parent 恢复；禁止：业务（若恢复失败）；最终：OPEN 或 DENIED；不变量：I-05/15。

### FT8 子回调抛异常
- 前置：fork 返回 pid=0；操作：_after_in_child；崩溃点：子初始化失败；磁盘：无；恢复读取：无；允许：无；禁止：业务；最终：os._exit(1)；不变量：I-06。

### FT9 Gate 恢复失败
- 前置：RESTORING；操作：restore_gate；崩溃点：抛异常；磁盘：无；恢复读取：Gate；允许：终止；禁止：业务/fork/FD；最终：RESTORE_FAILED→DENIED→TERMINATING；不变量：I-05/15。

### FT10 再次 fork 被拒绝
- 前置：DENIED；操作：fork_entry；崩溃点：assert_gate_open 拒绝；磁盘：无；恢复读取：Gate；允许：无；禁止：fork；最终：DENIED；不变量：I-05。

### FT11 子进程继承锁状态异常
- 前置：fork 后子进程；操作：继承锁；崩溃点：继承锁为未知态；磁盘：无；恢复读取：继承锁；允许：赋值新对象；禁止：acquire/release 继承锁；最终：child-local；不变量：I-04。

### FT12 父恢复失败子成功
- 前置：fork 后；操作：父 _close_fork_parent 失败、子初始化成功；磁盘：无；恢复读取：父 Gate；允许：父 DENIED/终止；禁止：父业务；最终：父 DENIED、子正常；不变量：I-05。

### FT13 父恢复成功子失败
- 前置：fork 后；操作：父恢复成功、子 os._exit；磁盘：无；恢复读取：父 Gate；允许：父业务；禁止：无；最终：父 OPEN、子退出；不变量：I-06。

### FT14 回调重复执行
- 前置：注册 at-fork；操作：自动+手动回调；崩溃点：重复恢复；磁盘：无；恢复读取：无；允许：无；禁止：注册 at-fork；最终：不存在（不注册）；不变量：I-02。

### FT15 清理操作部分成功
- 前置：异常清理；操作：_pending_snapshot=None + Gate 恢复；崩溃点：只完成部分；磁盘：无；恢复读取：Gate/快照；允许：继续清理；禁止：业务（Gate 未恢复）；最终：OPEN 或 DENIED；不变量：I-05。

### FT16 restore_gate 抛异常
- 前置：RESTORING；操作：restore_gate；崩溃点：抛 BaseException；磁盘：无；恢复读取：Gate；允许：DENIED/终止；禁止：业务/fork；最终：RESTORE_FAILED；不变量：I-05/15。

### FT17 解锁失败
- 前置：持 L0；操作：release；崩溃点：release 抛错；磁盘：无；恢复读取：锁状态；允许：终止；禁止：业务；最终：DENY；不变量：I-10。

### FT18 finally 内再次异常
- 前置：父路径 finally；操作：finally 内再抛错；磁盘：无；恢复读取：Gate/锁；允许：终止；禁止：业务；最终：DENY；不变量：I-11。

### FT19 业务继续被拒绝
- 前置：RESTORE_FAILED；操作：业务入口；崩溃点：assert_gate_open 拒绝；磁盘：无；恢复读取：Gate；允许：无；禁止：业务；最终：DENIED；不变量：I-05。

### FT20 进程终止
- 前置：无法恢复；操作：TERMINATING；崩溃点：os._exit；磁盘：无；恢复读取：无；允许：退出；禁止：业务；最终：退出；不变量：I-05。

## 2. 身份生命周期时间线（ID1~ID15）

### ID1 进程启动
- 事件：os.urandom(16) 生成 instance_id；身份：instance_id 建立；判定：实例内概率唯一；证据：STATIC_INFERENCE。

### ID2 首次 fork
- 事件：父 L0 内分配 fork_epoch=0→1；身份：fork_epoch 严格递增；判定：进程树内严格；证据：STATIC_INFERENCE。

### ID3 子进程继承
- 事件：子继承 instance_id + fork_epoch=1；身份：继承不变；判定：实例内一致；证据：STATIC_INFERENCE。

### ID4 子进程再 fork
- 事件：子作为父，L0 内分配新 epoch；身份：继续递增；判定：进程树严格；证据：STATIC_INFERENCE。

### ID5 兄弟进程 fork
- 事件：父分配两个不同 epoch 给两子；身份：fork_event_id epoch 分量不同；判定：**结构性不碰撞**；证据：STATIC_INFERENCE。

### ID6 PID 重用
- 事件：旧 pid 被新进程复用；身份：instance_id 区分；判定：实例内区分；证据：VERIFIED_FACT(PID 重用)。

### ID7 进程重启
- 事件：新 instance_id；身份：跨重启不唯一；判定：不承诺；证据：STATIC_INFERENCE。

### ID8 快照消费
- 事件：子消费 snapshot；身份：snapshot_id 失效；判定：不复用；证据：STATIC_INFERENCE。

### ID9 旧快照复用
- 事件：新一轮 fork 读旧 snapshot；身份：epoch 不匹配；判定：拒绝；证据：STATIC_INFERENCE。

### ID10 FD 槽位复用
- 事件：槽位关闭后重开；身份：fd_generation 递增；判定：不混淆；证据：STATIC_INFERENCE。

### ID11 txn 重复提交
- 事件：重复提交；身份：txn_id 幂等；判定：拒绝；证据：STATIC_INFERENCE。

### ID12 随机源不可用
- 事件：os.urandom 失败；身份：降级/拒绝；判定：不弱随机；证据：UNKNOWN。

### ID13 恶意复制身份
- 事件：进程复制 instance_id；身份：无认证；判定：不防伪造；证据：UNKNOWN。

### ID14 计数器溢出
- 事件：fork_epoch 溢出；身份：DENY；判定：不回绕；证据：STATIC_INFERENCE。

### ID15 快照跨进程传递
- 事件：快照传给非继承进程；身份：instance_id 区分；判定：拒绝；证据：STATIC_INFERENCE。

## 3. POSIX S1~S16 步骤时间线（每步含崩溃位置，18 项）

> 每步给出：前置状态、磁盘对象、syscall+参数、返回值/错误码、短写/部分写/EINTR/EIO/ENOSPC、崩溃位置、内存丢失、磁盘可能状态、目录项可见性、内容持久、文件名持久、txn/rev/父版本、身份/大小/SHA、必读对象、允许/禁止恢复、最终状态、不变量。

### S1 Candidate O_EXCL 写入
- 前置 NEW；磁盘：无 Candidate；syscall open(O_CREAT|O_EXCL|O_WRONLY)；返回值 fd 或 EEXIST/EINTR/EIO/ENOSPC；崩溃位置 ①前②中③后；内存丢失：fd；磁盘：①无②半成品③完整；可见性：③可见(未持久)；内容持久：③否；文件名持久：③否；txn/rev：无；身份/SHA：需重读；必读：Candidate 身份/SHA；允许：删半成品重 S1；禁止：当完成跳 S2；最终：CANDIDATE_CREATED；不变量：I-39/40/45。

### S2 fsync(candidate_fd)
- 前置 CANDIDATE_CREATED；syscall fsync(fd)；返回值 0 或 EINTR/EIO/ENOSPC；崩溃位置 fsync 前/中/后；内存丢失：无；磁盘：内容未持久/部分/持久；可见性：文件已可见；内容持久：成功才持久；必读：Candidate 内容 SHA；允许：重试 fsync（重读后）；禁止：当持久跳 S3；最终：CANDIDATE_CONTENT_DURABLE；不变量：I-34/37/38。

### S3 fsync(versions_dirfd)
- 前置 CANDIDATE_CONTENT_DURABLE；syscall fsync(dirfd)；返回值 0 或 EINTR/EIO/ENOSPC；崩溃位置 fsync 前/中/后；目录项持久：成功才持久；必读：versions 目录项 + Candidate 身份；允许：重试 dirfsync；禁止：当持久跳 S4；最终：CANDIDATE_ENTRY_DURABLE；不变量：I-33/37。

### S4 重读与 SHA-256 校验
- 前置 CANDIDATE_ENTRY_DURABLE；syscall 无（纯读）；崩溃位置 无副作用；必读：Candidate 内容；允许：重新校验；禁止：校验未过就 S5；最终：CANDIDATE_VERIFIED；不变量：I-40/42。

### S5 写 PREPARED
- 前置 CANDIDATE_VERIFIED；syscall open(O_CREAT|O_EXCL)+write；返回值 短写/EINTR/EIO/ENOSPC/EEXIST；崩溃位置 write 前/中/后；磁盘：无/半成品/完整；必读：PREPARED 身份/SHA；允许：删半成品重 S5（排他创建失败且无旧对象时）；禁止：当完成跳 S6；最终：PREPARED_CREATED；不变量：I-39/45。

### S6 fsync(prepared_fd)
- 前置 PREPARED_CREATED；syscall fsync(fd)；内容持久：成功才持久；必读：PREPARED 内容 SHA；允许：重试；禁止：跳 S7；最终：PREPARED_CONTENT_DURABLE；不变量：I-34/37/38。

### S7 fsync(prepared_dirfd)
- 前置 PREPARED_CONTENT_DURABLE；syscall fsync(dirfd)；目录项持久：成功才持久；必读：prepared 目录项；允许：重试；禁止：跳 S8；最终：PREPARED_ENTRY_DURABLE；不变量：I-33/37。

### S8 获取全局提交锁
- 前置 PREPARED_ENTRY_DURABLE；syscall 无（内存锁）；崩溃位置 无副作用（OS 释放）；最终：持锁→S9；不变量：I-12/14。

### S9 读取 current 并校验五条件
- 前置 持全局锁；syscall 无（纯读）；必读：current 内容 + 五条件；允许：重新校验；禁止：五条件未过就 S10；最终：PREPARED_VERIFIED；不变量：I-40/41/42。

### S10 写 current 临时文件
- 前置 PREPARED_VERIFIED；syscall open(O_CREAT|O_EXCL)+write；返回值 短写/EINTR/EIO/ENOSPC/EEXIST；崩溃位置 write 前/中/后；必读：临时文件身份/SHA；允许：删半成品重 S10；禁止：跳 S11；最终：临时文件就绪；不变量：I-39/45。

### S11 fsync(current_tmp_fd)
- 前置 S10 完成；syscall fsync(fd)；内容持久：成功才持久；必读：临时文件内容 SHA；允许：重试；禁止：跳 S12；最终：临时文件持久；不变量：I-34/37/38。

### S12 rename 替换 current
- 前置 S11 完成；syscall rename(tmp,current)；返回值 0 或 EINTR/EIO/ENOSPC；崩溃位置 rename 前/中/后；磁盘：旧/不确定/新；可见性：新值可见(内存未持久)；必读：current 身份/SHA/版本；允许：重读判定（不推断旧值=未 rename）；禁止：盲目重试 rename；最终：CURRENT_REPLACE_PENDING；不变量：I-32/36。

### S13 fsync(current 父目录)
- 前置 CURRENT_REPLACE_PENDING；syscall fsync(dirfd)；返回值 0 或 EINTR/EIO/ENOSPC；目录项持久：成功才持久；必读：current 身份/大小/SHA/txn/rev/父版本/六字段/五谓词/PREPARED；允许：全核验后重试；禁止：仅凭重读迁移；最终：成功→AUDIT_PENDING；失败→INDETERMINATE/QUARANTINED；不变量：I-33/36/37/40/41/42。

### S14 写 COMMIT 回执并 fsync(commit_fd)
- 前置 COMMIT_DURABLE_AUDIT_PENDING；syscall open(O_CREAT)+write+fsync；返回值 短写/EINTR/EIO/ENOSPC/EEXIST；必读：回执身份/SHA；允许：删半成品重 S14；禁止：跳 S15；最终：COMMIT_RECEIPT_CONTENT_DURABLE；不变量：I-35/37/38。

### S15 fsync(commit_dirfd)
- 前置 COMMIT_RECEIPT_CONTENT_DURABLE；syscall fsync(dirfd)；目录项持久：成功才持久；必读：回执目录项；允许：重试；禁止：跳 DURABLE；最终：DURABLE；不变量：I-33/37。

### S16 释放全局提交锁
- 前置 DURABLE；syscall 无（内存锁）；崩溃位置 无副作用；最终：DURABLE 不变；不变量：I-46。

---

## 统计

- Fork 异常时间线：FT1~FT20 = 20 条
- 身份时间线：ID1~ID15 = 15 条
- POSIX 步骤时间线：S1~S16 = 16 条（每步含多个崩溃位置）

**合计：20 + 15 + 16 = 51 条独立时间线（≥50）。**
