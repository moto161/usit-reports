# USIT-STAGE-B-DESIGN-05.28 全局不变量（INVARIANTS）

> 独立材料。共 60 条全局不变量（I-01~I-60）。
> 每条含：唯一编号 / 精确定义 / 适用状态 / 前置条件 / 证明依据 / 反例 / 违反后果 / 恢复策略。
> 证据等级：DESIGN_PROVEN / VERIFIED_STATIC / STATIC_INFERENCE / NOT_VERIFIED / NOT_TESTED。

## 全局不变量

### I-01 Gate 状态单值
- 定义：任意时刻 `_gate` 有且仅有一个状态值，且属于七态集合。
- 适用：全生命周期。前置：模块初始化完成。
- 依据：单例变量 + 受控写点。反例：Gate 双值/未初始化。
- 违反后果：拒绝门行为未定义。恢复：重置为 DENIED。

### I-02 不注册 at-fork 回调（唯一责任机制）
- 定义：不调用 `os.register_at_fork()`，仅手动恢复，杜绝重复执行。
- 依据：DESIGN_PROPOSAL。反例：自动+手动重复。
- 违反后果：恢复动作重复执行。恢复：移除 at-fork 注册。

### I-03 子进程不释放父进程锁
- 定义：子进程不得 acquire/release 继承的父锁。
- 依据：POSIX fork 锁语义。反例：子释放父锁。
- 违反后果：父锁状态破坏。恢复：子重建本地锁。

### I-04 子进程锁本地重建
- 定义：fork 后子进程为每把锁赋值新对象。
- 依据：DESIGN_PROPOSAL。反例：子沿用继承锁。
- 违反后果：跨进程锁误用。恢复：重赋锁对象。

### I-05 非 OPEN 态拒绝所有受控入口
- 定义：Gate≠OPEN 时 enter_op 抛 GateClosed。
- 依据：DESIGN_PROVEN（原子准入）。反例：非 OPEN 仍放行。
- 违反后果：坏状态业务继续。恢复：置 DENIED。

### I-06 子进程初始化失败即退出
- 定义：`_after_in_child` 异常 → os._exit(1)。
- 依据：DESIGN_PROPOSAL。反例：子带坏状态继续。
- 违反后果：子进程坏状态。恢复：重启子进程。

### I-07 Gate 无对外写接口
- 定义：`_gate` 写点仅 freeze/restore_gate/deny_fork。
- 依据：DESIGN_PROVEN。反例：调用方改 Gate。
- 违反后果：绕过拒绝门。恢复：审计写点。

### I-08 Gate 检查与 in_flight 登记原子
- 定义：二者同一 `_admit_lock` 临界区。
- 依据：DESIGN_PROVEN。反例：TOCTOU。
- 违反后果：冻结不完整。恢复：加锁。

### I-09 in_flight 非负
- 定义：in_flight ≥ 0 恒成立。
- 依据：配对 enter/exit。反例：负值。
- 违反后果：冻结永不归零。恢复：审计配对。

### I-10 协调锁最终必被释放
- 定义：fork_entry 正常/异常路径协调锁均释放或进程终止。
- 依据：DESIGN_PROVEN（最外层 finally）。反例：锁永久持有。
- 违反后果：后续 fork 死锁。恢复：终止。

### I-11 finally 二次异常不覆盖安全终态
- 定义：Gate 终态由赋值序决定，不依赖异常传播。
- 依据：DESIGN_PROVEN。反例：finally 异常覆盖 DENIED 赋值。
- 违反后果：非终态继续。恢复：独立记录异常。

### I-12 全局提交锁保护 current 读改写全程
- 定义：S8~S16 持 `_commit_lock`。
- 依据：DESIGN_PROVEN。反例：锁外写 current。
- 违反后果：并发修改。恢复：加锁。

### I-13 冻结等待 in_flight 归零
- 定义：freeze 在 `_cond` 等待 in_flight==0。
- 依据：DESIGN_PROVEN。反例：冻结时在途操作未完成。
- 违反后果：快照不完整。恢复：等待/超时回滚。

### I-14 锁顺序 _admit_lock → _commit_lock
- 定义：禁止反向获取，防死锁。
- 依据：DESIGN_PROPOSAL。反例：反向锁序。
- 违反后果：死锁。恢复：统一锁序。

### I-15 恢复失败必进 DENIED 或 TERMINATING
- 定义：restore_gate 失败 → DENIED；无法释放 → TERMINATING。
- 依据：DESIGN_PROVEN。反例：恢复失败仍 OPEN。
- 违反后果：坏状态业务。恢复：置 DENIED。

### I-16 父进程 L0 不漏释放不双释放
- 定义：协调锁释放恰好一次。
- 依据：DESIGN_PROVEN。反例：漏/双释放。
- 违反后果：死锁或未定义。恢复：审计。

### I-17 快照冻结完成后才 fork
- 定义：FROZEN 后 os.fork()。
- 依据：DESIGN_PROPOSAL。反例：未冻结 fork。
- 违反后果：快照不一致。恢复：重冻结。

