# USIT-STAGE-B-DESIGN-05.29 反例（COUNTEREXAMPLES.md）

> 全部反例。共 140 个唯一编号：
> CI1~CI30（身份 30，见 IDENTITY-COUNTEREXAMPLES.md）、CE-RC1~CE-RC20（回执归属冲突 20）、
> CE-IDEM1~CE-IDEM20（重复提交/幂等 20）、CE-X1~CE-X30（身份×事务交叉 30）、CE-P1~CE-P40（POSIX 反例 40）。

> 身份反例 CI1~CI30 详见 IDENTITY-COUNTEREXAMPLES.md，此处不重复。

## 1. 回执归属冲突反例（CE-RC1~CE-RC20）

### CE-RC1 回执四元组冲突
- 场景：同 (txn_id,revision) 但 inode/SHA 不同；结论 QUARANTINED。

### CE-RC2 回执 SHA 与 current 不一致
- 场景：回执指向旧 current；结论 QUARANTINED。

### CE-RC3 回执半成品短写
- 场景：S14 短写崩溃；结论 身份确认后删重建。

### CE-RC4 回执重复创建
- 场景：崩溃重放；结论 幂等跳过。

### CE-RC5 回执覆盖他人回执
- 场景：O_TRUNC；结论 禁止覆盖。

### CE-RC6 回执目录项未持久当完成
- 场景：S15 未做；结论 必须 S15。

### CE-RC7 回执身份不可确认
- 场景：inode 不符；结论 QUARANTINED。

### CE-RC8 回执 txn_id 缺失
- 场景：缺字段；结论 校验拒绝。

### CE-RC9 回执 revision 缺失
- 场景：缺字段；结论 校验拒绝。

### CE-RC10 回执 parent_revision 冲突
- 场景：父版本不符；结论 QUARANTINED。

### CE-RC11 回执仅凭文件名判定归属（05.28 不足）
- 场景：文件名相同但内容不同；结论 需四元组证明。

### CE-RC12 回执仅凭 txn_id 字符串判定
- 场景：txn_id 相同但非本次；结论 需四元组。

### CE-RC13 回执内容部分匹配判定归属
- 场景：部分匹配；结论 需全匹配。

### CE-RC14 回执被外部替换
- 场景：身份变化；结论 QUARANTINED。

### CE-RC15 回执被复制伪造
- 场景：无认证；结论 UNKNOWN。

### CE-RC16 恢复进程无法确认创建者
- 场景：创建者未知；结论 QUARANTINED。

### CE-RC17 多事务同 revision
- 场景：revision 冲突；结论 QUARANTINED。

### CE-RC18 辅助元数据当所有权证明（反例）
- 场景：仅创建时间/大小；结论 不能单独作证。

### CE-RC19 已提交但回执未完成当 DURABLE
- 场景：缺回执；结论 AUDIT_PENDING。

### CE-RC20 EEXIST 盲目删除（反例）
- 场景：EEXIST 即删；结论 先核验归属。

## 2. 重复提交及幂等性反例（CE-IDEM1~CE-IDEM20）

### CE-IDEM1 无唯一键重复提交
- 场景：无 (txn_id,revision)；结论 需唯一键。

### CE-IDEM2 唯一键缺 inode
- 场景：缺 inode/dev；结论 需四元组。

### CE-IDEM3 唯一键缺 SHA
- 场景：缺哈希；结论 需四元组。

### CE-IDEM4 匹配则幂等成功
- 场景：四元组全匹配；结论 幂等。

### CE-IDEM5 不匹配则覆盖（反例）
- 场景：不匹配覆盖；结论 QUARANTINED。

### CE-IDEM6 短写后重复
- 场景：半成品后重试；结论 身份确认后重建。

### CE-IDEM7 fsync 失败重复
- 场景：fsync 失败重试；结论 重读核验后重试。

### CE-IDEM8 回执与 current 不一致重复
- 场景：SHA 不一致；结论 QUARANTINED。

### CE-IDEM9 多进程重复提交
- 场景：并发提交；结论 advisory 串行 + 幂等。

### CE-IDEM10 崩溃后重复提交
- 场景：崩溃重放；结论 幂等识别。

### CE-IDEM11 txn_id 跨实例重复
- 场景：两实例同 txn_id；结论 txn_id 含 instance 分量。

### CE-IDEM12 revision 跨实例重复
- 场景：两实例同 revision；结论 instance 绑定。

### CE-IDEM13 幂等条件不完整
- 场景：缺 inode/SHA；结论 四元组完整。

### CE-IDEM14 重复提交误删他人回执
- 场景：删错回执；结论 核验归属。

### CE-IDEM15 幂等判定忽略内容完整性
- 场景：半成品当完成；结论 需内容完整。

### CE-IDEM16 幂等判定忽略文件类型
- 场景：目录当回执；结论 需 regular file。

### CE-IDEM17 幂等判定忽略并发修改
- 场景：回执被并发改；结论 QUARANTINED。

### CE-IDEM18 已提交事务重复提交
- 场景：重复提交已 DURABLE；结论 幂等拒绝。

### CE-IDEM19 回执唯一键跨重启不持久
- 场景：重启后唯一键丢失；结论 txn_id 持久化。

### CE-IDEM20 幂等协议缺 QUARANTINED 分支
- 场景：冲突无处理；结论 增加 QUARANTINED。

## 3. 身份×事务交叉反例（CE-X1~CE-X30）

### CE-X1 txn_id 不含 instance 分量
- 场景：跨实例 txn 混淆；结论 txn_id 绑定 instance。

