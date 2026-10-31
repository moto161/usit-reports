# COUNTEREXAMPLE-RESULTS.md — 反例结果（05.44）

工单：USIT-STAGE-B-DESIGN-05.44
工作包：B44-09（定点反例）
证据等级：STATIC_ANALYSIS（本轮不执行运行时实验）

---

## 1. 本轮反例统计（JSON 实际解析结果）

| 指标 | 值 |
|---|---|
| RAW_ENTRY_TOTAL | 26 |
| UNIQUE_ID_TOTAL | 26 |
| UNIQUE_DEFECT_TOTAL | 26（本轮无跨缺陷归并） |
| PASS | 15 |
| FAIL | 9 |
| NOT_PROVEN | 2 |
| NOT_TESTED_RUNTIME | 0（本轮全部静态推演） |

ID 唯一性：26 个 case_id 全部唯一，无重复。

---

## 2. 类别分布

| 类别 | 数量 |
|---|---|
| TXN_ORDER_AND_WAL_REPLAY | 4 |
| REQUEST_ID_ATOMIC_IDEMPOTENCY | 6 |
| RECOVERY_BARRIER_AND_ADMISSION | 4 |
| RESOURCE_RELEASE_SERIALIZATION | 5 |
| POST_COMMIT_FAILURE_CLOSURE | 6 |
| COUNTEREXAMPLE_REDESIGN | 1 |

---

## 3. FAIL 反例（暴露 05.43 真实缺陷，05.44 修复）

本轮 9 条 FAIL 反例直接指向 05.43 真实伪代码缺陷：

| case_id | 缺陷（05.43 行为） | 05.44 修复 |
|---|---|---|
| B44-ORD-001 | 恢复按 txn_id 升序，顺序反转 | commit_seq 单调 + 快照重放 |
| B44-ORD-002 | COMMIT 快照覆盖缺陷 | commit_seq 单调 + 快照重放 |
| B44-ORD-003 | first_commit 静默选取 | ISOLATED 隔离 |
| B44-ID-001 | req_id 锁外查询，并发重复提交 | 锁内查询 + 原子幂等 |
| B44-BAR-001 | in_recovery TOCTOU，并发恢复与 WAL 修改 | barrier_lock + 计数器 |
| B44-RES-001 | 幂等查询并发重复 operation | 锁内查询 + 资源锁串行化 |
| B44-PC-002 | mark_committed_pending_apply 无第二层处理 | 三层失败处理 |
| B44-PC-003 | wal_append(APPLY) 抛异常向上抛 | COMMITTED_PENDING_APPLY |
| B44-PC-005 | persist_isolation_record 持久化失败未定义 | 三层失败处理 |

这 9 条 status=FAIL 表示：**该缺陷在 05.43 中真实存在，05.44 已修复**。design_result 字段记录了「05.43 行为(FAIL) → 05.44 修复(PASS)」的完整轨迹。

---

## 4. NOT_PROVEN 反例（维持 DENY）

| case_id | 未证明项 | 动作 |
|---|---|---|
| B44-BAR-004 | 跨进程恢复屏障 | DENY |
| B44-RES-003 | 跨进程资源锁一致性 | ISOLATED |

---

## 5. 验证方式说明

- 本轮全部 `verification_method = STATIC_ANALYSIS`，`requires_runtime_verification = false`。
- 严格区分静态推演与运行时验证：**本轮不执行运行时实验**，静态 PASS 不代表运行时 PASS。
- 反例从 05.43 真实伪代码逐步推演得出，非文字重复结论。

---

## 6. 历史统计（05.44 叠加）

| 口径 | 值 |
|---|---|
| 历史累计（05.33～05.43）RAW | 493 |
| 历史累计唯一 ID | 493 |
| 历史累计唯一缺陷 | 491 |
| 本轮新增 RAW | 26 |
| 本轮新增唯一 ID | 26 |
| 本轮新增唯一缺陷 | 26 |
| **05.44 累计 RAW** | **519** |
| **05.44 累计唯一 ID** | **519** |
| **05.44 累计唯一缺陷** | **517** |

状态累计：

| 状态 | 历史 | 本轮 | 累计 |
|---|---|---|---|
| PASS | 485 | 15 | 500 |
| FAIL | 5 | 9 | 14 |
| NOT_PROVEN | 3 | 2 | 5 |
| 合计 | 493 | 26 | 519 |

（历史两处原始 status 标 PASS 但修正为 FAIL 的 B35-MASK-001/002，修正记录保留，不覆盖原始 status。）
