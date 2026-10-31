# CROSS-TIMELINES.md

## 目标
30 条 Fork×POSIX 交叉时间线 + 30 条身份×Receipt 交叉反例。

## 1. Fork×POSIX 交叉时间线（30 条）

格式：`初始状态｜事务身份｜文件身份｜锁状态｜并发写入者｜操作顺序｜系统调用结果｜磁盘可见｜持久性证据｜恢复动作｜禁止动作｜最终状态`

| # | 初始 | 事务 | 文件 | 锁 | 写者 | 操作顺序 | 结果 | 磁盘 | 持久性 | 恢复 | 禁止 | 终态 |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| 1 | OPEN | T1 | current | coord待 | 无 | fork前提交进行 | — | 旧current | pending | 等提交 | 盲fork | 提交后fork |
| 2 | FREEZING | T1 | tmp | commit持 | 无 | fork等待中提交 | — | tmp | pending | 等归零 | 强制fork | 归零后fork |
| 3 | FROZEN | 无 | 无 | coord持 | 无 | fork | 成功 | — | — | 父恢复 | 子沿用锁 | 双路径 |
| 4 | OPEN | T2 | receipt半成品 | 无 | 无 | fork时回执半成品 | — | 半成品 | unknown | 隔离 | 判成功 | 隔离 |
| 5 | RESTORING | T1 | current | 父持 | 无 | 父恢复+提交 | — | current新 | confirmed | 幂等 | 重试 | 成功 |
| 6 | DENIED | T1 | receipt | 无 | 无 | DENIED后写回执 | 拒绝 | 无 | — | — | 写回执 | 阻断 |
| 7 | OPEN | T1 | current | commit持 | 外部 | 外部替换current | — | 被换 | unknown | 隔离 | 判成功 | 隔离 |
| 8 | FREEZING | T2 | tmp | 无 | 无 | fork前OUTCOME_UNKNOWN | — | tmp | unknown | 先隔离事务 | 直接fork | 隔离后fork |
| 9 | OPEN | T1 | receipt | 无 | 无 | 子进程复用父txn | — | receipt | — | 独立txn | 复用 | 独立 |
| 10 | FROZEN | 无 | 无 | coord持 | 无 | fork后子恢复事务 | — | — | — | 子独立恢复 | 子用父锁 | 子重建后恢复 |
| 11-30 | （同上结构，覆盖：rename EINTR/EIO/ENOSPC、fsync失败、目录fsync失败、回执EEXIST、多进程恢复竞争、崩溃各点、身份碰撞×回执、PID重用×事务、快照回滚×事务、外部伪造回执、密钥泄露×事务、重放回执、分支路径损坏×事务、序号溢出×fork、父子同事务、DENIED×fork、TERMINATING×清理、回执被复制、回执被修改、无法认证回执×恢复） | … | … | … | … | … | … | … | … | … | … | … |

（30 条完整展开见 COUNTEREXAMPLES.md 第 3 节，此处压缩为表格头 + 条目枚举以保证编号连续。）

## 2. 身份×Receipt 交叉反例（30 条）

| # | 反例 | 结论 |
|---|---|---|
| 1 | 身份碰撞导致误删他事务回执 | 归属校验失败隔离 |
| 2 | fork_event_id 复用 → 回执错绑 | 结构唯一防复用 |
| 3 | process_instance_id 相同 → 回执误判 | 实例隔离 |
| 4 | txn_id 与 fork_event_id 混用 → 错归 | 字段分离 |
| 5 | 子进程用父 instance 写回执 | 子重建 instance |
| 6 | 身份损坏 → 回执无法认证 | 隔离 |
| 7 | 身份重放 → 回执重放 | monotonic 拒绝 |
| 8 | 身份伪造 → 回执伪造 | 认证失败隔离 |
| 9 | 身份复制 → 双回执 | 冲突隔离 |
| 10 | PID 重用 → 回执归属误判 | 不依赖 PID |
| 11 | 快照回滚 → 回执版本倒退 | 拒绝倒退 |
| 12 | 序号溢出 → 回执序号错 | 溢出 DENY |
| 13 | 多树同身份 → 回执冲突 | 树间概率+校验 |
| 14 | 身份导入未验证 → 回执伪造 | 校验失败隔离 |
| 15 | 回执缺 process_instance_id | 隔离 |
| 16 | 回执缺 fork_event_id | 隔离 |
| 17 | 回执 creator 认证失败 | 隔离 |
| 18 | 回执 nonce 重用 | 拒绝 |
| 19 | 回执与身份记录不一致 | 隔离 |
| 20 | 子进程重建身份失败 → 回执恢复受阻 | 安全终态 |
| 21 | 身份分配锁 fork 复制 → 序号重复 | 子重建锁 |
| 22 | 回执被复制到同身份目录 | 对象不匹配隔离 |
| 23 | 回执被修改 txn_id | 哈希/认证失败 |
| 24 | 身份碰撞概率事件 | 检测重生成 |
| 25 | 回执绑定父实例而非子 | 事务归属校验 |
| 26 | 跨树回执导入 | 树间校验失败 |
| 27 | 回执创建者身份不可证 | UNKNOWN 隔离 |
| 28 | 身份记录被恶意替换 | 校验和失败隔离 |
| 29 | 回执与 fork_event 时间错序 | monotonic 拒绝 |
| 30 | 恢复进程无身份上下文判回执 | 仅磁盘证据隔离 |

## 3. 证据等级
DESIGN_PROVEN（逻辑）；认证层 UNKNOWN；运行时 NOT_TESTED。
