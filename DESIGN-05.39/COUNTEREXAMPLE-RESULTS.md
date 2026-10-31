# COUNTEREXAMPLE-RESULTS.md — 反例验证报告（05.39）

证据等级：VERIFIED_STATIC（全部静态推演，非运行时测试）

---

## 1. 本轮统计

| 指标 | 值 |
|---|---|
| TOTAL | 55（本轮新增） |
| VALID | 55 |
| PASS | 55 |
| FAIL | 0 |
| NOT_PROVEN | 0 |
| NOT_TESTED_STATIC | 0 |
| NOT_TESTED_RUNTIME | 55 |

## 2. 类别分布

| 类别 | 数量 | 对应问题 |
|---|---|---|
| TOKEN_ATOMIC | 20 | B39-01 Token 原子事务 |
| FENCING | 20 | B39-02 资源端 Fencing |
| VERSION_EXHAUST | 15 | B39-03 版本耗尽 |

## 3. 负向场景覆盖

- TOKEN_ATOMIC：两线程并发消费同一 token、admit 崩溃窗口、token 回绕、账本损坏、旧 token 复活。
- FENCING：旧 owner 复活、旧 token 迟到、资源端重启、token 回滚、并发接管、两步 TOCTOU。
- VERSION_EXHAUST：版本耗尽、并发递增、快照回滚、epoch 丢失、ABA、静默回绕。

## 4. 累计统计（权威口径，见 HISTORICAL-COUNTEREXAMPLE-AUDIT.md §6）

```
RAW_ENTRY_TOTAL: 275
UNIQUE_ID_TOTAL: 275
UNIQUE_DEFECT_TOTAL: 273
DUPLICATE_ID: 0
INVALID: 0
VALID: 275
PASS: 272
FAIL: 2
NOT_PROVEN: 1
NOT_TESTED_STATIC: 0
NOT_TESTED_RUNTIME: 275
```

## 5. 验证方法声明

全部静态设计推演（STATIC_DEDUCTION），未执行真实 Fork、锁竞争、POSIX 事务或运行时测试（工单禁止）。`NOT_TESTED_RUNTIME` 如实标注。

---

## 证据来源

- 反例数据：COUNTEREXAMPLES.json
- 历史审计：HISTORICAL-COUNTEREXAMPLE-AUDIT.md
- 证明等级：VERIFIED_STATIC
