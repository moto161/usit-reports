# USIT-STAGE-B-DESIGN-05.28 状态时间线（TIMELINES）

> 独立材料。共 105 条唯一编号时间线：
> FT1~FT25（Fork 异常 25 条）、ID1~ID20（身份 20 条）、CR1~CR30（POSIX 崩溃恢复 30 条）、
> UC1~UC15（S12/S13 不确定提交 15 条）、GT1~GT10（多级 fork 分支推演 10 条）、FX1~FX20（Fork×POSIX 交叉 20 条）。
> 每条含：前置条件、事件/操作、崩溃点或触发点、身份/磁盘状态、允许/禁止动作、终态、不变量。

## 1. Fork 异常时间线（FT1~FT25）

### FT1 准入检查拒绝
- 前置：Gate≠OPEN；事件：业务入口 enter_op()；触发点：临界区 Gate 检查；锁：未持；Gate：非 OPEN；in_flight：0；终态：DENIED；不变量：I-05。

### FT2 协调锁获取失败
- 前置：Gate=OPEN；事件：acquire(_fork_coord_lock)；触发点：acquire 抛错；锁：未持；Gate：OPEN；in_flight：0；业务：否（fork 未发生）；终态：OPEN；不变量：I-10/11。

### FT3 FREEZING 失败（in_flight 未归零超时）
- 前置：持协调锁；事件：freeze()；触发点：等待超时；锁：协调锁持；Gate：OPEN（回滚）；in_flight：>0；终态：OPEN；不变量：I-13。

### FT4 in_flight 非零（等待中）
- 前置：FREEZING；事件：已准入操作完成递减；触发点：归零前；锁：协调锁持；Gate：FREEZING；in_flight：>0；终态：FREEZING；不变量：I-13。

### FT5 FROZEN 失败（快照生成失败）
- 前置：FREEZING；事件：make_snapshot；触发点：快照异常；锁：协调锁持；Gate：RESTORE_FAILED→DENIED；终态：DENIED；不变量：I-05。

### FT6 fork 系统调用失败
- 前置：FROZEN；事件：os.fork() 抛 OSError；锁：释放；Gate：OPEN（_close_fork_parent 恢复）或 DENIED；终态：OPEN/DENIED；不变量：I-05/15。

### FT7 父回调抛异常（_after_in_parent）
- 前置：fork 返回 pid>0；事件：_after_in_parent 抛错；锁：释放（最外层 finally）；Gate：先 DENIED 再尝试恢复；终态：OPEN（恢复成功）或 DENIED；不变量：I-05/15。

### FT8 子回调抛异常（_after_in_child）
- 前置：pid=0；事件：子初始化失败；锁：无；Gate：无（子退出）；终态：os._exit(1)；不变量：I-06。

### FT9 Gate 恢复失败
- 前置：RESTORING；事件：restore_gate 抛错；锁：释放；Gate：DENIED；终态：DENIED→TERMINATING；不变量：I-05/15。

### FT10 再次 fork 被拒绝
- 前置：DENIED；事件：fork_entry；触发点：enter_op 拒绝；Gate：DENIED；终态：DENIED；不变量：I-05。

### FT11 子进程继承锁状态异常
- 前置：fork 后子进程；事件：继承 _admit_lock/_coord_lock；触发点：继承锁未知态；锁：子进程本地重赋；Gate：子进程 OPEN（重建）；终态：child-local；不变量：I-04。

### FT12 父恢复失败子成功
- 前置：fork 后；事件：父 _close_fork_parent 失败、子初始化成功；锁：父释放；Gate：父 DENIED；终态：父 DENIED、子正常；不变量：I-05。

### FT13 父恢复成功子失败
- 前置：fork 后；事件：父恢复成功、子 os._exit；Gate：父 OPEN；终态：父 OPEN、子退出；不变量：I-06。

### FT14 回调重复执行
- 前置：不注册 at-fork；事件：仅手动恢复；终态：不存在（不注册）；不变量：I-02。

### FT15 清理操作部分成功
- 前置：异常清理；事件：_pending_snapshot=None + Gate 恢复部分完成；Gate：OPEN 或 DENIED；终态：OPEN/DENIED；不变量：I-05。

### FT16 restore_gate 抛 BaseException
- 前置：RESTORING；事件：restore_gate 抛 BaseException；Gate：先 DENIED（赋值序保证）；终态：RESTORE_FAILED；不变量：I-05/15。

