# USIT-STAGE-B-DESIGN-05.14-CROSS-REVIEW｜端到端交叉审查记录

> READ-ONLY DESIGN REVISION ONLY · 日期：2026-10-31
> 审查范围：P0-02~P0-06 跨章节一致性 + 八条端到端链路 + 20 条不变量

## 一、八条端到端链路审查

| 链路 | 前置条件 | 操作顺序 | 锁约束 | 对象身份 | 持久化边界 | 并发行为 | 崩溃点 | 恢复动作 | 不变量 | 结论 |
|------|---------|---------|-------|---------|-----------|---------|--------|---------|--------|------|
| L1 锁→Lease→fork→父子退出 | 有效锁 | 锁→Lease→fork→回调→退出 | flock 排他 | fd/OFD | flock 成功 | 单线程 fork | fork 中/后 | child 重置锁系统 | I-03/04 | PASS |
| L2 锁身份→Candidate→PREPARED→发布→revision | 有效锁 | 版本文件→校验→PREPARED→指针→COMMIT | link 占有 | 版本文件 inode | PREPARED 先于 rename | 两事务抢指针 | rename 前后 | 按 current 为准 | I-05/06/07 | PASS |
| L3 Journal意图→qtmp→qfinal→源处理→审计 | 意图持久化 | intent→复制→qfinal→源处理→审计 | 源不可变/排他 | 源 inode+哈希 | qtmp→qfinal rename | 源并发写 | 复制中/发布中 | 哈希+身份校验 | I-09/10 | PASS-WITH-ISSUES |
| L4 DELETE_INTENT→…→CLEANED | 意图持久化 | intent→rename→REMOVED→CLEANED | O_EXCL tombstone | tombstone inode+哈希 | rename→REMOVED | 并发删除 | rename 后 | INDETERMINATE 若 REMOVED 未写 | I-11/12 | PASS-WITH-ISSUES |
| L5 崩溃→重启→恢复→归因 | 全链路证据 | 各步 | — | — | 各步 fsync | — | 任意 | 按证据链归因 | I-08/13 | PASS-WITH-ISSUES |
| L6 两事务竞争同一对象 | 并发事务 | 竞争 | link/O_EXCL | — | 竞争点 | 抢占 | 竞争点 | 失败方放弃重试 | I-07/18 | PASS |
| L7 路径替换/目标存在/摘要不符/审计不可用 | 异常 | — | — | 身份/哈希 | — | — | 异常点 | DENY/INDETERMINATE | I-08/13 | PASS |
| L8 同 UID 外部进程绕过协议改文件 | 恶意进程 | — | 无（无法阻止）| — | — | 任意 | 任意 | 身份异常检测+拒绝 | I-17 | **NOT_VERIFIED** |

**结论**：L1~L7 在遵守协议边界内无已知矛盾；L8 是本设计已知能力边界，如实标注 NOT_VERIFIED。

## 二、20 条设计不变量审查

主文档 §6 列出 I-01~I-20，每条均给出证明章节、失效场景、拒绝策略。逐条核验：

- I-01~I-04（锁 + fork）：由 §2.5/§2.9/§1.5/§1.6 证明，无矛盾。
- I-05~I-07（Candidate 发布）：由 §3.2/§3.3/§3.4 证明，link 占有标记闭环。
- I-08/I-13（恢复归因 + 不确定拒绝）：贯穿 §3.7/§5.5/§4.2，一致。
- I-09/I-10（quarantine 一致性）：由 §4.4/§4.5 证明。
- I-11/I-12（删除归因）：由 §5.2/§5.5/§5.7 证明。
- I-14/I-15（不冒充验证 + 平台限定）：全文 NOT_TESTED 标注一致。
- I-16/I-17（外部 fork + 恶意进程边界）：由 §1.7/§2.3/§2.4 证明。
- I-18（不丢更新）：由 §3.7 C2 证明。
- I-19（quarantine ≠ retire 授权）：由 §4.4 证明。
- I-20（tombstone 不提前删）：由 §5.4/§5.5 证明。

**20 条不变量全部有对应推演证据，无"未证明的 PASS"。**

## 三、最小反例（每个 FAIL/边界给出可复现时间线）

| 反例 | 问题 | 时间线 |
|------|------|--------|
| inode 分裂 | P0-03 | A 持 inode-A 锁 → B unlink+recreate 得 inode-B → 双持（主文档 §2.2）|
| 旧 FD 写入 | P0-05 | rename 前 open(O_WRONLY) → rename 后旧 fd 继续写同一 inode（§4.5 W5）|
| REMOVED 未持久化 | P0-06 | rename 后崩溃，tombstone 在但 REMOVED 无 → INDETERMINATE（§5.7 D2）|
| 读-比较-写丢更新 | P0-04 | 两事务读同 revision 各自写（§3.1）|

## 四、审查结论

跨章节无已知逻辑矛盾；四项反例均已正面处理（或诚实标注边界）。整体 = **PASS-WITH-ISSUES（设计级）**，恶意同 UID 进程场景 NOT_VERIFIED。
