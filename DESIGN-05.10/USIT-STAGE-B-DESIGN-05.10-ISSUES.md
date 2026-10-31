# USIT-STAGE-B-DESIGN-05.10｜问题清单（ISSUES）

> 阶段 B · DESIGN-05.10 · READ-ONLY
> 前置：DESIGN-05.9 独立审查结论 **FAIL**
> 本清单逐项记录 P0-01～P0-04 的修订结果、状态、证据章节与剩余风险。
> 状态取值：PASS / FAIL / NOT_VERIFIED / NOT_TESTED / UNKNOWN / BLOCKED。禁止把未验证项写成 PASS。

---

## 1. 05.9 硬伤 → 05.10 修正对照总表

| 编号 | 05.9 审查指出的硬伤 | 05.10 修正 | 状态 |
|------|---------------------|-----------|------|
| A1 | 用 `fcntl(F_GETLK)` 返回 PID 证明"自身持有记录锁" | 该假设不成立；改用 OFD 锁 + ProcessMutex；`assert_lock_owned` 不再查自身锁 | PASS（设计级）|
| A2 | 传统 fcntl 进程关联记录锁语义混乱（close 任一 fd 释放全部锁）| 明确区分进程关联锁 vs OFD 锁生命周期，锁定 OFD 锁方案 | PASS（设计级）|
| A3 | 无进程内线程互斥层 | 新增 ProcessMutex（threading.Lock，按 auth_id）| PASS（设计级）|
| A4 | `_atomic_replace(src, None)` 未定义伪接口 | 改为 `rename src -> src.corrupt`，七步隔离协议 | PASS（设计级）|
| A5 | 清理状态机两套表达矛盾（CLAIMED/… vs DELETE_INTENT/REMOVED）| 统一为单一状态机 | PASS（设计级）|
| A6 | 清理先删对象后写审计，丢失恢复依据 | 改为意图先行（DELETE_INTENT 先持久化）| PASS（设计级）|
| A7 | 审计记录只存路径/哈希/状态字符串 | 改为终态快照（完整不可变 COMMITTED/ABORTED 摘要）| PASS（设计级）|
| A8 | CAS 无有效互斥绑定 | CAS 绑定授权锁，读-比较-写全程持锁 | PASS（设计级）|
| A9 | Windows 虚构等价 F_GETLK 接口 | 明确 Windows 无该接口，靠 dwShareMode=0 独占打开间接保证 | PASS（设计级，NOT_VERIFIED）|

---

## 2. P0-01｜锁协议基础

| 项 | 内容 | 状态 |
|----|------|------|
| 问题 | 05.9 用 F_GETLK 证明自身锁，POSIX 语义混乱，无线程互斥 | — |
| 修订 | §1.1～1.9：OFD 锁 + ProcessMutex；FD 唯一所有权；fork 重新 acquire；异常/退出闭环 | PASS（设计级）|
| 证据 | 主文档 §1.1 锁区分表、§1.2 方案、§1.3 ProcessMutex、§1.4 FD 规则、§1.5 fork、§1.6 异常表、§1.7 Windows、§1.8 接口契约 | — |
| 前置条件 | Linux≥3.15/glibc≥2.20 支持 OFD 锁 | — |
| 失败行为 | 不支持 OFD → 显式降级 posix-nofd（独立锁文件 + ProcessMutex），标记 NOT_VERIFIED | — |
| DENY 条件 | assert_lock_owned 任一层验证失败即 DENY | — |
| 剩余风险 | OFD 锁在 macOS/BSD 不支持；Windows 独占打开保证边界未实测 | NOT_VERIFIED |

**P0-01 结论：PASS（设计级）**，平台运行时行为 NOT_VERIFIED。

---

## 3. P0-02｜Candidate 提交与恢复协议

| 项 | 内容 | 状态 |
|----|------|------|
| 问题 | candidate 无 PREPARED 时盲目重建；提交持久化顺序不完整 | — |
| 修订 | §2.1 排他创建；§2.2 创建顺序；§2.3 十步提交协议；§2.4 父目录 fsync；§2.5 原件保留；§2.6 唯一动作表；§2.7 恢复矩阵 | PASS（设计级）|
| 证据 | 主文档 §2.1～§2.7 | — |
| 前置条件 | 以 P0-01 锁接口为前提 | — |
| 失败行为 | 哈希/revision 冲突、替换失败、目录同步失败 → 隔离 + DENY | — |
| DENY 条件 | 恢复矩阵中证据不足一律隔离 + DENY | — |
| 剩余风险 | 自动重放提交（正式文档旧版 + PREPARED 有效）为设计决定，未运行时验证 | NOT_TESTED |

**P0-02 结论：PASS（设计级）**，恢复行为 NOT_TESTED。

---

## 4. P0-03｜CAS 原子性与 Journal 隔离恢复

