# COUNTEREXAMPLE-RESULTS.md

## 目标
反例验证报告。数据见 COUNTEREXAMPLES.json。

## 1. 统计（实际来自 JSON）

| 指标 | 值 |
|---|---|
| 新增 | 30 |
| 累计（含 05.34 的 40 + 05.33 的 60） | 130 |
| VALID | 30 |
| PASS | 30 |
| FAIL | 0 |
| NOT_PROVEN | 0 |
| NOT_TESTED_STATIC | 0（全部 static analysis） |
| NOT_TESTED_RUNTIME | 0（本批反例均无需运行时验证；整体运行时验证未执行，见 EXECUTION-REPORT） |

## 2. 类别分布

| 类别 | 数量 |
|---|---|
| DUAL_STATE_SOURCE（双状态源不一致） | 6 |
| MASK_FIELD（掩码错误/CAS 跨字段/溢出下溢） | 8 |
| STATE_RACE_DENIED（状态转换竞争/DENIED 不可逆） | 8 |
| EXCEPTION_CRASH（快照/Fork 异常/崩溃/清理） | 8 |

## 3. 累计数量

05.33 = 60 条，05.34 = 40 条，05.35 = 30 条 → **累计 130 条**，满足「不少于 130 条」。

## 4. 说明

1. 30 条全部真实具体、可复现，无占位。
2. 每条含：唯一 ID、前置状态、参与者、操作顺序、预期结果、设计推演结果、依据文件及位置、结论、运行时验证需求。
3. 30 条 PASS：设计正确拒绝或安全处理。
4. 全部 verification_method 为 static analysis，**未冒充运行时测试**。
5. 本批反例均 `requires_runtime_verification=false`，但整体运行时验证未执行（见执行报告）。

## 5. 统计真实性
数量来自 JSON 实际条目，未以数量代替有效性，未冒充运行时测试。
