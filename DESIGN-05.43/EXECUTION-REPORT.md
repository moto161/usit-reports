# EXECUTION-REPORT.md — 执行报告（05.43）

## 1. 基本信息

| 项 | 值 |
|---|---|
| 工单编号 | USIT-STAGE-B-DESIGN-05.43 |
| 执行时间 | 2026-10-31 |
| 工作区 | `usit_p0_05.43_reports/`（项目外交付目录） |
| 基线 commit | `43e2e65`（05.42 文档 commit） |
| 交付 commit | 见 HASH-MANIFEST.txt 及 GitHub 提交记录 |
| 执行模式 | READ-ONLY ANALYSIS + DESIGN REVISION |

---

## 2. B43-01～B43-09 逐项结果

| 编号 | 结论 | 证据文件/章节 |
|---|---|---|
| B43-01 WAL 恢复严格幂等 | PASS | WAL-RECOVERY-IDEMPOTENCY.md §3/§5 |
| B43-02 提交前不变式验证 | PASS | TRANSACTION-PRECOMMIT-VALIDATION.md §2 |
| B43-03 COMMIT 后异常与重试 | PASS | POST-COMMIT-ERROR-AND-RETRY.md §3/§4 |
| B43-04 txn_lock 唯一所有权 | PASS | LOCK-OWNERSHIP-AND-CALL-GRAPH.md §2/§5 |
| B43-05 资源端释放异常与恢复 | PASS | RESOURCE-FAILURE-AND-RECOVERY.md §3/§4 |
| B43-06 资源端 Owner 授权 | PASS | RESOURCE-OWNER-AUTHORIZATION.md §3 |
| B43-07 Token 身份字段统一 | PASS | TOKEN-IDENTITY-CANONICAL-MODEL.md §2 |
| B43-08 跨文档编号一致性 | PASS | CROSS-DOC-CONSISTENCY.md §1 |
| B43-09 反例重新设计 | PASS | COUNTEREXAMPLES.json（33 条含 FAIL） |

---

## 3. WAL 恢复伪代码逐行审查

见 WAL-RECOVERY-IDEMPOTENCY.md §3。关键结论：

- `group_by_txn_id` 后按 txn_id 升序重放，每个 txn_id 只处理一次。
- 只有 `COMMIT` 记录触发 apply，`APPLY` 记录不触发。
- `applied_txn_ids` 集合去重，apply 后 add，入口再查，双重保险。
- COMMIT+APPLY 双记录重放反例（B43-WAL-001）证明 COUNT/generation/VERSION 只增加一次。

---

## 4. 提交点、候选状态验证、APPLY 发布顺序

见 TRANSACTION-PRECOMMIT-VALIDATION.md §2。唯一时序：

```
读旧状态 → 构造候选 → COMMIT 前验证不变式 → PREPARE fsync → COMMIT fsync → APPLY 发布
```

COMMIT fsync 是唯一提交点，之后不回滚。

---

## 5. 每个崩溃窗口和恢复动作

见 CRASH-AND-CONCURRENCY-MATRIX.md。事务提交、恢复幂等、锁并发、资源释放、版本耗尽五类矩阵全覆盖。

---

## 6. 锁调用图、锁所有权、出口审计

见 LOCK-OWNERSHIP-AND-CALL-GRAPH.md。唯一持锁层级 = run_txn，try/finally 保证所有出口释放，逐出口审计表见 §4。

---

## 7. 资源端 Owner 认证、Fencing、幂等、异常恢复

见 RESOURCE-OWNER-AUTHORIZATION.md + RESOURCE-FAILURE-AND-RECOVERY.md。三重校验 + Fencing 串行化 + req_id 幂等 + isolate_release 隔离。

---

## 8. 反例实际解析统计

| 指标 | 值 |
|---|---|
| 本轮 RAW | 33 |
| 唯一 ID | 33 |
| PASS | 28 |
| FAIL | 3 |
| NOT_PROVEN | 2 |
| NOT_TESTED_RUNTIME | 0 |

JSON 解析通过，ID 唯一，分类统计与 COUNTEREXAMPLE-RESULTS.md 一致。

---

## 9. 历史累计统计

| 口径 | 累计 |
|---|---|
| RAW | 493 |
| 唯一 ID | 493 |
| 唯一缺陷 | 491 |
| PASS | 485 |
| FAIL | 5 |
| NOT_PROVEN | 3 |

详见 HISTORICAL-COUNTEREXAMPLE-AUDIT.md。

---

## 10. 源码修改清单

**零修改**。未修改任何 `usit/*.py` 源码、配置、数据库及历史交付。

---

## 11. 运行时测试 / Fork / 构建 / 部署

**全部未执行**。本轮仅静态推演 + 文档修订 + 反例生成 + SHA256 计算。

---

## 12. SHA256 与字节数核验

见 HASH-MANIFEST.txt。全部交付文件 SHA256 与字节数以 Git blob（LF 口径）计算。

---

## 13. 未解决问题及影响

见 ISSUES.md。六项跨进程/跨重启能力 NOT_PROVEN，维持 DENY/ISOLATED。

---

## 14. 最终结论

**PASS-WITH-ISSUES**。

- 九项 B42 缺陷全部闭合，单进程安全不变式闭合。
- 跨进程/跨重启能力维持 NOT_PROVEN → DENY/ISOLATED。

## 15. 审批状态

| 项 | 状态 |
|---|---|
| IMPLEMENT-01 | 不批准（等待独立复审） |
| A/B Merge | 不批准 |
| Source Retire | 不批准 |
