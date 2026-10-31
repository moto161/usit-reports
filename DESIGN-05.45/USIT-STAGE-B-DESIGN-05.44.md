# USIT-STAGE-B-DESIGN-05.44.md — 主文档

**工单编号**：USIT-STAGE-B-DESIGN-05.44
**前置版本**：DESIGN-05.43
**执行模式**：READ-ONLY ANALYSIS + DESIGN REVISION
**目标**：修复 B43-R01～R10，闭合单进程事务安全协议。

---

## 一、执行边界

1. 只读取历史设计、分析与修订设计文档，修改项目外交付目录中的设计文档。
2. 未修改任何 `usit/*.py` 源码。
3. 未执行真实 Fork、构建、部署、运行时测试、真实资源操作、第三方请求。
4. 未执行 IMPLEMENT-01、A/B Merge、Source Retire。
5. 未将静态推演写成运行时验证。
6. 先修协议，再做反例。

---

## 二、五项核心修复结果总览

| 编号 | 修复项 | 结论 | 证据文档 |
|---|---|---|---|
| B44-01 | 事务 ID 与提交顺序 | PASS | TXN-ORDER-AND-WAL-REPLAY.md |
| B44-02 | req_id 原子幂等 | PASS | REQUEST-ID-ATOMIC-IDEMPOTENCY.md |
| B44-03 | 恢复屏障与事务准入 | PASS | RECOVERY-BARRIER-AND-ADMISSION.md |
| B44-04 | 资源端释放与接管串行化 | PASS | RESOURCE-RELEASE-SERIALIZATION.md |
| B44-05 | COMMIT 后异常、WAL 完整性、状态查询 | PASS | POST-COMMIT-FAILURE-CLOSURE.md + WAL-INTEGRITY-AND-CHECKPOINT.md |

---

## 三、核心设计结论

### 1. 事务顺序与快照（B44-01）

- 引入 `commit_seq`：全局单调提交顺序权威，锁内分配。
- COMMIT 记录保存完整候选状态快照。
- 恢复按 commit_seq 升序直接重放快照，不依赖 txn_id。

### 2. req_id 原子幂等（B44-02）

- req_id 唯一性检查在事务串行化边界内完成。
- 同一 req_id 并发请求，只有一个事务提交，其余返回 ALREADY_PROCESSED。

### 3. 恢复屏障与事务准入（B44-03）

- 引入 barrier_lock + 计数器模型，替代布尔 TOCTOU。
- 恢复排空在途事务，屏障建立后无新事务进入。

### 4. 资源端串行化（B44-04）

- 幂等查询、授权校验、Fencing、req_id 占用、operation、WAL 提交在同一资源锁内完成。
- 接管与释放串行化，旧 Owner 无法继续执行。

### 5. COMMIT 后异常与 WAL 完整性（B44-05）

- publish_atomic 原子发布，三层失败处理。
- query_txn 校验 COMMIT/APPLY 一致性。
- WAL 截断/压缩采用 CHECKPOINT 机制。

---

## 四、未证明能力（继续 DENY/ISOLATED）

跨进程 WAL 并发写、跨重启恢复、跨进程可信身份传递、operation 非幂等自动重放、线程持锁崩溃恢复、跨进程资源锁一致性、跨进程恢复屏障、跨进程 Fencing —— 全部 NOT_PROVEN，维持 DENY/ISOLATED（详见 ISSUES.md）。

---

## 五、反例统计

本轮 26 条（PASS=15 / FAIL=9 / NOT_PROVEN=2），ID 唯一，含 9 条 FAIL 暴露 05.43 真实缺陷。历史累计 RAW=519、唯一 ID=519、唯一缺陷=517。

---

## 六、最终结论

**PASS-WITH-ISSUES**。五项 B43 缺陷全部闭合，单进程事务安全协议闭合；跨进程/跨重启能力维持 NOT_PROVEN → DENY/ISOLATED。

不进入 IMPLEMENT-01，不 A/B Merge，不 Source Retire。等待 ChatGPT 独立复审。
