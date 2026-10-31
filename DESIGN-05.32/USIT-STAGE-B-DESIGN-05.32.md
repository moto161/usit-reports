# USIT-STAGE-B-DESIGN-05.32

## 0. 工单与结论摘要

| 项 | 值 |
|---|---|
| 工单编号 | USIT-STAGE-B-DESIGN-05.32 |
| 前置工单 | DESIGN-05.31 |
| 工作类型 | STRICT READ-ONLY ANALYSIS + DESIGN REVISION |
| 本轮目标 | 修复 05.31 复审 FAIL 四项，不扩大范围 |
| 整体结论 | PASS-WITH-ISSUES（B32-01/02/03 闭合；B32-04 Break-glass 审计信任根 NOT_PROVEN，功能禁用） |

本轮不授权任何源码实现，不批准 IMPLEMENT-01。定点修订四项。

---

## 1. 四项修订结论

| 编号 | 问题 | 结论 |
|---|---|---|
| B32-01 | Fork 锁重复释放及父进程生命周期不清 | 唯一所有者 + 唯一释放责任矩阵，彻底闭合 |
| B32-02 | coord 与 admit/commit 嵌套矛盾 | 唯一权威锁顺序，消除嵌套冲突 |
| B32-03 | 150 条反例存在占位内容 | 重建 120+ 条真实可复现反例 |
| B32-04 | Break-glass 审计信任根不完整 | 无可信外部信任根 → NOT_PROVEN，紧急恢复功能禁用 |

---

## 2. B32-01｜Fork 锁生命周期彻底闭合

05.31 发现的问题：`with self._fork_coord_lock:` 退出自动释放，`_parent_path()` 可能再次释放 → 重复释放。

核心修订原则：
- **每把锁只有一个所有者**（owner）。
- **每次获取锁对应唯一一个释放责任人**（release owner）。
- 区分：自动释放（with 退出）、显式释放（明确 release）、子进程清理继承状态。
- 父进程与子进程分别拥有独立锁实例，互不释放对方锁。

权威定义见 `FORK-LOCK-LIFECYCLE.md`，父子路径见 `FORK-PARENT-CHILD-PATHS.md`。

---

## 3. B32-02｜唯一全局锁顺序

05.31 矛盾：一处 `with _fork_coord_lock: with _admit_lock:`，另一处规定 coord 不得与业务锁嵌套。

本轮裁决（唯一权威，禁止保留两套规则）：

**Fork 协调锁 `_fork_coord_lock` 与准入锁 `_admit_lock`、提交锁 `_commit_lock` 不允许嵌套获取。**

修正后的 fork 协调结构：coord 锁内只做 Gate 状态检查与 `admit_closed` 置位（通过**独立的 `_gate_state_lock`** 保护 Gate 字段写入，而非在 coord 内获取 admit 锁）。admit 锁的获取与释放由业务线程独立进行，不进入 coord 锁作用域。

权威锁顺序见 `GLOBAL-LOCK-ORDER.md`，等待图与死锁证明见 `LOCK-WAIT-FOR-GRAPH.md`。

---

## 4. B32-03｜重建真实有效反例

05.31 的 `cond 1`、`path 1` 占位内容作废。重建 120+ 条真实反例，每条含完整字段（case_id / category / preconditions / actors / initial_lock_state / execution_steps / wait_for_edges / expected_result / design_result / evidence_refs / verification_method / status）。

见 `COUNTEREXAMPLES.json` 与 `COUNTEREXAMPLE-RESULTS.md`。

---

## 5. B32-04｜Break-glass 审计信任根

05.31 提出 append-only 日志 + 哈希链，但未证明不可回滚性。

本轮结论：**当前只读环境无可信外部信任根**。本地 append-only 文件、本地数据库 + 哈希链均不能提供严格防回滚保证（快照/磁盘恢复可整体回滚哈希链）。哈希链本身不是绝对不可篡改。

因此：
- Break-glass 审计信任根：**NOT_PROVEN**。
- 紧急恢复功能：**保持禁用**（审计完整性无法确认时拒绝执行）。
- 不自动解除 DENIED，不让普通管理员绕过授权流程。

详见 `BREAK-GLASS-TRUST-ROOT.md`。

---

## 6. 继承 05.31 安全约束（不弱化）

- 身份防回滚 NOT_PROVEN → 相关功能禁用。
- Receipt 认证 NOT_PROVEN → 事务恢复/幂等成功确认禁用。
- 未认证 Receipt → QUARANTINED。
- DENIED 不被普通流程自动恢复。
- Break-glass 独立授权 + 审计 + 新 Epoch。
- 不将 UNKNOWN/NOT_PROVEN 改写成 VERIFIED/DESIGN_PROVEN。
- 不用随机 UUID/普通计数器/普通自增冒充防回滚证明。

---

## 7. 最终状态

- 整体结论：**PASS-WITH-ISSUES**（B32-01/02/03 闭合；B32-04 NOT_PROVEN，功能禁用）。
- IMPLEMENT-01：NOT_APPROVED
- A/B Merge：NOT_APPROVED
- Source Retire：DENY
- 下一步：STOP，等待 ChatGPT 独立复审。
