# USIT-STAGE-B-DESIGN-05.15-DIFF｜相对 05.14 逐项差异

> READ-ONLY DESIGN REVISION ONLY · 日期：2026-10-31

## 总览

05.15 相对 05.14 的核心变化：**修复 ChatGPT 独立复审证实的四项协议错误**——假 CAS、fork 死锁、sealed≠不可变、tombstone 对象模型矛盾。

| # | 05.14 设计 | 05.15 修订 | 差异性质 |
|---|-----------|-----------|---------|
| P0-02 | before 无锁证明不充分，未建锁顺序 | 全局锁升序 + 禁持锁 fork + 无锁快照 | 修复死锁反例 |
| P0-04 | `link(current, current.lock.<txn_id>)` 假 CAS | 单一固定 `.commit-lock` + flock 临界区真 CAS | **废弃假 CAS，重构** |
| P0-05 | sealed/append-only/O_EXCL 当不可变 | 六状态机 + SEALED 持久化证据 + 重新验证 | 重构对象模型 |
| P0-06 | 源文件 rename 到 tombstone 路径 = tombstone 元数据 | 分离源对象与 tombstone 元数据 | 修复对象模型矛盾 |

## 逐项差异

### P0-02

| 维度 | 05.14 | 05.15 |
|------|-------|-------|
| 死锁处理 | 仅证 before 正常阻塞非死锁 | 全局锁升序 + 禁持锁 fork + before 无锁快照 |
| 锁顺序 | 无 | 5 级锁顺序（gate→registry→fd→lease→pmutex）|
| 最小反例 | 未处理 A 持 Registry / B 持 ForkGate | §2.4 给出并消除 |

### P0-04

| 维度 | 05.14 | 05.15 |
|------|-------|-------|
| CAS 原语 | `link(current, current.lock.<txn_id>)` | flock(LOCK_EX) 临界区 |
| 竞争对象 | 每事务独立锁文件名（假 CAS）| 单一固定 `.commit-lock` |
| 丢失更新 | 会（T1/T2 同时 link 成功）| 不会（flock 串行化 check+publish）|

### P0-05

| 维度 | 05.14 | 05.15 |
|------|-------|-------|
| 不可变定义 | sealed/append-only/O_EXCL 约定 | WRITABLE→SEALING→SEALED 状态机 |
| 旧 FD 写入 | 未充分处理 | §3.4 明确 rename 不撤销旧 FD，需协作或检测 |
| 持久化证据 | 无 | SEAL_INTENT + SEALED 记录 + sealed 哈希 |

### P0-06

| 维度 | 05.14 | 05.15 |
|------|-------|-------|
| 对象模型 | 源文件 rename = tombstone 元数据（矛盾）| 源对象 / 隔离对象 / tombstone 元数据 / journal 四对象分离 |
| 无覆盖发布 | O_EXCL 与 rename 混用 | 区分 O_EXCL 创建 vs renameat2 RENAME_NOREPLACE |
| TOMBSTONED 含义 | 含元数据 | 仅指"源对象被 rename 走" |

## 冻结内容不变

- 源码 scanner.py / cli.py / file_understanding.py 未修改（哈希见执行报告）。
- 阻断 IMPLEMENT-01、P0-BLOCK-1/2、P1-BLOCK-3、F1/P0.8/F2 全部保持。
- 05.14 原始交付物未删除、未覆盖、未修改。
