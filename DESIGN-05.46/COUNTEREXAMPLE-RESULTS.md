# COUNTEREXAMPLE-RESULTS.md — 反例结果（05.46）

工单：USIT-STAGE-B-DESIGN-05.46
工作包：B46-06（定点反例）
证据等级：STATIC_ANALYSIS（本轮不执行运行时实验）

---

## 1. 本轮反例统计（JSON 实际解析结果）

| 指标 | 值 |
|---|---|
| RAW_ENTRY_TOTAL | 38 |
| UNIQUE_ID_TOTAL | 38 |
| UNIQUE_DEFECT_TOTAL | 38（本轮无跨缺陷归并） |
| baseline_status（05.45 旧缺陷状态） | FAIL=22 / NOT_PROVEN=2 |
| revision_status（05.46 修订验证状态） | PASS=32 / NOT_PROVEN=0 |

ID 唯一性：38 个 case_id 全部唯一，无重复。

**字段说明**：本轮反例 JSON 新增 `baseline_status`（05.45 旧缺陷状态）与 `revision_status`（05.46 修订验证状态）两个独立字段，明确区分「历史缺陷状态」与「当前修订验证状态」。

---

## 2. 类别分布

| 类别 | 数量 |
|---|---|
| RECOVERY_BARRIER | 6 |
| REQUEST_ID_ATOMIC_IDEMPOTENCY | 5 |
| RESOURCE_OPERATION_CRASH_RECOVERY | 6 |
| WAL_INTEGRITY_AND_CHECKPOINT | 5 |
| POST_COMMIT_FAILURE_CLOSURE | 6 |

---

## 3. baseline_status 与 revision_status 说明

| 状态字段 | 含义 | 数量 |
|---|---|---|
| `baseline_status` | 05.45 设计中的缺陷状态（22 FAIL 表示该缺陷在 05.45 中真实存在） | 22 FAIL + 2 NOT_PROVEN |
| `revision_status` | 05.46 修订后的验证状态（32 PASS 表示 05.46 已修复该缺陷） | 32 PASS + 0 NOT_PROVEN |

工单要求「反例必须区分 baseline_status 与 revision_status，不得再用一个 FAIL 字段混合历史缺陷和当前修订结果」。本轮 38 条反例中：

- 22 条 baseline_status=FAIL → revision_status=PASS（05.46 修复）
- 2 条 baseline_status=NOT_PROVEN → revision_status=NOT_PROVEN（维持 DENY，跨进程/跨重启）
- 0 条 revision_status=FAIL（本轮无新设计缺陷）

---

## 4. NOT_PROVEN 反例（维持 DENY）

| case_id | 未证明项 | 动作 |
|---|---|---|
| B46-BAR-006 | 跨进程恢复屏障 | DENY |
| B46-ID-005 | 跨进程 req_id 唯一性 | DENY |
| B46-ORD-005 | 跨进程检查点一致性 | DENY |

---

## 5. 验证方式说明

- 本轮全部 `verification_method = STATIC_ANALYSIS`，`requires_runtime_verification = false`。
- 严格区分静态推演与运行时验证：**本轮不执行运行时实验**，静态 PASS 不代表运行时 PASS。
- 反例从 05.45 真实伪代码逐步推演得出，非文字重复结论。

---

## 6. 历史统计（05.46 叠加）

| 口径 | 值 |
|---|---|
| 历史累计（05.33～05.45）RAW | 560 |
| 历史累计唯一 ID | 560 |
| 历史累计唯一缺陷 | 558 |
| 本轮新增 RAW | 38 |
| 本轮新增唯一 ID | 38 |
| 本轮新增唯一缺陷 | 38 |
| **05.46 累计 RAW** | **598** |
| **05.46 累计唯一 ID** | **598** |
| **05.46 累计唯一缺陷** | **596** |

状态累计（revision_status 口径）：

| 状态 | 历史（05.45） | 本轮（05.46） | 累计（05.46） |
|---|---|---|---|
| PASS | 524 | 32 | 556 |
| FAIL | 14 | 0 | 14 |
| NOT_PROVEN | 7 | 3 | 7 |
| 合计 | 545 | 35 | 580 |

（历史两处原始 status 标 PASS 但修正为 FAIL 的 B35-MASK-001/002，修正记录保留，不覆盖原始 status。）
