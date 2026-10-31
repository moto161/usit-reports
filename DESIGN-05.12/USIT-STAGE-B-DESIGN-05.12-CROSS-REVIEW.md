# USIT-STAGE-B-DESIGN-05.12｜端到端交叉审查记录（CROSS-REVIEW）

> 阶段 B · DESIGN-05.12 · 静态交叉审查
> 对象：主文档六项 P0 协议 + 12 条设计不变量
> 方法：逐链路推演崩溃点、并发竞争、异常退出、恢复重入

---

## 1. 审查范围

对以下七条链路做端到端静态推演，每条链路给出：前置条件、操作顺序、持久化边界、并发约束、崩溃点、恢复动作、安全不变量、无法证明时的拒绝动作。

---

## 2. 链路 1：锁获取 → Lease 创建 → fork → 父子退出 → 锁释放

| 项 | 内容 |
|----|------|
| 前置条件 | 锁文件目录可信（0o700，属主授权用户）|
| 操作顺序 | acquire_lock（openat + fstat + flock/OFD）→ register → 构建 Lease → 使用 → release |
| 持久化边界 | 无（锁是运行时资源，不持久化）|
| 并发约束 | ForkGate 串行化 fork 与 acquire/release |
| 崩溃点 | fork 后子进程继承 fd |
| 恢复动作 | after_in_child 重置：作废 Lease → close fd（不 LOCK_UN）→ 重建 ProcessMutex |
| 安全不变量 | I-03（子进程不误操作父锁）|
| 拒绝动作 | 平台无锁原语 → DENY |

**结论：无矛盾。** 关键修复点（子进程只 close 不 LOCK_UN）在 §1.4/§2.6 均有依据。

---

## 3. 链路 2：锁身份验证 → Candidate 准备 → PREPARED → 无覆盖发布 → revision 更新

| 项 | 内容 |
|----|------|
| 前置条件 | 目录句柄已打开，锁已持有 |
| 操作顺序 | §4.3 POSIX 或 §4.4 Windows：tmp 完整写入 → fsync → 校验 → 无覆盖发布 → fsync 目录 → 复核 |
| 持久化边界 | 每个 rename 后 fsync 父目录 |
| 并发约束 | 授权锁保护，无覆盖原语拒绝并发发布 |
| 崩溃点 | 发布中、发布后 COMMITTED 前 |
| 恢复动作 | §4.7 矩阵，全量身份校验 |
| 安全不变量 | I-02/I-04/I-05 |
| 拒绝动作 | 目标已存在/身份不匹配/摘要不匹配 → DENY |

**结论：无矛盾。** 半成品暴露已通过"先 tmp 后发布"消除（I-04）。

---

## 4. 链路 3：Journal 意图持久化 → qtmp → qfinal → 源对象处理 → 审计完成

| 项 | 内容 |
|----|------|
| 前置条件 | quarantine_id 已生成 |
| 操作顺序 | §5.3：intent → copy → 校验 → 发布 qfinal → COMPLETE → 源处理 |
| 持久化边界 | intent 先于 qfinal；每个 rename 后 fsync |
| 并发约束 | 授权锁保护 |
| 崩溃点 | 各阶段 |
| 恢复动作 | §5.7 矩阵，intent 证明归属 |
| 安全不变量 | I-07 |
| 拒绝动作 | 无 intent 的孤立 qfinal → INDETERMINATE，不覆盖不删除 |

**结论：无矛盾。** 顺序错误（先 qfinal 后审计）已通过"intent 先于 qfinal"消除。

---

## 5. 链路 4：DELETE_INTENT → TOMBSTONED → UNLINK_INTENT → unlink → REMOVED → CLEANED

| 项 | 内容 |
|----|------|
| 前置条件 | 终态快照已持久化 |
| 操作顺序 | §6.4：DELETE_INTENT → rename → TOMBSTONED → UNLINK_INTENT → unlink → REMOVED |
| 持久化边界 | 每状态 fsync |
| 并发约束 | 单清理者（授权锁）|
| 崩溃点 | unlink 后 REMOVED 前（关键）|
| 恢复动作 | UNLINK_INTENT 已持久化 → 补写 REMOVED（§6.5）|
| 安全不变量 | I-08/I-09 |
| 拒绝动作 | 无 UNLINK_INTENT 的 tombstone 消失 → INDETERMINATE |

**结论：无矛盾。** 05.11 的状态矛盾（tombstone 要求存在 vs 实际 unlink）通过新增 UNLINK_INTENT 状态消除。

---

## 6. 链路 5：任意阶段崩溃 → 重启 → 恢复识别 → 归因 → 安全继续或拒绝

| 项 | 内容 |
|----|------|
| 前置条件 | 持久化证据完整 |
| 操作顺序 | 恢复程序重读 journal/intent/快照 |
| 持久化边界 | — |
| 并发约束 | 恢复也需 acquire 授权锁 |
| 崩溃点 | 恢复过程中再次崩溃 |
| 恢复动作 | 幂等重判（各矩阵均覆盖）|
| 安全不变量 | I-06/I-10 |
| 拒绝动作 | 证据不足 → DENY/INDETERMINATE |

**结论：无矛盾。** 所有恢复矩阵均保证幂等重入。

---

## 7. 链路 6：两个并发事务竞争同一对象 / 同一 Candidate 目标 / 同一 quarantine 目标

| 项 | 内容 |
|----|------|
| 前置条件 | 授权锁 |
| 操作顺序 | 事务 A 先获锁，事务 B 等待 |
| 持久化边界 | — |
| 并发约束 | 授权锁互斥 + 无覆盖原语 |
| 崩溃点 | — |
| 恢复动作 | 后到者失败或幂等跳过 |
| 安全不变量 | I-05 |
| 拒绝动作 | 目标已存在 → DENY |

**结论：无矛盾。** 授权锁 + 无覆盖原语双重保证。

---

## 8. 链路 7：路径被替换 / 目标已存在 / 摘要不一致 / 审计存储不可用

| 项 | 内容 |
|----|------|
| 前置条件 | — |
| 操作顺序 | 身份校验失败 → 拒绝 |
| 持久化边界 | — |
| 并发约束 | — |
| 崩溃点 | — |
| 恢复动作 | DENY/INDETERMINATE |
| 安全不变量 | I-02/I-06 |
| 拒绝动作 | 身份不匹配 → DENY；审计不可用 → DENY |

**结论：无矛盾。** 全部异常路径都有明确拒绝动作。

---

## 9. 交叉审查总结

| 审查项 | 结论 |
|--------|------|
| 七条链路崩溃点覆盖 | 全部覆盖 |
| 恢复动作唯一性 | 每崩溃点唯一动作或明确 DENY |
| 不变量在崩溃/并发下成立 | 是 |
| 跨章节锁语义一致 | 是（P0-02~06 复用 P0-01 锁接口）|
| 跨章节矛盾 | 无 |

**交叉审查结论：PASS（设计级，无已知矛盾）。**

---

## 10. 保留的 NOT_VERIFIED / NOT_TESTED 项

- flock/OFD 锁平台语义：NOT_VERIFIED
- register_at_fork / ForkGate：NOT_TESTED
- renameat2 / MoveFileEx 无覆盖语义：NOT_VERIFIED
- Windows 目录持久化 / 断电：NOT_VERIFIED
- tombstone 并发/外部删除：NOT_VERIFIED

**这些项不因设计级通过而改变状态。**
