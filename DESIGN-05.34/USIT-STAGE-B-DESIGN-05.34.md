# USIT-STAGE-B-DESIGN-05.34

## 0. 工单与结论摘要

| 项 | 值 |
|---|---|
| 工单编号 | USIT-STAGE-B-DESIGN-05.34 |
| 前置工单 | DESIGN-05.33 |
| 基线 | commit 1ad81b6 |
| 工作类型 | READ-ONLY ANALYSIS + DESIGN REVISION |
| 本轮目标 | 修复 05.33 复审发现的五项并发/异常/原子字/生命周期证明缺口 |
| 整体结论 | PASS-WITH-ISSUES（B34-01~04 协议层闭合；锁外写入者边界 UNKNOWN，身份/Receipt/Break-glass NOT_PROVEN 继承） |

只改设计文档与证明，不改任何项目源码。

---

## 1. 五项问题修订结论

| 编号 | 问题 | 结论 |
|---|---|---|
| B34-01 | 条件变量丢失唤醒 | generation 计数 + 锁内重查 predicate，闭合丢失唤醒窗口 |
| B34-02 | Fork/快照异常路径闭合 | 完整异常状态机 + 清理责任矩阵 + 父子隔离 |
| B34-03 | 原子状态字与计数器边界 | field-aware CAS + 溢出/下溢/非法组合拒绝策略 |
| B34-04 | Commit/Receipt/锁外写入者冻结覆盖 | 写入入口清单 + 全生命周期证明；无法枚举入口标 UNKNOWN |
| B34-05 | 反例与证据质量 | 40 条新增反例，累计 ≥100 |

---

## 2. B34-01｜条件变量丢失唤醒

05.33 的 `admit_count==0` 检查与 `_wait_admit_drain()` 之间存在丢失唤醒窗口。

修复：引入 **generation 计数器**（`admit_epoch`），业务退出时在**持锁**下递减 admit_count 并递增 admit_epoch；冻结者等待的 predicate 为 `(admit_count==0)`，且**检查与等待注册在同一锁内原子完成**。

- predicate 由 `_drain_lock`（Mutex）+ `_drain_cond`（Condition）保护。
- 冻结者：持锁 → 检查 predicate → 若满足立即返回，否则 wait（wait 自动释放锁，唤醒后重新持锁重查）。
- 最后一个业务退出：持锁 → admit_count 递减 → notify_all。
- 通知先于等待：generation 使冻结者 wait 前若已归零则不进入 wait，直接返回。

完整协议见 `FORK-FREEZE-PROTOCOL.md`。

---

## 3. B34-02｜Fork 与快照异常路径闭合

为 `_take_snapshot()` / `_do_fork()` / 父初始化 / 子初始化 / 锁重建 / 身份重建 / Receipt 检查 / 清理失败等定义完整异常状态机与清理顺序。

关键规则：父进程严格不执行子进程专属初始化；清理失败不误恢复 OPEN、不继续 Fork。`os.fork()` 异常按其真实行为（抛 `OSError`）设计，不把抽象 `pid=-1` 当正常失败机制。

详见 `FORK-EXCEPTION-STATE-MACHINE.md`。

---

## 4. B34-03｜原子状态字与计数器边界

状态字 `_admit_state` 字段：bit0 Gate、bit1–15 admit_count、bit16 frozen。给出掩码、取值范围、合法组合、溢出(32767)/下溢/非法位拒绝策略、CAS 失败重试与活锁处理、field-aware CAS（禁止裸 `state+1`）。

补全 `commit_count` / `receipt_write_count` 的增减线性化点、事务归属、异常/崩溃处理、归零通知、不一致检测隔离。

详见 `ATOMIC-STATE-WORD.md`。

---

## 5. B34-04｜写入入口冻结覆盖证明

建立完整写入入口清单（提交/提交确认/Receipt 创建更新持久化重试/事务恢复补偿/后台任务回调定时任务异步线程/锁外写入者），逐项证明每个入口进入 admit 生命周期，commit_count/receipt_write_count 不能在 admit 生命周期外增加或写。

无法枚举/证明受控的锁外写入者继续标 UNKNOWN/NOT_PROVEN，且**只要存在未覆盖入口，Fork 必须 DENY**。

详见 `WRITE-ENTRY-COVERAGE.md`。

---

## 6. B34-05｜反例与证据质量

新增 40 条反例（覆盖丢失唤醒/虚假唤醒/通知先于等待/超时取消/快照失败/Fork 异常/父子初始化失败/清理失败/计数器溢出下溢/CAS 竞争/非法状态/绕过 Gate/后台写入/锁外写入/崩溃恢复/重复清理），累计 ≥100 条。

统计区分 `VALID / PASS / FAIL / NOT_PROVEN / NOT_TESTED_STATIC / NOT_TESTED_RUNTIME`。运行时未执行，NOT_TESTED_RUNTIME 如实统计（非 0）。

见 `COUNTEREXAMPLES.json` 与 `COUNTEREXAMPLE-RESULTS.md`。

---

## 7. 继承的安全约束

- 跨快照身份唯一性 NOT_PROVEN。
- Receipt 创建者认证 NOT_PROVEN。
- DENIED 不自动重开。
- Break-glass 独立授权 + 审计根，无法证明则禁用。
- UNKNOWN/NOT_PROVEN 不包装为 PASS。
- Source Retire 继续 DENY。

---

## 8. 最终状态

- 整体结论：**PASS-WITH-ISSUES**。
- IMPLEMENT-01：NOT_APPROVED
- A/B Merge：NOT_APPROVED
- Source Retire：DENY
- 下一步：STOP，等待 ChatGPT 独立复审。
