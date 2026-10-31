# USIT-STAGE-B-DESIGN-05.29 时间线（TIMELINES.md）

> 全部时间线及交叉推演。共 165 条唯一编号：
> FT1~FT30（Fork 异常 30）、CR1~CR40（POSIX 崩溃 40）、RU1~RU25（rename 不确定 25）、
> DU1~DU25（目录持久不确定 25）、EW1~EW20（外部写入者 20）、FX1~FX30（Fork×POSIX 30）。

## 1. Fork 异常时间线（FT1~FT30）

### FT1 Fork 入口失败
- 前置 OPEN｜触发 fork_entry 前置失败｜持锁 无｜Gate OPEN｜业务 是｜终态 OPEN。

### FT2 业务登记失败（Gate≠OPEN）
- 前置 FREEZING｜触发 enter_op｜Gate FREEZING｜业务 否｜终态 拒绝。

### FT3 in_flight 自锁（已消除）
- 说明：Fork 不计入 _biz_in_flight，结构上不存在自锁。

### FT4 Freeze 超时
- 前置 FREEZING｜触发 wait 超时｜Gate OPEN（回滚）｜终态 OPEN。

### FT5 Freeze 中断
- 前置 FREEZING｜触发 KeyboardInterrupt｜Gate OPEN（回滚）｜终态 OPEN。

### FT6 Gate 写入失败
- 前置 FREEZING｜触发赋值异常｜Gate DENIED｜终态 DENIED。

### FT7 协调锁获取失败
- 前置 REQUESTED｜Gate OPEN｜终态 OPEN。

### FT8 协调锁释放失败
- 前置 PARENT_RESTORE｜Gate DENIED｜终态 TERMINATING。

### FT9 父恢复失败
- 前置 PARENT_RESTORE｜Gate DENIED｜终态 DENIED。

### FT10 子重建失败
- 前置 CHILD_REINIT｜Gate 无｜终态 os._exit(1)。

### FT11 finally 二次异常
- 前置 PARENT_RESTORE｜Gate 赋值序决定｜主异常不丢失｜终态 OPEN/DENIED。

### FT12 清理部分成功
- Gate OPEN 或 DENIED｜终态 OPEN/DENIED。

### FT13 嵌套准入
- in_flight +N｜计数配对递减｜终态 OPEN。

### FT14 多线程并发 Fork
- 协调锁串行｜后到者拒绝｜终态 单 fork。

### FT15 Fork 与提交并发
- 独立锁｜提交属业务计数｜无死锁。

### FT16 C 扩展不可见线程
- 无法证明无 C 线程｜DENIED。

### FT17 第三方回调失败
- 附加异常｜父 DENIED/子退出。

### FT18 DENIED 自动重开（已消除）
- DENIED 不可逆，仅人工 manual_reenable()。

### FT19 子进程继承锁
- 子重建新锁对象｜child-local。

### FT20 进程退出前清理失败
- TERMINATING｜os._exit/abort。

### FT21 fork 系统调用失败
- 前置 FROZEN｜os.fork 抛 OSError｜Gate OPEN 或 DENIED。

### FT22 _before_fork 失败
- Gate OPEN（回滚）｜终态 OPEN。

### FT23 _after_in_parent 失败
- Gate 先 DENIED 再尝试恢复。

### FT24 _after_in_child 失败
- 子 os._exit(1)。

### FT25 restore_gate 抛 BaseException
- 先 DENIED（赋值序）｜RESTORE_FAILED。

### FT26 父恢复失败子成功
- 父 DENIED、子正常。

### FT27 父恢复成功子失败
- 父 OPEN、子退出。

### FT28 回调重复执行
- 不注册 at-fork，仅手动｜不存在重复。

### FT29 业务继续被拒绝
- RESTORE_FAILED 后业务入口拒绝。

### FT30 进程终止
- 无法恢复｜os._exit。

## 2. POSIX 崩溃恢复时间线（CR1~CR40）

### CR1~CR16（S1~S8 阶段崩溃，见下简表）
- CR1 S1 open 前崩溃 → NEW
- CR2 S1 write 中 → 半成品 Candidate
- CR3 S1 完成后未 fsync → CANDIDATE_CREATED
- CR4 S2 fsync 前 → CANDIDATE_CREATED
- CR5 S2 fsync 中(EINTR/EIO) → 重读后重试
- CR6 S3 dirfsync 前 → CANDIDATE_CONTENT_DURABLE
- CR7 S4 校验损坏 → 删重 S1
- CR8 S5 write 中 → 半成品 PREPARED
- CR9 S5 EEXIST → 核验归属
- CR10 S6 fsync 中 → 重读重试
- CR11 S7 dirfsync 前 → PREPARED_CONTENT_DURABLE
- CR12 S8 锁前 → PREPARED_ENTRY_DURABLE
- CR13 S9 版本冲突 → QUARANTINED
- CR14 S10 write 中 → 半成品临时文件
- CR15 S11 fsync 中 → 重读重试
- CR16 S12 rename 前 → CURRENT_TMP_VERIFIED

