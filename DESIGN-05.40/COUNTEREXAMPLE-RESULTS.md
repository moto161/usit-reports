# COUNTEREXAMPLE-RESULTS.md — 反例结果（05.40 权威）

证据等级：VERIFIED_STATIC（由 JSON 逐条机器核验）
目标：修复 B39-R04——给出 05.40 新增反例的逐条统计与静态结论。

---

## 1. 本轮新增反例（05.40）

| 类别 | 数量 | 覆盖缺陷 |
|---|---|---|
| TOKEN_IDENTITY | 20 | B40-01（Token 身份/槽位复用/旧 Token 复活） |
| ATOMIC_WIDTH | 20 | B40-02（统一 128 位访问/字段隔离/非法状态） |
| VERSION_EXHAUST | 15 | B40-03（全入口版本耗尽/回绕/复用） |
| FENCING | 20 | B40-04（资源端原子释放/接管/幂等） |
| **合计** | **75** | B40-01～B40-04 |

---

## 2. 逐条状态分布（05.40 新增）

| 指标 | 值 |
|---|---|
| RAW_ENTRY_TOTAL | 75 |
| UNIQUE_ID_TOTAL | 75 |
| UNIQUE_DEFECT_TOTAL | 75（本轮无跨缺陷归并） |
| DUPLICATE_ID | 0 |
| INVALID | 0 |
| VALID | 75 |
| PASS | 75 |
| FAIL | 0 |
| NOT_PROVEN | 0 |
| NOT_TESTED_STATIC | 0 |
| NOT_TESTED_RUNTIME | 75 |

> 说明：全部 75 条反例均为**静态推演**（STATIC verification_method），`requires_runtime_verification` 字段标记是否需要运行时验证。当前未执行任何运行时测试，故 `NOT_TESTED_RUNTIME = 75`。

---

## 3. 反例质量核验（含负向竞态）

本轮反例覆盖的负向场景（非理想路径）：

- TOKEN_IDENTITY：槽位复用、旧 Token 复活、并发分配同槽位、重复消费、崩溃窗口、generation 耗尽、账本损坏、跨重启 generation 回退。
- ATOMIC_WIDTH：64/128 位重叠修改、未对齐、非法保留位、无条件 commit、整字算术、重试耗尽、LIVELOCK、跨进程/跨重启 NOT_PROVEN。
- VERSION_EXHAUST：全入口耗尽、回绕、快照回滚、generation 复用、epoch 外检查、跨重启单调性。
- FENCING：两步 TOCTOU、接管并发、旧 Owner 复活、重复释放、响应丢失、部分释放、资源端重启、flock/fcntl 误判、心跳超时误用。

---

## 4. 静态结论 vs 运行时结论

- 静态推演（STATIC）结论 = PASS，表示**设计文档层面**的协议闭合。
- 运行时验证**未执行**，`NOT_TESTED_RUNTIME = 75`。
- 不得将静态 PASS 等同于运行时 PASS（继承历轮约束）。

---

## 5. 与历史反例的关系

历史反例（05.33–05.38）共 220 条，05.39 新增 55 条，05.40 新增 75 条。累计统计见 HISTORICAL-COUNTEREXAMPLE-AUDIT.md。

---

## 证据来源

- 反例 JSON：COUNTEREXAMPLES.json（75 条，Python 逐条生成并校验 ID 唯一）
- 统计复算：见 EXECUTION-REPORT.md §D
- 证明等级：VERIFIED_STATIC + NOT_TESTED_RUNTIME