### FT17 解锁失败
- 前置：持协调锁；事件：release 抛错；锁：无法释放；Gate：DENIED；终态：TERMINATING；不变量：I-10。

### FT18 finally 内再次异常
- 前置：父路径 finally；事件：finally 内再抛错；Gate：终态由赋值序决定（非异常传播决定）；终态：OPEN/DENIED；不变量：I-11。

### FT19 业务继续被拒绝
- 前置：RESTORE_FAILED；事件：业务入口；触发点：assert_gate_open 拒绝；终态：DENIED；不变量：I-05。

### FT20 进程终止
- 前置：无法恢复；事件：TERMINATING → os._exit；终态：退出；不变量：I-05。

### FT21 _before_fork 回调失败（新增 A1）
- 前置：OPEN；事件：_before_fork 抛错；锁：释放；Gate：OPEN（回滚，冻结未完成）；in_flight：0；业务：否（fork 未发生）；终态：OPEN；不变量：I-05/13。

### FT22 Gate 状态写入失败（新增 A5）
- 前置：FREEZING；事件：_gate 赋值抛异常；锁：释放；Gate：未知→DENIED；终态：DENIED；不变量：I-05/07。

### FT23 协调锁释放失败（新增 A7）
- 前置：RESTORING；事件：release 抛错；锁：无法释放；Gate：DENIED；终态：TERMINATING；不变量：I-10/11。

### FT24 多线程同时申请 fork（新增 A15）
- 前置：OPEN，两线程 fork_entry；事件：协调锁串行化；后到者：enter_op 后协调锁 acquire 失败或 freeze 拒绝；终态：单 fork 成功、另一拒绝；不变量：I-12/14。

### FT25 fork 期间迟到 FD 操作（新增 A16）
- 前置：FREEZING；事件：新线程 enter_op；触发点：临界区 Gate≠OPEN 拒绝；in_flight：冻结者等待中；终态：拒绝新准入；不变量：I-13。

## 2. 身份生命周期时间线（ID1~ID20）

### ID1 进程启动生成 instance_id
- 事件：os.urandom(16)；身份：process_instance_id 建立；判定：实例内概率唯一；证据：STATIC_INFERENCE。

### ID2 首次 fork 分配 fork_epoch
- 事件：父 fork 时刻自增 seq=1；身份：fork_epoch=1；判定：父内严格；证据：STATIC_INFERENCE。

### ID3 子进程继承 instance_id
- 事件：子继承 instance_id + fork_epoch=1；判定：实例树内共享 instance_id；证据：STATIC_INFERENCE。

### ID4 子进程再 fork（父内序号独立）
- 事件：子作为父，自己的 seq 从 1 起；判定：父内严格，与祖父不共享计数器；证据：STATIC_INFERENCE。

### ID5 兄弟进程 fork（结构性区分）
- 事件：父分配 seq=1、seq=2 给两子；fork_event_id = (父instance, seq, rand)；判定：**结构性不碰撞**（seq 不同）；证据：STATIC_INFERENCE。

### ID6 多级 fork 分支（instance 前缀区分）
- 事件：A 创建 C，C fork_event_id=(A_instance, seq_A)；B fork_event_id=(R_instance, 2)；A_instance≠R_instance；判定：不碰撞；证据：STATIC_INFERENCE。

### ID7 PID 重用
- 事件：旧 pid 复用；身份：instance_id 区分；判定：实例内区分；证据：VERIFIED_FACT。

### ID8 进程重启
- 事件：新 instance_id；判定：跨重启不唯一；证据：STATIC_INFERENCE。

### ID9 快照消费失效
- 事件：子消费 snapshot；snapshot_id 失效；判定：不复用；证据：STATIC_INFERENCE。

### ID10 旧快照复用拒绝
- 事件：新 fork 读旧 snapshot；epoch 不匹配；判定：拒绝；证据：STATIC_INFERENCE。

### ID11 FD 槽位复用
- 事件：槽位关后重开；fd_generation 递增；判定：不混淆；证据：STATIC_INFERENCE。

### ID12 txn 重复提交
- 事件：重复提交；txn_id 幂等；判定：拒绝；证据：STATIC_INFERENCE。

### ID13 随机源不可用
- 事件：os.urandom 失败；判定：降级拒绝（DENY）；证据：UNKNOWN。

### ID14 恶意复制身份
- 事件：进程复制 instance_id；判定：无认证不防；证据：UNKNOWN。

### ID15 计数器溢出
- 事件：seq 溢出；判定：DENY 不回绕；证据：STATIC_INFERENCE。

