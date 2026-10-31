# GLOBAL-PROTOCOL-INDEX.md

## 目标
建立 05.31 全局协议索引，覆盖 12 项协议，记录权威定义与引用关系。

格式：`协议编号｜权威定义文档｜引用文档｜关键不变量｜验证证据｜当前状态`

| 协议编号 | 权威定义文档 | 引用文档 | 关键不变量 | 验证证据 | 当前状态 |
|---|---|---|---|---|---|
| P-01 Fork 生命周期 | FORK-PARENT-CHILD-ISOLATION.md | FORK-LOCK-ORDER-UNIFIED.md | I-FPC-1 | 状态转换表 + 不变量证明 | DESIGN_PROVEN |
| P-02 父进程恢复 | PARENT-RESTORE-PROTOCOL.md (05.30) | FORK-PARENT-CHILD-ISOLATION.md | I-PR-1~4 | 11 情况覆盖 | DESIGN_PROVEN |
| P-03 子进程重建 | FORK-PARENT-CHILD-ISOLATION.md | FORK-LOCK-OWNERSHIP.md (05.30) | 子重建全部锁 | 7 锁重建表 | DESIGN_PROVEN |
| P-04 锁顺序 | FORK-LOCK-ORDER-UNIFIED.md（权威） | FORK-LOCK-LIFECYCLE/OWNERSHIP/LOCK-ORDER-PROOF | 全局顺序唯一 | 顺序表 + 20 死锁反例 | DESIGN_PROVEN |
| P-05 锁所有权 | FORK-LOCK-OWNERSHIP.md (05.30) | FORK-LOCK-ORDER-UNIFIED.md | 父子独立 | 7 锁所有权表 | DESIGN_PROVEN |
| P-06 身份唯一性 | IDENTITY-ANTI-ROLLBACK.md | PROCESS-TREE-IDENTITY.md (05.30) | 单次运行内结构唯一 | branch_path 归纳证明 | 单次 DESIGN_PROVEN；跨快照 NOT_PROVEN |
| P-07 快照恢复 | IDENTITY-ANTI-ROLLBACK.md | — | 防回滚锚点缺失 | 4 方案分析 | NOT_PROVEN |
| P-08 Receipt 创建与认证 | RECEIPT-AUTH-BOUNDARY.md | RECEIPT-AUTHENTICITY.md (05.30) | 认证未建立则禁用 | 6 层证据 + 禁用清单 | NOT_PROVEN（认证） |
| P-09 Receipt 幂等 | RECEIPT-IDEMPOTENCY.md (05.30) | RECEIPT-AUTH-BOUNDARY.md | 全证据匹配才成功 | 30 反例 | DESIGN_PROVEN（协议），认证依赖 NOT_PROVEN |
| P-10 DENIED 状态机 | DENIED-BREAK-GLASS-STATE-MACHINE.md | DENIED-STATE-MACHINE.md (05.30) | I-BG-1~5 | 状态转换 + 反例 | DESIGN_PROVEN |
| P-11 Break-glass 恢复 | DENIED-BREAK-GLASS-STATE-MACHINE.md | — | 双人审批 + 审计 + 新 Epoch | 转换表 | DESIGN_PROVEN |
| P-12 全局安全不变量 | INVARIANTS.md (05.30) | 各协议文档 | I-001~I-080 | 80 条 | DESIGN_PROVEN |

## 2. 权威文档唯一性

- 锁顺序：FORK-LOCK-ORDER-UNIFIED.md 是唯一权威，其余引用。
- 身份防回滚：IDENTITY-ANTI-ROLLBACK.md 是唯一权威。
- Receipt 认证边界：RECEIPT-AUTH-BOUNDARY.md 是唯一权威。
- DENIED：DENIED-BREAK-GLASS-STATE-MACHINE.md 是唯一权威。

## 3. 冲突检查结论

见 CROSS-DOCUMENT-CONSISTENCY.md。本轮已消除 05.30 的主要矛盾（DENIED 语义、锁顺序、身份唯一性边界）。
