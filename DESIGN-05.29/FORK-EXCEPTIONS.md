# USIT-STAGE-B-DESIGN-05.29 — Fork 异常矩阵（FORK-EXCEPTIONS.md）

> B29-01 支撑文档。重建完整异常矩阵（F05 + 20 项异常）。

每条记录：`初始状态｜触发事件｜执行步骤｜持有锁｜锁所有者｜in_flight｜资源所有权｜异常类型｜Gate 状态｜是否允许业务继续｜最终终态`。

## 1. 异常矩阵（20 项）

### E1 Fork 入口失败
- 初始 OPEN｜触发 fork_entry 前置失败｜步骤 准入检查｜持锁 无｜锁主 无｜in_flight 0｜资源 无｜异常 ForkDenied｜Gate OPEN｜业务 是｜终态 OPEN。

### E2 普通操作登记失败
- 初始 OPEN｜触发 enter_op 中 Gate≠OPEN｜步骤 准入｜持锁 _admit_lock｜锁主 业务线程｜in_flight 不变｜资源 无｜异常 GateClosed｜Gate 非 OPEN｜业务 否｜终态 拒绝。

### E3 in_flight 自锁（05.28 缺陷，本轮已消除）
- 初始 —｜触发 Fork 计入业务计数｜步骤 —｜说明：本轮 Fork 不计入 _biz_in_flight，此异常**结构上不存在**｜终态 不存在。

### E4 Freeze 超时
- 初始 FREEZING｜触发 wait 超时｜持锁 _admit_lock（wait 释放）｜in_flight >0｜异常 FreezeTimeout｜Gate OPEN（回滚）｜业务 是（回滚后）｜终态 OPEN。

### E5 Freeze 被中断
- 初始 FREEZING｜触发 KeyboardInterrupt｜持锁 _admit_lock｜in_flight >0｜异常 中断｜Gate OPEN（回滚）｜业务 是｜终态 OPEN。

### E6 Gate 写入失败
- 初始 FREEZING｜触发 _gate 赋值异常｜持锁 _admit_lock｜异常 写入失败｜Gate 未知→DENIED｜业务 否｜终态 DENIED。

### E7 锁获取失败（_fork_coord_lock）
- 初始 REQUESTED｜触发 acquire 抛错｜持锁 无｜异常 获取失败｜Gate OPEN｜业务 是｜终态 OPEN。

### E8 锁释放失败（_fork_coord_lock）
- 初始 PARENT_RESTORE｜触发 release 抛错｜持锁 持有中｜异常 释放失败｜Gate DENIED｜业务 否｜终态 TERMINATING。

### E9 父进程恢复失败
- 初始 PARENT_RESTORE｜触发 _after_in_parent 抛错｜持锁 协调锁｜异常 恢复失败｜Gate DENIED｜业务 否｜终态 DENIED。

### E10 子进程重建失败
- 初始 CHILD_REINIT｜触发 _child_reinit 抛错｜持锁 无｜异常 重建失败｜Gate 无（子退出）｜业务 否｜终态 os._exit(1)。

### E11 finally 二次异常
- 初始 PARENT_RESTORE｜触发 finally 内抛错｜异常 附加异常｜Gate 由赋值序决定｜业务 否｜终态 OPEN/DENIED（主异常不丢失）。

### E12 清理部分成功
- 初始 异常清理｜触发 清理部分完成｜异常 附加异常｜Gate OPEN 或 DENIED｜业务 视 Gate｜终态 OPEN/DENIED。

### E13 嵌套准入
- 初始 OPEN｜触发 嵌套 enter_op｜持锁 无｜in_flight +N｜异常 无｜Gate OPEN｜业务 是｜终态 OPEN（计数配对递减）。

### E14 多线程并发 Fork
- 初始 OPEN｜触发 两线程 fork_entry｜持锁 协调锁串行｜异常 后到者拒绝｜Gate 单 fork 冻结｜业务 否（冻结后）｜终态 单 fork 成功、另一拒绝。

### E15 Fork 与提交并发
- 初始 OPEN｜触发 fork + 提交同时｜持锁 独立锁｜异常 无｜Gate FREEZING（提交属业务计数）｜业务 提交完成才冻结｜终态 无死锁。

### E16 C 扩展不可见线程
- 初始 OPEN｜触发 无法证明无 C 线程｜异常 ForkDenied｜Gate DENIED｜业务 否｜终态 DENIED。

### E17 第三方回调失败
- 初始 PARENT_RESTORE/CHILD_REINIT｜触发 回调抛错｜异常 附加异常｜Gate DENIED（父）/子退出｜业务 否｜终态 DENIED/os._exit。

### E18 DENIED 自动重开（05.28 缺陷，本轮已消除）
- 初始 DENIED｜触发 普通恢复路径｜说明：DENIED 不可逆，仅 manual_reenable() 人工授权可恢复，自动路径**不包含** DENIED→OPEN｜终态 保持 DENIED。

### E19 子进程继承锁
- 初始 CHILD_REINIT｜触发 子继承父锁对象｜处理 重建新锁对象｜异常 无｜Gate 子 OPEN（重建后）｜业务 否（重建前）｜终态 child-local。

### E20 进程退出前清理失败
- 初始 TERMINATING｜触发 清理抛错｜异常 附加异常｜Gate TERMINATING｜业务 否｜终态 os._exit/abort。

## 2. 结论

20 项异常路径全部有确定终态；自锁（E3）与 DENIED 自动重开（E18）在本轮**结构性消除**。协议层 DESIGN_PROVEN，运行时 NOT_TESTED。
