# USIT-STAGE-B-DESIGN-05.22-DIFF｜相对 05.21 逐项差异

> 阶段 B · DESIGN-05.22 · 相对 DESIGN-05.21 的修订差异
> 生成：2026-10-31（GMT+8）

---

## 修订总览

| 项 | 05.21 状态 | 05.22 修订 | 性质 |
|----|-----------|-----------|------|
| P0-02 | `_fork_coord_lock` 由 after_in_parent 与 fork_entry finally 双重释放 | 唯一释放者 = fork_entry finally，钩子不触碰 | **双释放修复（B22-01）** |
| P0-02 | 快照存储/所有权/生命周期未定义 | fork_epoch 单调 + 单槽位 + epoch 校验 | **快照协议补证（B22-02）** |
| P0-04 | 最终态 DURABLE 与 DURABLE（进行中）混用 | 七态状态机，COMMIT_IN_PROGRESS 独立 | **状态机重构（B22-03）** |
| P0-04 | 崩溃矩阵未覆盖 10 类位置 | S1~S16 完整矩阵（10 类位置无省略）| 补全 |
| 交付 | ZIP 回执缺真实哈希/核验 | RECEIPT 记录 ZIP 哈希 + 真实解压比对 | 补证（B22-04）|
| 交付 | 源码未修改缺本轮基线 | 本轮前后基线采集 | 补证（B22-05）|

---

## P0-02 差异

| 维度 | 05.21 | 05.22 |
|------|-------|-------|
| 锁释放责任 | after_in_parent 释放 + fork_entry finally 释放（双释放）| fork_entry finally 唯一释放 |
| 钩子是否触碰 coord 锁 | after_in_parent 释放 coord（错误）| 钩子不取不释 coord |
| 快照存储 | 未定义 | 进程级单槽位 `_pending_snapshot`（L0 保护）|
| 快照对应关系 | 未定义 | fork_epoch 单调 + epoch 校验，一一对应 |
| 快照生命周期 | 未定义 | freeze 生成 → before/child 读 → parent/finally 清空 |

## P0-04 差异

| 维度 | 05.21 | 05.22 |
|------|-------|-------|
| 状态机 | 六态，DURABLE 与 DURABLE（进行中）混用 | 七态，COMMIT_IN_PROGRESS 独立 |
| 初始态 | DURABLE（进行中）| COMMIT_IN_PROGRESS |
| 最终态 | DURABLE | DURABLE（仅 current+回执都持久）|
| 崩溃矩阵 | 未覆盖 10 类位置 | 10 类位置全覆盖（含 ①~⑩）|
| S16 语义 | 未明确 | 明确 S16 锁释放非持久化步骤，崩溃仍 DURABLE |

## 数量变化

| 项 | 05.21 | 05.22 |
|----|-------|-------|
| 不变量 | 45 条 | 45 条（修订 I-02/I-03/I-07~09/I-20/I-21/I-30）|
| 端到端链路 | 20 条 | 20 条 |
| 时间线 | 40 条 | 40 条 |
| 反例 | 30 个 | 30 个 |

---

## 结论

本轮定点修复：B22-01 双释放、B22-02 快照协议、B22-03 状态机混用、B22-04/B22-05 交付证据。无源码修改，IMPLEMENT-01 NOT_APPROVED。

---