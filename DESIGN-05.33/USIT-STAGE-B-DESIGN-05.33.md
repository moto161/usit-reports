# USIT-STAGE-B-DESIGN-05.33

## 0. 工单与结论摘要

| 项 | 值 |
|---|---|
| 工单编号 | USIT-STAGE-B-DESIGN-05.33 |
| 前置工单 | DESIGN-05.32 |
| 基线 | commit 9909447 |
| 工作类型 | STRICT READ-ONLY ANALYSIS + DESIGN REVISION |
| 本轮目标 | 修复 05.32 复审 FAIL 三项，不扩大范围 |
| 整体结论 | PASS-WITH-ISSUES（B33-01/02/03 闭合；身份/Receipt/Break-glass NOT_PROVEN 继续继承） |

本轮只处理 B33-01/02/03，不进入实现。

---

## 1. 05.32 复审 FAIL 三个问题与修订结论

| 编号 | 问题 | 修订结论 |
|---|---|---|
| B33-01 | `_fork_coordinate()` 持 coord 时调用业务锁 | coord 与业务锁彻底解耦，冻结协议重构 |
| B33-02 | Wait-for Graph 说 coord 孤立，但实际等业务归零 | 准入原子化，明确线性化点，消除竞争窗口 |
| B33-03 | Receipt 锁 Fork 期间可能仍被持有，快照一致性未闭合 | 业务/提交/Receipt/快照一致性重新定义，父子事务所有权闭合 |

---

## 2. B33-01｜Fork 冻结协议重构

核心：`_fork_coord_lock` 不与任何业务锁（`_admit_lock`/`_commit_lock`/Receipt 锁/`_gate_state_lock`）同时持有，也不在 coord 作用域内间接调用获取业务锁的函数。

方案：**原子状态转换 + 独立短临界区**。Gate 与准入计数通过**单一原子字**（CAS）更新，消除「先检查 Gate 再增计数」的分离窗口。coord 锁仅用于**串行化冻结发起者**（保证同一时刻只有一个冻结者），其作用域内不获取任何业务锁。

详见 `FORK-FREEZE-PROTOCOL.md`。

---

## 3. B33-02｜准入竞争窗口消除

通过「Gate + 准入计数」合并为单一原子状态字，准入的线性化点 = 原子 CAS 成功点。冻结的线性化点 = 原子 CAS 置 FREEZING 成功点。

关键不变量：**冻结 CAS 成功之后，任何新的业务准入 CAS 必然失败**（因为准入 CAS 的前置条件包含 Gate==OPEN，而 Gate 已不再是 OPEN）。

详见 `GATE-ADMISSION-ATOMICITY.md` 与 `FORK-CONCURRENCY-TIMELINE.md`。

---

## 4. B33-03｜业务/提交/Receipt/快照一致性

重新定义 Fork 前置条件，逐项明确哪些状态必须归零、允许保留、必须隔离、阻止 Fork。建立业务归零判定表、Receipt 状态与 Fork 许可矩阵、快照一致性不变量、父子事务所有权表。

详见 `BUSINESS-RECEIPT-SNAPSHOT-CONSISTENCY.md` 与 `FORK-TRANSACTION-OWNERSHIP.md`。

---

## 5. 全局锁协议与死锁证明

唯一权威锁清单、调用链（含间接）、Wait-for Graph、死锁环检测。coord 不再被描述为「无等待边」——Fork 冻结者等待**业务归零条件变量**，该等待的语义是「等待计数归零」，不涉及持有业务锁，因此不形成锁环。详见 `GLOBAL-LOCK-ORDER.md` 与 `LOCK-WAIT-FOR-GRAPH.md`。

---

## 6. 反例验证

60+ 条真实反例（20 Gate/准入竞争 + 20 Fork 冻结锁等待 + 20 业务/提交/Receipt/快照交错）。见 `COUNTEREXAMPLES.json` 与 `COUNTEREXAMPLE-RESULTS.md`。

---

## 7. 继承的安全约束（不弱化）

- 身份防回滚 NOT_PROVEN → 功能禁用。
- Receipt Creator Authentication NOT_PROVEN → 事务恢复/幂等成功确认禁用。
- 未认证 Receipt → QUARANTINED。
- DENIED 不被普通流程自动恢复。
- Break-glass 审计信任根 NOT_PROVEN → 紧急恢复禁用。
- 不将 UNKNOWN/NOT_PROVEN 改写为 DESIGN_PROVEN。
- 不把静态推演写成运行时验证。

---

## 8. 最终状态

- 整体结论：**PASS-WITH-ISSUES**（B33-01/02/03 闭合；身份/Receipt/Break-glass NOT_PROVEN 继续继承）。
- IMPLEMENT-01：NOT_APPROVED
- A/B Merge：NOT_APPROVED
- Source Retire：DENY
- 下一步：STOP，等待 ChatGPT 独立复审。
