# USIT-STAGE-B-DESIGN-05.35

## 0. 工单与结论摘要

| 项 | 值 |
|---|---|
| 工单编号 | USIT-STAGE-B-DESIGN-05.35 |
| 前置工单 | DESIGN-05.34 |
| 基线 | commit 57878fc |
| 工作类型 | READ-ONLY ANALYSIS + DESIGN REVISION |
| 本轮目标 | 修复 05.34 复审发现的三个核心阻断，不扩展实现范围 |
| 整体结论 | PASS-WITH-ISSUES（B35-01/02/03 闭合；锁外写入者 UNKNOWN，身份/Receipt/Break-glass NOT_PROVEN 继承） |

只改设计文档，不改源码。

---

## 1. 三个核心阻断修订结论

| 编号 | 问题 | 结论 |
|---|---|---|
| B35-01 | `_admit_state` 与 `_admit_count` 两套状态源 | 统一为单一原子状态字，删除独立安全判断副本 |
| B35-02 | 状态字掩码/字段更新/DENIED 保护 | 逐位复核，修复 15 位掩码，DENIED 不可逆保护 |
| B35-03 | 异常路径与崩溃恢复闭合 | 统一异常状态机 + 崩溃恢复协议 + 幂等清理 |

---

## 2. B35-01｜统一准入状态源

05.34 存在 `_admit_state`（原子字）与 `_admit_count`（独立计数器）两套源，可能冻结判断不一致。

修复：**唯一权威准入状态源 = 单一原子状态字 `_admit_state`**。删除独立 `_admit_count`。条件变量等待协议**只观察 `_admit_state` 内的 count 字段**，禁止使用独立滞后计数器。

准入增加/退出/Gate 关闭/冻结完成全部通过同一原子字 CAS 完成。详见 `ADMISSION-STATE-SINGLE-SOURCE.md`。

---

## 3. B35-02｜原子状态字完整修复

- 位布局：bit0=Gate、bit1–15=admit_count（15 位）、bit16=frozen、bit17=DENIED。
- 修复掩码：15 位 count 掩码 = `0x7FFE`（覆盖 bit1–15 全部 15 位），不得遗漏最高位。
- field-aware CAS，禁止裸 `state+1`。
- DENIED 放入状态字内，且**任何 unfreeze/rollback/恢复/父子初始化都不得清除 DENIED 位**（用独立 CAS，前置条件 gate!=DENIED）。

详见 `ATOMIC-STATE-WORD.md` 与 `STATE-TRANSITION-MATRIX.md`。

---

## 4. B35-03｜异常路径与崩溃恢复闭合

统一异常状态机覆盖：快照失败/Fork 失败或中断/父子初始化失败/锁重建失败/身份重建失败/Receipt 验证失败/commit/receipt idle 断言失败/清理回滚恢复失败/线程异常取消/进程崩溃强制终止/系统崩溃重启恢复。

关键规则：清理幂等；清理失败进入隔离不恢复 OPEN；线程 finally 不覆盖进程崩溃；崩溃后通过磁盘证据发现遗留记录、识别事务归属、完成隔离；无法确认遗留事务已提交则不自动重试。

详见 `FORK-EXCEPTION-STATE-MACHINE.md` 与 `CRASH-RECOVERY-PROTOCOL.md`。

---

## 5. 历史安全约束继承（不弱化）

- 跨快照身份唯一性 NOT_PROVEN。
- Receipt 创建者认证 NOT_PROVEN → 隔离/禁用。
- DENIED 不自动重开。
- Break-glass 无独立可信授权审计根 → 禁用。
- 锁外写入者 UNKNOWN/NOT_PROVEN → Fork DENY。
- Commit/Receipt 写入入口未全部证明受控 → Fork DENY。
- Source Retire DENY；IMPLEMENT-01/A-B Merge NOT_APPROVED。

---

## 6. 反例与交叉验证

新增 30 条反例（累计 ≥130），覆盖双状态源不一致、掩码错误、CAS 跨字段、溢出/下溢、状态转换竞争、DENIED 竞争、CAS 活锁、快照/Fork 异常、finally 未执行、崩溃重启、清理重复、遗留计数等。

见 `COUNTEREXAMPLES.json` 与 `COUNTEREXAMPLE-RESULTS.md`。

---

## 7. 最终状态

- 整体结论：**PASS-WITH-ISSUES**。
- IMPLEMENT-01：NOT_APPROVED
- A/B Merge：NOT_APPROVED
- Source Retire：DENY
- 下一步：STOP，等待 ChatGPT 独立复审。
