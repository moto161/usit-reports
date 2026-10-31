# CROSS-DOC-CONSISTENCY.md

## 目标
跨文档一致性审查。

## 1. 检查范围

05.35 全部交付文档 + 引用的 05.34/05.33 文档。

## 2. 冲突项及处理

| # | 冲突 | 处理 | 结果 |
|---|---|---|---|
| 1 | 05.34 双状态源 `_admit_state` + `_admit_count` | 05.35 统一单一原子状态字 | 已消除 |
| 2 | 05.34 15 位掩码可能遗漏最高位 | 05.35 明确 COUNT_MASK=0x7FFE 覆盖 bit1–15 | 已消除 |
| 3 | 05.34 DENIED 未放入状态字/保护不足 | 05.35 DENIED 位（bit17）入状态字 + 不可逆保护 | 已消除 |
| 4 | 05.34 崩溃恢复 finally 覆盖不足 | 05.35 崩溃恢复协议（磁盘证据） | 已消除 |

## 3. 冲突数量

- 发现冲突：4 项。
- 已处理：4 项。
- 未解释冲突：0。

## 4. 关键一致性检查结果

| 检查项 | 结果 |
|---|---|
| 准入状态唯一权威来源 | ✅（单一 `_admit_state`） |
| 位布局/掩码/伪代码一致 | ✅（COUNT_MASK=0x7FFE 统一） |
| DENIED 各文档不可逆 | ✅（状态字 DENIED 位不可清） |
| 异常路径终态与资源所有权一致 | ✅（异常状态机 + 崩溃恢复） |
| 反例结论由伪代码支持 | ✅ |

## 5. 安全约束继承

| 约束 | 继承 |
|---|---|
| 跨快照身份唯一性 NOT_PROVEN | ✅ |
| Receipt 认证 NOT_PROVEN | ✅ |
| DENIED 不自动重开 | ✅ |
| Break-glass 无可信根则禁用 | ✅ |
| 锁外写入者 UNKNOWN → Fork DENY | ✅ |
| Commit/Receipt 入口未全覆盖 → Fork DENY | ✅ |
| Source Retire DENY / IMPLEMENT-01 NOT_APPROVED | ✅ |

## 6. 结论
跨文档无关键冲突。安全约束全部继承。

## 7. 证据等级
VERIFIED_STATIC；运行时 NOT_TESTED。
