# USIT-STAGE-B-DESIGN-05.29 设计差异审查（DESIGN-DIFF.md）

> 逐项对照 05.28，记录保留/修改/撤销/未证明/待复审。

| 项目 | 05.28 原结论 | 05.28 复审发现 | 05.29 修订 | 证明状态 |
|------|-------------|---------------|-----------|---------|
| Fork in_flight | 原子准入 | 存在自锁（F01）| 业务/Fork 计数域分离 | DESIGN_PROVEN |
| 上下文管理器 | `__enter__()` 直调 | 生命周期不完整（F02）| 完整 with 协议 | DESIGN_PROVEN |
| 锁释放 | finally 释放 | 所有权不明确（F03）| 锁所有权模型 | DESIGN_PROVEN |
| DENIED | 失败关闭 | 可被重开（F04）| 不可逆终态 + 人工授权 | DESIGN_PROVEN |
| 异常传播 | finally 覆盖 | 原始异常丢失（F05）| 主/附加异常分离 | DESIGN_PROVEN |
| process_instance_id | 存在继承描述 | 继承/重建冲突 | 子重建 instance_id，继承 tree_id | DESIGN_PROVEN |
| fork_epoch | 父内序号 | 冒充全局计数器 | 仅父内严格 | DESIGN_PROVEN |
| fork_event_id | 结构+随机 | 唯一性需复核 | tree_id+父 instance+seq | DESIGN_PROVEN |
| commit_lock | 全局提交锁 | 无法约束锁外写入者 | advisory 边界明确 + QUARANTINED | DESIGN_PROVEN（合作者前提）|
| rename | 不确定态 | 结果/持久性混淆 | 双维度状态机 | DESIGN_PROVEN |
| Receipt | 幂等回执 | 半成品归属不足 | 四元组归属证明 | DESIGN_PROVEN |

## 保留的设计

- POSIX S1~S16 步骤定义（保留编号）。
- 六字段、五谓词、五条件定义。
- EINTR 不推断未执行（保留并强化）。
- 15 项恢复安全门（保留并逐项证明必要性）。
- Windows ReplaceFileW 1175/1176/1177 语义（保留）。

## 修改的设计

- Fork 生命周期（新增 DRAINING 阶段，业务/Fork 分流）。
- 锁所有权模型（新增锁依赖图与死锁证明）。
- 身份模型（八层分离，instance/tree 分离）。
- 状态机（20→23 态，拆不确定态）。
- 回执唯一键（二元组→四元组）。

## 撤销的设计

- 05.28 "父侧全局单调 fork_epoch 结构性消除碰撞"。
- 05.28 "fork_event_id 进程树内严格唯一"（改为树内结构 + 跨树概率）。
- 05.28 "仅凭 _commit_lock 约束所有 current 写入者"。
- 05.28 三项 DESIGN_PROVEN（全部重新证明，非自动继承）。

## 尚未证明的设计

- 锁外写入者防护（NOT_VERIFIED，advisory 本质）。
- 跨主机/跨重启唯一性（NOT_VERIFIED，不承诺）。
- 随机源失效降级行为（UNKNOWN）。

## 需后续独立审查事项

- 三核心阻断项（B29-01/02/03）的协议层证明是否成立。
- advisory 部署契约是否可接受。
- 是否允许进入 IMPLEMENT-01。
