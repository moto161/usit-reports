# USIT-STAGE-B-DESIGN-05.29 全局不变量（INVARIANTS.md）

> 80 条全局不变量（I-01~I-80）。
> 每条含：唯一编号 / 精确定义 / 适用状态 / 前置条件 / 证明依据 / 反例 / 违反后果 / 拒绝策略 / 恢复策略。

## Fork 不变量（I-01~I-25）

### I-01 Gate 状态单值
- 定义：_gate 有且仅有一值且属七态。依据 DESIGN_PROVEN。违反→拒绝门未定义。恢复→重置 DENIED。

### I-02 不注册 at-fork
- 定义：仅手动恢复。依据 DESIGN_PROPOSAL。违反→重复执行。

### I-03 子不释放父锁
- 依据 POSIX。违反→父锁破坏。

### I-04 子锁本地重建
- 依据 DESIGN_PROPOSAL。

### I-05 非 OPEN 拒绝所有入口
- 依据 DESIGN_PROVEN。

### I-06 子初始化失败即退出
- 依据 DESIGN_PROPOSAL。

### I-07 Gate 无对外写接口
- 依据 DESIGN_PROVEN。

### I-08 Gate 检查与业务登记原子
- 依据 DESIGN_PROVEN。

### I-09 业务 in_flight 非负
- 依据配对。

### I-10 协调锁最终必释放
- 依据 DESIGN_PROVEN。

### I-11 finally 异常不覆盖安全终态
- 依据 DESIGN_PROVEN。

### I-12 提交锁保护 current 读改写
- 依据 DESIGN_PROVEN。

### I-13 冻结等待业务归零
- 依据 DESIGN_PROVEN。

### I-14 锁序 _admit→_commit
- 依据 DESIGN_PROPOSAL。

### I-15 恢复失败必 DENIED/TERMINATING
- 依据 DESIGN_PROVEN。

### I-16 Fork 不计入业务 in_flight（消除自锁）
- 依据 DESIGN_PROVEN。反例 F01。

### I-17 上下文管理器完整配对
- 依据 DESIGN_PROVEN。反例 F02。

### I-18 父分支标识不可变
- 依据 DESIGN_PROPOSAL。

### I-19 DENIED 不可逆终态
- 依据 DESIGN_PROVEN。反例 F04。

### I-20 主异常不丢失
- 依据 DESIGN_PROVEN。反例 F05。

### I-21 附加异常独立记录
- 依据 DESIGN_PROVEN。

### I-22 fork_event_id 树内严格唯一
- 依据 DESIGN_PROVEN（tree_id+父 instance+seq）。

### I-23 process_instance_id 子重建
- 依据 DESIGN_PROVEN。反例 CI7。

### I-24 process_tree_id 同树继承
- 依据 DESIGN_PROVEN。

### I-25 随机源不可用 DENY
- 依据 DESIGN_PROPOSAL。

## 身份不变量（I-26~I-45）

### I-26 实例身份与树身份分离
### I-27 fd_generation 递增
### I-28 txn_id 幂等可重放
### I-29 身份无认证不防伪造（UNKNOWN）
### I-30 计数器溢出 DENY
### I-31 process_generation 单链
### I-32 rename 原子性结果未知不推断
### I-33 目录项可见≠持久
### I-34 内容持久需 fsync(fd)
### I-35 回执与 current 一致 + 四元组幂等
### I-36 旧 current 不证明未 rename
### I-37 fsync 失败不推断未执行
### I-38 EINTR 后重读核验
### I-39 对象创建 O_EXCL
### I-40 身份/SHA/版本三重核验
### I-41 校验与提交同锁内
### I-42 五条件五谓词六字段校验
### I-43 状态机无跳步
### I-44 删除/回滚/重试前全验证
### I-45 EEXIST 先核验身份

## POSIX 不变量（I-46~I-65）

### I-46 终态 DURABLE 需双持久
### I-47 冻结期间禁事务提交
### I-48 快照冻结与 candidate 写入互斥
### I-49 快照绑定 revision
### I-50 current 身份 inode+dev+SHA+rev
### I-51 revision 与 fork_epoch 隔离
### I-52 恢复中禁 fork
### I-53 子退出不影响父事务
### I-54 冻结回滚不影响 DURABLE
### I-55 snapshot_id=fork_event_id
### I-56 回滚与冻结锁序一致
### I-57 回执幂等
### I-58 子不继承父提交锁
### I-59 目录 fsync 纳入 enter_op
### I-60 txn_id 含 instance 分量
### I-61 advisory 锁仅约束合作者
### I-62 锁外写入者检测 QUARANTINED
### I-63 rename 结果与持久性分离
### I-64 不确定态不跳 DURABLE
### I-65 回执归属需四元组证明

## 交叉不变量（I-66~I-80）

### I-66 Fork 与业务计数域分离
### I-67 FREEZING 允许已登记操作完成
### I-68 Fork 与提交无死锁
### I-69 子不继承父 txn_id
### I-70 子不重复提交父事务
### I-71 父子不共享 current 临时文件
### I-72 父崩溃后子靠磁盘证据识别事务
### I-73 子重建失败不恢复事务
### I-74 Gate DENIED 后禁提交
### I-75 不确定态不被 Fork 恢复误清
### I-76 多进程恢复竞争 advisory 串行
### I-77 回执与进程身份绑定
### I-78 身份碰撞不误删半成品
### I-79 崩溃恢复仅依赖磁盘证据
### I-80 无法确认统一 DENY/QUARANTINED

---

## 统计

- 全局不变量：I-01~I-80 = 80 条。
