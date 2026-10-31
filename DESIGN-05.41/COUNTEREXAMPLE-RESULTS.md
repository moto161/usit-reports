# COUNTEREXAMPLE-RESULTS.md — 反例结果（05.41 权威）

证据等级：VERIFIED_STATIC（由 JSON 逐条机器核验）
目标：给出 05.41 新增反例的逐条统计与静态结论。

---

## 1. 本轮新增反例（05.41）

| 类别 | 数量 | 覆盖缺陷 |
|---|---|---|
| TOKEN_MULTI_ACTIVE | 15 | B41-01（多 Token 并存、独立消费） |
| TOKEN_SLOT_GENERATION | 15 | B41-01（每槽位独立代际、复用、耗尽） |
| COUNT_STATE_INVARIANT | 10 | B41-02（COUNT 与位图不变式、revoke 规则） |
| VERSION_MONOTONICITY | 10 | B41-03（版本单调、回退/跳跃/回绕） |
| RESOURCE_FENCING | 10 | B41-04（统一串行化、旧 Owner 拒绝） |
| **合计** | **60** | B41-01～B41-04 |

---

## 2. 逐条状态分布（05.41 新增）

| 指标 | 值 |
|---|---|
| RAW_ENTRY_TOTAL | 60 |
| UNIQUE_ID_TOTAL | 60 |
| UNIQUE_DEFECT_TOTAL | 60（本轮无跨缺陷归并） |
| DUPLICATE_ID | 0 |
| INVALID | 0 |
| VALID | 60 |
| PASS | 60 |
| FAIL | 0 |
| NOT_PROVEN | 0 |
| NOT_TESTED_STATIC | 0 |
| NOT_TESTED_RUNTIME | 60 |

> 说明：全部 60 条为静态推演，`requires_runtime_verification=False` 标记，未执行运行时测试，`NOT_TESTED_RUNTIME=60`。

---

## 3. 指定反例 A–J 覆盖核验

| 指定 | case_id | 覆盖 |
|---|---|---|
| A. Token0 分配后 Token1 分配，Token0 仍有效 | B41-TMA-002 | ✓ |
| B. 16 Token 同时活动，每个独立消费 | B41-TMA-003 | ✓ |
| C. 消费 Token0 不影响 Token1 | B41-TMA-004 | ✓ |
| D. 撤销 Token0 后 COUNT 与位图一致 | B41-CSI-004/005 | ✓ |
| E. 同一 Token 并发消费只成功一次 | B41-TMA-011（复用）+ TOKEN-ATOMIC-TRANSACTION.md §4 | ✓ |
| F. 两个 Owner 并发接管释放，旧 Owner 不越权 | B41-FEN-002 | ✓ |
| G. 释放成功响应丢失，重试 ALREADY_RELEASED | B41-FEN-004 | ✓ |
| H. 版本回退/跳跃必须拒绝 | B41-VER-003/004 | ✓ |
| I. generation 耗尽所有分配入口拒绝 | B41-TSG-003 | ✓ |
| J. 状态字非法组合不得 CAS 提交 | B41-CSI-009（UNDERFLOW）/ COUNT 校验 | ✓ |

---

## 4. 静态结论 vs 运行时结论

- 静态推演 PASS = 设计文档层面协议闭合。
- 运行时验证未执行，NOT_TESTED_RUNTIME=60。
- 不得将静态 PASS 等同运行时 PASS。

---

## 证据来源

- 反例 JSON：COUNTEREXAMPLES.json（60 条，Python 生成并校验 ID 唯一）
- 统计复算：EXECUTION-REPORT.md §G
- 证明等级：VERIFIED_STATIC + NOT_TESTED_RUNTIME
