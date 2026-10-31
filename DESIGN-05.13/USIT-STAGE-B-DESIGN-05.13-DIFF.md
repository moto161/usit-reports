# USIT-STAGE-B-DESIGN-05.13｜设计差异清单（DIFF）

> 阶段 B · DESIGN-05.12 → DESIGN-05.13 逐项差异
> 本清单是**设计文档差异**，不是源码差异，也不伪装成源码差异。
> 本轮针对五项缺陷定向修复，不推倒重做 05.12，不重复已通过的 P0-01。

---

## 0. 总体差异概览

| 维度 | DESIGN-05.12 | DESIGN-05.13 |
|------|-------------|--------------|
| fork child 回调 | 调用 snapshot_fds()（碰锁）| prepare 快照 tuple，child 只读不碰锁 |
| ForkGate 重建 | 只重建 ProcessMutex | ForkGate 及全部锁对象重建 |
| 锁文件稳定性 | 依赖目录权限 | 永久锁文件 + 身份变化全体 DENY |
| PREPARED 顺序 | 与发布重叠 | PREPARED 先持久化再发布 |
| 源内容一致性 | 身份相同≈内容相同 | 双哈希三方校验 + 独立 retire |
| 删除归因 | UNLINK_INTENT 补 REMOVED | tombstone 不 unlink，诚实 INDETERMINATE |

---

## 1. P0-02 差异

| 子项 | 05.12 | 05.13 |
|------|-------|-------|
| child 回调 | 调用 `snapshot_fds()`（持锁）| 读 prepare 快照的不可变 tuple |
| prepare 回调 | before=None（空）| before 快照 FD 列表 |
| ForkGate | 未重建 | 重建 ForkGate + 全部锁 |
| 子进程 DENY 标记 | 无 | `_fork_reset_failed` 拒绝后续锁操作 |

---

## 2. P0-03 差异

| 子项 | 05.12 | 05.13 |
|------|-------|-------|
| 稳定性来源 | 目录权限（0o700）| 永久锁文件协议 |
| 锁文件删除 | 允许（满足条件）| 协议禁止 unlink/replace |
| 身份变化 | 路径 vs 句柄比较 | 全体停止 + DENY（不对准新对象）|
| 同用户竞争 | 未明确 | §2.5 显式分析 |

---

## 3. P0-04 差异

| 子项 | 05.12 | 05.13 |
|------|-------|-------|
| PREPARED 时机 | 与发布重叠 | 阶段 B 先于阶段 C |
| commit_state 字段 | 无 | 新增 |
| 双事务竞争 | 无矩阵 | §3.6 完整矩阵 |
| CAS | 未明确 | §3.7 受保护 CAS |

---

## 4. P0-05 差异

| 子项 | 05.12 | 05.13 |
|------|-------|-------|
| 内容一致性 | 复制前后 sha256 比对 | Q4 三方校验（qtmp==src==intent）|
| 源处理阶段 | 复用 QUARANTINE_INTENT | 独立 SOURCE_RETIRE_INTENT |
| 处理前校验 | 无 | §4.6 处理前再校验内容 |

---

## 5. P0-06 差异

| 子项 | 05.12 | 05.13 |
|------|-------|-------|
| unlink 归因 | UNLINK_INTENT 补 REMOVED | 承认 unlink 不可归因 |
| 删除方式 | unlink tombstone | tombstone 不 unlink（保留为证据）|
| 崩溃恢复 | unlink 后补 REMOVED | unlink 后 INDETERMINATE |

---

## 6. 新增内容

- §0 缺陷定位表（逐项引用 05.12 原文章节）。
- §6 不变量扩展到 I-01～I-15。

---

## 7. 未变化部分

- P0-01：不重复修订（已通过）。
- P1-02 / P1-03：状态保持。
- 三个阻断项、F1/P0.8/F2、IMPLEMENT-01：维持阻断。

---

## 8. 结论

DESIGN-05.13 相对 05.12 的差异全部落在五项 P0 缺陷消除，均为设计契约修订，不涉及源码、测试、配置、运行时数据、A/B 工作区或冻结基线变更。