### I-18 fork_event_id 父分支标识不可变
- 定义：父 instance 分量只读。
- 依据：DESIGN_PROPOSAL。反例：篡改分支标识。
- 违反后果：追溯断裂。恢复：重建。

### I-19 进程内计数器严格单调
- 定义：seq/epoch/revision 不回绕。
- 依据：DESIGN_PROVEN。反例：回绕。
- 违反后果：身份重复。恢复：DENY 溢出。

### I-20 身份概率与结构严格区分
- 定义：结构性唯一与概率性唯一分开声明。
- 依据：DESIGN_PROVEN。反例：随机标严格。
- 违反后果：唯一性误承诺。恢复：更正声明。

### I-21 跨重启/全局唯一不承诺
- 定义：无持久化/协调器，明确不承诺。
- 依据：DESIGN_PROPOSAL。反例：隐式承诺。
- 违反后果：身份误用。恢复：显式标注。

### I-22 fork_event_id 进程树内严格唯一
- 定义：同树内兄弟/多代 fork_event_id 不碰撞。
- 依据：DESIGN_PROVEN（instance 前缀 + 父 seq）。反例：CE-ID1。
- 违反后果：身份碰撞。恢复：加结构分量。

### I-23 PID 重用不混淆身份
- 定义：instance_id 独立于 pid。
- 依据：DESIGN_PROVEN。反例：仅 pid 作身份。
- 违反后果：旧身份误判。恢复：instance_id 隔离。

### I-24 跨重启身份不复用
- 定义：重启换新 instance_id，seq 归零。
- 依据：DESIGN_PROPOSAL。反例：复用旧身份。
- 违反后果：跨重启碰撞。恢复：新 instance_id。

### I-25 随机源不可用 DENY 不降级
- 定义：os.urandom 失败 → DENY。
- 依据：DESIGN_PROPOSAL。反例：降级弱随机。
- 违反后果：可预测身份。恢复：拒绝。

### I-26 快照绑定进程树，跨树拒绝
- 定义：snapshot_id 校验 instance_id。
- 依据：DESIGN_PROPOSAL。反例：跨树快照复用。
- 违反后果：快照错用。恢复：校验拒绝。

### I-27 fd_generation 槽位递增
- 定义：槽位每次分配 generation+1。
- 依据：DESIGN_PROPOSAL。反例：复用不递增。
- 违反后果：FD 混淆。恢复：递增。

### I-28 txn_id 幂等且可重放校验
- 定义：同 txn_id 重复提交被拒绝。
- 依据：DESIGN_PROVEN。反例：重复提交生效。
- 违反后果：重复事务。恢复：幂等校验。

### I-29 身份无认证不防伪造（UNKNOWN）
- 定义：无认证机制，恶意复制身份不防。
- 依据：UNKNOWN。反例：假设防伪造。
- 违反后果：身份被冒充。恢复：显式 UNKNOWN。

### I-30 计数器溢出 DENY 不回绕
- 定义：seq/epoch 达上限 → DENY。
- 依据：DESIGN_PROVEN。反例：回绕。
- 违反后果：身份重复。恢复：拒绝。

### I-31 process_generation 仅单链语义
- 定义：代际计数不用于全局唯一。
- 依据：DESIGN_PROPOSAL。反例：当全局唯一。
- 违反后果：代际混淆。恢复：限定语义。

### I-32 rename 原子性，结果未知不推断
- 定义：rename 成功/失败原子，EINTR 后结果未知。
- 依据：VERIFIED_STATIC（POSIX）。反例：CE-P4/P5。
- 违反后果：误判 rename。恢复：重读身份。

### I-33 目录项可见 ≠ 断电持久
- 定义：dirfsync 成功才持久（受设备限制）。
- 依据：VERIFIED_STATIC。反例：CE-P5。
- 违反后果：断电丢数据。恢复：dirfsync。

### I-34 文件内容持久需 fsync(fd)
- 定义：内容持久靠文件 fsync。
- 依据：VERIFIED_STATIC。反例：写后当持久。
- 违反后果：内容丢失。恢复：fsync(fd)。

### I-35 回执与 current 一致且唯一键幂等
- 定义：回执 SHA==current SHA，唯一键幂等。
- 依据：DESIGN_PROVEN。反例：CE-R1/R2。
- 违反后果：审计断裂。恢复：QUARANTINED。

### I-36 旧 current 不证明 rename 未发生
- 定义：读到旧值不推断 rename 未执行。
- 依据：DESIGN_PROVEN。反例：CE-P4。
- 违反后果：盲目重试 rename。恢复：全核验。

### I-37 fsync 失败不推断未执行（EINTR/EIO）
- 定义：fsync 返回错误后状态未知。
- 依据：VERIFIED_STATIC。反例：CE-P3。
- 违反后果：误判持久。恢复：重读核验。

### I-38 EINTR 后必须重读核验再重试
- 定义：EINTR 重试前重读对象身份。
- 依据：DESIGN_PROVEN。反例：盲目重试。
- 违反后果：身份变化未检。恢复：重读。

### I-39 对象创建用 O_EXCL 排他
- 定义：Candidate/PREPARED/临时文件/回执排他创建。
- 依据：DESIGN_PROVEN。反例：覆盖创建。
- 违反后果：覆盖他人对象。恢复：O_EXCL。

