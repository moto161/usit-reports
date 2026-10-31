# COUNTEREXAMPLE-RESULTS.md — 反例验证报告（05.38）

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
| NOT_TESTED_RUNTIME | 30 |

## 2. 类别分布（覆盖 B38-01/02/03）

| 类别 | 数量 | 对应问题 |
|---|---|---|
| FENCING | 10 | B38-01 清理资源 Fencing |
| COND_CAS | 10 | B38-02 原子条件 CAS |
| VERSION_ATOMIC | 10 | B38-03 恢复版本原子性 |

## 3. 负向场景覆盖

本轮反例刻意包含失败路径与负向场景（非全理想路径）：
- 旧执行者复活并发释放（B38-FENCE-001/003）。
- Fencing 不支持资源的接管（B38-FENCE-002）。
- 心跳超时≠停止证明（B38-FENCE-004）。
- 释放结果不确定 → ISOLATED（B38-FENCE-005）。
- CAS 重试绕过 DENIED（B38-CAS-001/002/003）。
- 无 token / 重复 token 释放（B38-CAS-004/005）。
- CAS 失败无限重试（B38-CAS-006）。
- ABA 场景（B38-RECOV-002）。
- 版本回绕 / 断电（B38-RECOV-007/009）。
- 证据过期（B38-RECOV-008）。

---

## 4. 统一统计（权威口径，与 HISTORICAL-COUNTEREXAMPLE-AUDIT.md 一致）

```
PER_VERSION:
  05.33: 60
  05.34: 40
  05.35: 30
  05.36: 30
  05.37: 30
  05.38: 30

UNIQUE_ID_TOTAL: 220
UNIQUE_DEFECT_TOTAL: 218
DUPLICATE_ID: 0
INVALID: 0
VALID: 220
PASS: 217
FAIL: 2
NOT_PROVEN: 1
NOT_TESTED_STATIC: 0
NOT_TESTED_RUNTIME: 220
```

---

## 5. 验证方法声明

全部静态设计推演（STATIC_DEDUCTION），未执行真实 Fork、POSIX 事务、锁竞争或运行时测试（工单禁止）。`NOT_TESTED_RUNTIME` 如实标注。

---

## 证据来源

- 反例数据：COUNTEREXAMPLES.json
- 历史审计：HISTORICAL-COUNTEREXAMPLE-AUDIT.md
- 证明等级：VERIFIED_STATIC
