# USIT-STAGE-B-DESIGN-05.25 与 05.24 差异（DIFF）

> 仅列 05.24 → 05.25 实质性差异。

## 1. P0-02｜Fork 异常控制流（B25-01）

**05.24**：
- `restore_gate()` 在 except 块中裸调用，若抛异常则 `release(_fork_coord_lock)` 被跳过 → 锁永久持有；
- 子进程初始化失败仅 `_deny_protected_ops()` 后 `return 0`，仍带坏状态返回业务栈；
- 仅检查"主线程"，被误当"单线程进程"；
- 保留"两线程并发 fork"场景，与主线程限制矛盾；
- 05.24 已选手动接管但残留 at-fork 表述。

**05.25**：
- `_safe_restore_gate()` 内部 try/except 吞异常，`release(L0)` 无条件执行；
- 子进程初始化失败强制 `os._exit(1)`，不带坏状态继续；
- 新增 `is_known_single_threaded_process()`，多线程进程永久 fail-closed 拒绝 fork；
- **删除"两线程并发 fork"场景**（多线程进程入口即拒绝）；
- 明确完全手动接管、不注册 at-fork 回调，单一机制。

## 2. P0-02｜Epoch 身份重构（B25-02）

**05.24**：`process_generation + pid + fork_epoch + fd_generation` 直接宣称全局唯一身份。

**05.25**：
- **废除全局唯一宣称**；
- 三方案比较（A 局部 / B 实例随机 ID / C 协调器），**采用方案 B**；
- 新增 `process_instance_id`（128-bit 随机）作为实例内唯一身份，跨 fork 继承、跨重启不唯一；
- 唯一性精确分级：进程内唯一 / 实例内唯一 / 事务内可持久，**无字段宣称跨重启或全局唯一**；
- 明确随机源不可用降级/拒绝、恶意复制身份无认证。

## 3. P0-04｜S1~S16 矩阵（B25-03）

**05.24**：§3.3 S12-G 仍写"current 旧值 → rename 未发生 → 重试 S12"，与 §4.1 矛盾。

**05.25**：
- **彻底删除 S12-G 矛盾推断**，改为"核验身份/哈希判定真实状态，不直接重试 S12"；
- 状态机从七态重构为 **11 态**（新增 NEW/CANDIDATE_CREATED/CANDIDATE_DURABLE/PREPARED_CREATED/PREPARED_DURABLE/CURRENT_REPLACE_PENDING）；
- 新增自动删除/重试/回滚的 8 项证明门槛。

## 4. P0-04｜目录 fsync 统一（B25-04）

**05.24**：目录 fsync 恢复按目录对象不同分散描述，状态映射不统一。

**05.25**：S3/S7/S13/S15 四目录统一语义 + 统一恢复决策（回答 8 个问题），状态映射一致。

## 5. 交付（B25-05 / B25-06）

- **B25-05**：新增 `USIT-STAGE-B-DESIGN-05.25-ZIP-RECEIPT.txt` 作为独立交付物（外部回执），记录 ZIP 自身哈希 + 完整核验日志；
- **B25-06**：记录采集命令/时间/工作区/Git HEAD（本工作区非 git 仓库，记为 N/A）。

## 6. 术语统一

证据等级标注 `VERIFIED_FACT / STATIC_INFERENCE / DESIGN_PROPOSAL / UNKNOWN / CONFLICT`；11 态状态名；`process_instance_id + process_generation + fork_epoch + fd_generation + snapshot_id + txn_id` 六字段规范。
