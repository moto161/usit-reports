# COUNTEREXAMPLE-RESULTS.md — 反例结果（05.42 权威）

证据等级：VERIFIED_STATIC（由 JSON 逐条机器核验）
目标：给出 05.42 新增反例的逐条统计与静态结论。

---

## 1. 本轮新增反例（05.42）

| 类别 | 数量 | 覆盖缺陷 |
|---|---|---|
| TRANSACTION_WAL | 15 | B42-01（WAL 提交、崩溃窗口、恢复重放） |
| COUNT_INVARIANT | 10 | B42-02（revoke 减 COUNT、不变式） |
| LOCK_LIFECYCLE | 10 | B42-03（锁生命周期、try/finally） |
| TOKEN_IDENTITY | 5 | B42-04（owner/resource 校验） |
| RESOURCE_IDEMPOTENCY | 10 | B42-05（幂等释放、consumed_records） |
| **合计** | **50** | B42-01～B42-05 |

---

## 2. 逐条状态分布（05.42 新增）

| 指标 | 值 |
|---|---|
| RAW_ENTRY_TOTAL | 50 |
| UNIQUE_ID_TOTAL | 50 |
| UNIQUE_DEFECT_TOTAL | 50 |
| DUPLICATE_ID | 0 |
| INVALID | 0 |
| VALID | 50 |
| PASS | 50 |
| FAIL | 0 |
| NOT_PROVEN | 0 |
| NOT_TESTED_STATIC | 0 |
| NOT_TESTED_RUNTIME | 50 |

---

## 3. 负向竞态覆盖（非理想路径）

本轮反例重点覆盖复审点名的负向场景：

- 事务日志各持久化步骤的崩溃窗口（B42-WAL-001～015）。
- 正常历史 generation 不得清零（B42-WAL-006/007）。
- revoke 破坏不变式（B42-INV-003）、锁泄漏（B42-LOCK-004）、身份校验不全（B42-TOK-005）、幂等误判（B42-RES-002）——这些是**针对 05.41 缺陷的反例**，design_result 明确标注「05.41 缺陷，05.42 修复」。

---

## 4. 静态 vs 运行时

- 静态推演 PASS = 设计文档层面协议闭合。
- 运行时验证未执行，NOT_TESTED_RUNTIME=50。
- 不得将静态 PASS 等同运行时 PASS。

---

## 证据来源

- 反例 JSON：COUNTEREXAMPLES.json（50 条，ID 唯一）
- 统计复算：EXECUTION-REPORT.md §G
- 证明等级：VERIFIED_STATIC + NOT_TESTED_RUNTIME
