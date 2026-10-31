# USIT-STAGE-B-DESIGN-05.27 与 05.26 差异（DIFF）

> 仅列 05.26 → 05.27 实质性差异。

## 1. P0-02｜Fork 异常控制流（B27-01）

**05.26**：`finally` 只清理变量+释放锁，未保证 Gate 恢复；`_fork_allowed=False` 当唯一防线。

**05.27**：
- 新增 `_close_fork_parent()` 统一负责 `_after_in_parent` + Gate 恢复，失败→DENIED；
- 明确锁释放 ≠ Gate 恢复 ≠ 业务继续（三件事独立）；
- 新增**统一拒绝门** `assert_gate_open()`，所有业务/FD/fork 入口强制检查，证明调用方无法绕过；
- 区分"永久禁 fork"与"永久禁业务"（RESTORE_FAILED→DENIED 永久禁业务）；
- Gate 状态机扩为七态（增 RESTORE_FAILED/TERMINATING/DENIED）。

## 2. P0-02｜身份唯一性（B27-02）

**05.26**：`fork_event_id` 高熵随机却标"实例内唯一"，随机≠严格唯一。

**05.27**：
- 唯一性承诺明确分级：进程内/进程树**严格**（单调计数器）；实例内/主机**概率**（随机+结构）；跨重启/全局**不承诺**；
- 兄弟碰撞通过"父侧全局严格单调 fork_epoch 分配"**结构性消除**（不再依赖随机）；
- 给出 128-bit 随机位数与碰撞概率（生日界 N²/2^129）；
- 九字段逐项标注"结构性/概率性"。

## 3. P0-04｜状态机与 S1~S16（B27-03）

**05.26**：S4 直接转 PREPARED_CREATED；S2/S3 混为一个持久化状态；S13 名称不统一；S9 五条件/六字段/五谓词未定义；EINTR 当"未执行"。

**05.27**：
- 状态机 11 态 → **16 态**（拆 CANDIDATE_CONTENT/ENTRY、PREPARED_CONTENT/ENTRY、CANDIDATE_VERIFIED、PREPARED_VERIFIED、COMMIT_RECEIPT_CONTENT_DURABLE）；
- S9 五条件、Candidate 六字段、五谓词完整定义；
- EINTR 纠正：返回后状态未知，重试前必须重读核验；
- 安全门扩至 15 项（增"操作是否已发生但结果未知"）。

## 4. 交付（B27-04/B27-05）

- 新增 4 份独立文档：TIMELINES / COUNTEREXAMPLES / CROSS-MODULE-CHAINS / INVARIANTS；
- 交付文件 12 → **16 份**；
- 数量机器统计唯一编号。

## 5. 术语统一

证据等级标注；Gate 七态；身份九字段（唯一性分级）；POSIX 16 态；15 项安全门。
