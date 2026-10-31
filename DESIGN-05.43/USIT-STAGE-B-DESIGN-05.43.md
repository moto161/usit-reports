# USIT-STAGE-B-DESIGN-05.43.md — 主文档

**工单编号**：USIT-STAGE-B-DESIGN-05.43
**前置版本**：DESIGN-05.42
**执行模式**：READ-ONLY ANALYSIS + DESIGN REVISION
**目标**：修复 B42-R01～R09，建立唯一、可执行、无歧义的事务提交、恢复和资源端释放协议。

---

## 一、执行边界

1. 只读取历史设计、分析与修订设计文档，修改项目外交付目录中的设计文档。
2. 未修改任何 `usit/*.py` 源码。
3. 未执行真实 Fork、构建、部署、运行时测试、真实资源释放、第三方请求。
4. 未执行 IMPLEMENT-01、A/B Merge、Source Retire。
5. 未将静态推演写成运行时验证。
6. 先修协议，再做反例。

---

## 二、九项修复结果总览

| 编号 | 修复项 | 结论 | 证据文档 |
|---|---|---|---|
| B43-01 | WAL 恢复严格幂等 | PASS | WAL-RECOVERY-IDEMPOTENCY.md |
| B43-02 | 提交前不变式验证 | PASS | TRANSACTION-PRECOMMIT-VALIDATION.md |
| B43-03 | COMMIT 后异常与重试 | PASS | POST-COMMIT-ERROR-AND-RETRY.md |
| B43-04 | txn_lock 唯一所有权 | PASS | LOCK-OWNERSHIP-AND-CALL-GRAPH.md |
| B43-05 | 资源端释放异常与恢复 | PASS | RESOURCE-FAILURE-AND-RECOVERY.md |
| B43-06 | 资源端 Owner 授权 | PASS | RESOURCE-OWNER-AUTHORIZATION.md |
| B43-07 | Token 身份字段统一 | PASS | TOKEN-IDENTITY-CANONICAL-MODEL.md |
| B43-08 | 跨文档编号一致性 | PASS | CROSS-DOC-CONSISTENCY.md |
| B43-09 | 反例重新设计 | PASS | COUNTEREXAMPLES.json |

---

## 三、核心设计结论

### 1. WAL 恢复严格幂等（B43-01）

- COMMIT 是提交记录，APPLY 是应用进度标记，**不是第二笔状态变更**。
- 恢复按 txn_id 分组，每个 txn_id 只 apply 一次，用 `applied_txn_ids` 集合去重。
- 只有 `COMMIT` 记录触发 apply，`APPLY` 记录被忽略。

### 2. 提交前不变式验证（B43-02）

- 唯一时序：读旧状态 → 构造候选 → **COMMIT 前验证全部不变式** → PREPARE fsync → COMMIT fsync → APPLY。
- COMMIT 后禁止回滚；APPLY 只发布已验证候选状态。

### 3. COMMIT 后异常（B43-03）

- 四态：ABORT / UNKNOWN / COMMITTED_PENDING_APPLY / OK。
- COMMIT 后异常返回 COMMITTED_PENDING_APPLY，不返回 ERROR 诱导重试。
- `query_txn` 查询接口 + `req_id → txn_id` 幂等键。

### 4. 锁唯一所有权（B43-04）

- 唯一持锁层级 = `run_txn`，上层 admit/release 不持锁，消除自锁。
- try/finally 语义保证所有出口释放锁。

### 5. 资源端释放异常（B43-05）

- operation 与 consumed_records 的真实提交边界明确；operation 不可回滚。
- 异常落到 `isolate_release` 伪代码，持久化隔离标记，不自动重放。

### 6. 资源端 Owner 授权（B43-06）

- 三重校验：slot 活动 + generation 匹配 + owner 匹配 + resource 匹配。
- owner 来自可信上下文注入，非调用方自报。

### 7. Token 身份统一（B43-07）

- 身份 = `(slot, generation, owner_id, resource_id)`，state_version 为审计字段。

---

## 四、未证明能力（继续 DENY/ISOLATED）

跨进程 WAL 并发写、跨重启恢复、跨进程可信身份传递、operation 非幂等自动重放、线程持锁崩溃恢复、跨进程 Fencing —— 全部 NOT_PROVEN，维持 DENY/ISOLATED（详见 ISSUES.md）。

---

## 五、反例统计

本轮 33 条（PASS=28 / FAIL=3 / NOT_PROVEN=2），ID 唯一，含 3 条 FAIL 暴露 05.42 真实缺陷。历史累计 RAW=493、唯一 ID=493、唯一缺陷=491。

---

## 六、最终结论

**PASS-WITH-ISSUES**。九项 B42 缺陷全部闭合，单进程安全不变式已闭合；跨进程/跨重启能力维持 NOT_PROVEN → DENY/ISOLATED。

不进入 IMPLEMENT-01，不 A/B Merge，不 Source Retire。等待 ChatGPT 独立复审。
