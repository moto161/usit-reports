# USIT-STAGE-B-DESIGN-05.27 反例（COUNTEREXAMPLES）

> 独立材料。共 40 个反例（CID1~CID40）。每个含：错误方案、具体反例过程、违反的不变量、修复措施、修复后验证边界。

## 1. Fork 反例（CID1~CID15）

| # | 错误方案 | 反例过程 | 违反不变量 | 修复措施 | 验证边界 |
|---|---------|---------|-----------|---------|---------|
| CID1 | 子进程释放 coord 锁 | 子执行 release(L0) 但未持有 | I-03 | pid==0 分流 | 运行时 NOT_TESTED |
| CID2 | 子释放继承 gate 锁 | 子 release 继承锁 | I-04 | 赋值新对象 | NOT_TESTED |
| CID3 | 父双释放 L0 | except+finally 都释放 | I-10 | 互斥分支 | NOT_TESTED |
| CID4 | fail-open 吞异常 | restore 失败仅设标志继续业务 | I-05 | fail-closed | NOT_TESTED |
| CID5 | 子失败返回业务栈 | os._exit 前 return 0 | I-06 | os._exit(1) | NOT_TESTED |
| CID6 | 主线程当单线程 | active_count==1 误判 | I-08 | 单独判定 | UNKNOWN(C 线程) |
| CID7 | 多线程并发 fork | 两线程同时 fork | I-07 | fail-closed 拒绝 | NOT_TESTED |
| CID8 | at-fork 回调重复 | 自动+手动都恢复 | I-02 | 不注册回调 | NOT_TESTED |
| CID9 | 依赖回调异常传播 | 假设回调异常被捕获 | I-02 | 手动接管 | NOT_TESTED |
| CID10 | 跨进程释放锁 | 子释放父锁 | I-03 | 锁所有权表 | NOT_TESTED |
| CID11 | 锁释放当 Gate 恢复 | finally 只释放锁 | I-15 | _close_fork_parent | NOT_TESTED |
| CID12 | _fork_allowed 当唯一防线 | 调用方捕获异常绕过 | I-05 | assert_gate_open 强制 | NOT_TESTED |
| CID13 | 永久禁 fork 当永久禁业务 | 只关 fork 不关业务 | I-05 | RESTORE_FAILED→DENIED | NOT_TESTED |
| CID14 | EINTR 当未执行 | 盲目重试不核验 | I-39 | 重试前核验 | NOT_VERIFIED |
| CID15 | 子进程继承锁 acquire | 子 acquire 继承锁 | I-04 | 不 acquire | NOT_TESTED |

## 2. 身份反例（CID16~CID30）

| # | 错误方案 | 反例过程 | 违反不变量 | 修复措施 | 验证边界 |
|---|---------|---------|-----------|---------|---------|
| CID16 | 全局唯一宣称 | generation+pid+epoch 当全局唯一 | I-22 | 精确分级 | NOT_VERIFIED |
| CID17 | fork_epoch 当全局唯一 | 跨进程 epoch=1 碰撞 | I-19 | 进程树内严格 | NOT_TESTED |
| CID18 | 兄弟进程碰撞（05.26）| 兄弟各自 epoch=1 | I-20 | 父侧全局分配 | NOT_TESTED |
| CID19 | 不同父进程碰撞 | 两父 epoch=1 | I-20 | 随机分量 | 概率 |
| CID20 | PID 当永久唯一 | PID 重用碰撞 | I-23 | instance_id | VERIFIED_FACT(PID 重用) |
| CID21 | 系统重启复用身份 | 旧 instance_id 复用 | I-22 | 新随机 | 跨重启不承诺 |
| CID22 | 随机当严格唯一 | UUID"通常不重复"当证明 | I-22 | 概率性声明 | 概率 |
| CID23 | 协调器不可用 | 方案 C 失效 | I-22 | 不采用协调器 | 无此依赖 |
| CID24 | 随机源弱 | 弱随机碰撞 | I-29 | 降级/拒绝 | NOT_VERIFIED |
| CID25 | 恶意复制身份 | 进程复制 instance_id | I-30 | 不防伪造 | UNKNOWN |
| CID26 | epoch 溢出回绕 | 计数器回绕 ABA | I-28 | DENY | NOT_TESTED |
| CID27 | 子 epoch 重置 0 | 子重置计数器 | I-19 | 继承值 | NOT_TESTED |
| CID28 | txn_id 不绑定事务 | 重复提交 | I-24 | 绑定+幂等 | NOT_TESTED |
| CID29 | revision 回绕 | 版本回绕冲突 | I-25 | 严格递增 | NOT_TESTED |
| CID30 | 旧快照复用 | epoch 不匹配仍用 | I-21 | 拒绝 | NOT_TESTED |

## 3. POSIX/Windows/交付反例（CID31~CID40）

| # | 错误方案 | 反例过程 | 违反不变量 | 修复措施 | 验证边界 |
|---|---------|---------|-----------|---------|---------|
| CID31 | 仅凭 current 重读迁移 | rename 后未 fsync 就迁移 | I-36 | fsync 成功才迁 | NOT_VERIFIED |
| CID32 | 目录可见当持久 | 目录项可见误判持久 | I-33 | 区分可见/持久 | NOT_VERIFIED |
| CID33 | 盲目重试 rename | S13 失败直接 rename | I-36 | 核验身份 | NOT_VERIFIED |
| CID34 | 旧值直接重试 S12 | 读到旧值续 S12 | I-36 | 核验身份 | NOT_VERIFIED |
| CID35 | 盲目回滚 | 不确定态回滚 | I-43 | 禁止 | NOT_VERIFIED |
| CID36 | 盲目删除 Candidate | 未证明归属删除 | I-45 | 15 项证明 | NOT_VERIFIED |
| CID37 | 状态不确定启动新事务 | INDETERMINATE 开新事务 | I-43 | 禁止 | NOT_VERIFIED |
| CID38 | fsync 重试成功直接 DURABLE | 只 S13 成功标 DURABLE | I-38 | S13+S15 分步 | NOT_VERIFIED |
| CID39 | Windows 目录持久标 PASS | 套用 POSIX 结论 | I-47 | NOT_VERIFIED | NOT_VERIFIED |
| CID40 | 自审当独立复审 | 执行器自评当通过 | I-50 | 等 ChatGPT | — |

---

## 统计

**反例合计：40 个（CID1~CID40）。** 每个含错误方案、反例过程、违反不变量、修复措施、验证边界，无重复编号。
