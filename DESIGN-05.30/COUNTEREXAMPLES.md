# COUNTEREXAMPLES.md

## 目标
全局反例汇总（交叉时间线完整展开 + 身份×Receipt 反例 + Receipt 反例）。

## 1. Fork×POSIX 交叉时间线完整展开（30 条）

格式：`初始状态｜事务身份｜文件身份｜锁状态｜并发写入者｜操作顺序｜系统调用结果｜磁盘可见｜持久性证据｜恢复动作｜禁止动作｜最终状态`

| # | 初始 | 事务 | 文件 | 锁 | 写者 | 操作顺序 | 结果 | 磁盘 | 持久性 | 恢复 | 禁止 | 终态 |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| 1 | OPEN | T1 | current | coord待 | 无 | fork前提交进行 | rename前 | 旧current | pending | 等提交完成 | 盲fork | 提交后fork |
| 2 | FREEZING | T1 | tmp | commit持 | 无 | fork等待中提交 | 进行中 | tmp | pending | 等归零 | 强制fork | 归零后fork |
| 3 | FROZEN | 无 | 无 | coord持 | 无 | fork | 成功 | — | — | 父恢复 | 子沿用锁 | 双路径 |
| 4 | OPEN | T2 | receipt半 | 无 | 无 | fork时回执半成品 | — | 半成品 | unknown | 隔离 | 判成功 | 隔离 |
| 5 | RESTORING | T1 | current | 父持 | 无 | 父恢复+提交 | 成功 | current新 | confirmed | 幂等 | 重试 | 成功 |
| 6 | DENIED | T1 | receipt | 无 | 无 | DENIED后写回执 | 拒绝 | 无 | — | — | 写回执 | 阻断 |
| 7 | OPEN | T1 | current | commit持 | 外部 | 外部替换current | — | 被换 | unknown | 隔离 | 判成功 | 隔离 |
| 8 | FREEZING | T2 | tmp | 无 | 无 | fork前OUTCOME_UNKNOWN | — | tmp | unknown | 先隔离事务 | 直接fork | 隔离后fork |
| 9 | OPEN | T1 | receipt | 无 | 无 | 子复用父txn | — | receipt | — | 独立txn | 复用 | 独立 |
| 10 | FROZEN | 无 | 无 | coord持 | 无 | fork后子恢复事务 | — | — | — | 子独立恢复 | 子用父锁 | 子重建后恢复 |
| 11 | OPEN | T1 | current | 无 | 无 | rename EINTR | EINTR | 未知 | unknown | 确认结果 | 盲重试 | 隔离 |
| 12 | OPEN | T1 | current | 无 | 无 | rename EIO | EIO | 未知 | unknown | 隔离 | 盲重试 | 隔离 |
| 13 | OPEN | T1 | current | 无 | 无 | rename ENOSPC | ENOSPC | 旧current | — | 报告 | 盲重试 | 失败 |
| 14 | OPEN | T1 | current | 无 | 无 | rename成功fsync(fd)失败 | fd失败 | current新 | fd unknown | fsync重试 | 判未发生 | 隔离 |
| 15 | OPEN | T1 | current | 无 | 无 | fsync(dirfd)失败 | dir失败 | current新 | dir unknown | dir fsync重试 | 判未发生 | 隔离 |
| 16 | OPEN | T1 | receipt | 无 | 无 | 回执EEXIST完整 | EEXIST | receipt | confirmed | 幂等 | 删除 | 成功(可认证) |
| 17 | OPEN | T1 | receipt | 无 | 无 | 回执EEXIST半成品 | EEXIST | 半成品 | unknown | 隔离 | 删除 | 隔离 |
| 18 | OPEN | T1 | receipt | 无 | 多进程 | 多进程恢复同事务 | 竞争 | receipt | — | 排他锁 | 并发恢复 | 单恢复者 |
| 19 | OPEN | T1 | current | 无 | 无 | rename前崩溃 | 崩溃 | 旧current | — | 重新开始 | 判成功 | 未提交 |
| 20 | OPEN | T1 | current | 无 | 无 | rename后fsync前崩溃 | 崩溃 | current新 | unknown | 确认+fsync | 判未发生 | 隔离 |
| 21 | OPEN | T1 | current | 无 | 无 | fsync后回执前崩溃 | 崩溃 | current新 | confirmed | 补回执 | 判失败 | 幂等 |
| 22 | OPEN | T1 | receipt | 无 | 无 | 回执写入中崩溃 | 崩溃 | 半成品 | unknown | 隔离 | 判成功 | 隔离 |
| 23 | OPEN | T1 | receipt | 无 | 无 | 回执写后崩溃 | 崩溃 | 完整 | confirmed | 幂等 | 判失败 | 成功(可认证) |
| 24 | OPEN | T1 | receipt | 无 | 无 | 身份碰撞×回执 | — | receipt | — | 检测重生成 | 沿用 | 重生成 |
| 25 | OPEN | T1 | current | 无 | 无 | PID重用×事务 | — | current | — | instance区分 | 用PID | 正确 |
| 26 | OPEN | T1 | receipt | 无 | 无 | 快照回滚×事务 | — | receipt旧 | — | 拒绝倒退 | 重放 | 拒绝 |
| 27 | OPEN | T1 | receipt | 无 | 外部 | 外部伪造回执 | — | receipt伪 | — | 认证失败 | 判成功 | 隔离 |
| 28 | OPEN | T1 | receipt | 无 | 无 | 密钥泄露×事务 | — | receipt | — | 轮换隔离 | 继续用旧钥 | 隔离 |
| 29 | OPEN | T1 | receipt | 无 | 无 | 重放回执 | — | receipt旧 | — | nonce拒绝 | 判成功 | 拒绝 |
| 30 | OPEN | T1 | receipt | 无 | 无 | 分支路径损坏×事务 | — | receipt | — | 校验失败 | 判成功 | 隔离 |

## 2. 身份×Receipt 反例（30 条）

见 CROSS-TIMELINES.md 第 2 节（30 条，编号连续）。

## 3. Receipt 反例（30 条）

见 RECEIPT-IDEMPOTENCY.md 第 3 节（30 条）。

## 4. 身份反例（35 条）

见 IDENTITY-COUNTEREXAMPLES.md（35 条）。

## 5. 死锁反例（20 条）

见 LOCK-ORDER-PROOF.md（20 条）。

## 汇总
- Fork×POSIX 时间线：30
- 身份×Receipt 反例：30
- Receipt 反例：30
- 身份反例：35
- 死锁反例：20