### ID16 快照跨进程传递（非继承）
- 事件：快照传给非继承进程；instance_id 不匹配；判定：拒绝；证据：STATIC_INFERENCE。

### ID17 fork_event_id 随机后缀降级
- 事件：随机源不可用，仅结构分量；判定：降级拒绝而非弱随机；证据：STATIC_INFERENCE。

### ID18 父分支标识在 fork 后保持
- 事件：子进程 fork_event_id 的父 instance 分量固定；判定：跨多代可追溯父链；证据：STATIC_INFERENCE。

### ID19 txn_id 持久化重放
- 事件：崩溃后重放 txn_id；判定：可重放校验；证据：STATIC_INFERENCE。

### ID20 revision 跨重启重读
- 事件：崩溃后重读 revision；判定：可重读；证据：STATIC_INFERENCE。

## 3. POSIX 崩溃恢复时间线（CR1~CR30）

> 每条：前置状态、崩溃位置、磁盘可能状态、必读对象、允许/禁止恢复、终态、不变量。

### CR1 S1 open 前崩溃
- 前置 NEW；崩溃：open 前；磁盘：无 Candidate；必读：无；允许：重 S1；禁止：当完成；终态：NEW；不变量：I-39。

### CR2 S1 write 中崩溃（半成品 Candidate）
- 前置 NEW；崩溃：write 中；磁盘：半成品 Candidate 目录项；必读：Candidate 身份/SHA；允许：删半成品重 S1；禁止：当完成跳 S2；终态：CANDIDATE_CREATED（半成品待删）；不变量：I-39/45。

### CR3 S1 完成后崩溃（未 fsync）
- 前置 NEW；崩溃：write 完成；磁盘：完整 Candidate 未持久；必读：内容 SHA；允许：删后重 S1 或 S2 fsync；禁止：当已持久；终态：CANDIDATE_CREATED；不变量：I-34。

### CR4 S2 fsync 前崩溃
- 前置 CANDIDATE_CREATED；崩溃：fsync 前；磁盘：内容未持久；必读：Candidate SHA；允许：重 S2；禁止：跳 S3；终态：CANDIDATE_CREATED；不变量：I-34。

### CR5 S2 fsync 中崩溃（EINTR/EIO）
- 前置 CANDIDATE_CREATED；崩溃：fsync 中；磁盘：内容持久状态未知；必读：重读 Candidate SHA；允许：重读后重试 fsync；禁止：盲目重试；终态：CANDIDATE_CONTENT_DURABLE 或重试；不变量：I-37/38。

### CR6 S3 fsync(dir) 前崩溃
- 前置 CANDIDATE_CONTENT_DURABLE；崩溃：dirfsync 前；磁盘：目录项未持久；必读：versions 目录项；允许：重 S3；禁止：跳 S4；终态：CANDIDATE_CONTENT_DURABLE；不变量：I-33。

### CR7 S4 校验前崩溃（Candidate 内容损坏）
- 前置 CANDIDATE_ENTRY_DURABLE；崩溃：校验发现 SHA 不匹配；磁盘：Candidate 损坏；必读：Candidate 内容；允许：删重 S1；禁止：跳 S5；终态：CANDIDATE_VERIFIED 失败→删；不变量：I-40/42。

### CR8 S5 写 PREPARED 中崩溃（半成品）
- 前置 CANDIDATE_VERIFIED；崩溃：write 中；磁盘：半成品 PREPARED；必读：PREPARED 身份；允许：删半成品重 S5（身份匹配）；禁止：当完成；终态：PREPARED_CREATED（待删）；不变量：I-39/45。

### CR9 S5 EEXIST（旧 PREPARED 残留）
- 前置 CANDIDATE_VERIFIED；崩溃：EEXIST；磁盘：已存在 PREPARED；必读：已存在对象身份+txn；允许：身份匹配删后重写；禁止：身份不明盲目删；终态：QUARANTINED（若无法确认）；不变量：I-45。

### CR10 S6 fsync(prepared_fd) 中崩溃
- 前置 PREPARED_CREATED；崩溃：fsync 中；磁盘：内容持久未知；必读：PREPARED SHA；允许：重读后重试；禁止：盲目重试；终态：PREPARED_CONTENT_DURABLE 或重试；不变量：I-34/37。

