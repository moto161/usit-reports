# USIT-STAGE-B-DESIGN-05.12｜问题清单（ISSUES）

> 阶段 B · DESIGN-05.12 · READ-ONLY
> 前置：DESIGN-05.11-INDEPENDENT-REVIEW（结论 FAIL）
> 状态取值：PASS / FAIL / NOT_VERIFIED / NOT_TESTED / UNKNOWN / BLOCKED。

---

## 1. 六项缺陷分类与修复结果

| 编号 | 独立审查指出的缺陷 | 缺陷类别 | 修复方案 | 状态 |
|------|-------------------|---------|---------|------|
| P0-01 | flock 继承语义矛盾 | A（确定矛盾）| §1 纠正 fork 语义 | PASS（设计级）|
| P0-02 | fork 回调契约缺失 | B（契约缺失）| §2 LockHandleRegistry + ForkGate | PASS（设计级）|
| P0-03 | 路径身份 TOCTOU 未消除 | A+B（竞态）| §3 全程句柄 + dirfd/openat | PASS（设计级）|
| P0-04 | Windows 半成品暴露 | A+B | §4 先 tmp 后无覆盖发布 | PASS（设计级）|
| P0-05 | 先 qfinal 后审计，孤立副本 | A（顺序错误）| §5 意图先于 qfinal | PASS（设计级）|
| P0-06 | tombstone 状态矛盾 | A（确定矛盾）| §6 UNLINK_INTENT 状态 | PASS（设计级）|

**缺陷类别说明**：A=确定矛盾（逻辑必有一错），B=契约缺失（应定义未定义），C=未验证风险（保留 NOT_VERIFIED）。分类依据见主文档 §0。

---

## 2. 逐项问题、修复、剩余风险

### P0-01
- 原缺陷：05.11 §1.2 "子进程不继承 flock" 与 "fork 复制 fd 引用同一 OFD" 矛盾。
- 修复：主文档 §1.1 纠正为"子进程继承 OFD 引用，共享锁"；§1.2 三种锁原语所有权模型；§1.4 close/LOCK_UN/退出影响矩阵；§1.5 fork 前后状态表。
- 剩余风险：flock/OFD 锁平台语义未实测 → NOT_VERIFIED。

### P0-02
- 原缺陷：`_inherited_lock_fds` 无登记机制；`_in_lock_critical_section` 布尔标志不可靠。
- 修复：§2.2 LockHandleRegistry 单一登记表；§2.4 ForkGate 锁互斥替代布尔标志；§2.6 子进程重置顺序；§2.7 外部 fork 拒绝策略。
- 剩余风险：register_at_fork 行为未测试 → NOT_TESTED。

### P0-03
- 原缺陷：检查后到使用之间路径可被替换。
- 修复：§3.2 全程句柄原则；§3.3 openat/dirfd 序列；§3.4 竞态分析；§3.5 目录可信前提。
- 剩余风险：Windows 句柄语义、目录并发未实测 → NOT_VERIFIED。

### P0-04
- 原缺陷：CREATE_NEW 空占位暴露半成品。
- 修复：§4.2 先 tmp 后发布；§4.3 POSIX renameat2/link 无覆盖；§4.4 Windows MoveFileEx 不带 REPLACE；§4.7 恢复矩阵。
- 剩余风险：MoveFileEx/renameat2 语义未实测 → NOT_VERIFIED。

### P0-05
- 原缺陷：先发布 qfinal 后写审计，崩溃留孤立副本。
- 修复：§5.3 意图（QUARANTINE_INTENT）先于 qfinal；§5.5 副本隔离与源处理分两阶段；§5.7 恢复矩阵。
- 剩余风险：Windows 断电/目录持久化 → NOT_VERIFIED。

### P0-06
- 原缺陷：§6.3 要求 tombstone 存在，§6.4 实际 unlink，矛盾。
- 修复：§6.3 新增 UNLINK_INTENT 状态；§6.4 顺序调整；§6.5 恢复矩阵解决矛盾；§6.6 三态归因表。
- 剩余风险：tombstone 并发/外部删除 → NOT_VERIFIED。

---

## 3. 剩余风险汇总

1. **平台语义未实测**：flock fork/close 语义、OFD 锁、renameat2、MoveFileEx、Windows 目录持久化——全部 NOT_VERIFIED。
2. **fork 行为未测试**：register_at_fork、ForkGate 串行化——NOT_TESTED。
3. **Windows 无目录 fsync 等价**：断电持久性无法证明——NOT_VERIFIED，强一致模式需 DENY。
4. **审计存储同故障域**：不承诺审计存储自身故障可靠。
5. **全部设计级契约**：真实实现前不得视为已验证。

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

- P0-01～P0-06：**PASS（设计级）**，运行时 NOT_VERIFIED / NOT_TESTED。
- 交叉审查：**PASS（设计级）**，见 CROSS-REVIEW.md。
- 三个阻断项、F1/P0.8/F2、IMPLEMENT-01：**维持阻断，未解除**。

**本工单不授权实现、不授权测试、不解除门禁。**
