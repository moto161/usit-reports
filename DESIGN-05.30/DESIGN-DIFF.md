# DESIGN-DIFF.md

## 目标
05.29 / 05.30 差异审查。

## 1. 逐项差异表

| 05.29 问题 | 修订目标 | 修订结果 | 证明状态 |
|---|---|---|---|
| DENIED 被重新打开 | 不可逆状态机 | DENIED 不可逆，人工授权独立协议 | DESIGN_PROVEN |
| process_tree_id 概率唯一 | 根身份唯一性证明 | 如实标 PROBABILISTIC | DESIGN_PROVEN（边界明确） |
| branch identity 不充分 | 结构性分支路径 | 引入 branch_path 结构唯一 | DESIGN_PROVEN（树内） |
| Receipt 四元组冒充真实性 | 创建者认证 | 六层证据分离，认证层 UNKNOWN | NOT_PROVEN（认证层） |
| EEXIST 归属不足 | 事务归属证明 | 9 种来源分类 + 安全门 | DESIGN_PROVEN |
| Fork 锁获取路径缺失 | 完整生命周期 | 15 步完整伪代码 | DESIGN_PROVEN |
| 父子锁所有权不清 | 独立状态建模 | 7 把锁所有权表 + 重建规则 | DESIGN_PROVEN |
| 统计数量冲突 | 统一编号和统计 | 机器统计一致 | VERIFIED_STATIC |
| 执行报告统计缺失 | 实际统计结果 | 执行报告含真实统计 | VERIFIED_STATIC |

## 2. 保留 / 修改 / 撤销 / 新增

### 保留
- Fork 业务计数域 `_biz_in_flight` 与协调分离。
- POSIX 双维度不确定态。
- 23 态状态机骨架。
- 回执幂等判定框架。

### 修改
- DENIED 状态机（修复 restore→OPEN）。
- 身份唯一性（branch_path 结构）。
- Receipt 证据分层。
- Fork 锁生命周期。

### 撤销
- 05.29 的「DENIED→restore→OPEN」路径。
- 05.29 的「四元组证明创建者真实性」结论。
- 05.29 的「parent_instance_id 参与严格唯一」结论。

### 新增
- branch_path 结构字段。
- 六层 Receipt 证据（A~F）。
- 身份分配锁、Receipt 写入锁。
- manual_reenable 管理协议。

## 3. 尚未证明

- Receipt 创建者认证（D 层）。
- 树间身份全局严格唯一。
- 锁外写入者不存在性。
- 所有运行时行为。

## 4. 下一轮处理

- 认证层闭合（需密钥基础设施）。
- 运行时验证（需单独授权）。

## 5. 说明
05.29 的 DESIGN_PROVEN 不自动继承为 05.30 的 DESIGN_PROVEN；本轮各结论独立标注证据等级。
