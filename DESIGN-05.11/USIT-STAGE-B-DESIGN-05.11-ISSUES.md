# USIT-STAGE-B-DESIGN-05.11｜问题清单（ISSUES）

> 阶段 B · DESIGN-05.11 · READ-ONLY
> 前置：DESIGN-05.10 独立审查结论 **FAIL**
> 状态取值：PASS / FAIL / NOT_VERIFIED / NOT_TESTED / UNKNOWN / BLOCKED。禁止把未验证项写成 PASS。

---

## 1. 05.10 六项缺口 → 05.11 修正对照

| # | 05.10 缺口 | 05.11 修正 | 状态 |
|---|-----------|-----------|------|
| G1 | `posix-nofd` 降级仅"创建独立锁文件"，无真实锁原语 | 区分载体/原语，降级选 flock，无锁则 DENY | PASS（设计级）|
| G2 | 错误假设 fork 后 Registry 为空、threading.Lock 会自动释放 | register_at_fork 清理 + 重建 ProcessMutex | PASS（设计级）|
| G3 | 未定义锁文件身份/路径替换/TOCTOU | 三层身份 + 句柄 vs 路径对象比较 + DENY | PASS（设计级）|
| G4 | O_EXCL 被误当 rename 保护；恢复仅凭 SHA-256 | 无覆盖提交原语 + transaction_id/revision 全量绑定 | PASS（设计级）|
| G5 | 隔离"幂等跳过"未重校验；rename 后未 fsync 目录 | 副本重校验 + 每次 rename 后 fsync 父目录 | PASS（设计级）|
| G6 | 终态快照只存摘要；删除归因不分 ALREADY_ABSENT | 完整终态内容 + tombstone 三态归因 | PASS（设计级）|

---

## 2. P0-01｜POSIX 无 OFD 支持时的跨进程锁原语

| 项 | 内容 | 状态 |
|----|------|------|
| 问题 | 05.10 posix-nofd 降级无真实锁原语 | — |
| 修订 | §1.1 载体/原语区分；§1.2 flock vs fcntl 比较；§1.3 flock 实现；§1.4 平台标记 | PASS（设计级）|
| 证据 | 主文档 §1.1～§1.5 | — |
| 失败行为 | flock 亦不可用 → DENY，不进入无锁状态 | — |
| 剩余风险 | flock 降级语义未实测；macOS/BSD 支持情况 NOT_VERIFIED | NOT_VERIFIED |

**P0-01 结论：PASS（设计级）**，运行时 NOT_VERIFIED。

---

## 3. P0-02｜fork 生命周期

| 项 | 内容 | 状态 |
|----|------|------|
| 问题 | 05.10 错误假设 fork 后 Registry 为空 | — |
| 修订 | §2.1 纠错；§2.2 继承资源表；§2.3 register_at_fork；§2.4 持锁 fork 策略；§2.5 场景表；§2.6 清理责任 | PASS（设计级）|
| 证据 | 主文档 §2.1～§2.7 | — |
| 失败行为 | 子进程 close 锁 FD 不调 LOCK_UN（避免释放父锁）；持锁 fork DENY | — |
| 剩余风险 | register_at_fork 实际行为未测试 | NOT_TESTED |

**P0-02 结论：PASS（设计级）**，fork 行为 NOT_TESTED。

---

## 4. P0-03｜锁文件身份与 TOCTOU

| 项 | 内容 | 状态 |
|----|------|------|
| 问题 | 未定义身份/路径替换/TOCTOU，P1-01 直接影响 P0-01 | — |
| 修订 | §3.1 三层身份；§3.2 操作规则；§3.3 身份检测；§3.4 清理规则；§3.5 Windows 职责 | PASS（设计级）|
| 证据 | 主文档 §3.1～§3.6 | — |
| 失败行为 | 身份不一致立即 DENY，禁止提交 | — |
| 剩余风险 | Windows 句柄身份、并发删除窗口 NOT_VERIFIED | NOT_VERIFIED |

**P0-03 结论：PASS（设计级）**，运行时 NOT_VERIFIED。

---

## 5. P0-04｜Candidate 无覆盖提交与事务身份恢复

