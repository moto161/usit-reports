# USIT-STAGE-B-DESIGN-05.31

## 0. 工单与结论摘要

| 项 | 值 |
|---|---|
| 工单编号 | USIT-STAGE-B-DESIGN-05.31 |
| 前置工单 | DESIGN-05.30 |
| 工作类型 | STRICT READ-ONLY ANALYSIS + DESIGN REVISION |
| 本轮目标 | 修复 05.30 复审发现的六项协议级缺陷，形成无内部矛盾的 05.31 |
| 整体结论 | PASS-WITH-ISSUES（身份防回滚与 Receipt 认证如实 NOT_PROVEN，功能保持禁用） |

本轮为**只读设计修订**，不批准任何实现。定点修订六项，不扩大范围。

---

## 1. 六项定点修订结论

| 编号 | 问题 | 修订结论 |
|---|---|---|
| B31-01 | 父进程误执行子进程初始化 | 父子执行路径严格隔离，状态转换表 + 不变量证明 |
| B31-02 | Fork 锁获取顺序冲突 | 统一权威锁顺序表，单一权威定义 |
| B31-03 | 快照恢复导致身份计数器回滚 | 无可信防回滚锚点 → **NOT_PROVEN**，功能禁用 |
| B31-04 | Receipt 创建者认证 UNKNOWN | 认证未闭合 → 功能禁用清单 + 重新启用条件 |
| B31-05 | DENIED 与 manual_reenable 冲突 | Break-glass 语义重定义，无语义冲突 |
| B31-06 | 文档间协议矛盾 | 全局协议索引 + 跨文档一致性检查 |

---

## 2. B31-01｜Fork 父子执行路径严格隔离

核心修订：父进程路径与子进程路径**完全分离**，父进程绝不执行子进程专属初始化（锁重建、身份重建、Gate 重建）。

- 父进程路径：保留原有锁/身份/Gate/资源所有权，仅释放协调锁并恢复 Gate。
- 子进程路径：清理继承锁状态 → 重建子进程专属锁/身份/Gate。
- Fork 失败路径：明确资源释放、状态恢复、安全终态。
- 未知 PID / 异常退出 / 部分初始化失败：明确处理规则。

详见 `FORK-PARENT-CHILD-ISOLATION.md`。与 `PARENT-RESTORE-PROTOCOL`（05.30）一致性已检查，无冲突。

---

## 3. B31-02｜Fork 锁顺序与生命周期统一

对 `_fork_coord_lock` / `_admit_lock` / `_commit_lock` / 身份分配锁 / Receipt 写入锁做完整梳理，产出**唯一权威锁顺序表**：

```
身份分配锁 → 准入锁 → 提交锁 → 业务资源锁 → Receipt 写入锁
Fork 协调锁：独立串行化，不与业务锁嵌套
```

所有文档引用此权威定义，不再重复产生冲突规则。详见 `FORK-LOCK-ORDER-UNIFIED.md`。

---

## 4. B31-03｜身份唯一性与快照防回滚

05.30 的 `branch_path + local_fork_sequence` 依赖序号永不回退，但恢复旧快照可回退序号。

本轮分析四种候选方案（可信外部单调计数器 / 不可回滚持久化锚点 / 外部身份注册租约 / 其他可验证防回滚机制），结论：

**当前只读环境无任何可信防回滚锚点。** 随机 UUID、普通文件计数器、普通数据库自增字段均不能提供严格防回滚证明。因此：

- 身份防回滚唯一性：**NOT_PROVEN**。
- 依赖防回滚身份的功能：**保持禁用**。
- 不得用概率唯一冒充严格防回滚。

详见 `IDENTITY-ANTI-ROLLBACK.md`。

---

## 5. B31-04｜Receipt 认证未建立时的功能边界

05.30 已标 Creator Authentication 为 UNKNOWN，本轮落实为**明确功能禁用边界**：

- 认证设施未建立时，事务提交恢复、幂等成功确认**必须禁用**（不判成功）。
- QUARANTINED 不自动恢复，不视为认证成功。
- 不得将 UNKNOWN 包装为非核心遗留。
- 列出功能禁用清单及重新启用条件。

详见 `RECEIPT-AUTH-BOUNDARY.md`。

---

## 6. B31-05｜DENIED 与紧急恢复语义统一

05.30 的「DENIED 永久不可逆」与 `manual_reenable()` 可恢复 OPEN 存在语义冲突。本轮重定义：

- DENIED：**禁止普通协议自动恢复**（不可逆于普通路径）。
- Break-glass 紧急恢复：独立的、受控的、双人审批的、审计不可篡改的、创建新安全 Epoch 的管理协议。
- 恢复后：旧授权/旧 Receipt/旧事务全部失效或隔离。
- 不再同时使用「绝对永久不可逆」与「允许人工恢复」两种矛盾定义。

详见 `DENIED-BREAK-GLASS-STATE-MACHINE.md`。

---

## 7. B31-06｜跨文档一致性验证

建立全局协议索引，覆盖 12 项协议，每项记录权威定义文档、引用文档、关键不变量、验证证据、当前状态。详见 `GLOBAL-PROTOCOL-INDEX.md` 与 `CROSS-DOCUMENT-CONSISTENCY.md`。

---

## 8. 强制反例验证

机器可读反例数据 `COUNTEREXAMPLES.json`，覆盖 150 条（20 Fork 父子 + 20 锁死锁 + 30 快照回滚 + 30 Receipt 伪造 + 20 DENIED + 30 跨文档冲突）。结果见 `COUNTEREXAMPLE-RESULTS.md`。

---

## 9. 最终状态

- 整体结论：**PASS-WITH-ISSUES**（协议层闭合；身份防回滚、Receipt 认证如实 NOT_PROVEN，功能禁用）。
- IMPLEMENT-01：NOT_APPROVED
- A/B Merge：NOT_APPROVED
- Source Retire：DENY
- 下一步：STOP，等待 ChatGPT 独立复审。
