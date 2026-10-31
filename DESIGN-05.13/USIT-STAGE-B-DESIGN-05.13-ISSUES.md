# USIT-STAGE-B-DESIGN-05.13｜问题清单（ISSUES）

> 阶段 B · DESIGN-05.13 · READ-ONLY
> 前置：DESIGN-05.12 独立审查结论 FAIL
> 修订范围：P0-02、P0-03、P0-04、P0-05、P0-06（P0-01 已通过，不重复）
> 状态取值：PASS / FAIL / NOT_VERIFIED / NOT_TESTED / UNKNOWN / BLOCKED。

---

## 1. 五项缺陷分类与修复结果

| 编号 | 缺陷 | 类别 | 修复方案 | 状态 |
|------|------|------|---------|------|
| P0-02 | child 回调调用带锁方法死锁 | 确定缺陷 | §1 prepare 快照 tuple，child 禁碰锁 | PASS（设计级）|
| P0-03 | 目录权限无法防同用户替换锁文件 | 错误假设 | §2 永久锁文件 + 身份变化全体 DENY | PASS（设计级）|
| P0-04 | PREPARED 顺序与恢复矩阵矛盾 | 顺序矛盾 | §3 PREPARED 先持久化再发布 | PASS（设计级）|
| P0-05 | inode 身份相同≠内容相同 | 错误等同 | §4 双哈希三方校验 + 独立 retire 阶段 | PASS（设计级）|
| P0-06 | UNLINK_INTENT 只证意图不证事实 | 归因过度 | §5 tombstone 不 unlink，诚实处理归因边界 | PASS（设计级）|

---

## 2. 逐项问题、修复、剩余风险

### P0-02
- 原缺陷：05.12 §2.6 child 回调调用 `snapshot_fds()`（内部 `with self._lock`），继承锁可能 locked → 子进程死锁。
- 修复：§1.3 child 回调禁碰锁；§1.4 prepare 阶段快照 FD 为不可变 tuple，child 只读 tuple；§1.5 ForkGate 也重建。
- 剩余风险：register_at_fork / fork 行为未测试 → NOT_TESTED。

### P0-03
- 原缺陷：05.12 §3.5 假设目录权限能防同用户进程替换锁文件，实际不能。
- 修复：§2.3 永久锁文件 + 禁 unlink/replace；§2.6 身份变化全体停止 DENY。
- 剩余风险：同用户恶意进程、Windows 身份语义 → NOT_VERIFIED。

### P0-04
- 原缺陷：05.12 §4.6/§4.7 PREPARED 持久化时机与发布重叠，恢复证据未先于崩溃点生成。
- 修复：§3.2/§3.3 PREPARED（阶段 B）先于发布（阶段 C）持久化；§3.8 恢复矩阵重排。
- 剩余风险：CAS/无覆盖发布语义 → NOT_VERIFIED。

### P0-05
- 原缺陷：05.12 §5.3~5.5 把 inode 身份不变当成内容不变。
- 修复：§4.3 双哈希三方校验；§4.5 SOURCE_RETIRE_INTENT 独立阶段；§4.6 处理前再校验。
- 剩余风险：外部写入场景 → NOT_VERIFIED。

### P0-06
- 原缺陷：05.12 §6.5 用 UNLINK_INTENT 存在 + tombstone 不存在归因为本事务删除。
- 修复：§5.6 承认 unlink 不可归因；§5.7 tombstone 不 unlink（保留为删除证据）；§5.8 诚实三态表。
- 剩余风险：tombstone 保留方案、并发删除 → NOT_VERIFIED。

---

## 3. 剩余风险汇总

1. **平台语义未实测**：flock/OFD 锁、renameat2、MoveFileEx、Windows 目录持久化、同用户进程竞争——全部 NOT_VERIFIED。
2. **fork 行为未测试**：register_at_fork、prepare 快照、child 重建——NOT_TESTED。
3. **外部写入无法阻止**：源对象被绕过协议的外部进程修改，内容校验能暴露但无法阻止——NOT_VERIFIED。
4. **删除归因边界**：unlink 不可归因是文件系统本质，采用 tombstone 不 unlink 方案规避，但 tombstone 占用磁盘需归档流程——NOT_VERIFIED。
5. **审计存储同故障域**：不承诺审计存储自身故障可靠。
6. **全部设计级契约**：真实实现前不得视为已验证。

---

## 4. P1 与既有阻断项

| 编号 | 状态 |
|------|------|
| P1-01 | 设计级闭合（P0-03），运行时 NOT_VERIFIED |
| P1-02 | NOT_TESTED |
| P1-03 | 保持原状态，不执行 |
| P0-BLOCK-1 | 未解决 |
| P0-BLOCK-2 | 未解决 |
| P1-BLOCK-3 | 未解决 |
| F1 / P0.8 / F2 | 不解除 |
| IMPLEMENT-01 | STOP |

---

## 5. 结论

- P0-02～P0-06：**PASS（设计级）**，运行时 NOT_VERIFIED / NOT_TESTED。
- 交叉审查：**PASS（设计级）**，见 CROSS-REVIEW.md。
- 三个阻断项、F1/P0.8/F2、IMPLEMENT-01：**维持阻断，未解除**。

**本工单不授权实现、不授权测试、不解除门禁。**
