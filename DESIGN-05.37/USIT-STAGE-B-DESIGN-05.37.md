# USIT-STAGE-B-DESIGN-05.37.md — 主设计文档

冻结状态机、清理所有权与崩溃恢复专项修订

**执行模式**：严格 READ-ONLY ANALYSIS + DESIGN REVISION
**基线版本**：DESIGN-05.36（commit `511f59d`）
**交付目录**：`DESIGN-05.37`
**最终结论**：PASS-WITH-ISSUES

---

## 1. 目标与范围

定点修复 05.36 独立复审发现的五项问题，不扩大范围，不进入实现：

| 编号 | 05.36 问题 | 05.37 结论 |
|---|---|---|
| B37-01 | 三套冻结描述不一致、矩阵 T4 DENIED 通配冲突 | **PASS（统一）** |
| B37-02 | 清理锁外释放无所有权、永久失败无限重试 | **PASS（修复）** |
| B37-03 | 恢复 commit(state) 无条件覆盖、allowed 未定义 | **PASS（修复）** |
| B37-04 | 准入整字算术 cur+(1<<1) 违反 field-aware 规范 | **PASS（修复）** |
| B37-05 | 反例累计 60+40+30=130≠160 计算错误 | **PASS（逐轮核验）** |

---

## 2. B37-01｜冻结状态机统一

05.36 三套描述（一次 CAS 关 Gate+FROZEN / 三阶段 / 两阶段 DENIED 通配）互相矛盾。05.37 统一为唯一权威协议（FREEZE-STATE-MACHINE.md）：

- 冻结分三阶段：① 关 Gate（`cas gate 1→0`）→ ② 等待 count 归零 → ③ 设 FROZEN（`cas_frozen`）。
- `cas_frozen()` 原子检查 `gate==0 且 DENIED==0 且 count==0` 三项，修复矩阵 T4 的 DENIED 通配 `*` 冲突。
- DENIED 优先级 `DENIED > FROZEN > FREEZING > OPEN`。
- 8 条并发时间线证明 DENIED 一旦提交不可清除。

---

## 3. B37-02｜清理所有权与并发执行

05.36 清理锁外释放无所有权证明、永久失败无限重试。05.37 修复：

- 7 态清理状态机，增加 `CLEANUP_CLAIMED`。
- 执行所有权（owner_id + ownership_version + resource_set）原子取得，同一资源只由一个执行者释放。
- 重试上限 `MAX_RETRY=3` + 指数退避，永久失败/状态不明 → ISOLATED，不无限循环。
- 清理者崩溃接管协议（心跳超时 + 资源状态核查 + 版本递增）。

详见 IDEMPOTENT-CLEANUP-PROTOCOL.md、CLEANUP-OWNERSHIP-MATRIX.md。

---

## 4. B37-03｜崩溃恢复状态重建与 CAS 保护

05.36 恢复用 `commit(state)` 无条件覆盖、`allowed`/`denied_evidence` 未定义。05.37 修复：

- 明确恢复证据清单（来源/时间/版本/可信等级）。
- `allowed` 由 `compute_allowed()` 的可验证条件计算，无未定义变量。
- 恢复用 `cas_with_version` 带版本校验，禁止无条件 commit；CAS 失败重读重验。
- DENIED 证据 TOCTOU 防护：提交前取 DENIED 并集，绝不写 0。
- 证据不足/冲突/版本变化/来源不可信 → DENY。

详见 RECOVERY-CAS-PROTOCOL.md、CRASH-RECOVERY-PROTOCOL.md。

---

## 5. B37-04｜统一准入字段操作

05.36 的 `nv = cur + (1 << COUNT_SHIFT)` 违反 field-aware 规范。05.37 统一：

- 唯一接口 `field_cas`，所有字段更新用「掩码清除 + 移位写入」，禁止整字算术。
- 掩码互不重叠，证明字段隔离（count 更新不污染 gate/frozen/DENIED）。
- 准入/退出/冻结/解冻/恢复全部走 field_cas。
- CAS 失败重读重算，LIVELOCK（8 次）降级，不无限重试。

详见 ADMISSION-FIELD-OPERATIONS.md。

---

## 6. B37-05｜历史反例累计核验

逐轮机器核验（见 HISTORICAL-COUNTEREXAMPLE-AUDIT.md）：

- 05.33=60、05.34=40、05.35=30、05.36=30，合计 160 条历史 + 本轮 30 = 190。
- 唯一 ID 无重复，无效条目 0。
- 05.35 的 B35-MASK-001/002 错误 PASS（0x7FFE）修正为 FAIL（历史缺陷），保留原始记录。
- 05.36 的 `60+40+30=130≠160` 计算错误已纠正。

---

## 7. 继承的安全约束（未弱化）

- 唯一准入状态源；COUNT_MASK=0xFFFE、移位 1、MAX=32767。
- DENIED 不可被普通恢复/rollback/unfreeze/父子初始化清除。
- 跨快照身份唯一性 NOT_PROVEN；Receipt 认证 NOT_PROVEN；Break-glass NOT_PROVEN；锁外写入者 UNKNOWN。
- Fork 在共享介质/未排除旧写入者时 DENY。
- Source Retire DENY；IMPLEMENT-01 / A/B Merge NOT_APPROVED。

---

## 8. 验收结果

| 验收项 | 结论 |
|---|---|
| 冻结状态机唯一权威 | PASS |
| DENIED 与 FROZEN 原子保护 | PASS |
| 清理唯一执行所有权、不无限重试 | PASS |
| 清理者崩溃接管可证明安全 | PASS |
| 恢复版本校验 CAS、不覆盖并发 DENIED | PASS |
| 旧写入者未排除不重置计数 | PASS |
| 准入字段操作统一、无整字算术 | PASS |
| 历史反例逐条核验、统计可复现 | PASS |
| 新增反例 ≥30 | PASS（30） |
| 跨文档无关键矛盾 | PASS |
| 05.36 基线与源码不变 | PASS |
| SHA256 清单 | PASS |

**整体结论：PASS-WITH-ISSUES**

ISSUES：身份防回滚、Receipt 认证、Break-glass 审计根、锁外写入者枚举四项历史 NOT_PROVEN/UNKNOWN 仍存在，对应功能禁用。不构成本轮协议缺陷，但阻止整体 PASS。

---

## 9. 交付文件清单

见 HASH-MANIFEST.txt。共 15 份交付文档 + 反例 JSON。

---

## 10. 最终状态

- IMPLEMENT-01：NOT_APPROVED
- A/B Merge：NOT_APPROVED
- Source Retire：DENY

**下一步：STOP，等待 ChatGPT 独立复审。**