| 项 | 内容 | 状态 |
|----|------|------|
| 问题 | 05.9 revision 比较逻辑无效、`_atomic_replace(src, None)` 伪接口 | — |
| 修订 | §3.1 CAS 绑定锁；§3.2 完整原子性前置；§3.3 并发证明；§3.4 失败唯一动作；§3.5 临时文件；§3.6 七步隔离；§3.7 唯一动作表；§3.8 平台写入顺序 | PASS（设计级）|
| 证据 | 主文档 §3.1～§3.8 | — |
| 前置条件 | 所有 journal 写入入口持同一授权锁 | — |
| 失败行为 | CAS_FAILED 不静默覆盖、不自动改写、不无条件重试 | — |
| DENY 条件 | 审计失败、隔离目录不可用、原文件变化 → DENY | — |
| 剩余风险 | Windows 断电持久性无法证明 | NOT_VERIFIED |

**P0-03 结论：PASS（设计级）**，Windows 断电持久化 NOT_VERIFIED。

---

## 5. P0-04｜清理归因与终态审计证据

| 项 | 内容 | 状态 |
|----|------|------|
| 问题 | 清理先删对象后写审计；审计只存路径/哈希；状态机两套表达 | — |
| 修订 | §4.1 审计存储独立性（同 FS 同故障域）；§4.2 终态快照；§4.3 五步清理；§4.4 不存在证明；§4.5 审计自身持久化；§4.6 唯一动作；§4.7 统一状态机 | PASS（设计级）|
| 证据 | 主文档 §4.1～§4.7 | — |
| 前置条件 | 终态快照在 DELETE_INTENT 前持久化 | — |
| 失败行为 | 审计损坏、身份/哈希变化 → DENY | — |
| DENY 条件 | 证据不足、对象不存在但无法证明删除目标 → DENY | — |
| 剩余风险 | 审计存储不具备独立故障保护（同故障域），仅承诺不被清理流程误删 | NOT_VERIFIED |

**P0-04 结论：PASS（设计级）**，审计存储独立故障保护 NOT_VERIFIED（明确限制范围）。

---

## 6. 交叉一致性检查

| 项 | 结果 | 状态 |
|----|------|------|
| P0-02～04 只用 P0-01 锁接口 | §5.7 确认，未引入第二套锁语义 | PASS（设计级）|
| 四套状态转换图无矛盾 | §5.2 | PASS（设计级）|
| 端到端时序图覆盖全部持久化边界 | §5.1 | PASS（设计级）|
| Windows/POSIX 差异分别说明 | §5.3、§5.4 | PASS（设计级）|

---

## 7. P1 与既有阻断项（本轮不关闭）

| 编号 | 状态 | 说明 |
|------|------|------|
| P1-01 锁文件替换/TOCTOU | NOT_VERIFIED | 未被四项 P0 修订直接闭合 |
| P1-02 analyze_file 行为等价 | NOT_TESTED | 本轮不执行测试 |
| P1-03 A/B 三文件合并 | 保持原状态 | 不执行合并 |
| P0-BLOCK-1 存储事务实现+故障注入 | 未解决 | 设计级修订不替代实现 |
| P0-BLOCK-2 analyze_file 一致性 | 未解决 | 同上 |
| P1-BLOCK-3 A/B 三文件分叉 | 未解决 | 同上 |
| F1 / P0.8 / F2 | 不解除 | — |
| IMPLEMENT-01 | STOP | — |

---

## 8. 剩余风险汇总

1. **POSIX 平台覆盖**：OFD 锁仅在 Linux≥3.15 可用，macOS/BSD 降级为独立锁文件（posix-nofd），互斥语义与 OFD 不同，标记 NOT_VERIFIED。
2. **Windows 断电持久性**：无法证明 FlushFileBuffers + MoveFileEx 在断电后一定持久化，标记 NOT_VERIFIED，强持久化模式下不满足则 DENY。
3. **自动重放提交**：恢复矩阵中"PREPARED 有效 + 正式文档旧版 + candidate 完整 → 自动重放提交"为设计决定，未经运行时验证，NOT_TESTED。
4. **审计存储故障域**：审计记录与事务文件同 FS 同故障域，不承诺审计存储自身故障时可靠，仅承诺不被清理流程误删。
5. **全部为设计级契约**：锁、事务、CAS、清理均为伪代码/状态机，真实实现与故障注入前不得视为已验证。

---

## 9. 结论

- P0-01～P0-04：**PASS（设计级）**，运行时行为 NOT_VERIFIED / NOT_TESTED。
- 交叉一致性：**PASS（设计级）**。
- 三个阻断项、F1/P0.8/F2、IMPLEMENT-01：**维持阻断，未解除**。

**本工单不授权实现、不授权测试、不解除门禁。**