| 项 | 内容 | 状态 |
|----|------|------|
| 问题 | O_EXCL 被误当 rename 保护；恢复仅凭 SHA-256 推断 | — |
| 修订 | §4.1 身份区分；§4.2 无覆盖原语；§4.3 字段绑定；§4.4 全量验证；§4.5 恢复组合；§4.6 幂等 | PASS（设计级）|
| 证据 | 主文档 §4.1～§4.7 | — |
| 失败行为 | 摘要相同但 txn_id 不同 → DENY；无覆盖提交失败 → DENY | — |
| 剩余风险 | renameat2/link 兜底、MoveFileEx 语义 NOT_VERIFIED | NOT_VERIFIED |

**P0-04 结论：PASS（设计级）**，运行时 NOT_VERIFIED。

---

## 6. P0-05｜Journal Quarantine 持久化

| 项 | 内容 | 状态 |
|----|------|------|
| 问题 | 隔离"幂等跳过"未重校验；rename 后未 fsync 父目录 | — |
| 修订 | §5.1 四对象绑定；§5.2 rename 后 fsync 目录；§5.3 Windows 边界；§5.4 重校验；§5.5 复制中变化 DENY；§5.6 崩溃表；§5.7 审计失败；§5.8 函数契约 | PASS（设计级）|
| 证据 | 主文档 §5.1～§5.9 | — |
| 失败行为 | 副本损坏/身份不明 → DENY；原文件复制中变化 → DENY | — |
| 剩余风险 | Windows 断电持久性 NOT_VERIFIED | NOT_VERIFIED |

**P0-05 结论：PASS（设计级）**，Windows 断电 NOT_VERIFIED。

---

## 7. P0-06｜终态快照与删除归因

| 项 | 内容 | 状态 |
|----|------|------|
| 问题 | 快照只存摘要；删除归因不分 ALREADY_ABSENT | — |
| 修订 | §6.1 完整快照；§6.2 快照先行；§6.3 三态区分；§6.4 tombstone；§6.5 分别定义；§6.6 要求；§6.7 限制；§6.8 状态机 | PASS（设计级）|
| 证据 | 主文档 §6.1～§6.9 | — |
| 失败行为 | 无法归因记 INDETERMINATE/ALREADY_ABSENT，不记 REMOVED | — |
| 剩余风险 | tombstone rename/unlink 并发 NOT_VERIFIED；审计存储同故障域 | NOT_VERIFIED |

**P0-06 结论：PASS（设计级）**，运行时 NOT_VERIFIED。

---

## 8. 交叉一致性

| 项 | 结果 | 状态 |
|----|------|------|
| 三路径互斥原语明确 | §7.1 | PASS（设计级）|
| 线程/进程生命周期一致 | §7.2 | PASS（设计级）|
| 路径身份 TOCTOU 覆盖 | §7.3 | PASS（设计级）|
| 持久化顺序完整 | §7.4 | PASS（设计级）|
| 恢复唯一动作 | §7.5 | PASS（设计级）|
| 删除归因可证明 | §7.6 | PASS（设计级）|
| 崩溃边界分别说明 | §7.7 | PASS（设计级）|
| PASS 项不变量+反例 | §7.8 | PASS（设计级）|

---

## 9. P1 与既有阻断项

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

## 10. 剩余风险汇总

1. **flock 降级语义**：macOS/BSD 不支持 OFD 锁，降级 flock，其 close/fork 语义与 OFD 锁有差异，未实测，NOT_VERIFIED。
2. **fork 行为**：register_at_fork 清理逻辑未测试，NOT_TESTED。
3. **Windows 断电持久性**：无目录 fsync 等价物，断电后目录项可能丢失，NOT_VERIFIED，强持久化模式需 DENY。
4. **renameat2/link 无覆盖兜底**：POSIX 无 renameat2 时用 link 两步法，未实测，NOT_VERIFIED。
5. **tombstone 删除归因**：并发删除窗口未实测，NOT_VERIFIED。
6. **审计存储同故障域**：不承诺审计存储自身故障可靠。

---

## 11. 结论

- P0-01～P0-06：**PASS（设计级）**，运行时行为 NOT_VERIFIED / NOT_TESTED。
- 交叉一致性：**PASS（设计级）**。
- 三个阻断项、F1/P0.8/F2、IMPLEMENT-01：**维持阻断，未解除**。

**本工单不授权实现、不授权测试、不解除门禁。**