### CR11 S7 fsync(prepared_dir) 前崩溃
- 前置 PREPARED_CONTENT_DURABLE；崩溃：dirfsync 前；磁盘：目录项未持久；必读：prepared 目录项；允许：重 S7；终态：PREPARED_CONTENT_DURABLE；不变量：I-33。

### CR12 S8 获取全局锁前崩溃
- 前置 PREPARED_ENTRY_DURABLE；崩溃：锁获取前；磁盘：PREPARED 已持久；必读：PREPARED+current；允许：重 S8；终态：PREPARED_ENTRY_DURABLE；不变量：I-12。

### CR13 S9 校验发现版本冲突
- 前置 持全局锁；崩溃：C1 不匹配；磁盘：current 已被他人改；必读：current revision；允许：重读重新校验；禁止：跳 S10；终态：QUARANTINED（若持续冲突）；不变量：I-40/41。

### CR14 S10 写 current 临时文件中崩溃
- 前置 PREPARED_VERIFIED；崩溃：write 中；磁盘：半成品临时文件；必读：临时文件身份；允许：删半成品重 S10；禁止：当完成；终态：CURRENT_TMP_CREATED（待删）；不变量：I-39/45。

### CR15 S11 fsync(tmp) 中崩溃
- 前置 CURRENT_TMP_CREATED；崩溃：fsync 中；磁盘：内容持久未知；必读：临时文件 SHA；允许：重读后重试；终态：CURRENT_TMP_CONTENT_DURABLE 或重试；不变量：I-34/37。

### CR16 S12 rename 前崩溃
- 前置 CURRENT_TMP_VERIFIED；崩溃：rename 前；磁盘：current 旧值 + 完整临时文件；必读：临时文件身份；允许：重 S12（校验后）；禁止：未校验 rename；终态：CURRENT_TMP_VERIFIED；不变量：I-32/36。

### CR17 S12 rename 中崩溃（结果未知）
- 前置 CURRENT_TMP_VERIFIED；崩溃：rename EINTR；磁盘：current 旧值或新值（未知）；必读：current 身份/SHA/rev；允许：重读判定；禁止：盲目重试 rename；终态：CURRENT_REPLACE_PENDING / COMMIT_INDETERMINATE；不变量：I-32/36。

### CR18 S13 fsync(dir) 中崩溃
- 前置 CURRENT_REPLACE_PENDING；崩溃：dirfsync 中；磁盘：current 新值目录项未持久；必读：current 全身份+五条件+六字段+五谓词+PREPARED；允许：全核验后重试；禁止：仅凭重读迁移；终态：COMMIT_INDETERMINATE / AUDIT_PENDING；不变量：I-33/36/37。

### CR19 S13 读到旧 current（不推断 rename 未发生）
- 前置 CURRENT_REPLACE_PENDING；事件：dirfsync 失败后重读得旧值；判定：**旧值不证明 rename 未发生**；允许：全核验；禁止：盲目重试 rename；终态：COMMIT_INDETERMINATE；不变量：I-32/36。

### CR20 S13 读到新 current（不推断已持久）
- 前置 CURRENT_REPLACE_PENDING；事件：重读得新值；判定：**新值不证明已持久**；允许：dirfsync 重试；禁止：当 DURABLE；终态：COMMIT_INDETERMINATE；不变量：I-33/36。

### CR21 S14 写回执中崩溃
- 前置 COMMIT_DURABLE_AUDIT_PENDING；崩溃：write 中；磁盘：半成品回执；必读：回执身份；允许：删半成品重 S14；终态：COMMIT_RECEIPT_CREATED（待删）；不变量：I-35/45。

### CR22 S14 EEXIST（回执已存在）
- 前置 COMMIT_DURABLE_AUDIT_PENDING；崩溃：EEXIST；必读：已存在回执唯一键+内容；允许：匹配则幂等成功；禁止：不匹配覆盖；终态：幂等完成 或 QUARANTINED；不变量：I-35。

### CR23 S14 fsync(commit_fd) 中崩溃
- 前置 COMMIT_RECEIPT_CREATED；崩溃：fsync 中；磁盘：回执内容持久未知；必读：回执 SHA；允许：重读后重试；终态：COMMIT_RECEIPT_CONTENT_DURABLE 或重试；不变量：I-35/37。

### CR24 S15 fsync(commit_dir) 前崩溃
- 前置 COMMIT_RECEIPT_CONTENT_DURABLE；崩溃：dirfsync 前；磁盘：回执目录项未持久；必读：回执目录项；允许：重 S15；终态：COMMIT_RECEIPT_CONTENT_DURABLE；不变量：I-33。

