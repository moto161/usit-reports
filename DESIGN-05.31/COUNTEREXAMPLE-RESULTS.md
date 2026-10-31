# COUNTEREXAMPLE-RESULTS.md

## 目标
反例验证结果。机器可读数据见 COUNTEREXAMPLES.json（150 条）。

## 1. 总数统计（实际来自 JSON）

| 类别 | 数量 | 要求 | 达标 |
|---|---|---|---|
| Fork 父子路径反例 | 20 | 20 | ✅ |
| 锁顺序及死锁反例 | 20 | 20 | ✅ |
| 快照回滚与身份复用反例 | 30 | 30 | ✅ |
| Receipt 伪造/复制/重放/认证失败反例 | 30 | 30 | ✅ |
| DENIED 及紧急恢复反例 | 20 | 20 | ✅ |
| 跨文档协议冲突反例 | 30 | 30 | ✅ |
| **合计** | **150** | 150 | ✅ |

## 2. 结论分布

| 结论 | 数量 | 说明 |
|---|---|---|
| PASS | 90 | 协议层闭合（Fork 隔离、锁顺序、DENIED、跨文档） |
| NOT_PROVEN | 60 | 身份防回滚（30）+ Receipt 认证（30）如实未证明 |
| FAIL | 0 | 无 |
| 合计 | 150 | |

## 3. 关键反例触发与拒绝/降级说明

### Fork 父子路径（PASS）
- 触发：pid==0 分支执行子初始化。
- 拒绝：父进程 `_parent_path` 不引用任何 `_child_*`/`_rebuild_*`。
- 证据：FORK-PARENT-CHILD-ISOLATION.md 状态转换表 + I-FPC-1 证明。

### 锁顺序（PASS）
- 触发：coord 与 admit/commit 嵌套。
- 拒绝：权威锁顺序表禁止嵌套。
- 证据：FORK-LOCK-ORDER-UNIFIED.md。

### 快照回滚（NOT_PROVEN）
- 触发：恢复旧快照回退 local_fork_sequence。
- 降级：无可信防回滚锚点 → 功能禁用，不冒充证明。
- 证据：IDENTITY-ANTI-ROLLBACK.md 四方案分析。

### Receipt 认证（NOT_PROVEN）
- 触发：伪造/复制/重放/认证失败。
- 降级：QUARANTINED，不判成功，功能禁用。
- 证据：RECEIPT-AUTH-BOUNDARY.md。

### DENIED/紧急恢复（PASS）
- 触发：直接 DENIED→OPEN、无审计恢复。
- 拒绝：双人审批 + 不可篡改审计 + 新 Epoch。
- 证据：DENIED-BREAK-GLASS-STATE-MACHINE.md。

### 跨文档冲突（PASS）
- 触发：锁顺序/身份/DENIED 定义不一致。
- 拒绝：GLOBAL-PROTOCOL-INDEX 单一权威定义。
- 证据：GLOBAL-PROTOCOL-INDEX.md + CROSS-DOCUMENT-CONSISTENCY.md。

## 4. 结论

150 条反例全部有明确设计验证结果（PASS 或 NOT_PROVEN），无「全部通过」的虚构结论。60 条 NOT_PROVEN 如实保留，对应功能保持禁用。

## 5. 证据等级
反例数据为机器生成 + 人工逻辑复核；运行时行为 NOT_TESTED。
