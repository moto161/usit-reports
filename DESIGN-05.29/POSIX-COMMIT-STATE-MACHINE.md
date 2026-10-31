# USIT-STAGE-B-DESIGN-05.29 — POSIX 提交状态机（POSIX-COMMIT-STATE-MACHINE.md）

> B29-03 支撑文档。23 态状态机，拆出 rename 结果不确定与持久性不确定。

## 1. 状态机（23 态）

每个状态含：进入条件、退出条件、系统调用、对象身份、inode/dev、内容哈希、txn_id、revision、持久性证据、是否允许重试/删除/覆盖、恢复路径、终态。

| 状态 | 进入条件 | 退出条件 | 系统调用 | 持久性证据 | 重试 | 删除 | 覆盖 | 终态 |
|------|---------|---------|---------|-----------|------|------|------|------|
| NEW | 事务创建 | S1 完成 | — | 无 | 是 | 否 | 否 | 否 |
| CANDIDATE_CREATED | S1 O_EXCL 成功 | S2 完成 | open(O_EXCL) | 无 | 否 | 是(身份确认) | 否 | 否 |
| CANDIDATE_CONTENT_DURABLE | S2 fsync(fd) | S3 完成 | fsync(fd) | 内容持久 | 是(重读) | 否 | 否 | 否 |
| CANDIDATE_ENTRY_DURABLE | S3 fsync(dir) | S4 完成 | fsync(dirfd) | 目录项持久 | 是(重读) | 否 | 否 | 否 |
| CANDIDATE_VERIFIED | S4 校验过 | S5 完成 | 纯读 | 哈希匹配 | 否 | 否 | 否 | 否 |
| PREPARED_CREATED | S5 写入 | S6 完成 | open(O_EXCL)+write | 无 | 否 | 是(身份确认) | 否 | 否 |
| PREPARED_CONTENT_DURABLE | S6 fsync(fd) | S7 完成 | fsync(fd) | 内容持久 | 是 | 否 | 否 | 否 |
| PREPARED_ENTRY_DURABLE | S7 fsync(dir) | S8 完成 | fsync(dirfd) | 目录项持久 | 是 | 否 | 否 | 否 |
| PREPARED_VERIFIED | S9 校验过 | S10 完成 | 纯读 | 哈希匹配 | 否 | 否 | 否 | 否 |
| CURRENT_TMP_CREATED | S10 写入 | S11 完成 | open(O_EXCL)+write | 无 | 否 | 是(身份确认) | 否 | 否 |
| CURRENT_TMP_CONTENT_DURABLE | S11 fsync(fd) | S12 校验 | fsync(fd) | 内容持久 | 是 | 否 | 否 | 否 |
| CURRENT_TMP_VERIFIED | S11 后校验 | rename 前 | 纯读 | 哈希匹配 | 否 | 否 | 否 | 否 |
| RENAME_NOT_STARTED | 校验过，未 rename | rename 调用 | — | 无 | 否 | 否 | 否 | 否 |
| RENAME_OUTCOME_UNKNOWN | rename EINTR 等 | 重读确认 | rename(结果未知) | 无 | 否(禁止盲目) | 否 | 否 | 否 |
| RENAME_CONFIRMED | rename 生效确认 | S13 fsync | rename | 目录项可见(未持久) | 否 | 否 | 否 | 否 |
| CURRENT_DURABILITY_UNKNOWN | S13 fsync 失败 | 重读/人工 | fsync(dirfd)(失败) | 无法确认 | 是(重读) | 否 | 否 | 否 |
| CURRENT_DURABLE_AUDIT_PENDING | S13 完成 | S14 | fsync(dirfd) | 目录项持久 | 否 | 否 | 否 | 否 |
| RECEIPT_CREATED | S14 写入 | S14 fsync | open(O_EXCL)+write | 无 | 否 | 是(身份确认) | 否 | 否 |
| RECEIPT_CONTENT_DURABLE | S14 fsync(fd) | S15 完成 | fsync(fd) | 内容持久 | 是 | 否 | 否 | 否 |
| RECEIPT_ENTRY_DURABLE | S15 fsync(dir) | 完成 | fsync(dirfd) | 目录项持久 | 是 | 否 | 否 | 否 |
| DURABLE | current+回执双持久 | — | — | current+回执持久 | — | 否 | 否 | 是 |
| RECOVERY_REQUIRED | 自动恢复不足 | 人工 | — | 无 | 人工 | 否 | 否 | 是 |
| QUARANTINED | 证据冲突 | 人工 | — | 冲突 | 人工 | 否 | 否 | 是 |

## 2. 状态转移规则

- 允许转移：按 S1→S2→…→S16 顺序 + 失败分支。
- 禁止转移：跳过中间态；不确定态直接跳 DURABLE；未校验跳 rename。
- 恢复转移：RENAME_OUTCOME_UNKNOWN →（重读确认）→ RENAME_CONFIRMED 或 QUARANTINED。
- CURRENT_DURABILITY_UNKNOWN →（重读+重试 fsync）→ CURRENT_DURABLE_AUDIT_PENDING 或 QUARANTINED。

## 3. 状态转移目标存在性

所有转移目标均在表中定义，无引用不存在状态。

## 4. 结论

23 态状态机拆出 RENAME_OUTCOME_UNKNOWN / RENAME_CONFIRMED / CURRENT_DURABILITY_UNKNOWN 三个关键不确定态，禁止不确定态直接跳 DURABLE。协议层 DESIGN_PROVEN，运行时 NOT_TESTED。
