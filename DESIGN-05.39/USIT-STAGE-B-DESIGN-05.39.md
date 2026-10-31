# USIT-STAGE-B-DESIGN-05.39.md — 主设计文档

准入令牌原子事务、资源端 Fencing、版本回绕与交付完整性专项修订

**执行模式**：严格 READ-ONLY ANALYSIS + DESIGN REVISION
**前置版本**：DESIGN-05.38（commit `e782471`）
**交付目录**：`DESIGN-05.39`
**最终结论**：PASS-WITH-ISSUES

---

## 1. 目标与范围

修复 ChatGPT 独立复审 05.38 发现的 B38-R01～R08，形成伪代码一致、反例可复算的 05.39 设计：

| 编号 | 05.38 缺陷 | 05.39 结论 |
|---|---|---|
| B39-01 | Token 与 COUNT 非原子（崩溃窗口 + 并发重复消费） | **PASS（内嵌 128 位 CAS）** |
| B39-02 | flock/fcntl 误标 Fencing、两步 TOCTOU | **PASS（能力分级 + 单接口）** |
| B39-03 | 版本静默回绕 vs 文字 DENY | **PASS（VERSION_EXHAUSTED_DENY）** |
| B39-04 | 185/188/218 矛盾 + HASH 错误 | **PASS（统一口径 + blob 哈希）** |

---

## 2. B39-01｜准入令牌与 COUNT 原子一致性

05.38 的 `admit_tokens` 独立字典导致两个竞态：admit 崩溃窗口（COUNT 改但 token 未入账）、release 并发重复消费（`token in dict` 非原子）。

05.39 采用**方案 A：Token 内嵌状态字**（扩展为 128 位原子字）：
- `admit()` 的单条 128 位 CAS 原子完成「COUNT+1 + token_seq+1 + active 位图置位」。
- `release(token)` 的单条 128 位 CAS 原子完成「验证 token + COUNT−1 + active 位图清除」。
- 消除崩溃窗口与并发重复消费。方案 B（独立账本 + WAL）仅作宽度不足时的退化路径，且跨进程 NOT_PROVEN。

详见 TOKEN-ATOMIC-TRANSACTION.md、TOKEN-OWNERSHIP-AND-LIFECYCLE.md、TOKEN-CRASH-RECOVERY.md、TOKEN-CONCURRENCY-MATRIX.md。

---

## 3. B39-02｜资源端 Fencing 协议

05.38 把 flock/fcntl/fd 误归为 FENCING_SUPPORTED，且保留两步 accept+release TOCTOU。

05.39 修正：
- 五概念严格区分：软件层 Owner 检查 / OS 锁生命周期 / 资源端 Fencing Token / 资源端原子释放 / 幂等操作。
- flock/fcntl/fd 降级为 **OS_LOCK_ONLY**，禁止接管释放。
- 唯一接口 `release_with_fencing_token()`，Token 比较与释放一步原子完成，消除 TOCTOU。
- 不支持资源端 Token 的资源 → ISOLATED；心跳超时只触发调查。

详见 RESOURCE-FENCING-PROTOCOL.md、RESOURCE-FENCING-CAPABILITY-MATRIX.md、RESOURCE-RELEASE-ATOMICITY.md。

---

## 4. B39-03｜版本耗尽与 ABA

05.38 的 `(old_version + 1) & 0xFFFFFFFF` 在 version==0xFFFFFFFF 时静默回绕，与「回绕 DENY」矛盾。

05.39 修正：
- 版本耗尽检查在**提交路径内**：`if old_version == 0xFFFFFFFF: return 'VERSION_EXHAUSTED_DENY'`。
- 删除静默回绕；ABA 通过「禁止回绕」消除残余风险。
- epoch 移除（除非进入 CAS 比较值）。

详见 VERSION-EXHAUSTION-PROTOCOL.md、RECOVERY-VERSION-ATOMICITY.md。

---

## 5. B39-04｜历史统计、缺陷去重与 HASH 完整性

- 统计统一口径：RAW_ENTRY_TOTAL=220、UNIQUE_ID_TOTAL=220、UNIQUE_DEFECT_TOTAL=218（历史）。
- 消除 185/188/218 矛盾（185 是笔误，删除）。
- HASH 差异根因：git `core.autocrlf=true` 导致工作区 CRLF(18949B) vs blob LF(18352B)，05.38 HASH 清单哈希了错误对象。05.39 对 git blob 哈希。

详见 HISTORICAL-COUNTEREXAMPLE-AUDIT.md。

---

## 6. 继承的安全约束（未弱化）

- 跨快照身份唯一性：NOT_PROVEN。
- Receipt 创建者认证：NOT_PROVEN。
- Break-glass 审计信任根：NOT_PROVEN。
- 锁外写入者：UNKNOWN。
- DENIED 不可清除；COUNT_MASK=0xFFFE。
- Source Retire DENY；IMPLEMENT-01 / A/B Merge NOT_APPROVED。

---

## 7. 验收结果

| 验收项 | 结论 |
|---|---|
| B39-01 Token 原子一致性（无双消费/无 COUNT 永久错） | PASS |
| B39-02 资源端 Fencing（能力分级、无 TOCTOU） | PASS |
| B39-03 版本耗尽（禁止回绕） | PASS |
| B39-04 统计与 HASH（可复算、blob 哈希） | PASS |
| 跨文档一致性 | PASS |
| 历史安全限制保留 | PASS |
| 新增反例 ≥55（含负向竞态） | PASS（55） |

**整体结论：PASS-WITH-ISSUES**

ISSUES：跨进程 Token/HW CAS、版本回绕 ABA、Fencing 不支持资源等 NOT_PROVEN 项仍存在，对应功能 DENY/ISOLATED。不构成本轮协议缺陷，但阻止整体 PASS。

---

## 8. 交付文件清单

见 HASH-MANIFEST.txt。共 17 份交付文档 + 反例 JSON。

---

## 9. 最终状态

- IMPLEMENT-01：NOT_APPROVED
- A/B Merge：NOT_APPROVED
- Source Retire：DENY

**下一步：STOP，等待 ChatGPT 独立复审。**
