# WRITE-ENTRY-COVERAGE.md

## 目标
证明所有写入入口受同一冻结准入协议约束；无法枚举的锁外写入者标 UNKNOWN，Fork DENY。

## 1. 写入入口清单

| # | 写入入口 | 进入 admit 生命周期 | 增减计数 | 覆盖 |
|---|---|---|---|---|
| 1 | 业务提交 | CAS 准入成功 | commit_count++ | ✅ |
| 2 | 提交确认 | 同上 | commit_count-- | ✅ |
| 3 | Receipt 创建 | 同上 | receipt_write_count++ | ✅ |
| 4 | Receipt 更新 | 同上 | receipt_write_count++ | ✅ |
| 5 | Receipt 持久化 | 同上 | receipt_write_count++ | ✅ |
| 6 | Receipt 重试 | 同上 | receipt_write_count++ | ✅ |
| 7 | 事务恢复写入 | CAS 准入 | commit_count++ | ✅ |
| 8 | 补偿写入 | CAS 准入 | commit_count++ | ✅ |
| 9 | 后台任务 | CAS 准入 | 对应计数 | ✅ |
| 10 | 回调 | CAS 准入 | 对应计数 | ✅ |
| 11 | 定时任务 | CAS 准入 | 对应计数 | ✅ |
| 12 | 异步工作线程 | CAS 准入 | 对应计数 | ✅ |
| 13 | 锁外写入者（其他进程/管理员） | **无法枚举** | **无** | ❌ UNKNOWN |

## 2. 生命周期图

```
入口 ──CAS准入(admit_state, OPEN,count+1)──► 已准入
  └─CAS失败(gate=FREEZING)──► 拒绝
已准入 ──执行写入──► commit_count++/receipt_write_count++ ──完成──► 计数-- ──CAS退出──► 结束
```

## 3. 逐项证明

**命题**：commit_count / receipt_write_count 不能在 admit 生命周期之外增加或执行写入。

**证明**（对每个受控入口）：
1. 入口在写入前必须先 `CAS(admit_state, gate==OPEN, count+1)` 成功。
2. CAS 成功是执行写入的**唯一前提**（代码路径强制）。
3. 冻结 CAS 成功置 gate==FREEZING 后，新准入 CAS 失败 → 无新写入者。
4. 因此冻结后，commit_count/receipt_write_count 不再增加，已有写入者递减归零。

结论：受控入口全部覆盖，冻结成功 ⟹ 所有受保护写入停止。

## 4. 覆盖边界

| 边界 | 覆盖 |
|---|---|
| 跨线程（同进程） | ✅（共享 admit_state） |
| 跨进程（同主机遵守协议） | 部分（需跨进程共享状态） |
| 子进程继承 | ✅（子重建独立状态） |
| 恢复路径 | ✅（CAS 准入） |
| 锁外写入者 | ❌ UNKNOWN |

## 5. 关键结论与硬性规则

1. 受控写入入口（1–12）全部由 admit 生命周期覆盖。
2. 锁外写入者（入口 13）无法枚举、无法证明受控 → **UNKNOWN / NOT_PROVEN**。
3. **只要存在未覆盖写入入口，Fork 必须 DENY**，不得凭「静态计数器归零」推定安全。
4. 因此：在锁外写入者无法排除的部署环境中，**Fork 功能 DENY**；仅在可证明无锁外写入者的受控环境（如单进程、封闭目录、文件系统权限隔离）才允许 Fork。

## 6. 证据等级
受控入口覆盖 DESIGN_PROVEN（静态）；锁外写入者 UNKNOWN/NOT_PROVEN；运行时 NOT_TESTED。