### CR25 S15 完成后崩溃（回执目录项持久但 current 状态待确认）
- 前置 COMMIT_RECEIPT_CONTENT_DURABLE；崩溃：S15 后；磁盘：current+回执目录项持久；必读：current+回执全身份；允许：判定 DURABLE；终态：DURABLE；不变量：I-33/35。

### CR26 S16 释放锁后崩溃
- 前置 DURABLE；崩溃：锁释放后；磁盘：DURABLE；允许：无（已完成）；终态：DURABLE；不变量：I-46。

### CR27 证据冲突（身份不匹配）
- 前置 任意恢复；事件：目录项身份 vs 记录身份冲突；判定：QUARANTINED；终态：QUARANTINED；不变量：I-40/42。

### CR28 并发事务残留（旧 txn 重放）
- 前置 恢复；事件：发现旧 txn 残留；必读：txn_id+revision；允许：判定归属；禁止：盲目删除；终态：RECOVERY_REQUIRED 或 QUARANTINED；不变量：I-41。

### CR29 EINTR 后重读发现对象身份变化
- 前置 任意 fsync/rename EINTR；事件：重读身份与记录不符；判定：QUARANTINED；终态：QUARANTINED；不变量：I-37/38。

### CR30 设备写缓存导致持久性不可确认
- 前置 任意 fsync 成功；事件：设备写缓存；判定：fsync 成功受设备边界限制；终态：标记 DEVICE_OR_STORAGE_LIMIT；不变量：I-33/37（NOT_VERIFIED）。

## 4. 多级 fork 分支推演（GT1~GT10）

### GT1 根 R 创建 A、B
- R 内 seq：A=1、B=2；A=(R,1)、B=(R,2)；不碰撞（seq 不同）。

### GT2 A 创建 C、D
- A 内 seq：C=1、D=2；C=(A,1)、D=(A,2)；不碰撞（A 内 seq 独立）。

### GT3 B 创建 E
- B 内 seq：E=1；E=(B,1)；与 C=(A,1) 前缀 A≠B，不碰撞。

### GT4 跨分支同 seq 不同 instance
- C=(A,1)、E=(B,1)：seq 同为 1 但父 instance 不同 → 不碰撞。

### GT5 五代深链
- R→A→C→F→G，每代 fork_event_id 前缀为该代父 instance；全程可追溯。

### GT6 兄弟多代交错
- A、B 各自繁衍，任意两节点 fork_event_id 前缀不同或 seq 不同 → 不碰撞。

### GT7 PID 重用后新进程 fork
- 新进程 instance_id 全新，旧 pid 复用不影响身份；不碰撞。

### GT8 进程崩溃后重启再 fork
- 重启新 instance_id，旧 fork_event_id 不复用；跨重启不承诺唯一但结构不误用。

### GT9 随机源不可用时的分支
- 结构分量（instance+seq）仍可区分兄弟/多代；随机后缀缺失仅影响跨树概率；整体 DENY 而非降级。

### GT10 计数器溢出边界
- seq 达 2^64-1 后 DENY，不回绕，不产生重复。

## 5. S12/S13 不确定提交场景（UC1~UC15）

> 每条：rename/fsync 结果不确定时的判定、必读、允许/禁止、终态。

### UC1 rename EINTR 后 current 为新值
- 判定：rename 可能已生效；禁止：盲目重 rename；终态：进入 S13。

### UC2 rename EINTR 后 current 为旧值
- 判定：不推断未发生；允许：全核验；终态：COMMIT_INDETERMINATE。

### UC3 S13 fsync EINTR
- 判定：持久状态未知；允许：重读+全核验后重试 fsync；终态：INDETERMINATE。

### UC4 S13 fsync EIO
- 判定：设备错误；允许：人工裁决；终态：RECOVERY_REQUIRED。

### UC5 S13 fsync ENOSPC
- 判定：空间不足；禁止：盲目重试；终态：RECOVERY_REQUIRED。

### UC6 S13 重读 current 身份不匹配
- 判定：身份冲突；终态：QUARANTINED。

### UC7 S13 重读 SHA 不匹配
- 判定：内容损坏；终态：QUARANTINED。

### UC8 S13 重读 txn/rev 冲突
- 判定：事务冲突；终态：QUARANTINED。

### UC9 rename 后 current 新值但目录项未持久
- 判定：不可当 DURABLE；允许：重 S13；终态：INDETERMINATE。

