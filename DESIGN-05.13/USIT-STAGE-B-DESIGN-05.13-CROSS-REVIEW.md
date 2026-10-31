# USIT-STAGE-B-DESIGN-05.13｜端到端交叉审查记录（CROSS-REVIEW）

> 阶段 B · DESIGN-05.13 · 静态交叉审查
> 对象：主文档五项 P0 协议 + 15 条设计不变量
> 方法：逐链路推演崩溃点、并发竞争、异常退出、恢复重入、绕过协议的外部进程

---

## 1. 八条链路审查

### 链路 1：多线程持 Lease → 受控 fork → 父子回调 → 子进程重初始化 → 父子退出

| 项 | 内容 |
|----|------|
| 前置条件 | 父进程多线程，某线程持 Lease |
| 操作顺序 | fork_controlled → before 快照 FD tuple → fork → after_in_child 读 tuple + close + 重建 |
| 锁约束 | ForkGate 串行化 fork 与 acquire/release |
| 对象身份 | prepare 快照的 tuple 不可变 |
| 持久化边界 | 无（运行时资源）|
| 并发行为 | 其他线程持 Mutex 不影响（child 重建，不 acquire）|
| 崩溃点 | fork 后子进程回调 |
| 恢复动作 | child 禁碰锁，读 tuple，重建全部锁对象 |
| 安全不变量 | I-03、I-04 |
| 拒绝条件 | `_fork_reset_failed` → DENY |

**结论：无矛盾。** child 回调不再调用任何带锁方法。

### 链路 2：多进程打开锁文件 → 身份验证 → 加锁 → 路径替换 → 并发竞争

| 项 | 内容 |
|----|------|
| 前置条件 | 永久锁文件已创建 |
| 操作顺序 | openat + fstat + 基准比对 + flock/OFD 加锁 + 复核 |
| 锁约束 | OS 锁互斥 |
| 对象身份 | dev+ino（POSIX）/ volume+index（Windows）|
| 持久化边界 | 无 |
| 并发行为 | 同用户恶意进程 unlink 重建 → 身份变化 |
| 崩溃点 | 身份变化时刻 |
| 恢复动作 | 全体停止 + DENY，不对准新对象 |
| 安全不变量 | I-02 |
| 拒绝条件 | 身份不一致 → DENY |

**结论：无矛盾。** 永久锁文件 + 身份变化全体停止，阻断 inode-A/inode-B 双锁路径。

### 链路 3：Candidate 临时文件 → PREPARED → 无覆盖发布 → revision → 崩溃恢复

| 项 | 内容 |
|----|------|
| 前置条件 | 授权锁持有 |
| 操作顺序 | 阶段 A（tmp+candidate）→ B（PREPARED）→ C（发布）→ D（COMMITTED）|
| 锁约束 | 授权锁 + 受保护 CAS |
| 对象身份 | candidate_identity + sha256 |
| 持久化边界 | 每个 rename 后 fsync 父目录 |
| 并发行为 | 双事务 CAS 串行 |
| 崩溃点 | 各阶段边界 |
| 恢复动作 | §3.8 矩阵，证据先于崩溃点生成 |
| 安全不变量 | I-05、I-06、I-07、I-08 |
| 拒绝条件 | 目标已存在/身份不匹配/摘要不匹配 → DENY |

**结论：无矛盾。** PREPARED 先持久化再发布，恢复证据完整。

### 链路 4：QUARANTINE_INTENT → qtmp → qfinal → COMPLETE → SOURCE_RETIRE_INTENT → 源处理

| 项 | 内容 |
|----|------|
| 前置条件 | quarantine_id 已生成 |
| 操作顺序 | Q1 intent → Q2 校验 → Q3 复制 → Q4 三方校验 → Q5 发布 → Q6 COMPLETE → 源处理 |
| 锁约束 | 授权锁 |
| 对象身份 | source_identity + source_sha256 |
| 持久化边界 | intent 先于 qfinal |
| 并发行为 | 源被并发写 → Q4/Q6 校验拦截 |
| 崩溃点 | 各阶段 |
| 恢复动作 | §4.8 矩阵 |
| 安全不变量 | I-09、I-10 |
| 拒绝条件 | 内容不一致 → DENY/INDETERMINATE |

