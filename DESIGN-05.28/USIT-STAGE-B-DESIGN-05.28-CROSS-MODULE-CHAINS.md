# USIT-STAGE-B-DESIGN-05.28 跨模块链路（CROSS-MODULE-CHAINS）

> 独立材料。共 30 条跨模块链路（L1~L30），覆盖 Fork、Gate、FD、Candidate、PREPARED、current、临时文件、事务身份、fsync、rename、回执、恢复、审计之间的依赖关系。
> 每条含：链路路径、关键依赖、失效后果、对应不变量。

## 跨模块链路

### L1 业务入口 → enter_op → Gate 检查 → FD 操作
- 路径：业务 API → enter_op（原子准入）→ Gate OPEN 检查 → 资源操作。
- 依赖：Gate 状态与 in_flight 登记原子化。
- 失效后果：TOCTOU 竞态，FD 操作绕过冻结。
- 不变量：I-05/13。

### L2 fork_entry → enter_op → 协调锁 → freeze → fork
- 路径：fork 请求 → 准入 → 串行化 → 冻结 → 系统 fork。
- 依赖：协调锁串行 + 冻结等待 in_flight 归零。
- 失效后果：并发 fork 或冻结不完整。
- 不变量：I-12/13/14。

### L3 freeze → FROZEN → make_snapshot → fork_event_id
- 路径：冻结完成 → 生成快照 → 绑定 fork_event_id。
- 依赖：快照与 fork 事件一一对应。
- 失效后果：快照身份错乱。
- 不变量：I-55。

### L4 fork_event_id → snapshot_id → 子进程消费
- 路径：fork_event_id 作为 snapshot_id 传给子进程。
- 依赖：snapshot_id 恒等于 fork_event_id。
- 失效后果：快照与 fork 脱钩。
- 不变量：I-55。

### L5 父 instance_id + 父 seq → fork_event_id 分支唯一
- 路径：身份分配 → fork_event_id 进程树唯一。
- 依赖：父内 seq 严格 + 父 instance 前缀。
- 失效后果：兄弟/多代碰撞。
- 不变量：I-22。

### L6 process_instance_id → txn_id 绑定
- 路径：实例身份 → 事务身份绑定。
- 依赖：txn_id 含 instance 分量。
- 失效后果：跨实例事务混淆。
- 不变量：I-60。

### L7 Candidate 写入 → S1 O_EXCL → S2 fsync(fd) → S3 fsync(dir)
- 路径：候选对象创建 → 内容持久 → 目录项持久。
- 依赖：文件 fsync 与目录 fsync 分离。
- 失效后果：持久性误判。
- 不变量：I-34/33。

### L8 Candidate → 六字段 → 五谓词 → S4 校验
- 路径：候选对象字段完整性校验。
- 依赖：六字段五谓词完整定义。
- 失效后果：损坏 Candidate 被接受。
- 不变量：I-40/42。

### L9 PREPARED 写入 → S6/S7 fsync → S9 五条件
- 路径：PREPARED 持久 → 提交前校验。
- 依赖：五条件与 current 比对。
- 失效后果：版本/事务/哈希冲突未检。
- 不变量：I-40/41。

### L10 S9 五条件 → S10 临时文件 → S11 fsync → S12 rename
- 路径：校验通过 → 临时文件 → rename 替换。
- 依赖：临时文件持久后 rename，rename 前重读校验。
- 失效后果：损坏内容成为 current。
- 不变量：I-32/39/40。

### L11 S12 rename → S13 fsync(dir) → 目录项持久
- 路径：rename 生效 → 目录持久化。
- 依赖：rename 原子性 + dirfsync。
- 失效后果：current 目录项断电丢失。
- 不变量：I-32/33。

### L12 S13 → S14 回执 → S15 fsync(dir) → DURABLE
- 路径：current 持久 → 回执 → 回执目录持久 → 完成。
- 依赖：回执与 current 一致性。
- 失效后果：审计链断裂。
- 不变量：I-35/33。

### L13 回执唯一键 (txn_id, revision) → 幂等
- 路径：回执身份 → 幂等提交。
- 依赖：唯一键唯一 + 内容匹配。
- 失效后果：重复提交或身份冲突。
- 不变量：I-35/28。

