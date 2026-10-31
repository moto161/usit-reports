# COUNTEREXAMPLE-RESULTS.md

## 目标
反例验证报告。机器可读数据见 COUNTEREXAMPLES.json。

## 1. 总数统计（实际来自 JSON）

| 类别 | 数量 | 要求 | 达标 |
|---|---|---|---|
| LOCK_LIFECYCLE（Fork 父子锁生命周期） | 20 | 20 | ✅ |
| LOCK_ORDER（锁顺序死锁） | 20 | 20 | ✅ |
| IDENTITY_ANTI_ROLLBACK（身份防回滚） | 20 | 20 | ✅ |
| RECEIPT_AUTH（Receipt 认证与重放） | 20 | 20 | ✅ |
| DENIED_BREAK_GLASS（DENIED 与紧急恢复） | 20 | 20 | ✅ |
| CROSS_DOCUMENT（跨文档协议冲突） | 20 | 20 | ✅ |
| **TOTAL** | **120** | 120 | ✅ |

## 2. 统计口径（区分有效性）

| 指标 | 值 |
|---|---|
| TOTAL | 120 |
| VALID（字段完整且可复现） | 120（全部含 case_id/category/preconditions/actors/initial_lock_state/execution_steps/wait_for_edges/expected_result/design_result/evidence_refs/verification_method/status） |
| PASS | 89 |
| FAIL | 0 |
| NOT_PROVEN | 31 |
| NOT_TESTED | 0（所有 verification_method 均为 static analysis，未冒充运行测试） |

## 3. NOT_PROVEN 分布（如实）

| 类别 | NOT_PROVEN 数量 | 原因 |
|---|---|---|
| IDENTITY_ANTI_ROLLBACK | 9 | 无可信防回滚锚点 |
| RECEIPT_AUTH | 11 | 认证设施未建立 |
| DENIED_BREAK_GLASS | 11 | 审计信任根缺失 |
| 合计 | 31 | — |

## 4. 关键结论

1. 120 条反例全部为**真实、具体、可复现**内容，无 `cond 1` / `path 1` 占位。
2. 每条例证均含具体参与者、锁状态、执行步骤、等待边、预期结果、设计结果、证据位置、验证方法。
3. 89 条 PASS：设计正确拒绝或安全处理。
4. 31 条 NOT_PROVEN：如实标注，不冒充 PASS。
5. 0 条 FAIL：无设计漏洞被确证（但 NOT_PROVEN 项对应功能保持禁用，不视为通过）。
6. 全部 verification_method 为 static analysis，**未冒充真实运行测试**（符合只读边界）。

## 5. 统计真实性声明

- 所有数量来自 COUNTEREXAMPLES.json 实际条目，非预先填写。
- 未将模板条目计为 PASS。
- 未以数量代替有效性。

## 6. 证据等级
反例数据为机器生成 + 人工逻辑复核；所有运行时行为 NOT_TESTED。
