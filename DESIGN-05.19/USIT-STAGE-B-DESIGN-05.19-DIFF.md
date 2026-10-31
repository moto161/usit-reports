# USIT-STAGE-B-DESIGN-05.19-DIFF｜相对 05.18 逐项差异

> 阶段 B · DESIGN-05.19 · 相对 DESIGN-05.18 的修订差异
> 生成：2026-10-31（GMT+8）

---

## 修订总览

| 项 | 05.18 状态 | 05.19 修订 | 性质 |
|----|-----------|-----------|------|
| P0-02 | 快照后释放 `_fd_lock`/`_fork_coord_lock` 再 fork | FREEZING 态覆盖真正的 fork 系统调用窗口 | **协议重构（阻断修复）** |
| P0-04 | 补目录项 fsync + 崩溃矩阵，缺故障状态机 | 新增五态持久化故障状态机 + 完整五条件谓词 | 补强 |
| P0-05 | 撤回 +i 方案，Source Retire DENY | 保持 DENY，厘清"未证明"vs"不存在机制"边界 | 厘清表述 |
| P0-06 | 笼统列错误码 | 依据 Microsoft Learn 逐码覆盖 1175/1176/1177 | **依据官方文档重构** |

---

## P0-02 差异（核心）

| 维度 | 05.18 | 05.19 |
|------|-------|-------|
| 冻结窗口 | 快照生成后释放 `_fd_lock` 和 `_fork_coord_lock` | FREEZING 态保持覆盖 `os.fork()`，仅释放细粒度 `_fd_lock` |
| 快照一致性 | 快照后 FD 可能被其他线程修改（未门禁窗口）| FREEZING 态阻断所有受管 FD 操作入口 |
| FD 入口覆盖 | 未逐项审查 | 逐项审查 open/socket/accept/pipe/dup/fcntl/close/close_range/detach/fileno/继承/第三方 |
| 第三方 C fork/open | 提及 DENY，未细化 child 处理 | 明确 child 对未知 FD 不主动 close，标 UNKNOWN_FD |
| 结论 | DESIGN_PARTIAL | DESIGN_PARTIAL（协议层 DESIGN_PROVEN，运行时 NOT_TESTED，非受控入口 UNSUPPORTED/DENY）|

## P0-04 差异

| 维度 | 05.18 | 05.19 |
|------|-------|-------|
| 五条件校验 | 一句"五条件全部通过" | 逐项谓词 P1~P5（字段来源/比较对象/时点/锁保护/失败行为）|
| 持久化步骤 | 16 步（缺故障态）| 13+3 步明确 + 目录项 |
| 故障状态机 | 无 | DURABLE/NOT_DURABLE/COMMIT_INDETERMINATE/RECOVERY_REQUIRED/QUARANTINED |
| fsync 失败策略 | 未定义 | NOT_DURABLE 禁后续提交；不确定态禁回滚/重复提交 |

## P0-05 差异

| 维度 | 05.18 | 05.19 |
|------|-------|-------|
| DENY 结论 | 保持 | 保持（不制造 PASS）|
| 边界表述 | "本地 FS 无机制同时满足" | 区分"当前方案未证明"与"平台不存在可行机制（独立特权服务未验证）"|

## P0-06 差异（依据 Microsoft Learn）

| 维度 | 05.18 | 05.19 |
|------|-------|-------|
| 1175 | 未逐项 | "目标与替换文件都保持原名，无移动，可幂等重试" |
| 1176 | 未逐项 | 分两种情况：指定备份→无移动可重试；未指定备份→目标已不存在、替换以原名，部分移动须判定 |
| 1177 | 未逐项 | 替换文件原名（继承流属性），目标以不同名存在，部分移动须判定 |
| 状态矩阵 | 笼统 | 六态 PLACEHOLDER/PUBLISHED/REPLACEMENT_PRESENT/BACKUP_PRESENT/PARTIAL_MUTATION/INDETERMINATE |

## 不变量与链路

| 项 | 05.18 | 05.19 |
|----|-------|-------|
| 不变量 | 35 条 | 40 条（新增 I-08/I-09/I-12/I-14/I-18/I-19/I-29/I-32/I-33）|
| 端到端链路 | 15 条 | 18 条 |
| 时间线 | 57 条 | 45 条（P0-02/04/06 各 15）|

---

## 结论

本轮为定点修复：P0-02 闭合门禁窗口、P0-04 补故障状态机、P0-06 依据官方文档重构错误码矩阵、P0-05 保持 DENY 并厘清边界。无源码修改，无 IMPLEMENT-01。

---