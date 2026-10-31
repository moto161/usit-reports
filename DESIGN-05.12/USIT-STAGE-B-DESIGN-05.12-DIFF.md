# USIT-STAGE-B-DESIGN-05.12｜设计差异清单（DIFF）

> 阶段 B · DESIGN-05.11 → DESIGN-05.12 逐项差异
> 本清单是**设计文档差异**，不是源码差异，也不伪装成源码差异。
> 本轮针对独立审查六项缺陷做定向修复，不推倒重做 05.11。

---

## 0. 总体差异概览

| 维度 | DESIGN-05.11 | DESIGN-05.12 |
|------|-------------|--------------|
| flock fork 语义 | "子进程不继承"（矛盾）| 纠正为"子进程共享 OFD 锁" |
| fork 清理机制 | 未定义 `_inherited_lock_fds` + 布尔标志 | LockHandleRegistry + ForkGate 锁互斥 |
| 路径身份 | 检查后按路径操作（TOCTOU）| 全程句柄 + dirfd/openat |
| Candidate 提交 | Windows CREATE_NEW 空占位 | 先 tmp 后 MoveFileEx 无覆盖发布 |
| Journal 隔离 | 先 qfinal 后审计 | 意图（quarantine_id）先于 qfinal |
| Tombstone 归因 | 状态矛盾 | 新增 UNLINK_INTENT 消除矛盾 |

---

## 1. P0-01 差异

| 子项 | 05.11 | 05.12 |
|------|-------|-------|
| fork 语义 | 表格写"不继承"，与正文矛盾 | §1.1 纠正为"继承 OFD 引用共享锁" |
| LOCK_UN 语义 | 未明确定义 | §1.4 明确"解除整个 OFD 锁" |
| 状态表 | 无 fork 前后 FD/OFD/锁表 | §1.5 完整状态表 |

---

## 2. P0-02 差异

| 子项 | 05.11 | 05.12 |
|------|-------|-------|
| 继承 FD 登记 | `_inherited_lock_fds`（未定义）| LockHandleRegistry（单一登记表）|
| fork 前检查 | `_in_lock_critical_section` 布尔标志 | ForkGate 锁互斥（可证明）|
| 回调职责 | 仅 after_in_child | prepare/parent/child 三回调职责 |
| 外部 fork | 未定义 | 明确拒绝策略 |

---

## 3. P0-03 差异

| 子项 | 05.11 | 05.12 |
|------|-------|-------|
| 身份验证 | 检查句柄 vs 路径对象，比较后操作 | 全程句柄，openat/dirfd，不再回头用路径 |
| 目录可信 | 未定义 | §3.5 目录权限前提，不满足 DENY |
| Windows | 与 POSIX 混述 | §3.7 独立契约，不套 POSIX dirfd |

---

## 4. P0-04 差异

| 子项 | 05.11 | 05.12 |
|------|-------|-------|
| Windows 发布 | CREATE_NEW 空占位（暴露半成品）| MoveFileEx 不带 REPLACE 无覆盖 |
| POSIX 发布 | renameat2/link | 保留 + 补充完整序列（§4.3）|
| 提交顺序 | 未明确"先 tmp 完整" | §4.2 三步：tmp 完整→校验→无覆盖发布 |

---

## 5. P0-05 差异

| 子项 | 05.11 | 05.12 |
|------|-------|-------|
| 隔离顺序 | 先 qfinal 后审计 | 意图（QUARANTINE_INTENT）先于 qfinal |
| 事务关联 | 无 quarantine_id | 阶段 0 生成 quarantine_id 绑定 |
| 源对象处理 | 混在隔离流程 | 分两阶段（副本隔离 vs 源改名）|

---

## 6. P0-06 差异

| 子项 | 05.11 | 05.12 |
|------|-------|-------|
| 状态机 | DELETE_INTENT→TOMBSTONED→REMOVED | 新增 UNLINK_INTENT 状态 |
| 归因矛盾 | tombstone 要求存在但已 unlink | UNLINK_INTENT 持久化后 unlink 可归因 |
| 三态归因 | 未覆盖 tombstone 消失 | §6.6 完整决策表 |

---

## 7. 新增内容

- §0 缺陷分类方法论（A 确定矛盾 / B 契约缺失 / C 未验证风险）。
- §7 设计不变量集中证明（I-01～I-12）。
- CROSS-REVIEW.md（独立端到端交叉审查记录，05.11 无此文件）。

---

## 8. 未变化部分（明确不改）

- P1-02 / P1-03：状态保持。
- 三个阻断项、F1/P0.8/F2、IMPLEMENT-01：维持阻断。

---

## 9. 结论

DESIGN-05.12 相对 05.11 的差异全部落在六项 P0 缺陷的消除，均为**设计契约修订**，不涉及源码、测试、配置、运行时数据、A/B 工作区或冻结基线的变更。
