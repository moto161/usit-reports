# EXECUTION-REPORT.md — 执行报告（05.36）

## A. 执行信息

| 项 | 值 |
|---|---|
| 工单编号 | USIT-STAGE-B-DESIGN-05.36 |
| 执行时间 | 2026-10-31（本地时间） |
| 工作区 | C:\Users\Administrator\WorkBuddy\2026-10-15-22-09-47\usit_p0（只读） |
| 交付目录 | C:\Users\Administrator\WorkBuddy\2026-10-15-22-09-47\usit_p0_05.36_reports |
| 执行模式 | READ-ONLY ANALYSIS + DESIGN REVISION |
| 基线 | DESIGN-05.35（commit `2a9e985`） |

## B. 基线与证据

- 05.35 基线 commit `2a9e985` 核验：✅（`git log` 确认 HEAD）
- 冻结三文件哈希核验：scanner.py / cli.py / file_understanding.py 与历史基线一致（未变）。
- 源码 25 个 `usit/*.py` 文件 pre 基线哈希与 05.35 post 基线 diff 为空（仅标题行 PRE/POST 标签不同，属预期）。
- **源码零修改：确认。**

## C. B36-01｜计数器掩码修复

- 根因：05.35 使用 `0x7FFE`（仅 14 位，遗漏 bit15）。
- 数学验证（本机 Python 3.12 实跑）：`bit1..15 mask = 0xFFFE`；`0x7FFE` bit15=False；`0xFFFE` bit15=True；count=16384 时 `0x7FFE` 提取 0（错误），`0xFFFE` 提取 16384（正确）。
- 修复：`COUNT_MASK=0xFFFE`。边界 0→1 / 16383→16384 / 32767 / 下溢 全部重算。
- 结论：**PASS**。

## D. B36-02｜DENIED 原子保护

- `cas_frozen()` 修正为同一 CAS 内检查 gate==CLOSED、DENIED==0、count==0。
- 6 条并发时间线（FROZEN/unfreeze/rollback/恢复 vs DENIED、双设 DENIED、CAS 重试）全部证明 DENIED 一旦提交不可清除。
- DENIED 位入状态字 bit17，无写回 0 路径。
- 结论：**PASS**。

## E. B36-03｜清理幂等

- 6 态清理状态机，`_cleanup_done` 仅在 CLEANUP_COMPLETED 置 True。
- 资源独立所有权 + 独立释放状态；部分失败重试；状态不明隔离。
- 结论：**PASS**。

## F. B36-04｜崩溃恢复

- 状态字四种作用域矩阵；进程退出≠共享计数器可归零。
- 旧写入者排除证明 + 事务恢复决策树 + 证据冲突隔离。
- 共享介质归零路径因锁外写入者 UNKNOWN → NOT_PROVEN，Fork DENY。
- 结论：**PASS**（含如实标注的 NOT_PROVEN 边界）。

## G. 反例统计

| 指标 | 值 |
|---|---|
| TOTAL | 30（本轮新增） |
| VALID | 30 |
| PASS | 28 |
| FAIL | 2（记录 05.35 历史缺陷） |
| NOT_PROVEN | 0 |
| NOT_TESTED_STATIC | 0 |
| NOT_TESTED_RUNTIME | 30 |

累计 ≥ 160 条（05.33=60 + 05.34=40 + 05.36=30）。已复核 05.35 `B35-MASK-001` 错误 PASS，修正为 FAIL（历史）+ 修复后 PASS。

## H. 跨文档一致性

- 关键定义（COUNT_MASK/DENIED/frozen/清理状态机/单一状态源）全部一致。
- 5 处 05.35 冲突/缺陷全部消除。
- 历史 NOT_PROVEN/UNKNOWN 在各文档一致标注，无矛盾。
- 结论：无未解释冲突。

## I. 基线保护与零修改证明

- 项目源码、配置、数据库、运行文件：**零修改**。
- 未执行真实 Fork、POSIX 事务、构建、部署、运行时测试。
- 05.35 交付物未覆盖（新目录 `DESIGN-05.36`）。

## J. 禁止事项确认

- [x] 未修改源码
- [x] 未修改项目配置
- [x] 未构建 / 未编译
- [x] 未测试（运行时）
- [x] 未执行真实 Fork
- [x] 未执行真实 POSIX 事务
- [x] 未执行 Source Retire
- [x] 未执行 A/B Merge
- [x] 未执行 IMPLEMENT-01
- [x] 未覆盖历史交付物

## K. 验收明细

| 验收项 | 结果 |
|---|---|
| 计数器掩码正确 | PASS |
| 边界计算无跨字段污染 | PASS |
| 非零计数不判零 | PASS |
| DENIED 不可清除 | PASS |
| FROZEN 不绕过 DENIED | PASS |
| 清理幂等/不误标完成 | PASS |
| 崩溃恢复不擅重置共享计数器 | PASS |
| UNKNOWN 事务隔离 | PASS |
| 反例 ≥160 + 复核历史错误反例 | PASS |
| 跨文档无矛盾 | PASS |
| 基线/源码不变 | PASS |
| SHA256 清单 | PASS |

## L. 最终结论

- B36-01：PASS
- B36-02：PASS
- B36-03：PASS
- B36-04：PASS

**整体结论：PASS-WITH-ISSUES**

ISSUES：身份防回滚、Receipt 认证、Break-glass 审计根、锁外写入者枚举四项历史 NOT_PROVEN/UNKNOWN 仍存在，对应功能禁用。

- IMPLEMENT-01：NOT_APPROVED
- A/B Merge：NOT_APPROVED
- Source Retire：DENY

**下一步：STOP，等待 ChatGPT 独立复审。**
