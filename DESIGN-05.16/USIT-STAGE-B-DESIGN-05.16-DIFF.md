# USIT-STAGE-B-DESIGN-05.16-DIFF｜相对 05.15 逐项差异

> READ-ONLY DESIGN REVISION ONLY · 日期：2026-10-31

## 总览

05.16 相对 05.15 的核心变化：**修复 ChatGPT 复审证实的四项剩余缺陷**——FD 无锁枚举竞态、Candidate 未绑父版本、sealed 非强制、tombstone 非原子发布。

| # | 05.15 设计 | 05.16 修订 | 差异性质 |
|---|-----------|-----------|---------|
| P0-02 | 无锁枚举 FD 快照 tuple | FD generation 绑定 + 持锁冻结协调 | 修复竞态 |
| P0-04 | Candidate 不绑父版本；固定锁文件无身份校验 | 绑定 base_revision/txn_id/hash + 五条件校验 + 锁身份校验 | 补全 |
| P0-05 | 哈希校验 + 重新验证当不可变 | 强制不可变属性（chattr +i），无强制平台 DENY | 重构边界 |
| P0-06 | O_EXCL 创建后逐步写当原子发布；归档混用 | 临时文件+校验+无覆盖发布；归档区分原语 | 修复 |

## 逐项差异

### P0-02

| 维度 | 05.15 | 05.16 |
|------|-------|-------|
| FD 快照 | 无锁枚举 tuple | 持锁冻结 + generation 绑定 |
| 误关防护 | 无 | generation 校验，不匹配不关闭 |
| 第三方 C fork | 未明确 | 明确 DENY（不声称覆盖）|

### P0-04

| 维度 | 05.15 | 05.16 |
|------|-------|-------|
| Candidate 绑定 | 无父版本 | base_revision/base_txn_id/base_hash |
| 提交校验 | 仅 rev==cur+1 | 五条件（父版本三 + rev+1 + 内容哈希）|
| 锁身份 | 固定锁文件无校验 | fstat+fstatat+openat+O_NOFOLLOW 身份比对 |

### P0-05

| 维度 | 05.15 | 05.16 |
|------|-------|-------|
| 不可变保证 | sealed 约定 + 重新哈希 | chattr +i 强制（EPERM 阻止旧 FD）|
| 无强制平台 | 未明确 | 默认 DENY Source Retire |

### P0-06

| 维度 | 05.15 | 05.16 |
|------|-------|-------|
| 元数据发布 | O_EXCL 创建后逐步写 | 临时文件 + 校验 + renameat2 NOREPLACE |
| 归档 | "rename 同样走 O_EXCL"（混用）| 无覆盖发布原语，区分 O_EXCL 与 rename |

## 冻结内容不变

- 源码 scanner.py / cli.py / file_understanding.py 未修改（哈希见执行报告）。
- 阻断 IMPLEMENT-01、P0-BLOCK-1/2、P1-BLOCK-3、F1/P0.8/F2 全部保持。
- 05.15 原始交付物未删除、未覆盖、未修改。
