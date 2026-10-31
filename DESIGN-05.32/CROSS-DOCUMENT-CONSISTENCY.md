# CROSS-DOCUMENT-CONSISTENCY.md

## 目标
跨文档一致性报告。

## 1. 检查范围

对 05.32 全部交付文档 + 引用的 05.31 文档做交叉一致性检查。

## 2. 冲突项及处理

| # | 冲突 | 处理 | 结果 |
|---|---|---|---|
| 1 | 05.31 coord 与 admit/commit 嵌套矛盾 | GLOBAL-LOCK-ORDER 裁决：禁止嵌套，Gate 用独立 gate_state 锁 | 已消除 |
| 2 | 05.31 Fork 锁重复释放 | FORK-LOCK-LIFECYCLE 唯一释放责任人 | 已消除 |
| 3 | 05.31 反例占位内容 | 重建 120 条真实反例 | 已消除 |
| 4 | 05.31 Break-glass 审计信任根未证明 | BREAK-GLASS-TRUST-ROOT 如实 NOT_PROVEN | 已消除 |
| 5 | 锁重建/释放/所有权混淆 | FORK-LOCK-LIFECYCLE 明确区分 | 已消除 |
| 6 | coord 所有者不一致 | FORK-LOCK-LIFECYCLE 唯一所有者 | 已消除 |

## 3. 冲突数量统计

- 发现冲突：6 项（05.31 遗留）。
- 已处理：6 项。
- 未解释冲突：**0**。

## 4. 单一权威定义清单

| 主题 | 权威文档 |
|---|---|
| 锁顺序 | GLOBAL-LOCK-ORDER.md |
| 锁生命周期 | FORK-LOCK-LIFECYCLE.md |
| 父子路径 | FORK-PARENT-CHILD-PATHS.md |
| 死锁证明 | LOCK-WAIT-FOR-GRAPH.md |
| Break-glass 信任根 | BREAK-GLASS-TRUST-ROOT.md |
| 反例数据 | COUNTEREXAMPLES.json |

其他文档只引用权威定义，不重复产生冲突规则。

## 5. 05.31 安全约束继承检查

| 约束 | 05.32 是否继承 |
|---|---|
| 身份防回滚 NOT_PROVEN → 功能禁用 | ✅ |
| Receipt 认证 NOT_PROVEN → 功能禁用 | ✅ |
| 未认证 Receipt → QUARANTINED | ✅ |
| DENIED 不被普通流程自动恢复 | ✅ |
| Break-glass 独立授权 + 审计 + 新 Epoch | ✅ |
| 不将 UNKNOWN/NOT_PROVEN 改写为 VERIFIED/DESIGN_PROVEN | ✅ |
| 不用随机 UUID/普通计数器/普通自增冒充防回滚 | ✅ |

## 6. 结论

跨文档协议冲突已全部消除，无未解释冲突。05.31 安全约束全部继承，未弱化。

## 7. 证据等级
一致性检查：VERIFIED_STATIC；运行时：NOT_TESTED。
