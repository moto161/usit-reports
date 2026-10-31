# USIT-STAGE-B-DESIGN-05.21-CROSS-REVIEW｜执行器自审

> 阶段 B · DESIGN-05.21 · 执行器自审（非独立复审）
> 生成：2026-10-31（GMT+8）
> 立场：主动寻找设计自身反例，不复述设计作者证明

---

## 审查方法与权威依据

1. 依据 Microsoft Learn《ReplaceFileW function (winbase.h)》核验 1175/1176/1177。
2. 依据 chattr(1) 官方 man page 核验 `+i` 语义。
3. 依据 CPython fork 文档核验 fork 生命周期。
4. 本机 Windows（`hasattr(os,'fork')`=False），fork 结论为设计级推演，标 NOT_TESTED。

---

## P0-02 反例审查（锁顺序 + fork 职责）

| # | 反例 | 审查结论 |
|---|------|---------|
| C1 | 锁级别编号与获取顺序矛盾（05.20）| §1.2 重排 L0=coord/L1=gate/L2=fd，闭合 |
| C2 | fork 入口先拿高级锁再拿低级锁 | 重排后 L0→L1→L2 升序，无逆序，闭合 |
| C3 | freeze 自行 fork 导致递归 | §1.4 freeze 明确不 fork，闭合 |
| C4 | before 递归触发 fork | §1.4 before 只读无锁不 fork，闭合 |
| C5 | Gate 永久 FREEZING | §1.4 after_in_parent 恢复 OPEN + fork 失败 except 恢复，闭合 |
| C6 | child 继承锁死锁 | §1.4 child 直接赋值全新锁，闭合 |
| C7 | 父子 Registry 交叉污染 | §1.4 child 只读 snapshot 副本，闭合 |
| C8 | 第三方 C fork 绕过 | §1.7 DENY/UNSUPPORTED，闭合（诚实拒绝）|

**审查结论**：锁顺序矛盾与 fork 职责分离反例已闭合；第三方 C fork 明确 DENY。未发现受控路径未闭合反例。

## P0-04 反例审查（状态迁移 + 崩溃矩阵）

| # | 反例 | 审查结论 |
|---|------|---------|
| C9 | NOT_DURABLE 重试成功直接跳 DURABLE（05.20）| §2.4 按步骤恢复，禁止跳步，闭合 |
| C10 | current 持久回执未持久当 DURABLE | §2.3 保持 COMMIT_DURABLE_AUDIT_PENDING，闭合 |
| C11 | S12 rename 中崩溃盲目重试 | §2.5 重读 current 身份，闭合 |
| C12 | 崩溃矩阵未覆盖"开始前/执行中/完成后" | §2.5 7 类崩溃位置全覆盖，闭合 |
| C13 | 未定义状态迁移 | §2.3 完整迁移表，禁止跳步，闭合 |
| C14 | 未恢复即启动新事务 | §2.4 禁止，闭合 |

**审查结论**：状态迁移与崩溃矩阵反例闭合；Windows 持久性 NOT_VERIFIED（诚实标注）。

## P0-05/P0-06 反例审查

| # | 反例 | 审查结论 |
|---|------|---------|
| C15 | 把 +i 当强制不可变 | DENY 正确 |
| C16 | 1176 路径语义写错 | 保留 05.20 纠正（"在其原名"），正确 |
| C17 | 从错误码推断文件状态 | 保留禁止推断，正确 |

**审查结论**：P0-05 DENY 边界准确，P0-06 纠正保留正确。

---

## 交叉审查总结

- 本轮 P0-02 锁顺序矛盾、P0-04 状态迁移错误均已闭合。
- 未闭合项：第三方 C fork（UNSUPPORTED/DENY）、多线程 fork 运行时（NOT_TESTED）、Windows 目录持久化（NOT_VERIFIED）、Source Retire（DENY）、恶意同 UID（NOT_VERIFIED）。均诚实标注。
- **本 CROSS-REVIEW 是执行器自审，不是独立复审**，最终结论以 ChatGPT 独立复审为准。
- 整体结论与主设计 §7 一致：**PASS-WITH-ISSUES**。

---