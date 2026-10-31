# CROSS-DOC-CONSISTENCY.md

## 目标
跨文档一致性检查。

## 1. 检查范围

05.34 全部交付文档 + 引用的 05.33 文档。

## 2. 冲突项及处理

| # | 冲突 | 处理 | 结果 |
|---|---|---|---|
| 1 | 05.33 `admit_count==0` 检查与 wait 分离 → 丢失唤醒 | 05.34 持锁重查 predicate + generation | 已消除 |
| 2 | 05.33 抽象 `pid=-1` 当 fork 失败 | 05.34 按 os.fork 真实 OSError 语义 | 已消除 |
| 3 | 05.33 原子字字段操作未禁止裸加 | 05.34 field-aware CAS | 已消除 |
| 4 | 05.33 锁外写入者未明确 DENY 规则 | 05.34 未覆盖入口 Fork DENY | 已消除 |

## 3. 冲突数量

- 发现冲突：4 项。
- 已处理：4 项。
- 未解释冲突：0。

## 4. 单一权威定义

| 主题 | 权威文档 |
|---|---|
| 等待协议 | FORK-FREEZE-PROTOCOL.md |
| 异常状态机 | FORK-EXCEPTION-STATE-MACHINE.md |
| 原子状态字 | ATOMIC-STATE-WORD.md |
| 写入入口覆盖 | WRITE-ENTRY-COVERAGE.md |

## 5. 安全约束继承检查

| 约束 | 继承 |
|---|---|
| 跨快照身份唯一性 NOT_PROVEN | ✅ |
| Receipt 创建者认证 NOT_PROVEN | ✅ |
| DENIED 不自动重开 | ✅ |
| Break-glass 审计根未证明则禁用 | ✅ |
| UNKNOWN/NOT_PROVEN 不包装为 PASS | ✅ |
| Source Retire DENY | ✅ |

## 6. 结论
跨文档冲突已消除，无未解释冲突。安全约束全部继承。

## 7. 证据等级
VERIFIED_STATIC；运行时 NOT_TESTED。