**结论：无矛盾。** 双哈希三方校验阻断 H1→H2。

### 链路 5：DELETE_INTENT → TOMBSTONED → REMOVED_BY_THIS_CLEANUP → CLEANED

| 项 | 内容 |
|----|------|
| 前置条件 | 终态快照已持久化 |
| 操作顺序 | DELETE_INTENT → rename 到 tombstone → TOMBSTONED → REMOVED（tombstone 保留，不 unlink）|
| 锁约束 | 单清理者 |
| 对象身份 | tombstone 身份 + 哈希 |
| 持久化边界 | 每状态 fsync |
| 并发行为 | 外部删除 tombstone → INDETERMINATE |
| 崩溃点 | 各状态边界 |
| 恢复动作 | §5.8 三态表 |
| 安全不变量 | I-11、I-12 |
| 拒绝条件 | tombstone 消失且无归因 → INDETERMINATE |

**结论：无矛盾。** tombstone 不 unlink，删除归因有真实持久化证据。

### 链路 6：两事务竞争同一锁/Candidate/Journal/quarantine 目标/tombstone

| 项 | 内容 |
|----|------|
| 前置条件 | 授权锁 |
| 操作顺序 | 事务 A 先获锁，B 等待 |
| 锁约束 | 授权锁互斥 + 无覆盖原语 + CAS |
| 对象身份 | 各对象身份绑定 |
| 持久化边界 | — |
| 并发行为 | 后到者失败/幂等 |
| 崩溃点 | — |
| 恢复动作 | 后到者 DENY 或幂等跳过 |
| 安全不变量 | I-07 |
| 拒绝条件 | 目标已存在 → DENY |

**结论：无矛盾。**

### 链路 7：每个持久化边界发生进程崩溃/系统崩溃/磁盘失败/审计失败

| 项 | 内容 |
|----|------|
| 前置条件 | — |
| 操作顺序 | 各协议持久化边界 |
| 锁约束 | — |
| 对象身份 | — |
| 持久化边界 | 每 rename 后 fsync |
| 并发行为 | — |
| 崩溃点 | 每个持久化边界 |
| 恢复动作 | 各矩阵覆盖 |
| 安全不变量 | I-13 |
| 拒绝条件 | 证据不足 → DENY/INDETERMINATE |

**结论：无矛盾。**

### 链路 8：同用户其他进程绕过协议修改/替换/删除文件

| 项 | 内容 |
|----|------|
| 前置条件 | 外部进程不遵守协议 |
| 操作顺序 | 外部进程 unlink/replace/write |
| 锁约束 | 无法阻止 |
| 对象身份 | 身份/内容校验暴露 |
| 持久化边界 | — |
| 并发行为 | 破坏协议 |
| 崩溃点 | — |
| 恢复动作 | 身份/内容不一致 → DENY/INDETERMINATE |
| 安全不变量 | I-02、I-10、I-12 |
| 拒绝条件 | 身份/内容变化 → DENY/INDETERMINATE |

**结论：无矛盾。** 绕过协议的外部进程无法阻止，但身份+内容双校验能暴露并拒绝，不会伪造成功。

---

## 2. 交叉审查总结

| 审查项 | 结论 |
|--------|------|
| 八条链路崩溃点覆盖 | 全部覆盖 |
| 恢复动作唯一性 | 每崩溃点唯一动作或明确 DENY/INDETERMINATE |
| 不变量在崩溃/并发下成立 | 是 |
| 跨章节锁语义一致 | 是 |
| 跨章节矛盾 | 无 |

**交叉审查结论：PASS（设计级，无已知矛盾）。**

---

## 3. 保留的 NOT_VERIFIED / NOT_TESTED 项

- flock/OFD 锁、renameat2、MoveFileEx 平台语义：NOT_VERIFIED
- register_at_fork / prepare 快照 / child 重建：NOT_TESTED
- 同用户恶意进程、外部写入绕过协议：NOT_VERIFIED
- tombstone 保留 + 归档回收流程：NOT_VERIFIED
- Windows 目录持久化/断电：NOT_VERIFIED

**这些项不因设计级通过而改变状态。**