### L14 全局提交锁 → S8~S16 读改写 → 锁内 TOCTOU 防护
- 路径：锁保护 current 读改写全程。
- 依赖：校验与 rename 同锁内。
- 失效后果：并发修改 TOCTOU。
- 不变量：I-12/41。

### L15 锁丢失/崩溃 → fcntl 自动释放 → 磁盘证据判定
- 路径：崩溃后锁释放 → 靠磁盘状态恢复。
- 依赖：不假设锁仍持有。
- 失效后果：死锁或误判持锁。
- 不变量：I-12。

### L16 EINTR → 重读核验 → 重试/QUARANTINED
- 路径：中断 → 状态未知 → 重读 → 决策。
- 依赖：EINTR 不推断执行结果。
- 失效后果：误判操作已/未执行。
- 不变量：I-37/38。

### L17 恢复安全门 15 项 → 删除/重试/回滚/新事务
- 路径：所有恢复动作前逐项验证。
- 依赖：任一不可证则 DENY/QUARANTINED。
- 失效后果：破坏已有提交或恢复证据。
- 不变量：I-40/42/45。

### L18 Gate 状态 → assert_gate_open → 统一拒绝门
- 路径：Gate 状态驱动所有入口拒绝。
- 依赖：Gate 无对外写接口。
- 失效后果：调用方绕过拒绝。
- 不变量：I-05/07。

### L19 Gate 恢复失败 → 先 DENIED → 再尝试恢复 → 终态
- 路径：恢复异常不绕过 DENIED。
- 依赖：赋值序保证。
- 失效后果：带着非终态继续业务。
- 不变量：I-05/15。

### L20 协调锁释放 → 最外层 finally → 终止兜底
- 路径：锁释放不被前序异常跳过。
- 依赖：锁释放在最外层 finally。
- 失效后果：锁永久持有。
- 不变量：I-10/11。

### L21 快照 revision → 提交递增 → 旧快照失效
- 路径：快照绑定 revision，提交后失效。
- 依赖：revision 单调递增。
- 失效后果：旧快照误用。
- 不变量：I-49。

### L22 fd_generation → 槽位分配递增 → FD 身份
- 路径：FD 槽位代际递增防混淆。
- 依赖：每次分配递增。
- 失效后果：旧 FD 引用误指新对象。
- 不变量：I-27。

### L23 process_generation → 单链代际 → 不用于唯一性
- 路径：fork 代际计数。
- 依赖：仅单链语义。
- 失效后果：被误当全局唯一。
- 不变量：I-31。

### L24 revision 与 fork_epoch 命名空间隔离
- 路径：对象版本与 fork 计数分离。
- 依赖：独立递增。
- 失效后果：版本混淆。
- 不变量：I-51。

### L25 随机源不可用 → DENY → 不降级
- 路径：身份生成失败拒绝。
- 依赖：不降级弱随机。
- 失效后果：可预测身份。
- 不变量：I-25。

### L26 计数器溢出 → DENY → 不回绕
- 路径：seq/epoch 溢出拒绝。
- 依赖：不回绕。
- 失效后果：身份重复。
- 不变量：I-30。

### L27 设备写缓存 → 持久性边界 → NOT_VERIFIED
- 路径：fsync 成功受设备限制。
- 依赖：硬件行为。
- 失效后果：断电丢已"持久"数据。
- 不变量：I-33/37。

### L28 Windows ReplaceFileW → 1175/1176/1177 → 部分失败中间态
- 路径：Windows 替换协议边界。
- 依赖：官方文档 vs 运行时。
- 失效后果：跨卷/中间态未知。
- 不变量：I-33（Windows NOT_VERIFIED）。

### L29 崩溃恢复 → 重读 current+回执 → 状态判定
- 路径：恢复读取全对象 → 判定终态。
- 依赖：身份+SHA+rev 三重核验。
- 失效后果：误判恢复完成。
- 不变量：I-50。

### L30 审计回执 → current SHA 一致 → 可追溯
- 路径：回执记录 current SHA → 审计链。
- 依赖：回执 SHA == current SHA。
- 失效后果：审计断裂。
- 不变量：I-35/50。

---

## 统计

- 跨模块链路：L1~L30 = 30 条。
