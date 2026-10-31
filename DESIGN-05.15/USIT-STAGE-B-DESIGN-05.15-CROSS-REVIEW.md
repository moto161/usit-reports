# USIT-STAGE-B-DESIGN-05.15-CROSS-REVIEW｜反例审查记录

> READ-ONLY DESIGN REVISION ONLY · 日期：2026-10-31
> 性质：独立于主设计的反例审查，主动寻找反例，不只验证正常路径

## 一、四项 P0 的反例审查

### P0-02 反例

| 反例 | 结论 |
|------|------|
| A 持 Registry 锁，B 持 ForkGate 锁并 fork | 已消除：禁持锁 fork + before 无锁快照（§2.4）|
| 双线程各自持不同锁并 fork | 已消除：禁持锁 fork，入口拒绝（§2.8 T5）|
| before 回调拿锁导致死锁 | 已消除：before 无锁（§2.5）|

**未闭合**：外部代码绕过 API 直接持锁 fork 无法拦截 → 属"不支持"，文档化。

### P0-04 反例

| 反例 | 结论 |
|------|------|
| T1/T2 同时 link 不同锁文件名成功 → 双 rename 丢更新 | 已消除：废弃 link 假 CAS，改 flock 临界区（§1.2）|
| revision 检查与发布之间插入其他事务 | 已消除：flock 串行化 check+publish（§1.5）|
| 持锁后崩溃 | 已处理：OS 释放 flock，current 未变，按 current 判定（§1.7 X6）|

**未闭合**：Windows 的 LockFileEx 锁字节范围是否等价 flock 排他性，NOT_VERIFIED。

### P0-05 反例

| 反例 | 结论 |
|------|------|
| seal 前持可写 FD，seal 后继续写 | 协作 flock 阻止；恶意无法阻止，检测哈希变化（§3.4）|
| rename 不撤销旧 FD | 明确，需 sealed + 重新验证（§3.4）|
| mmap 映射写入 | 需 fsync 后重验，NOT_VERIFIED（§3.8 Y5）|

**未闭合**：恶意同 UID 进程的强写无法强制阻止（除非 chattr +i 特权），NOT_VERIFIED。

### P0-06 反例

| 反例 | 结论 |
|------|------|
| 源文件 rename = tombstone 元数据（对象矛盾）| 已消除：四对象分离（§4.2）|
| 两事务同时删除同一源 | 隔离对象 rename 竞争，一个成功（§4.8 Z2）|
| 两事务同时发布同名 tombstone 元数据 | O_EXCL，一个成功一个换 ID（§4.8 Z3）|
| rename 后崩溃 REMOVED 未持久化 | 元数据匹配补写 / 缺失则 INDETERMINATE（§4.8 Z4/Z5）|
| 外部替换 tombstone 元数据 | 身份不匹配 → INDETERMINATE（§4.8 Z6）|
| 审计写入失败 | INDETERMINATE（§4.8 Z9）|

**未闭合**：rename/fsync 崩溃语义 NOT_TESTED。

## 二、25 条不变量审查

主文档 §5 列出 I-01~I-25，每条有证明章节与拒绝策略。逐类核验：

- 锁 + fork（I-01~I-07）：§2 证明，全局锁顺序闭环。
- CAS + 提交（I-08~I-14）：§1 证明，flock 临界区闭环。
- sealed（I-15~I-19）：§3 证明，状态机 + 重新验证。
- tombstone（I-20~I-25）：§4 证明，对象分离 + 归因证据链。

**25 条不变量全部有对应推演证据，无"未证明的 PASS"。**

## 三、10 条端到端链路审查

主文档 §6 列出 E1~E10，每条含前置/状态转换/持久化边界/竞争点/崩溃点/恢复动作/结论。

- E1~E9：遵守协议边界内无已知矛盾。
- E10（恶意同 UID）：NOT_VERIFIED，如实标注。

## 四、审查结论

四项 P0 的已知反例均已正面处理或诚实标注边界；未发现新的、未处理的反例。整体 = **PASS-WITH-ISSUES（设计级）**，恶意同 UID 进程场景 NOT_VERIFIED。
