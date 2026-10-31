# USIT-STAGE-B-DESIGN-05.45.md — 主文档

**工单编号**：USIT-STAGE-B-DESIGN-05.45
**前置版本**：DESIGN-05.44
**执行模式**：READ-ONLY ANALYSIS + DESIGN REVISION
**目标**：修复 B44-R01～R05，形成可逐步推演、跨文档一致的单进程事务安全协议。

---

## 一、执行边界

1. 只读取历史设计、分析与修订设计文档，修改项目外交付目录中的设计文档。
2. 未修改任何 `usit/*.py` 源码。
3. 未执行真实 Fork、构建、部署、运行时测试、真实资源操作和第三方请求。
4. 未执行 IMPLEMENT-01、A/B Merge、Source Retire。
5. 未将静态推演写成运行时验证。
6. 先修协议，再做反例。

---

## 二、六项核心修复结果总览

| 编号 | 修复项 | 结论 | 证据文档 |
|---|---|---|---|
| B45-01 | 恢复屏障（Condition + 锁分离） | PASS | RECOVERY-BARRIER-AND-ADMISSION.md |
| B45-02 | 全局请求幂等（global_request_lock + 摘要校验） | PASS | REQUEST-ID-ATOMIC-IDEMPOTENCY.md |
| B45-03 | operation 预留/确认接入真实释放 | PASS | RESOURCE-OPERATION-CRASH-RECOVERY.md |
| B45-04 | 事务顺序与 WAL 恢复边界闭合 | PASS | TXN-ORDER-AND-WAL-REPLAY.md |
| B45-05 | 原子发布、查询与隔离持久化 | PASS | POST-COMMIT-FAILURE-CLOSURE.md |
| B45-09 | 反例重新设计（baseline/revision 分离） | PASS | COUNTEREXAMPLES.json |

---

## 三、核心设计结论

### 1. 恢复屏障（B45-01）

- Condition 机制替代 `threading.Lock().wait()`，双 Condition 分离无死锁。
- `recovery_barrier_guard()` 为真正上下文管理器，所有出口都解除屏障。
- 正常/异常出口均有确定行为。

### 2. 全局请求幂等（B45-02）

- 全局请求锁（`global_request_lock`）先于资源锁，req_id 原子占用在执行副作用前完成。
- 参数摘要校验在同一锁内完成，冲突直接 ISOLATED。
- 两个不同资源并发请求相同 req_id，最多一个进入副作用执行阶段。

### 3. operation 预留/确认接入真实释放（B45-03）

- `atomic_release()` 调式调用 `persist_reservation()` 和 `wal_append_fsync(confirm_rec, status='OP_CONFIRM')`。
- 预留持久化失败 → ABORT；确认持久化失败 → UNKNOWN（查 OP_RESERVATION 判断是否已执行，ISOLATED 或重放）。

### 4. 事务顺序与 WAL 恢复边界（B45-04）

- PREPARE 无 commit_seq，不参与 commit_seq 排序。
- commit_seq 耗尽（u64 最大值）永久拒绝。
- 检查点含完整状态快照 + 请求幂等索引 + 可验证摘要，跳过历史记录从快照恢复。

### 5. 原子发布、查询与隔离持久化（B45-05）

- `publish_atomic()` 采用不可变聚合快照 + 单指针切换，非锁内依次赋值。
- `query_txn()` 校验 COMMIT 唯一性、APPLY 唯一性、commit_seq 一致性、snapshot_hash 完整性。
- 三层失败处理，绝不返回普通 ERROR。

### 6. 反例重新设计（B45-09）

- 41 条反例区分 `baseline_status`（05.44 旧缺陷状态）与 `revision_status`（05.45 修订验证状态）。
- 24 条 baseline_status=FAIL → revision_status=PASS（05.45 修复）。
- 2 条 baseline_status=NOT_PROVEN → revision_status=NOT_PROVEN（维持 DENY）。

---

## 四、未证明能力（继续 DENY/ISOLATED）

十项跨进程/跨重启能力（跨进程 WAL 并发写、跨重启恢复、跨进程可信身份传递、operation 非幂等自动重放、线程持锁崩溃恢复、跨进程资源锁一致性、跨进程恢复屏障、跨进程 Fencing、跨进程预留/确认、跨进程检查点一致性）全部 NOT_PROVEN → 维持 DENY/ISOLATED（详见 ISSUES.md）。

---

## 五、反例统计

本轮 41 条（baseline_status=22 FAIL / 2 NOT_PROVEN；revision_status=24 PASS / 0 NOT_PROVEN），ID 唯一。历史累计 RAW=560、唯一 ID=560、唯一缺陷=558。

---

## 六、最终结论

**PASS-WITH-ISSUES**。六项 B44 缺陷全部闭合，单进程事务安全协议闭合；跨进程/跨重启能力维持 NOT_PROVEN → DENY/ISOLATED。

不进入 IMPLEMENT-01，不 A/B Merge，不 Source Retire。等待 ChatGPT 独立复审。
