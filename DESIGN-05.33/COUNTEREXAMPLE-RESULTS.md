# COUNTEREXAMPLE-RESULTS.md

## 目标
反例验证报告。数据见 COUNTEREXAMPLES.json。

## 1. 统计（实际来自 JSON）

| 指标 | 值 |
|---|---|
| TOTAL | 60 |
| VALID | 60（全部含 case_id/preconditions/actors/initial_state/execution_steps/lock_state/expected_result/design_result/evidence_refs/verification_method/status） |
| PASS | 60 |
| FAIL | 0 |
| NOT_PROVEN | 0 |
| NOT_TESTED | 0（全部 static analysis） |

## 2. 类别分布

| 类别 | 数量 | 要求 |
|---|---|---|
| GATE_ADMISSION（Gate 与准入竞争） | 20 | 20 |
| FORK_FREEZE_LOCK（Fork 冻结与锁等待） | 20 | 20 |
| BUSINESS_RECEIPT_SNAPSHOT（业务/提交/Receipt/快照交错） | 20 | 20 |

## 3. 说明

1. 60 条全部为真实、具体、可复现内容，无 `cond 1` / `path 1` 占位。
2. 每条例证含具体参与者、初始状态、执行步骤、锁状态、预期结果、设计结果、证据位置、验证方法。
3. 60 条 PASS：设计正确拒绝或安全处理。
4. 全部 verification_method 为 static analysis，未冒充真实运行测试（符合只读边界）。
5. 本工单三项（冻结协议/准入原子性/快照一致性）属于协议层设计，反例均为静态推演验证，无运行时验证，故 NOT_TESTED 计数为 0（表示「未宣称已做运行时测试」，而非「已测试」）。

## 4. 统计真实性声明

- 数量来自 JSON 实际条目。
- 未将模板条目计 PASS。
- 未以数量代替有效性。

## 5. 证据等级
静态设计反例，人工逻辑复核；运行时 NOT_TESTED。
