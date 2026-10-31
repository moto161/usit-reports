# USIT-STAGE-B-DESIGN-05.36.md — 主设计文档

状态字边界、DENIED 原子保护与清理恢复专项修订

**执行模式**：严格 READ-ONLY ANALYSIS + DESIGN REVISION
**基线版本**：DESIGN-05.35（commit `2a9e985`）
**交付目录**：`DESIGN-05.36`
**最终结论**：PASS-WITH-ISSUES

---

## 1. 目标与范围

定点修复 05.35 独立复审发现的四项问题，不扩大范围，不进入实现：

| 编号 | 05.35 问题 | 05.36 结论 |
|---|---|---|
| B36-01 | 计数器掩码 `0x7FFE` 遗漏 bit15（15 位 mask 应为 0xFFFE） | **PASS（已修复）** |
| B36-02 | `cas_frozen()` 未在原子转换中完整检查 DENIED | **PASS（已修复）** |
| B36-03 | `_cleanup_done` 提前置位导致资源泄漏、不可重试 | **PASS（已修复）** |
| B36-04 | 崩溃恢复计数器重置「随进程消失即归零」未经证明 | **PASS（已闭合，含 NOT_PROVEN 边界）** |

---

## 2. B36-01｜计数器掩码修复

### 根因

05.35 全文档将 bit1–15 的 15 位计数器掩码定义为 `0x7FFE`，遗漏 bit15。

```
bit1..15 全置 1 的掩码 = 0xFFFE（= 65534）
0x7FFE = 0b0111_1111_1111_1110  → 仅 bit1..14（14 位），bit15 = 0
0xFFFE = 0b1111_1111_1111_1110  → bit1..15（完整 15 位）
```

### 错误触发

真实计数 = 16384 时，存储字段 = `16384 << 1 = 0x8000`：
- `0x7FFE` 提取 → 0（**非零计数误判为零**）
- `0xFFFE` 提取 → 16384（正确）

后果：count ∈ [16384, 32767] 时准入排空被误判，Fork 可能在业务未停止时进行。

### 修复

```python
GATE_MASK   = 0x00001
COUNT_MASK  = 0x0FFFE   # bit1..15 全 15 位（05.36 修正）
COUNT_SHIFT = 1
COUNT_MAX   = 32767
FROZEN_MASK = 0x10000
DENIED_MASK = 0x20000
```

边界验证（十六进制 + 二进制）与 field-aware CAS 见 `ATOMIC-STATE-WORD.md`。修复后任何非零准入计数都不可能被提取为 0。

---

## 3. B36-02｜DENIED 与 FROZEN 原子保护

05.35 的 `cas_frozen()` 未在同一原子转换中检查 DENIED。05.36 修正：`cas_frozen()` 在**同一 CAS 的 expected 状态**上原子检查 `gate==CLOSED`、`DENIED==0`、`count==0` 三项。

并发时间线（FROZEN/unfreeze/rollback/恢复 vs DENIED、双线程设 DENIED、CAS 重试）逐一证明：只要 DENIED 已提交，后续普通操作不能恢复 OPEN 或清除 DENIED。

DENIED 位（bit17）入状态字，全系统不存在将 DENIED 写回 0 的操作路径。详见 `DENIED-ATOMIC-PROTECTION.md` 与 `STATE-TRANSITION-MATRIX.md` §3。

---

## 4. B36-03｜清理幂等与部分失败恢复

05.35 的 `_cleanup_done` 在资源释放前即置 True，释放失败后第二次调用直接返回，造成泄漏。

05.36 建立 6 态清理状态机（NOT_STARTED / IN_PROGRESS / PARTIAL / COMPLETED / FAILED / ISOLATED），核心不变量：
- `_cleanup_done` 仅在 `CLEANUP_COMPLETED` 置 True。
- 每项资源独立所有权 + 独立释放状态。
- 部分失败进入 PARTIAL 允许重试剩余；状态不明资源进入 ISOLATED 停止重试。

详见 `IDEMPOTENT-CLEANUP-PROTOCOL.md`。

---

## 5. B36-04｜崩溃恢复与计数器重置安全证明

状态字存在四种作用域（进程内单线程/多线程/Fork 副本/跨进程共享），进程退出后状态是否消失取决于存储介质，而非「进程是否退出」。

**强制规则**：不得仅凭某进程退出就推定共享计数器可归零。归零需满足「所有旧写入者已排除」+「无 UNKNOWN 事务」。跨进程共享介质场景下，因锁外写入者无法完备枚举（UNKNOWN），归零路径 NOT_PROVEN，Fork 保持 DENY。

事务恢复决策树：COMMITTED_CONFIRMED 安全；COMMITTED_UNCONFIRMED/IN_FLIGHT/UNKNOWN 隔离；QUARANTINED 保持隔离。详见 `CRASH-RECOVERY-PROTOCOL.md`。

---

## 6. 反例验证

本轮新增 30 条反例（MASK_FIELD=8 / DENIED_ATOMIC=8 / CLEANUP=7 / CRASH_RECOVERY=7），累计 ≥ 160 条。统计：VALID=30 / PASS=28 / FAIL=2（记录 05.35 历史缺陷）/ NOT_PROVEN=0 / NOT_TESTED_RUNTIME=30。

已重新复核 05.35 的 `B35-MASK-001` 错误 PASS 结论，修正为 FAIL（历史）+ 修复后 PASS。详见 `COUNTEREXAMPLE-RESULTS.md`。

---

## 7. 继承的安全约束（未弱化）

- 单一准入状态源（B35-01）继续继承。
- 跨快照身份唯一性：NOT_PROVEN，功能禁用。
- Receipt 创建者认证：NOT_PROVEN，事务恢复/幂等确认禁用。
- 未认证 Receipt：QUARANTINED。
- Break-glass 审计信任根：NOT_PROVEN，紧急恢复禁用。
- 锁外写入者：UNKNOWN，Fork 保持 DENY。
- DENIED 不可逆，普通路径不可清除。
- Source Retire：DENY。

---

## 8. 验收结果

| 验收项 | 结论 |
|---|---|
| 计数器掩码与字段定义完全正确 | PASS |
| 边界计算无跨字段污染 | PASS |
| 非零准入计数不可能判为零 | PASS |
| DENIED 普通并发路径不可清除 | PASS |
| FROZEN 转换不绕过 DENIED | PASS |
| 清理部分失败可安全重试、不误标完成 | PASS |
| 崩溃恢复不得未经证明重置共享计数器 | PASS |
| UNKNOWN 事务/旧写入者保持隔离 | PASS |
| 反例 ≥ 160 且复核 05.35 错误反例 | PASS |
| 跨文档无关键矛盾 | PASS |
| 05.35 基线与源码保持不变 | PASS |
| 全部交付文件 SHA256 | PASS |

**整体结论：PASS-WITH-ISSUES**

`PASS-WITH-ISSUES` 的 ISSUES 指：身份防回滚、Receipt 认证、Break-glass 审计根、锁外写入者枚举四项历史 NOT_PROVEN/UNKNOWN 仍存在，对应功能保持禁用。这些不构成本轮的协议缺陷，但阻止整体 PASS。

---

## 9. 交付文件清单

见 `HASH-MANIFEST.txt`。共 12 份交付文档 + 反例 JSON。

---

## 10. 最终状态

- IMPLEMENT-01：NOT_APPROVED
- A/B Merge：NOT_APPROVED
- Source Retire：DENY

**下一步：STOP，等待 ChatGPT 独立复审。**
