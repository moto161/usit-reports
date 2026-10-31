# COUNTEREXAMPLE-RESULTS.md — 反例结果（05.45）

工单：USIT-STAGE-B-DESIGN-05.45
工作包：B45-09（定点反例）
证据等级：STATIC_ANALYSIS（本轮不执行运行时实验）

---

## 1. 本轮反例统计（JSON 实际解析结果）

| 指标 | 值 |
|---|---|
| RAW_ENTRY_TOTAL | 41 |
| UNIQUE_ID_TOTAL | 41 |
| UNIQUE_DEFECT_TOTAL | 41（本轮无跨缺陷归并） |
| baseline_status（05.44 旧缺陷状态） | PASS=0 / FAIL=22 / NOT_PROVEN=2 |
| revision_status（05.45 修订验证状态） | PASS=24 / FAIL=0 / NOT_PROVEN=0 |

ID 唯一性：41 个 case_id 全部唯一，无重复。

**字段说明**：本轮反例 JSON 新增 `baseline_status`（05.44 旧缺陷状态）与 `revision_status`（05.45 修订验证状态）两个独立字段，明确区分「历史缺陷状态」与「当前修订验证状态」，不混用。

---

## 2. 类别分布

| 类别 | 数量 |
|---|---|
| RECOVERY_BARRIER | 8 |
| REQUEST_ID_ATOMIC_IDEMPOTENCY | 8 |
| RESOURCE_OPERATION_CRASH_RECOVERY | 7 |
| TXN_ORDER_AND_WAL_REPLAY | 12 |
| POST_COMMIT_FAILURE_CLOSURE | 6 |

---

## 3. baseline_status 与 revision_status 说明

| 状态字段 | 含义 | 数量 |
|---|---|---|
| `baseline_status` | 05.44 设计中的缺陷状态（FAIL=22 表示该缺陷在 05.44 中真实存在） | 22 FAIL + 2 NOT_PROVEN |
| `revision_status` | 05.45 修订后的验证状态（PASS=24 表示 05.45 已修复该缺陷） | 24 PASS + 0 NOT_PROVEN |

工单要求「反例必须区分 baseline_status（05.44 旧缺陷）与 revision_status（05.45 修订验证结果）」。本轮 41 条反例中：

- 24 条 baseline_status=FAIL → revision_status=PASS（05.45 修复）
- 2 条 baseline_status=NOT_PROVEN → revision_status=NOT_PROVEN（维持 DENY，跨进程/跨重启）
- 0 条 revision_status=FAIL（本轮无新设计缺陷）

---

## 4. NOT_PROVEN 反例（维持 DENY）

| case_id | 未证明项 | 动作 |
|---|---|---|
| B45-BAR-008 | 跨进程恢复屏障 | DENY |
| B45-RES-007 | 跨进程预留/确认 | ISOLATED |
| B45-ORD-012 | 跨进程 WAL 并发写/检查点一致性 | DENY |

---

## 5. 验证方式说明

- 本轮全部 `verification_method = STATIC_ANALYSIS`，`requires_runtime_verification = false`。
- 严格区分静态推演与运行时验证：**本轮不执行运行时实验**，静态 PASS 不代表运行时 PASS。
- 反例从 05.44 真实伪代码逐步推演得出，非文字重复结论。

---

## 6. 历史统计（05.45 叠加）

| 口径 | 值 |
|---|---|
| 历史累计（05.33～05.44）RAW | 519 |
| 历史累计唯一 ID | 519 |
| 历史累计唯一缺陷 | 517 |
| 本轮新增 RAW | 41 |
| 本轮新增唯一 ID | 41 |
| 本轮新增唯一缺陷 | 41 |
| **05.45 累计 RAW** | **560** |
| **05.45 累计唯一 ID** | **560** |
| **05.45 累计唯一缺陷** | **558** |

状态累计（revision_status 口径）：

| 状态 | 历史（05.33～05.44） | 本轮（05.45） | 累计（05.45） |
|---|---|---|---|
| PASS | 500 | 24 | 524 |
| FAIL | 14 | 0 | 14 |
| NOT_PROVEN | 5 | 2 | 7 |
| 合计 | 519 | 26 | 545 |

（历史两处原始 status 标 PASS 但修正为 FAIL 的 B35-MASK-001/002，修正记录保留，不覆盖原始 status。）
