# COUNTEREXAMPLE-RESULTS.md — 反例验证报告（05.37）

证据等级：VERIFIED_STATIC（全部静态推演，非运行时测试）

---

## 1. 本轮统计

| 指标 | 值 |
|---|---|
| TOTAL | 30（本轮新增） |
| VALID | 30 |
| PASS | 30 |
| FAIL | 0 |
| NOT_PROVEN | 0 |
| NOT_TESTED_STATIC | 0 |
| NOT_TESTED_RUNTIME | 30（未执行运行时测试，如实标注） |

## 2. 类别分布

| 类别 | 数量 | 对应问题 |
|---|---|---|
| FREEZE_UNIFY | 8 | B37-01 冻结状态机统一 |
| CLEANUP_OWNERSHIP | 8 | B37-02 清理所有权 |
| RECOVERY_CAS | 7 | B37-03 恢复 CAS |
| FIELD_OPS | 7 | B37-04 字段操作统一 |

## 3. 累计统计（含历史核验，权威口径见 HISTORICAL-COUNTEREXAMPLE-AUDIT.md）

```
PER_VERSION:
  05.33: 60
  05.34: 40
  05.35: 30
  05.36: 30
  05.37: 30

UNIQUE_TOTAL: 190
DUPLICATE_ID: 0
INVALID: 0
VALID: 190
PASS: 185（审计修正口径）
FAIL: 4（05.35 的 B35-MASK-001/002 修正 + 05.36 的 2 条历史记录）
NOT_PROVEN: 1（05.34 的 1 条）
NOT_TESTED_STATIC: 0
NOT_TESTED_RUNTIME: 190
```

## 4. 验证方法声明

本轮 30 条全部为静态设计推演（STATIC_DEDUCTION），通过伪代码 + 状态机 + CAS 协议推演验证，**未执行任何真实 Fork、POSIX 事务、锁竞争或运行时测试**（工单禁止）。`NOT_TESTED_RUNTIME=30` 如实标注。

## 5. 关键反例摘录

- `B37-FREEZE-004`：DENIED=1 时 cas_frozen 拒绝设置 frozen。
- `B37-CLEAN-001`：两个执行者竞争清理所有权，只有一个成功。
- `B37-RECOV-002`：恢复期间 DENIED 并发设置，恢复不得覆盖 DENIED。
- `B37-FIELD-001`：准入禁止 `cur+(1<<1)` 整字算术，必须 field_cas。

---

## 证据来源

- 反例数据：COUNTEREXAMPLES.json
- 历史核验：HISTORICAL-COUNTEREXAMPLE-AUDIT.md
- 证明等级：VERIFIED_STATIC
