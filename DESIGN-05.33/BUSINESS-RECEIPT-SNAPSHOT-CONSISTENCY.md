# BUSINESS-RECEIPT-SNAPSHOT-CONSISTENCY.md

## 目标
重新定义 Fork 前置条件，明确业务/提交/Receipt/快照一致性边界。

## 1. 状态清单与归零判定

| 状态 | 必须归零 | 允许保留 | 必须隔离 | 阻止 Fork |
|---|---|---|---|---|
| 运行中业务请求 | ✅ | — | — | ✅（未归零前） |
| 执行中提交 | ✅ | — | — | ✅ |
| 写入/更新中 Receipt | ✅ | — | — | ✅ |
| 已创建未完成 Receipt | — | — | ✅（半成品隔离） | 视认证 |
| 已提交未确认事务 | — | — | ✅（认证缺失时） | 视认证 |
| QUARANTINED Receipt | — | — | ✅ | 否（不参与快照） |
| 状态未知/恢复中事务 | — | — | ✅ | ✅（隔离后放行） |

## 2. 业务归零判定表

| 判定项 | 判定方法 | 结论 |
|---|---|---|
| admit_count | 读原子字 | ==0 则归零 |
| commit_count | 读原子提交计数 | ==0 则归零 |
| receipt_write_count | 读原子回执计数 | ==0 则归零 |
| 半成品 Receipt | 扫描回执目录 | 存在则隔离，不参与快照 |
| 未知事务 | 恢复协议判定 | UNKNOWN → 隔离，阻止 Fork |

## 3. Receipt 状态与 Fork 许可矩阵

| Receipt 状态 | Fork 许可 | 处理 |
|---|---|---|
| 无（idle） | ✅ 允许 | — |
| 写入中 | ❌ 阻止 | 等 receipt_write_count 归零 |
| 半成品 | ❌ 阻止 | 隔离后放行 |
| 完整可认证 | ✅ 允许 | 参与快照（但认证未建立 → 实际隔离） |
| 完整不可认证 | ✅ 允许（不信任） | 隔离，不参与成功判定 |
| QUARANTINED | ✅ 允许（不参与） | 隔离 |
| 状态未知 | ❌ 阻止 | 隔离后放行 |

## 4. 快照一致性不变量

- I-SNAP-1：快照生成时 admit_count==0。
- I-SNAP-2：快照生成时 commit_count==0。
- I-SNAP-3：快照生成时 receipt_write_count==0。
- I-SNAP-4：快照不包含 QUARANTINED / 半成品 / 未知事务的成功状态。
- I-SNAP-5：快照期间无数据变化（冻结态 + 业务归零保证）。
- I-SNAP-6：未认证 Receipt 不参与成功事务恢复或幂等成功确认。

## 5. 快照生成期间防变化

冻结态（gate=FREEZING）+ 业务归零 + commit/receipt 归零 → 无写者。快照读取一致。

## 6. Fork 失败后父进程状态恢复

- CAS unfreeze 回滚 gate→OPEN。
- 原子计数状态未变（未实际 fork）。
- 半成品/未知事务保持隔离，不被清理。

## 7. 子进程处理继承业务状态与 Receipt

- 子进程丢弃继承的 admit_state/commit/receipt 原子计数副本，重建为 0。
- 子进程不继承父进程的半成品/QUARANTINED 状态（重新扫描或保持隔离）。
- 子进程独立的事务身份，不复用父 txn_id。

## 8. 其他线程持有的锁在子进程中的状态

- 子进程丢弃继承锁副本，重建全部进程本地锁。
- OS 级锁不继承，需重取。

## 9. 防止父子同时拥有同一事务

- 事务所有权 = `(process_tree_id, branch_path, txn_id)` 三元组。
- 子进程 fork 后生成新 branch_path，txn_id 独立生成。
- 父子 txn_id 空间不相交 → 不存在父子双重拥有同一事务。

## 10. 硬性规则

- Receipt Creator Authentication 未证明时，任何未认证 Receipt 不参与成功事务恢复或幂等成功确认。
- QUARANTINED 不视为成功。
- 不为 Fork 快照擅自信任未知 Receipt。

## 11. 证据等级
快照一致性协议 DESIGN_PROVEN（逻辑）；Receipt 认证依赖 NOT_PROVEN；运行时 NOT_TESTED。