### I-40 身份/SHA/版本三重核验
- 定义：恢复/提交前核验身份+SHA+rev。
- 依据：DESIGN_PROVEN。反例：单重核验。
- 违反后果：对象错认。恢复：三重核验。

### I-41 版本校验与提交同锁内（防 TOCTOU）
- 定义：S9 校验与 S12 rename 同 `_commit_lock`。
- 依据：DESIGN_PROVEN。反例：CE-P27。
- 违反后果：并发修改。恢复：锁内。

### I-42 五条件五谓词六字段完整校验
- 定义：提交前全部校验通过。
- 依据：DESIGN_PROVEN。反例：缺校验。
- 违反后果：坏提交。恢复：补校验。

### I-43 状态机无跳步
- 定义：状态转移必经中间态。
- 依据：DESIGN_PROVEN。反例：CE-P30。
- 违反后果：证据断裂。恢复：严格转移。

### I-44 删除/回滚/重试前全验证
- 定义：15 项安全门全过才动作。
- 依据：DESIGN_PROVEN。反例：盲目删除。
- 违反后果：破坏提交。恢复：QUARANTINED。

### I-45 EEXIST 先核验身份再决定
- 定义：排他创建冲突先核验归属。
- 依据：DESIGN_PROVEN。反例：CE-P7。
- 违反后果：删错对象。恢复：QUARANTINED。

### I-46 终态 DURABLE 需 current+回执双持久
- 定义：current 目录项 + 回执目录项均持久。
- 依据：DESIGN_PROVEN。反例：缺回执当完成。
- 违反后果：审计链断。恢复：补回执。

### I-47 冻结期间禁止事务提交
- 定义：FREEZING/FROZEN 拒绝事务 enter_op。
- 依据：DESIGN_PROVEN。反例：CE-X3。
- 违反后果：快照不一致。恢复：拒绝。

### I-48 快照冻结与 candidate 写入互斥
- 定义：冻结等待 in_flight，未完成写入不计入。
- 依据：DESIGN_PROVEN。反例：半成品入快照。
- 违反后果：快照损坏。恢复：等待归零。

### I-49 快照绑定 revision，提交后失效
- 定义：revision 递增使旧快照失效。
- 依据：DESIGN_PROVEN。反例：旧快照仍有效。
- 违反后果：旧快照误用。恢复：失效检查。

### I-50 current 身份识别用 inode+dev+SHA+rev
- 定义：四者共同识别 current 身份。
- 依据：DESIGN_PROVEN。反例：靠文件名。
- 违反后果：rename 后错认。恢复：身份核验。

### I-51 revision 与 fork_epoch 命名空间隔离
- 定义：对象版本与 fork 计数独立。
- 依据：DESIGN_PROVEN。反例：CE-ID15。
- 违反后果：版本混淆。恢复：隔离。

### I-52 恢复中禁止 fork
- 定义：RESTORING/RESTORE_FAILED 拒绝 fork。
- 依据：DESIGN_PROVEN。反例：CE-X11。
- 违反后果：恢复中断。恢复：拒绝。

### I-53 子进程退出不影响父事务
- 定义：父子事务状态独立。
- 依据：DESIGN_PROPOSAL。反例：CE-X9。
- 违反后果：父事务丢失。恢复：独立。

### I-54 冻结回滚不影响 DURABLE 事务
- 定义：Gate 回滚仅 Gate，不影响已提交。
- 依据：DESIGN_PROPOSAL。反例：CE-X8。
- 违反后果：已提交被破坏。恢复：隔离。

### I-55 snapshot_id 恒等于 fork_event_id
- 定义：快照与 fork 事件一一对应。
- 依据：DESIGN_PROPOSAL。反例：CE-ID13。
- 违反后果：快照脱钩。恢复：绑定。

### I-56 回滚与冻结锁序一致防死锁
- 定义：统一 _admit_lock → _commit_lock。
- 依据：DESIGN_PROVEN。反例：CE-X6。
- 违反后果：死锁。恢复：统一锁序。

### I-57 回执幂等防重复提交
- 定义：已存在匹配回执则跳过写入。
- 依据：DESIGN_PROVEN。反例：重复回执。
- 违反后果：重复审计。恢复：幂等。

### I-58 子进程不继承父提交锁所有权
- 定义：子无父 _commit_lock 所有权。
- 依据：DESIGN_PROPOSAL。反例：CE-X4。
- 违反后果：锁混乱。恢复：子独立。

### I-59 目录 fsync 经 enter_op 纳入冻结
- 定义：目录 fsync 是受控操作，冻结等待。
- 依据：DESIGN_PROVEN。反例：CE-X10。
- 违反后果：冻结漏操作。恢复：纳入 enter_op。

### I-60 txn_id 含 instance 分量
- 定义：txn_id 绑定 process_instance_id。
- 依据：DESIGN_PROVEN。反例：CE-X1。
- 违反后果：跨实例混淆。恢复：绑定。

---

## 统计

- 全局不变量：I-01~I-60 = 60 条。
