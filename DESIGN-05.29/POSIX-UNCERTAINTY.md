# USIT-STAGE-B-DESIGN-05.29 — POSIX 不确定态双维度闭合（POSIX-UNCERTAINTY.md）

> B29-03 支撑文档。分离 rename 结果不确定（维度 A）与持久性不确定（维度 B），闭合不确定态（P02）。

## 1. 两个独立维度

**维度 A：操作结果**
- NOT_STARTED / IN_PROGRESS / CONFIRMED_APPLIED / CONFIRMED_NOT_APPLIED / OUTCOME_UNKNOWN

**维度 B：持久性**
- NOT_DURABLE / DURABILITY_PENDING / DURABLE_CONFIRMED / DURABILITY_UNKNOWN

## 2. 组合状态表

| 维度 A \ B | NOT_DURABLE | DURABILITY_PENDING | DURABLE_CONFIRMED | DURABILITY_UNKNOWN |
|-----------|-------------|-------------------|-------------------|-------------------|
| NOT_STARTED | 合法（未开始）| 非法 | 非法 | 非法 |
| IN_PROGRESS | 合法（进行中）| 合法（内容已写）| 非法 | 非法 |
| CONFIRMED_APPLIED | 合法（已应用未持久）| 合法（fsync 中）| **合法=DURABLE** | 合法（持久不确定）|
| CONFIRMED_NOT_APPLIED | 合法 | 非法 | 非法 | 非法 |
| OUTCOME_UNKNOWN | 合法 | 合法 | 非法 | 合法（不确定态）|

## 3. 禁止的推断

1. 操作是否发生不明确 → 不得标记提交成功。
2. 操作已发生但 fsync 失败 → 不得认定操作未发生。
3. 目录 fsync 失败 → 不得盲目重复 rename。
4. 不确定态 → 不得直接跳 DURABLE。
5. 不得仅凭进程内变量恢复磁盘事务。
6. 不得仅凭文件名判断当前对象身份。

## 4. rename 16 种情况建模

| # | 情况 | rename 已发生？| current 内容？| 目录项持久？| 重试 | 回滚 | 新事务 | 人工 |
|---|------|--------------|-------------|------------|------|------|--------|------|
| 1 | 正常返回成功 | 是（已生效）| 可核验 | 未（待 fsync）| 否 | 否 | 否 | 否 |
| 2 | EINTR | **未知** | 重读判定 | 未知 | 否(重读后) | 否 | 否 | 否 |
| 3 | EIO | 未知 | 重读 | 未知 | 否 | 否 | 否 | 是 |
| 4 | ENOSPC | 否（未发生）| 旧值 | 旧值 | 否 | 否 | 否 | 否 |
| 5 | 其他错误 | 视错误码 | 重读 | — | 视情况 | 否 | 否 | 视 |
| 6 | 已生效 + fsync 失败 | 是 | 新值 | **未知** | 否(不重 rename) | 否 | 否 | 否 |
| 7 | 未生效但结果不明 | 未知 | 旧值 | 旧值 | 否 | 否 | 否 | 否 |
| 8 | current 已是新版本 | 是 | 新值 | 待确认 | 否 | 否 | 否 | 否 |
| 9 | current 仍是旧版本 | 未知（不推断未发生）| 旧值 | 旧值 | 否 | 否 | 否 | 否 |
| 10 | current 内容与预期不一致 | 冲突 | 异常 | — | 否 | 否 | 否 | 是(QUARANTINED) |
| 11 | current inode 改变 | 可能已替换 | 新 | 待确认 | 否 | 否 | 否 | 否 |
| 12 | current 哈希改变 | 冲突 | 异常 | — | 否 | 否 | 否 | 是 |
| 13 | 外部写入者在恢复期改 current | 冲突 | 异常 | — | 否 | 否 | 否 | 是 |
| 14 | rename 前崩溃 | 否 | 旧值 | 旧值 | 是(重 S12) | 否 | 否 | 否 |
| 15 | rename 后 fsync 前崩溃 | 未知 | 重读 | 未知 | 否 | 否 | 否 | 否 |
| 16 | fsync 后回执前崩溃 | 是 | 新值 | 已持久 | 否(补回执) | 否 | 否 | 否 |

## 5. 关键结论

- rename EINTR → 结果未知，重读 current 身份（inode/dev+SHA+revision）判定，不推断。
- rename 已生效但 fsync 失败 → 操作结果 CONFIRMED_APPLIED，持久性 UNKNOWN，只重试 dirfsync，不重 rename。
- current 内容/inode/哈希异常 → QUARANTINED。

## 6. 结论

双维度状态表闭合不确定态；禁止不确定态直接跳 DURABLE。协议层 DESIGN_PROVEN，运行时 NOT_TESTED。
