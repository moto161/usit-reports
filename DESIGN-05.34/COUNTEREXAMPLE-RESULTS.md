# COUNTEREXAMPLE-RESULTS.md

## 目标
反例验证报告。数据见 COUNTEREXAMPLES.json。

## 1. 统计（实际来自 JSON）

| 指标 | 值 |
|---|---|
| TOTAL | 40 |
| VALID | 40（字段完整可复现） |
| PASS | 39 |
| FAIL | 0 |
| NOT_PROVEN | 1（锁外写入者） |
| NOT_TESTED_STATIC | 0（全部 static analysis，未宣称运行时） |
| NOT_TESTED_RUNTIME | 1（锁外写入者 requires_runtime_verification=true，如实未测试） |

## 2. 类别分布

| 类别 | 数量 |
|---|---|
| LOST_WAKEUP（丢失/虚假唤醒/通知先于等待/超时/取消） | 10 |
| FORK_EXCEPTION（快照/Fork/父子初始化/清理失败） | 10 |
| COUNTER_BOUNDARY（溢出/下溢/CAS 竞争/非法状态） | 10 |
| WRITE_ENTRY_COVERAGE（绕过 Gate/后台写入/锁外写入） | 10 |

## 3. 累计数量

本工单要求「新增 40 条，累计 ≥100」。05.33 已交付 60 条，本工单新增 40 条 → **累计 100 条**。05.33 的 60 条在 `DESIGN-05.33/COUNTEREXAMPLES.json`，本工单 40 条在 `DESIGN-05.34/COUNTEREXAMPLES.json`。

## 4. NOT_TESTED_RUNTIME 说明

- 本工单禁止真实 Fork 与运行时实验。
- 所有反例 verification_method 均为 static analysis。
- 锁外写入者反例（B34-BYP-010）requires_runtime_verification=true，如实标注 NOT_PROVEN，未执行运行时验证。
- **NOT_TESTED_RUNTIME = 1**，非 0，不暗示运行时覆盖完成。

## 5. 关键结论

1. 40 条反例真实具体、可复现，无占位。
2. 39 条 PASS：设计正确拒绝或安全处理。
3. 1 条 NOT_PROVEN：锁外写入者无法证明受控 → Fork DENY。
4. 运行时验证未执行，如实统计。

## 6. 统计真实性
数量来自 JSON 实际条目，未冒充，未以数量代替有效性。
