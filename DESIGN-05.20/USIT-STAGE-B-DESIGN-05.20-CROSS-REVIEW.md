# USIT-STAGE-B-DESIGN-05.20-CROSS-REVIEW｜执行器自审

> 阶段 B · DESIGN-05.20 · 执行器自审（非独立复审）
> 生成：2026-10-31（GMT+8）
> 立场：主动寻找设计自身反例，不复述设计作者证明

---

## 审查方法与权威依据

1. 依据 **Microsoft Learn《ReplaceFileW function (winbase.h)》**（Last updated 2023-06-01）核验 1175/1176/1177 最终文件状态。
2. 依据 **chattr(1) 官方 man page** 核验 `+i` 属性语义（承接 05.18）。
3. 依据 **CPython fork 文档** 核验 fork 生命周期。
4. 本机 Windows（`hasattr(os,'fork')` = False），fork 结论为设计级推演，标 NOT_TESTED。

---

## P0-02 反例审查（准入原子性）

| # | 反例 | 审查结论 |
|---|------|---------|
| C1 | 检查 Gate 后、计数前 FREEZING 介入 | §1.2 合并为同一临界区，消除 |
| C2 | in_flight 计数泄漏 | §1.3 RAII/finally 保证配对 |
| C3 | freeze 异常未恢复 OPEN | §1.3 catch 恢复 + signal_all |
| C4 | 第三方 C 扩展绕过 Gate open | §1.6 无法拦截，child 标 UNKNOWN_FD 不关闭 |
| C5 | 第三方 C 扩展直接 fork | §1.5 DENY/UNSUPPORTED |
| C6 | 线程取消打断计数 | §1.3 禁用取消临界区/RAII |

**审查结论**：受控路径的准入原子性反例已闭合；第三方 C 不可控入口明确 DENY。未发现受控路径未闭合反例。

## P0-04 反例审查（状态机）

| # | 反例 | 审查结论 |
|---|------|---------|
| C7 | 回执未持久误当不确定 | §2.3 拆出 COMMIT_DURABLE_AUDIT_PENDING，闭合 |
| C8 | rename 后目录未持久误当成功 | §2.6 COMMIT_INDETERMINATE，闭合 |
| C9 | 不确定态自动回滚 | §2.3 禁止，闭合 |
| C10 | 步骤编号不一致 | §2.5 统一 S1~S16，闭合 |
| C11 | 仅凭内存状态判断提交 | §2.7 强制重读磁盘证据，闭合 |

**审查结论**：POSIX 状态机反例闭合；Windows 持久性 NOT_VERIFIED（诚实标注）。

## P0-05 反例审查（保持 DENY）

| # | 反例 | 审查结论 |
|---|------|---------|
| C12 | 把 +i 当强制不可变 | chattr(1) 否决，DENY 正确 |
| C13 | 把权限/flock/哈希当强制不可变 | §4.2 五维度区分，正确 |
| C14 | 维度混用 | §4.2 严格区分，正确 |

**审查结论**：DENY 边界准确，无伪造 PASS。

## P0-06 反例审查（1176 纠正）

| # | 反例 | 审查结论 |
|---|------|---------|
| C15 | 1176 替换文件路径写错 | §3.3 依据官方原文纠正为"在其原始名"，闭合 |
| C16 | 从 FILE_NOT_FOUND 推断替换文件缺失 | §3.4 禁止推断，闭合 |
| C17 | 从 SHARING_VIOLATION 推断两文件都在 | §3.4 禁止推断，闭合 |
| C18 | 文件 ID 与内容哈希混用 | §3.5 三者正交，闭合 |
| C19 | 占位误认 PUBLISHED | §3.6 PLACEHOLDER ≠ PUBLISHED，闭合 |

**审查结论**：1176 路径语义纠正正确，错误码推断已禁止。

---

## 交叉审查总结

- 本轮 4 项 P0 协议层反例已闭合或明确 DENY。
- 未闭合项：第三方 C fork（UNSUPPORTED/DENY）、多线程 fork 运行时（NOT_TESTED）、Windows 目录持久化（NOT_VERIFIED）、Source Retire（DENY）、恶意同 UID（NOT_VERIFIED）。均诚实标注。
- **本 CROSS-REVIEW 是执行器自审，不是独立复审**，最终结论以 ChatGPT 独立复审为准。
- 整体结论与主设计 §8 一致：**PASS-WITH-ISSUES**。

---