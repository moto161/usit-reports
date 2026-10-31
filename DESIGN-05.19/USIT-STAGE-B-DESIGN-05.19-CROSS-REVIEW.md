# USIT-STAGE-B-DESIGN-05.19-CROSS-REVIEW｜独立交叉审查

> 阶段 B · DESIGN-05.19 · 独立于主设计的反例审查
> 生成：2026-10-31（GMT+8）
> 审查立场：主动寻找设计自身反例，不复述设计作者证明

---

## 审查方法与权威依据

1. 依据 **Microsoft Learn《ReplaceFileW function (winbase.h)》**（Last updated 2023-06-01）逐码核验 1175/1176/1177 的最终文件状态。
2. 依据 **chattr(1) 官方 man page**（e2fsprogs）核验 `+i` 属性对旧 FD 写与 rename 的实际语义（承接 05.18 结论）。
3. 依据 **CPython os.fork / os._fork / register_at_fork** 官方文档核验 fork 生命周期语义。
4. 本机为 Windows（`hasattr(os,'fork')` = False），**无法实际运行 fork 验证**，所有 fork 结论为设计级推演，标 NOT_TESTED。

---

## P0-02 反例审查（Fork 门禁）

| # | 反例 | 审查结论 |
|---|------|---------|
| C1 | FREEZING 态下，第三方 C 扩展仍可 open/close（不经 Gate）| **确认无法拦截**。child 仅对 generation 匹配的已知 FD 关闭，未知 FD 标 UNKNOWN_FD 不关闭，正确。但该环境必须 UNSUPPORTED/DENY |
| C2 | FREEZING 态下某线程已阻塞在 open 内部 | FREEZING 态先等待在途操作排空（§1.3 步骤 2c），排空后才生成快照，正确 |
| C3 | before 钩子只读快照，但 PyOS_BeforeFork 内部可能获取锁 | before 钩子本身不取锁，但 CPython 内部 PyOS_BeforeFork 会获取 GIL/import 锁——这是 CPython 自身行为，非本库锁，不构成本库锁逆序 |
| C4 | child 直接赋值全新锁对象，但 CPython 内部锁已由 AfterFork_Child 重置 | 一致，AfterFork_Child 重置 runtime 锁，本库锁由 child 重建，边界清晰 |
| C5 | 连续 fork 时 FREEZING 态残留 | §1.8 状态机：fork 后 parent 恢复 OPEN，child 置 child-local OPEN，无残留 |
| C6 | fork 失败时 FREEZING 态未恢复 | §1.8：parent FREEZING→OPEN→返回错误，幂等可重试 |

**P0-02 审查结论**：受控 fork 协议层反例均已闭合，但第三方 C fork/open 是不可控入口，必须 DENY。**未发现受控路径的未闭合反例。**

## P0-04 反例审查（CAS 持久化）

| # | 反例 | 审查结论 |
|---|------|---------|
| C7 | fsync 失败后仍提交后续事务 | 五态状态机 NOT_DURABLE 态禁提交，闭合 |
| C8 | rename 后目录项未持久，误判为成功 | COMMIT_INDETERMINATE 态，按 current 确认，闭合 |
| C9 | 不确定态自动回滚 | §2.4 明确禁止自动回滚，闭合 |
| C10 | 五条件笼统不列谓词 | §2.2 逐项 P1~P5，闭合 |
| C11 | Windows 目录持久化误当等价 POSIX | §2.7 明确 NOT_VERIFIED，闭合 |

**P0-04 审查结论**：POSIX 协议层闭合；Windows 持久性 NOT_VERIFIED（诚实标注）。

## P0-05 反例审查（保持 DENY）

| # | 反例 | 审查结论 |
|---|------|---------|
| C12 | 把 +i 当强制不可变 | chattr(1) 权威文档否决，DENY 正确 |
| C13 | 把权限/flock/哈希当强制不可变 | §4.2 明确仅协作语义，正确 |
| C14 | 过度断言"平台不存在任何机制" | §4.2 修正为"独立特权服务未验证"，正确 |

**P0-05 审查结论**：DENY 边界表述准确，无过度断言，无伪造 PASS。

## P0-06 反例审查（ReplaceFileW 错误码）

| # | 反例 | 审查结论 |
|---|------|---------|
| C15 | 从错误码推断文件存在性 | §3.3 明确"只能来自实际重新读取"，闭合 |
| C16 | 1176 未区分"指定备份/未指定备份" | §3.3 分两种情况，符合官方契约，闭合 |
| C17 | 1177 部分移动被盲目重试 | §3.3/§3.6 禁止盲目重试，INDETERMINATE，闭合 |
| C18 | 占位文件被误认有效 tombstone | §3.4 PLACEHOLDER ≠ PUBLISHED，闭合 |
| C19 | FlushFileBuffers 目录句柄误当 fsync(dirfd) | §3.5 明确 NOT_VERIFIED，闭合 |

**P0-06 审查结论**：1175/1176/1177 依据官方文档逐码闭合，部分移动场景 INDETERMINATE 判定正确。

---

## 交叉审查总结

- 本轮 4 项 P0 的协议层反例均已闭合或明确 DENY。
- **未闭合项**：第三方 C fork（UNSUPPORTED/DENY）、CPython 多线程 fork 运行时（NOT_TESTED）、Windows 目录持久化（NOT_VERIFIED）、Source Retire（DENY）、恶意同 UID（NOT_VERIFIED）。均诚实标注，无伪造 PASS。
- 整体结论与主设计 §8 一致：**PASS-WITH-ISSUES**。

---