### CR17 S12 rename 中（结果未知）
- rename EINTR｜current 旧或新（未知）｜重读身份｜COMMIT 不确定。

### CR18 S13 fsync 中
- current 新值目录项未持久｜全核验后重试。

### CR19 S13 读到旧 current
- 不推断 rename 未发生｜INDETERMINATE。

### CR20 S13 读到新 current
- 不推断已持久｜INDETERMINATE。

### CR21 S14 write 中
- 半成品回执｜身份确认后删。

### CR22 S14 EEXIST
- 核验四元组｜幂等或 QUARANTINED。

### CR23 S14 fsync 中
- 重读重试。

### CR24 S15 dirfsync 前
- 回执目录项未持久。

### CR25 S15 后崩溃
- current+回执双持久｜DURABLE。

### CR26 S16 释放锁后
- DURABLE。

### CR27 证据冲突（身份不匹配）
- QUARANTINED。

### CR28 并发事务残留
- 判定归属｜RECOVERY_REQUIRED/QUARANTINED。

### CR29 EINTR 后身份变化
- QUARANTINED。

### CR30 设备写缓存
- DEVICE_OR_STORAGE_LIMIT（NOT_VERIFIED）。

### CR31 锁外写入者改 current（新增）
- 检测 inode/SHA 异常｜QUARANTINED。

### CR32 回执被外部替换（新增）
- 身份不符｜QUARANTINED。

### CR33 临时文件被外部删除（新增）
- 身份缺失｜RECOVERY_REQUIRED。

### CR34 回执半成品归属不明（新增）
- 无法确认创建者｜QUARANTINED。

### CR35 rename 后 fsync 前崩溃且重读异常（新增）
- 结果未知 + 持久未知｜INDETERMINATE。

### CR36 多进程并发恢复同一事务（新增）
- advisory 锁竞争｜单恢复者｜其他 QUARANTINED。

### CR37 恢复进程无法确认创建者（新增）
- QUARANTINED。

### CR38 已提交但回执未完成（新增）
- 补回执 AUDIT_PENDING。

### CR39 目录 fsync 失败后盲目重试 rename（反例，新增）
- 禁止盲目重试｜INDETERMINATE。

### CR40 不确定态直接跳 DURABLE（反例，新增）
- 禁止｜保持不确定态。

## 3. rename 结果不确定时间线（RU1~RU25）

- RU1 rename 返回成功 → CONFIRMED_APPLIED，持久 NOT_DURABLE（待 fsync）
- RU2 rename EINTR → OUTCOME_UNKNOWN
- RU3 rename EIO → OUTCOME_UNKNOWN，人工
- RU4 rename ENOSPC → CONFIRMED_NOT_APPLIED
- RU5 rename 其他错误 → 视错误码
- RU6 已生效 + fsync 失败 → CONFIRMED_APPLIED + DURABILITY_UNKNOWN
- RU7 未生效但结果不明 → OUTCOME_UNKNOWN
- RU8 current 已新版本 → CONFIRMED_APPLIED
- RU9 current 仍旧版本 → OUTCOME_UNKNOWN（不推断未发生）
- RU10 current 内容不一致 → QUARANTINED
- RU11 current inode 改变 → 可能已替换，重读
- RU12 current 哈希改变 → QUARANTINED
- RU13 外部写入者恢复期改 current → QUARANTINED
- RU14 rename 前崩溃 → NOT_STARTED
- RU15 rename 后 fsync 前崩溃 → OUTCOME_UNKNOWN
- RU16 fsync 后回执前崩溃 → CONFIRMED_APPLIED + DURABLE_CONFIRMED
- RU17 rename 返回成功但 current 读取失败 → OUTCOME_UNKNOWN
- RU18 rename 后 inode 与记录不符 → QUARANTINED
- RU19 rename 重试前对象身份变化 → QUARANTINED
- RU20 重读得旧值即重试 rename（反例）→ 禁止
- RU21 重读得新值即当已持久（反例）→ 禁止
- RU22 rename EINTR 后盲目重试（反例）→ 禁止
- RU23 rename 结果未知直接标提交成功（反例）→ 禁止
- RU24 rename 已发生但 fsync 失败认定未发生（反例）→ 禁止
- RU25 目录 fsync 失败盲目重 rename（反例）→ 禁止

## 4. 目录持久性不确定时间线（DU1~DU25）

