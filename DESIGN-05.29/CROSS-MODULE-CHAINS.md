# USIT-STAGE-B-DESIGN-05.29 跨模块调用链（CROSS-MODULE-CHAINS.md）

> 30 条跨模块调用链（L1~L30），覆盖 Fork、Gate、身份、事务、current、fsync、rename、回执、恢复之间的依赖。

## 跨模块链路

### L1 业务入口 → enter_op → Gate 检查 → 资源操作
- 路径：业务 API → enter_op → Gate OPEN 检查 → 资源操作。不变量：I-05/13。

### L2 fork_entry → 准入 → 协调锁 → freeze → fork
- 路径：fork 请求 → 准入 → 串行化 → 冻结 → fork。不变量：I-12/13。

### L3 freeze → DRAINING → 业务 in_flight 归零
- 路径：冻结 → 排空 → FROZEN。不变量：I-13。

### L4 fork_event_id → snapshot_id → 子消费
- 路径：fork 事件 → 快照 → 子进程。不变量：I-55。

### L5 tree_id + 父 instance + 父 seq → fork_event_id
- 路径：身份分配 → fork 事件唯一。不变量：I-22。

### L6 process_instance_id → 子重建 → 初始化标志
- 路径：子 fork 后重建 instance_id，标志阻断未完成业务。不变量：I-23。

### L7 process_tree_id → 同树继承
- 路径：树身份跨代继承。不变量：I-31。

### L8 parent_instance_id → 父子关联
- 路径：子记录父 instance 作历史。不变量：I-18。

### L9 Candidate 写入 → 六字段 → 五谓词 → S4
- 路径：候选对象完整性。不变量：I-40/42。

### L10 PREPARED → S9 五条件 → current 比对
- 路径：提交前校验。不变量：I-40/41。

### L11 S10 临时文件 → S11 fsync → S12 rename
- 路径：临时文件 → rename 替换。不变量：I-32/39。

### L12 S12 rename → S13 dirfsync → 目录项持久
- 路径：rename → 持久化。不变量：I-32/33。

### L13 S13 → S14 回执 → S15 → DURABLE
- 路径：current 持久 → 回执 → 完成。不变量：I-35/33。

### L14 回执四元组 → 幂等
- 路径：(txn_id,revision,inode,SHA) 幂等。不变量：I-35/28。

### L15 全局提交锁 → 读改写 → 锁内 TOCTOU 防护
- 路径：锁保护全程。不变量：I-12/41。

### L16 advisory lock → 合作写入者约束
- 路径：flock/fcntl 约束合作者。不变量：I-61。

### L17 锁外写入者 → 检测 → QUARANTINED
- 路径：inode/SHA 异常 → 隔离。不变量：I-62。

### L18 EINTR → 重读核验 → 重试/QUARANTINED
- 路径：中断 → 未知 → 重读。不变量：I-37/38。

### L19 恢复安全门 15 项 → 删除/重试/回滚
- 路径：恢复前验证。不变量：I-40/42/45。

### L20 Gate 状态 → 统一拒绝门
- 路径：Gate 驱动拒绝。不变量：I-05/07。

### L21 Gate 恢复失败 → 先 DENIED → 再恢复
- 路径：赋值序保证。不变量：I-05/15。

### L22 协调锁释放 → 最外层 finally → 终止兜底
- 路径：锁释放不跳过。不变量：I-10/11。

### L23 快照 revision → 提交递增 → 旧快照失效
- 路径：版本绑定。不变量：I-49。

### L24 fd_generation → 槽位递增
- 路径：FD 代际。不变量：I-27。

### L25 revision 与 fork_epoch 命名空间隔离
- 路径：独立递增。不变量：I-51。

### L26 随机源不可用 → DENY
- 路径：不降级。不变量：I-25。

### L27 计数器溢出 → DENY
- 路径：不回绕。不变量：I-30。

### L28 设备写缓存 → 持久边界 → NOT_VERIFIED
- 路径：硬件边界。不变量：I-33/37。

### L29 崩溃恢复 → 重读 current+回执 → 状态判定
- 路径：磁盘证据。不变量：I-50。

### L30 审计回执 → current SHA → 可追溯
- 路径：审计链。不变量：I-35/50。

---

## 统计

- 跨模块链路：L1~L30 = 30 条。