### UC10 rename 后目录项持久但回执缺失
- 判定：current 已持久；允许：补回执 S14；终态：AUDIT_PENDING。

### UC11 并发事务在 S13 期间改 current
- 判定：锁外写入；五条件 C5 失败；终态：QUARANTINED。

### UC12 重试 rename 前对象身份变化
- 判定：对象被替换；禁止：重试；终态：QUARANTINED。

### UC13 重试 dirfsync 成功后直接当 DURABLE
- 判定：**错误**，重试成功仍需回执核验；禁止：跳 S14；终态：AUDIT_PENDING（非 DURABLE）。

### UC14 读到旧值即重试 rename
- 判定：**错误**，旧值≠未发生；禁止：盲目重试；终态：INDETERMINATE。

### UC15 读到新值即当已持久
- 判定：**错误**，新值≠已持久；禁止：跳 dirfsync；终态：INDETERMINATE。

## 6. Fork×POSIX 交叉场景（FX1~FX20）

### FX1 fork 冻结期间禁止事务提交
- Gate FREEZING 阻断 enter_op，事务提交入口被拒绝；不变量：I-13/47。

### FX2 快照冻结与 candidate 写入互斥
- freeze 等待 in_flight 归零，未完成 candidate 写入不计入快照；不变量：I-13/48。

### FX3 fork 后子进程 FD 状态独立
- 子进程继承 FD 但 fd_generation 独立重算；不变量：I-04/30。

### FX4 fork 后父进程事务锁状态
- 父恢复后事务锁可继续，子进程重建事务锁；不变量：I-12/14。

### FX5 快照引用 current 的 revision
- 快照绑定 fork 时 current revision，后续提交递增 revision 使旧快照失效；不变量：I-41/49。

### FX6 fork 期间 current 被 rename（跨进程）
- 子进程读 current 需身份校验，rename 后 inode 变化被检测；不变量：I-32/50。

### FX7 事务提交与 fork 的 epoch 关系
- 事务 revision 与 fork_epoch 独立命名空间，不混淆；不变量：I-51。

### FX8 恢复中禁止 fork
- RESTORING/RESTORE_FAILED 拒绝 fork_entry；不变量：I-05/52。

### FX9 子进程初始化失败不清除父事务
- 子 os._exit 不影响父进程事务状态；不变量：I-06/53。

### FX10 fork 冻结超时回滚不影响已提交事务
- freeze 超时回滚 Gate 到 OPEN，已 DURABLE 事务不受影响；不变量：I-54。

### FX11 快照与 txn_id 的绑定
- snapshot_id 与 fork_event_id 一致，txn_id 独立分配；不变量：I-55。

### FX12 多线程事务提交与 fork 冻结
- 冻结等待 in_flight，含事务提交的 enter_op 递减后才冻结；不变量：I-13/56。

### FX13 fork 后父进程回执幂等性
- 父进程恢复后继续 S14，回执唯一键防重复；不变量：I-35/57。

### FX14 子进程不参与父的提交协议
- 子进程无父的 _commit_lock 所有权，提交走父进程；不变量：I-12/58。

### FX15 fork 期间目录 fsync 进行中
- 目录 fsync 属事务路径，需 enter_op，冻结等待其完成；不变量：I-13/59。

### FX16 身份与事务交叉：txn_id 含 instance 分量
- txn_id 绑定 process_instance_id，跨实例不混淆；不变量：I-60。

### FX17 崩溃恢复后 fork 计数器重置
- 重启后 seq 归零，旧 fork_event_id 不复用；不变量：I-22/31。

### FX18 rename 与 fork 的目录项可见性
- 子进程看到的 current 是 fork 时刻快照，rename 后的新值不可见（若未重新读取）；不变量：I-32/50。

### FX19 父进程 DENIED 后子进程状态
- 父 DENIED 不影响已 fork 出的健康子进程；不变量：I-05/06。

### FX20 事务回滚与 fork 冻结的锁顺序
- 锁顺序 _admit_lock → _commit_lock，回滚与冻结不死锁；不变量：I-14/56。

---

## 统计

- Fork 异常：FT1~FT25 = 25 条
- 身份：ID1~ID20 = 20 条
- POSIX 崩溃恢复：CR1~CR30 = 30 条
- S12/S13 不确定提交：UC1~UC15 = 15 条
- 多级 fork 推演：GT1~GT10 = 10 条
- Fork×POSIX 交叉：FX1~FX20 = 20 条

**合计：25+20+30+15+10+20 = 120 条独立时间线。**