- DU1 S3 dirfsync 成功 → CANDIDATE_ENTRY_DURABLE
- DU2 S3 dirfsync EINTR → DURABILITY_UNKNOWN
- DU3 S3 dirfsync EIO → 人工
- DU4 S3 dirfsync ENOSPC → 人工
- DU5 S7 dirfsync 成功 → PREPARED_ENTRY_DURABLE
- DU6 S7 dirfsync EINTR → 未知
- DU7 S13 dirfsync 成功 → CURRENT_DURABLE_AUDIT_PENDING
- DU8 S13 dirfsync EINTR → 未知
- DU9 S13 dirfsync EIO → 人工
- DU10 S13 dirfsync ENOSPC → 人工
- DU11 S15 dirfsync 成功 → RECEIPT_ENTRY_DURABLE
- DU12 S15 dirfsync EINTR → 未知
- DU13 目录项可见但未持久 → DURABILITY_PENDING
- DU14 目录项可见当已持久（反例）→ 禁止
- DU15 fsync 失败当未发生（反例）→ 禁止
- DU16 重试 dirfsync 成功直接当 DURABLE（反例）→ 禁止（还需回执）
- DU17 dirfsync 重试前对象身份变化 → QUARANTINED
- DU18 dirfsync 重试后状态变化 → 重读
- DU19 dirfsync 期间进程崩溃 → 未知
- DU20 设备写缓存 → NOT_VERIFIED
- DU21 文件系统差异未标注（反例）→ 标注 POSIX/Linux/特定 FS
- DU22 dirfsync 失败不统一处理（反例）→ 统一协议
- DU23 重读 current 当断电持久证明（反例）→ 禁止
- DU24 目录项可见≠断电持久（原则）
- DU25 fsync 重试成功替代完整核验（反例）→ 禁止

## 5. 外部写入者并发时间线（EW1~EW20）

- EW1 同进程线程写 current → 受锁约束
- EW2 合作进程写 current → advisory 约束
- EW3 不合作进程写 current → 不受控
- EW4 管理员改 current → 不受控
- EW5 外部删除临时文件 → 不受控
- EW6 外部创建同名回执 → 不受控
- EW7 外部改变 inode → 检测
- EW8 检查与 rename 之间外部修改 → 锁外 TOCTOU
- EW9 两次读一致但期间有写又改回 → 无法检测
- EW10 哈希一致但被替换 → 概率非零
- EW11 恢复期外部改 current → QUARANTINED
- EW12 外部改回执 → QUARANTINED
- EW13 多进程恢复竞争 → advisory 串行
- EW14 管理员删除锁文件 → 锁失效
- EW15 FS 层异常 → 不受控
- EW16 检测到锁外痕迹 → QUARANTINED
- EW17 无法证明无锁外写入者 → UNKNOWN
- EW18 部署环境存在锁外写入者 → 前提不成立
- EW19 advisory 锁当强制锁（反例）→ 禁止
- EW20 仅凭 _commit_lock 约束所有写入者（反例，05.28 缺陷）→ 撤销

## 6. Fork×POSIX 交叉时间线（FX1~FX30）

- FX1 冻结期间禁止事务提交
- FX2 快照冻结与 candidate 写入互斥
- FX3 fork 后子 FD 状态独立
- FX4 fork 后父事务锁状态
- FX5 快照引用 current revision
- FX6 fork 期间 current 被 rename
- FX7 事务 revision 与 fork_epoch 独立
- FX8 恢复中禁止 fork
- FX9 子初始化失败不清父事务
- FX10 冻结超时不影响已提交事务
- FX11 快照与 txn_id 绑定
- FX12 多线程提交与冻结
- FX13 fork 后父回执幂等
- FX14 子不参与父提交协议
- FX15 fork 期间目录 fsync
- FX16 txn_id 含 instance 分量
- FX17 崩溃后 fork 计数器重置
- FX18 rename 与 fork 目录项可见性
- FX19 父 DENIED 后子状态
- FX20 回滚与冻结锁序
- FX21 fork 时存在未完成提交
- FX22 FREEZING 允许进行中事务完成
- FX23 fork 等待期间提交锁死锁
- FX24 子继承父提交锁状态
- FX25 子使用父 txn_id
- FX26 子重复提交父事务
- FX27 父子共享 current 临时文件
- FX28 父崩溃后子识别事务归属
- FX29 子重建身份失败后恢复事务
- FX30 身份碰撞导致误删半成品

---

## 统计

- Fork 异常：FT1~FT30 = 30 条
- POSIX 崩溃：CR1~CR40 = 40 条
- rename 不确定：RU1~RU25 = 25 条
- 目录持久不确定：DU1~DU25 = 25 条
- 外部写入者：EW1~EW20 = 20 条
- Fork×POSIX：FX1~FX30 = 30 条

**合计：30+40+25+25+20+30 = 170 条独立时间线。**