### CE-X2 快照绑定 revision 失效未检测
- 场景：旧快照未失效；结论 revision 递增失效。

### CE-X3 冻结期间事务提交未阻断
- 场景：FREEZING 提交继续；结论 enter_op 阻断。

### CE-X4 子进程参与父提交
- 场景：子操作父 _commit_lock；结论 不继承所有权。

### CE-X5 revision 与 fork_epoch 混淆
- 场景：混用；结论 独立命名空间。

### CE-X6 回滚与冻结死锁
- 场景：锁序错误；结论 _admit→_commit 固定。

### CE-X7 快照与 txn_id 解绑
- 场景：snapshot_id 独立；结论 =fork_event_id。

### CE-X8 冻结超时影响已提交
- 场景：回滚影响 DURABLE；结论 仅 Gate。

### CE-X9 子初始化失败清父事务
- 场景：连带清父；结论 独立。

### CE-X10 目录 fsync 冻结不等待
- 场景：冻结漏操作；结论 enter_op 纳入。

### CE-X11 恢复中 fork 未阻断
- 场景：RESTORING fork；结论 拒绝。

### CE-X12 父 DENIED 影响子
- 场景：传导到子；结论 独立 Gate。

### CE-X13 崩溃后 fork 计数器不重置
- 场景：seq 未归零；结论 归零+新 instance。

### CE-X14 rename 后子读旧 current 不校验
- 场景：无身份校验；结论 inode+dev+SHA。

### CE-X15 身份生成失败静默继续事务
- 场景：生成失败仍提交；结论 DENY。

### CE-X16 回执唯一键跨实例不唯一
- 场景：同键跨实例；结论 instance 分量。

### CE-X17 快照跨树传递
- 场景：非继承树快照；结论 拒绝。

### CE-X18 提交引用错误 epoch
- 场景：epoch 错位；结论 独立。

### CE-X19 恢复读 current 不核身份
- 场景：只读内容；结论 三重核验。

### CE-X20 回执绑定错误 current 版本
- 场景：revision 错；结论 校验。

### CE-X21 fork 时未完成提交
- 场景：fork 与提交并发；结论 提交属业务计数。

### CE-X22 FREEZING 允许事务完成
- 场景：进行中事务完成；结论 允许（已登记）。

### CE-X23 fork 等待提交锁死锁
- 场景：锁序；结论 独立锁无死锁。

### CE-X24 子继承父提交锁状态
- 场景：子继承；结论 不继承。

### CE-X25 子使用父 txn_id
- 场景：子复用父 txn；结论 子不继承 txn_id。

### CE-X26 子重复提交父事务
- 场景：子重提交；结论 子无父 txn 归属。

### CE-X27 父子共享 current 临时文件
- 场景：共享；结论 子独立，不共享。

### CE-X28 父崩溃后子识别事务归属
- 场景：父崩溃；结论 磁盘证据 + tree_id。

### CE-X29 子重建身份失败后恢复事务
- 场景：重建失败；结论 os._exit，不恢复。

### CE-X30 身份碰撞误删半成品
- 场景：碰撞删错；结论 四元组核验。

## 4. POSIX 反例（CE-P1~CE-P40）

### CE-P1 S4 校验未过进 PREPARED
- CE-P2 S2/S3 混态
- CE-P3 EINTR 当未执行
- CE-P4 S12 旧 current 推断未 rename
- CE-P5 S13 新 current 推断已持久
- CE-P6 重试 fsync 直接 DURABLE
- CE-P7 EEXIST 盲目删
- CE-P8 两次读一致当无并发
- CE-P9 rename 盲目重试
- CE-P10 回执普通覆盖
- CE-P11 回执缺幂等
- CE-P12 S10 无独立态
- CE-P13 临时文件名无唯一性
- CE-P14 rename 前未校验五条件
- CE-P15 目录 fsync 不统一
- CE-P16 删 Candidate 不验身份
- CE-P17 回滚不验归属
- CE-P18 新事务不查残留
- CE-P19 短写当完整
- CE-P20 文件/目录 fsync 混淆
- CE-P21 rename 结果未知当未发生
- CE-P22 仅凭 current 可读启动新事务
- CE-P23 设备写缓存未考虑
- CE-P24 文件系统差异未标注
- CE-P25 回执与 current 不一致未检
- CE-P26 锁外写入者未检测
- CE-P27 校验与提交间 TOCTOU
- CE-P28 临时文件 rename 前不校验
- CE-P29 半成品回执不验身份即删
- CE-P30 状态跳步
- CE-P31 提交锁当强制锁（05.28 缺陷）
- CE-P32 advisory 锁约束锁外写入者（05.28 缺陷）
- CE-P33 rename 结果与持久性混淆（05.28 缺陷）
- CE-P34 不确定态直接跳 DURABLE
- CE-P35 仅凭文件名判断对象身份
- CE-P36 仅凭进程内变量恢复磁盘事务
- CE-P37 回执半成品归属证明不足（05.28 缺陷）
- CE-P38 两次读/inode/哈希一致当无锁外写入者
- CE-P39 EEXIST 误删其他事务文件
- CE-P40 无法建立排他性仍继续提交

---

## 统计

- 身份反例：CI1~CI30 = 30 个（见 IDENTITY-COUNTEREXAMPLES.md）
- 回执归属冲突：CE-RC1~CE-RC20 = 20 个
- 重复提交/幂等：CE-IDEM1~CE-IDEM20 = 20 个
- 身份×事务：CE-X1~CE-X30 = 30 个
- POSIX：CE-P1~CE-P40 = 40 个

**合计：30+20+20+30+40 = 140 个独立反例。**
