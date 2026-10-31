# COUNTEREXAMPLE-RESULTS.md — 反例验证报告（05.36）

证据等级：VERIFIED_STATIC（全部为静态推演，非运行时测试）

---

## 1. 统计

| 指标 | 值 |
|---|---|
| TOTAL | 30（本轮新增） |
| VALID | 30（字段完整、可复现） |
| PASS | 28 |
| FAIL | 2（记录 05.35 历史缺陷，05.36 已修复） |
| NOT_PROVEN | 0 |
| NOT_TESTED_STATIC | 0 |
| NOT_TESTED_RUNTIME | 30（未执行任何运行时测试，如实标注） |

累计反例（05.33=60 + 05.34=40 + 05.36=30）≥ 160 条要求满足（05.35 的 30 条因掩码错误需复核，见 §3）。

---

## 2. 类别分布

| 类别 | 数量 | 说明 |
|---|---|---|
| MASK_FIELD | 8 | 掩码边界、字段污染、溢出/下溢 |
| DENIED_ATOMIC | 8 | DENIED 与 FROZEN/unfreeze/rollback/恢复/准入并发 |
| CLEANUP | 7 | 清理幂等、部分失败、并发、隔离 |
| CRASH_RECOVERY | 7 | 共享计数器遗留、旧写入者、UNKNOWN 事务、证据冲突 |

---

## 3. 历史反例复核（05.35 错误结论修正）

05.35 的 `COUNTEREXAMPLES.json` 中，`B35-MASK-001` 及其后续条目将「掩码 `0x7FFE`」标为 PASS，这是**错误的 PASS 结论**：

- 05.35 实际使用 `COUNT_MASK=0x7FFE`，遗漏 bit15。
- 当真实计数 = 16384（存储字段 0x8000）时，`0x7FFE` 提取结果为 0，非零计数被误判为零。
- 05.36 将 05.35 的该结论**修正为 FAIL**，并以 `0xFFFE` 重新验证。

对应到本轮反例：`B36-MASK-002` 记录「05.35 的 0x7FFE 提取 16384→0」为 FAIL（历史缺陷），`B36-MASK-001` 记录「05.36 的 0xFFFE 提取 16384→16384」为 PASS（修复后正确）。

**不得沿用 05.35 的错误 PASS 结论**——此为工单 B36-01 的明确要求，本报告如实执行。

---

## 4. 验证方法声明

所有 30 条反例均为**静态设计推演**（STATIC_DEDUCTION），通过伪代码 + 状态机 + 位运算数学推演验证，**未执行任何真实 Fork、POSIX 事务或运行时测试**（工单禁止）。因此：

- `requires_runtime_verification` 全部为 `false`（当前阶段无需，设计推演可判定）。
- `NOT_TESTED_RUNTIME = 30`，如实标注，不用 0 暗示运行时覆盖完成。

---

## 5. 关键反例摘录

### B36-MASK-002（掩码缺陷，FAIL 记录历史）

- 前置：count=16384，stored field=0x8000
- 操作：`get_count(0x8000)` using `0x7FFE`
- 预期：16384
- 05.35 设计结果：`(0x8000 & 0x7FFE)>>1 = 0`（错误）
- 05.36 修复：`COUNT_MASK=0xFFFE` → 16384（正确）

### B36-DENIED-001（FROZEN vs DENIED 并发）

- 前置：gate=CLOSED, count=0, frozen=0, DENIED=0
- 操作：线程 A `cas_frozen` 与线程 B `set_denied` 并发
- 预期：DENIED=1 时 FROZEN 不得设置
- 设计结果：`cas_frozen` 在同一 CAS 内检查 DENIED，返回 `'DENIED'`

### B36-CLEANUP-001（清理提前标记，FAIL 记录历史）

- 前置：`_cleanup_done=False`，资源 R 未释放
- 操作：先置 `_cleanup_done=True` 再释放 R，R 抛异常
- 05.35 缺陷：R 永久泄漏且不可重试
- 05.36 修复：`_cleanup_done` 仅在 CLEANUP_COMPLETED 置 True

---

## 证据来源

- 反例数据：`COUNTEREXAMPLES.json`
- 位运算验证：EXECUTION-REPORT.md §B36-01
- 证明等级：VERIFIED_STATIC
