# USIT-STAGE-B-DESIGN-05.46.md — 主文档

**工单编号**：USIT-STAGE-B-DESIGN-05.46
**前置版本**：DESIGN-05.45
**执行模式**：READ-ONLY ANALYSIS + DESIGN REVISION
**目标**：修复 B45-R01～R05，形成可逐步推演、跨文档一致的单进程事务安全协议。

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
| B46-01 | 恢复屏障（单一 `barrier_cond`） | PASS | RECOVERY-BARRIER-AND-ADMISSION.md |
| B46-02 | 全局请求幂等（global_request_lock + 完整参数） | PASS | REQUEST-ID-ATOMIC-IDEMPOTENCY.md |
| B46-03 | operation 预留/确认（七状态机） | PASS | RESOURCE-OPERATION-CRASH-RECOVERY.md |
| B46-04 | WAL 与检查点统一（完整快照 + 统一 recover()） | PASS | WAL-INTEGRITY-AND-CHECKPOINT.md |
| B46-05 | 原子发布、查询与隔离持久化（ImmutableSnapshot + 唯一性校验） | PASS | POST-COMMIT-FAILURE-CLOSURE.md |
| B46-06 | 反例重新设计（baseline/revision 分离） | PASS | COUNTEREXAMPLES.json |

---

## 三、核心设计结论

### 1. 恢复屏障（B46-01）

- 单一 `barrier_cond` 保护屏障标志、在途计数、进入/退出条件、通知/等待，消除双 Condition 不同步。
- 解锁分离，无死锁。
- 资源操作生命周期完整覆盖：从 req_id 占用到 OP_RELEASE/COMMIT/APPLY/ISOLATED。

### 2. 全局请求幂等（B46-02）

- 参数完整定义：`op`、`payload_hash`、`resource` 齐全，禁止未定义变量。
- `start_req()` 使用 WAL 事务持久化，ABORT/UNKNOWN。
- 锁顺序：`barrier_cond → global_request_lock → resource_lock → txn_guard`。

### 3. operation 预留/确认（B46-03）

- 七状态状态机：RESERVED_NOT_STARTED → OPERATION_STARTED → OPERATION_CONFIRMED → COMMITTED → RELEASED → UNKNOWN/ISOLATED。
- 预留后、operation 前崩溃：查 OP_STARTED，判断 operation 未执行，不重复。

### 4. WAL 与检查点统一（B46-04）

- CHECKPOINT 含完整状态快照 + TRT + req_id_index + operation_state_index + wal_position + snapshot_hash + checksum。
- 统一恢复入口 `recover()`，检查点存在时从快照恢复，不存在时从 WAL 起点恢复。

### 5. 原子发布、查询与隔离持久化（B46-05）

- `ImmutableSnapshot` 私有字段 + 受保护访问，真正冻结内部状态。
- `query_txn()` 校验唯一 COMMIT、唯一 APPLY、commit_seq 一致性、snapshot_hash。
- 三层失败处理，绝不返回普通 ERROR。

---

## 四、未证明能力（继续 DENY/ISOLATED）

十项跨进程/跨重启能力（跨进程 WAL 并发写、跨重启恢复、跨进程可信身份传递、operation 非幂等自动重放、线程持锁崩溃恢复、跨进程资源锁一致性、跨进程恢复屏障、跨进程 Fencing、跨进程预留/确认、跨进程检查点一致性）全部 NOT_PROVEN → 维持 DENY/ISOLATED（详见 ISSUES.md）。

---

## 五、反例统计

本轮 38 条（baseline_status：22 FAIL / 2 NOT_PROVEN；revision_status：32 PASS / 0 NOT_PROVEN），ID 唯一。历史累计 RAW=598、唯一 ID=598、唯一缺陷=596。

---

## 六、最终结论

**PASS-WITH-ISSUES**。六项 B45 缺陷全部闭合，单进程事务安全协议闭合；跨进程/跨重启能力维持 NOT_PROVEN → DENY/ISOLATED。

不进入 IMPLEMENT-01，不 A/B Merge，不 Source Retire。等待 ChatGPT 独立复审。
