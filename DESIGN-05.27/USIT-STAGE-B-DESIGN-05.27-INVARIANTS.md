# USIT-STAGE-B-DESIGN-05.27 全局不变量（INVARIANTS）

> 独立材料。共 50 条不变量（I-01~I-50）。每条含：定义、适用状态、证据等级、反例、违反后的拒绝策略。

| # | 定义 | 适用状态 | 证据等级 | 反例 | 拒绝策略 |
|---|------|---------|---------|------|---------|
| I-01 | 唯一 fork 入口 fork_entry | 全状态 | DESIGN_PROPOSAL | 多入口 | 手动接管 |
| I-02 | 不注册 at-fork 回调 | 全状态 | DESIGN_PROPOSAL | 注册回调 | 显式手动 |
| I-03 | L0 唯一释放者父进程 | Fork | DESIGN_PROPOSAL | 子释放 | pid==0 分流 |
| I-04 | 子进程不释放继承锁 | Child | DESIGN_PROPOSAL | 释放继承锁 | 赋值新对象 |
| I-05 | Gate 恢复失败 fail-closed | RESTORE_FAILED | DESIGN_PROPOSAL | fail-open | DENIED+终止 |
| I-06 | 子初始化失败 os._exit(1) | Child | DESIGN_PROPOSAL | 返回业务栈 | 强制终止 |
| I-07 | 多线程进程拒绝 fork | 准入 | DESIGN_PROPOSAL | 并发 fork | fail-closed |
| I-08 | 主线程 ≠ 单线程 | 准入 | STATIC_INFERENCE | 混淆 | 单独判定 |
| I-09 | fork 前禁止持业务锁 | Fork | DESIGN_PROPOSAL | 持锁 fork | DENY |
| I-10 | 父 L0 不双释放 | Fork | DESIGN_PROPOSAL | 双释放 | 互斥分支 |
| I-11 | 父 L0 不漏释放 | Fork | DESIGN_PROPOSAL | 漏释放 | except/finally 覆盖 |
| I-12 | 锁顺序 L0→L1→L2→L3 | 全状态 | DESIGN_PROPOSAL | 逆序 | 顺序检查 |
| I-13 | FREEZING 后无迟到登记 | FREEZING | DESIGN_PROPOSAL | 迟到 | 同临界区 |
| I-14 | 持锁 wait 释放重取 | FREEZING | DESIGN_PROPOSAL | 持锁 wait | 释放/重取 |
| I-15 | 锁释放 ≠ Gate 恢复 | Fork | DESIGN_PROPOSAL | 当恢复 | _close_fork_parent |
| I-16 | 第三方 C fork 不覆盖 | 准入 | OFFICIAL_RUNTIME_DOC | 当覆盖 | UNSUPPORTED/DENY |
| I-17 | posix_spawn 不覆盖 | 准入 | OFFICIAL_RUNTIME_DOC | 当覆盖 | UNSUPPORTED/DENY |
| I-18 | 嵌入/子解释器不覆盖 | 准入 | OFFICIAL_RUNTIME_DOC | 当覆盖 | UNSUPPORTED/DENY |
| I-19 | fork_epoch 进程树内严格 | Epoch | DESIGN_PROPOSAL | 跨分支碰撞 | 父侧全局分配 |
| I-20 | fork_event_id 结构+概率 | Identity | DESIGN_PROPOSAL | 兄弟碰撞 | 父侧 epoch 分量 |
| I-21 | snapshot_id = fork_event_id | Snapshot | DESIGN_PROPOSAL | 复用 | 事件绑定 |
| I-22 | 无字段承诺全局/跨重启唯一 | Identity | DESIGN_PROPOSAL | 全局宣称 | 精确分级 |
| I-23 | instance_id 实例内共享 | Identity | DESIGN_PROPOSAL | 当唯一 | 非唯一区分 |
| I-24 | txn_id 绑定事务可持久 | 事务 | DESIGN_PROPOSAL | 不绑定 | 绑定+幂等 |
| I-25 | revision 严格递增 | 事务 | DESIGN_PROPOSAL | 回绕 | 结构 |
| I-26 | epoch 递增在 L0 内 | Epoch | DESIGN_PROPOSAL | 锁外递增 | 移入临界区 |
| I-27 | 失败 fork 消耗 epoch | Epoch | DESIGN_PROPOSAL | 复用 | 消耗 |
| I-28 | epoch 溢出 DENY | Epoch | DESIGN_PROPOSAL | 回绕 | DENY |
| I-29 | 随机源不可用降级/拒绝 | Identity | DESIGN_PROPOSAL | 弱随机 | 降级/拒绝 |
| I-30 | 恶意复制身份无认证 | Identity | UNKNOWN | 复制 | 不防伪造 |
| I-31 | DURABLE 仅最终态 | 事务 | DESIGN_PROPOSAL | 进行中 | 16 态分离 |
| I-32 | rename 原子 ≠ 断电持久 | S12/S13 | AUTHORITATIVE_SPEC | 当持久 | fsync(dirfd) |
| I-33 | 目录可见 ≠ 断电持久 | S3/S7/S13/S15 | OS_DOCUMENTATION | 当持久 | fsync 成功 |
| I-34 | 文件 fsync ≠ 目录 fsync | fsync | AUTHORITATIVE_SPEC | 混用 | 分步 |
| I-35 | current 持久 ≠ 回执持久 | S13~S15 | DESIGN_PROPOSAL | 混用 | 分步 |
| I-36 | 旧值不推断 rename 未发生 | S12/S13 | DESIGN_PROPOSAL | 直接重试 | 核验身份 |
| I-37 | fsync(dirfd) 成功是目录持久必要条件 | fsync | OS_DOCUMENTATION | 省略 | 强制 dirfsync |
| I-38 | fsync 重试成功不直接 DURABLE | 恢复 | DESIGN_PROPOSAL | 跳步 | S13+S15 分步 |
| I-39 | 恢复前重读磁盘 | 恢复 | DESIGN_PROPOSAL | 信任内存 | 强制重读 |
| I-40 | 恢复前核验身份/大小/SHA | 恢复 | DESIGN_PROPOSAL | 未核验 | 强制核验 |
| I-41 | 恢复前核验 txn/rev/父版本 | 恢复 | DESIGN_PROPOSAL | 未核验 | 强制核验 |
| I-42 | 恢复前核验六字段/五谓词 | 恢复 | DESIGN_PROPOSAL | 未核验 | 强制核验 |
| I-43 | 状态不确定禁止新事务 | 恢复 | DESIGN_PROPOSAL | 启动新事务 | 禁止 |
| I-44 | 身份冲突 QUARANTINED | 恢复 | DESIGN_PROPOSAL | 不隔离 | QUARANTINED |
| I-45 | 删除/重试需 15 项证明 | 恢复 | DESIGN_PROPOSAL | 未证明删除 | 禁止 |
| I-46 | S16 锁释放非持久化 | S16 | DESIGN_PROPOSAL | 当持久化 | 区分 |
| I-47 | Windows 目录持久 NOT_VERIFIED | Windows | OS_DOCUMENTATION | 标 PASS | NOT_VERIFIED |
| I-48 | MANIFEST 哈希最终重算 | 交付 | DESIGN_PROPOSAL | 旧哈希 | 重算 |
| I-49 | 不用 Git SHA-1 冒充 SHA-256 | 交付 | DESIGN_PROPOSAL | 冒充 | 强制 SHA-256 |
| I-50 | ZIP 独立解压真实执行 | 交付 | DESIGN_PROPOSAL | 只看中央目录 | 真实解压 |

---

## 统计

**全局不变量合计：50 条（I-01~I-50）。** 每条含定义、适用状态、证据等级、反例、拒绝策略，无重复编号。
