# USIT-STAGE-B-DESIGN-05.20-EXECUTION-REPORT｜真实执行报告

> 阶段 B · DESIGN-05.20 · 实际执行报告
> 生成：2026-10-31（GMT+8）

---

## 1. 执行环境

| 项 | 值 |
|----|----|
| 工单编号 | USIT-STAGE-B-DESIGN-05.20 |
| 执行时间 | 2026-10-31 12:06~12:15（GMT+8）|
| 执行模式 | READ-ONLY DESIGN REVISION ONLY |
| 工作目录 | `C:\Users\Administrator\WorkBuddy\2026-10-15-22-09-47\usit_p0` |
| 输出目录 | `C:\Users\Administrator\WorkBuddy\2026-10-15-22-09-47\usit_p0_05.20_reports`（项目外）|
| 环境 | Windows · Python 3.12.15 · `hasattr(os,'fork')` = False |
| 权威来源 | Microsoft Learn ReplaceFileW；chattr(1) man page；CPython fork 文档 |

## 2. 实际读取的历史文件清单

| 文件 | 读取状态 |
|------|---------|
| DESIGN-05.19 主文档 | 已读取（grep 核验 1176 表述）|
| DESIGN-05.19 交付物清单 | 已读取 |
| Microsoft Learn ReplaceFileW 文档 | 已通过 FetchURL 重新读取（Last updated 2023-06-01）|
| DESIGN-05.11~05.18 | NOT_AVAILABLE（承接 05.19 已核验结论，本轮仅需 05.19 作前置）|

## 3. 权威文档清单与链接

| 文档 | 链接 | 支持结论 |
|------|------|---------|
| ReplaceFileW function (winbase.h) | https://learn.microsoft.com/en-us/windows/win32/api/winbase/nf-winbase-replacefilew | 1175/1176/1177 最终文件状态 |
| chattr(1) man page | man7.org（e2fsprogs）| +i 不阻旧 FD 写、阻止 rename |

## 4. 各 P0 修订与结论

| 项 | 修订内容 | 结论 |
|----|---------|------|
| P0-02 | 原子准入（检查+计数同临界区）、线性化点定义、RAII 计数配对、freeze 异常恢复 | DESIGN_PARTIAL（协议层 DESIGN_PROVEN）|
| P0-04 POSIX | 六态状态机（拆出 COMMIT_DURABLE_AUDIT_PENDING）、16 步统一编号、五谓词逐项 + TOCTOU | DESIGN_PROVEN |
| P0-04 Windows | 保留 NOT_VERIFIED | NOT_VERIFIED |
| P0-05 | 五维度严格区分，保持 DENY | DENY |
| P0-06 | 1176 路径语义纠正、错误码推断禁止、身份三要素正交 | DESIGN_PARTIAL / NOT_VERIFIED |

## 5. 不变量 / 链路 / 时间线 / 反例数量

| 项 | 数量 | 达标 |
|----|------|------|
| 编号不变量 | 45 条 | ≥45 ✅ |
| 端到端链路 | 20 条 | ≥20 ✅ |
| 时间线 | 60 条 | ≥40 ✅ |
| 反例 | 45 个 | ≥30 ✅ |

## 6. ISSUES 逐项状态

| 编号 | 状态 |
|------|------|
| ISS-01 第三方 C FD/fork | 未关闭（UNSUPPORTED/DENY）|
| ISS-02 多线程 fork 运行时 | 未关闭（NOT_TESTED）|
| ISS-03 恶意同 UID | 未关闭（NOT_VERIFIED）|
| ISS-04 Windows 目录持久化 | 未关闭（NOT_VERIFIED）|
| ISS-05 Source Retire | 未关闭（DENY）|
| ISS-06 ReplaceFileW 中间态 | 未关闭（NOT_VERIFIED）|
| ISS-07 嵌入/子解释器 fork | 未关闭（DENY）|

## 7. 文件变更清单

| 文件 | 变更 | 状态 |
|------|------|------|
| 8 份 05.20 交付文件 | 新建于项目外 `usit_p0_05.20_reports` | 新增 |
| USIT 源码/配置/历史交付物 | 未触碰 | 无变更 |

## 8. 源码未修改确认

| 项 | 状态 | 确认方法 |
|----|------|---------|
| 源码修改 | **NO** | 仅项目外创建 05.20 文件；冻结三文件哈希核验（见下）|

冻结源码哈希（与 05.19 冻结基线一致，未变）：
- `usit/scanner.py` = 58ba5f25483429ce7482ae1d852deabce6ea0490f2e106c6955c6ff5323b45e3
- `usit/cli.py` = 879a3735838e183f41dd08054095b514ffc0e81e2b43776b75a77e25d138b8d9
- `usit/file_understanding.py` = 71995a09da04427c028282cfeba17c6b12ebdbcae97796adc2998af7c455ce5b

## 9. 执行状态（强制字段）

| 项 | 状态 |
|----|------|
| 构建 | NOT_EXECUTED |
| 运行时测试 | NOT_EXECUTED |
| 故障注入 | NOT_EXECUTED |
| IMPLEMENT-01 | NOT_EXECUTED / NOT_APPROVED |
| 真实隔离/删除/归档/清理 | NOT_EXECUTED |
| 解除阻断 | 未解除 |
| 合并 A/B | 未合并 |

## 10. 交付文件 SHA-256 与字节数

| 文件 | 字节数 | SHA-256 |
|------|--------|---------|
| USIT-STAGE-B-DESIGN-05.20.md | 54330 | c3f5a6d8c4e6c17bd87ca43cd59c60e9d9a1f8f7db97eebd5c963d2ddd6922b2 |
| USIT-STAGE-B-DESIGN-05.20-ISSUES.md | 2937 | 71e04369cde32ae91f11ed438785d60f6808be53a1250f13847819d265357db9 |
| USIT-STAGE-B-DESIGN-05.20-DIFF.md | 2689 | d064d21992376380a4cd3303b66a1717c340150f8c8775ff42592d4749381416 |
| USIT-STAGE-B-DESIGN-05.20-CROSS-REVIEW.md | 3396 | 2a1124ed597a437a3e228e4dec3524f01f1c2e00b084181450bf7006351cd818 |
| USIT-STAGE-B-DESIGN-05.20-EXECUTION-REPORT.md | （见 MANIFEST）| （见 MANIFEST）|
| USIT-STAGE-B-DESIGN-05.20-HASH-MANIFEST.md | （见 RECEIPT）| （见 RECEIPT）|
| USIT-STAGE-B-DESIGN-05.20-HASH-RECEIPT.md | （见 RECEIPT）| （见 RECEIPT）|

## 11. 最终结论

**DESIGN-05.20 = PASS-WITH-ISSUES**

分项：
- P0-02：DESIGN_PARTIAL（协议层 DESIGN_PROVEN，运行时 NOT_TESTED，第三方 C UNSUPPORTED/DENY）
- P0-04 POSIX：DESIGN_PROVEN
- P0-04 Windows：NOT_VERIFIED
- P0-05：DENY（保持）
- P0-06：DESIGN_PARTIAL / NOT_VERIFIED
- 交付完整性：PASS（待 ZIP 核验后确认）
- IMPLEMENT-01：**NOT_APPROVED**

无伪造 PASS，无降低标准，无删除反例。等待 ChatGPT 独立复审。

---