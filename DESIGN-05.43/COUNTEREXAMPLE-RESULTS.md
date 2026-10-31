# COUNTEREXAMPLE-RESULTS.md — 反例结果（05.43）

工单：USIT-STAGE-B-DESIGN-05.43
工作包：B43-09（修复 B42-R08）
证据等级：STATIC_ANALYSIS（本轮不执行运行时实验）

---

## 1. 本轮反例统计（JSON 实际解析结果）

| 指标 | 值 |
|---|---|
| RAW_ENTRY_TOTAL | 33 |
| UNIQUE_ID_TOTAL | 33 |
| UNIQUE_DEFECT_TOTAL | 33（本轮无跨缺陷归并） |
| PASS | 28 |
| FAIL | 3 |
| NOT_PROVEN | 2 |
| NOT_TESTED_RUNTIME | 0（本轮全部静态推演） |

ID 唯一性：33 个 case_id 全部唯一，无重复。

---

## 2. 类别分布

| 类别 | 数量 |
|---|---|
| WAL_RECOVERY_IDEMPOTENCY | 6 |
| PRECOMMIT_VALIDATION | 4 |
| POST_COMMIT_ERROR | 3 |
| LOCK_OWNERSHIP | 4 |
| RESOURCE_FAILURE | 4 |
| OWNER_AUTHORIZATION | 4 |
| TOKEN_IDENTITY | 3 |
| VERSION_EXHAUSTION | 3 |
| RESOURCE_FENCING | 2 |

---

## 3. FAIL 反例（暴露 05.42 真实缺陷，05.43 修复）

工单要求「存在设计漏洞的反例必须标 FAIL，不得全部预设 PASS」。本轮 3 条 FAIL 反例直接指向 05.42 真实伪代码缺陷：

| case_id | 缺陷（05.42 行为） | 05.43 修复 |
|---|---|---|
| B43-LOCK-001 | admit 持 txn_lock 后又调 run_txn 内部再获取非重入锁 → 自锁死 | 唯一持锁层级 = run_txn，上层不持锁 |
| B43-ID-001 | state_version 既列入身份元组又声明不参与验证 | state_version 明确为审计字段，身份=4 字段 |
| B43-ID-002 | 调用方自报 owner_id 当已认证身份 | 用系统注入的 trusted_caller_owner |

这 3 条 status=FAIL 表示：**该缺陷在 05.42 中真实存在，05.43 已修复**。design_result 字段记录了「05.42 行为(FAIL) → 05.43 修复(PASS)」的完整轨迹。

---

## 4. NOT_PROVEN 反例（维持 DENY）

| case_id | 未证明项 | 动作 |
|---|---|---|
| B43-LOCK-004 | 线程持锁崩溃后锁遗留，跨线程 | DENY |
| B43-OWN-004 | 跨进程可信身份传递不可伪造 | 跨进程授权 DENY |

---

## 5. 验证方式说明

- 本轮全部 `verification_method = STATIC_ANALYSIS`，`requires_runtime_verification = false`。
- 严格区分静态推演与运行时验证：**本轮不执行运行时实验**，静态 PASS 不代表运行时 PASS。
- 反例从 05.42 真实伪代码逐步推演得出，非文字重复结论。

---

## 6. 历史统计（05.43 叠加）

| 口径 | 值 |
|---|---|
| 历史累计（05.33～05.42）RAW | 460 |
| 历史累计唯一 ID | 460 |
| 历史累计唯一缺陷 | 458 |
| 本轮新增 RAW | 33 |
| 本轮新增唯一 ID | 33 |
| 本轮新增唯一缺陷 | 33 |
| **05.43 累计 RAW** | **493** |
| **05.43 累计唯一 ID** | **493** |
| **05.43 累计唯一缺陷** | **491** |

状态累计（含历史 PASS=457 / FAIL=2 / NOT_PROVEN=1）：

| 状态 | 历史 | 本轮 | 累计 |
|---|---|---|---|
| PASS | 457 | 28 | 485 |
| FAIL | 2 | 3 | 5 |
| NOT_PROVEN | 1 | 2 | 3 |
| 合计 | 460 | 33 | 493 |

（历史两处原始 status 标 PASS 但修正为 FAIL 的 B35-MASK-001/002，修正记录保留，不覆盖原始 status，详见 HISTORICAL-COUNTEREXAMPLE-AUDIT.md。）
