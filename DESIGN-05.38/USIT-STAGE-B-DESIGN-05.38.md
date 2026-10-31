# USIT-STAGE-B-DESIGN-05.38.md — 主设计文档

清理资源 Fencing、原子条件 CAS 与恢复版本协议专项修订

**执行模式**：严格 READ-ONLY ANALYSIS + DESIGN REVISION
**前置基线**：DESIGN-05.37（commit `dede10a`）
**交付目录**：`DESIGN-05.38`
**最终结论**：PASS-WITH-ISSUES

---

## 1. 目标与范围

定点修复 05.37 独立复审发现的 4 项阻断问题，不扩大范围，不进入实现：

| 编号 | 05.37 问题 | 05.38 结论 |
|---|---|---|
| B38-01 | version 递增不能阻止在途释放 | **PASS（资源端 Fencing）** |
| B38-02 | 安全条件循环外检查一次 | **PASS（谓词纳入 CAS）** |
| B38-03 | 版本模型未定、ABA、递归 recover | **PASS（64 位合并原子字）** |
| B38-04 | PASS=187/185 冲突、缺陷去重 | **PASS（统一口径）** |

---

## 2. B38-01｜清理资源 Fencing

05.37 的 `ownership_version` 递增只能阻止「未来」释放，无法阻止「已发起」的在途释放。05.38 引入三个必须区分的概念：

- **软件层所有权校验**（进程内 owner_id/version）：只决定「是否发起」。
- **资源端 Fencing**（Token）：决定「资源端是否接受」，是跨进程安全的唯一依据。
- **操作幂等性**：仅对天然幂等释放有效。

修复要点：
- 释放流程 = 软件层校验 + 资源端 Fencing Token（`release_with_token`）。
- 心跳超时只触发「调查」，不单独作为「已停止」证明。
- Fencing 不支持的资源（unlink 临时文件等）→ NOT_PROVEN → 禁止接管释放 → ISOLATED。

详见 CLEANUP-FENCING-PROTOCOL.md、CLEANUP-OWNERSHIP-MATRIX.md、IDEMPOTENT-CLEANUP-PROTOCOL.md。

---

## 3. B38-02｜原子条件 CAS

05.37 的安全条件只在 CAS 循环外检查一次，提交点无原子保障。05.38 统一为 `conditional_cas(predicate, transform)`：

- 谓词与变换在**同一 CAS 内**基于同一次 `load()` 评估。
- CAS 失败重读重验，谓词结果永不过期。
- `admit`/`cas_frozen`/`cas_unfreeze` 的谓词都含 `¬denied`，DENIED 并发置位不可绕过。
- `release` 引入准入令牌，防止重复释放下溢。
- 重试上限 MAX_CAS_RETRY=8，禁止外层 while True 无限重试。

详见 ATOMIC-CONDITION-CAS-PROTOCOL.md、ADMISSION-FIELD-OPERATIONS.md、ADMISSION-OPERATION-MATRIX.md。

---

## 4. B38-03｜恢复版本原子性

05.37 的版本模型多选项未定、状态与版本更新可分离、ABA 未防护、递归 recover。05.38 选定**唯一权威模型**：

- 状态（bit0–31）+ 版本（bit32–63）合并为**64 位原子字**，单条 CAS 比较更新，二者不可分离。
- ABA 防护：cas_state_version 同时比较 (state, version)。
- recover 用 `while` + MAX_RECOVERY_RETRY=8 + 退避，禁止递归。
- DENIED 单调语义：new_state 的 DENIED = 状态字 DENIED ∪ 证据 DENIED，取并集。
- 跨进程/跨重启原子性未证明 → NOT_PROVEN → DENY。

详见 RECOVERY-VERSION-ATOMICITY.md、RECOVERY-CAS-PROTOCOL.md、CRASH-RECOVERY-PROTOCOL.md。

---

## 5. B38-04｜历史反例统计与缺陷去重

逐轮机器核验 + 缺陷聚类：

- 历史唯一 ID 190 条（05.33=60/05.34=40/05.35=30/05.36=30/05.37=30），无重复 ID。
- UNIQUE_DEFECT_TOTAL 通过聚类：D1「掩码 0x7FFE 遗漏 bit15」含 3 个 ID（B35-MASK-001/002、B36-MASK-002）→ 归 1 缺陷，故历史唯一缺陷 188。
- 05.37 的 PASS=187 vs 185 双口径已统一为「原始标注」口径（PASS=187）。
- 本轮新增 30 条，覆盖 B38-01/02/03 负向场景。

详见 HISTORICAL-COUNTEREXAMPLE-AUDIT.md、COUNTEREXAMPLE-RESULTS.md。

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
| B38-01 Fencing（无并发释放/不支持则 NOT_PROVEN） | PASS |
| B38-02 条件 CAS（谓词纳入、重试闭合） | PASS |
| B38-03 版本原子性（64 位合并、ABA、无递归） | PASS |
| B38-04 反例统计与缺陷去重 | PASS |
| 跨文档一致性 | PASS |
| 历史安全限制保留 | PASS |
| 新增反例 ≥30（含负向） | PASS（30） |
| 基线保护 | PASS |

**整体结论：PASS-WITH-ISSUES**

ISSUES：Fencing 不支持资源、跨进程 HW CAS 原子性、版本回绕 ABA 防护等 NOT_PROVEN 项仍存在，对应功能 DENY/ISOLATED。不构成本轮协议缺陷，但阻止整体 PASS。

---

## 8. 交付文件清单

见 HASH-MANIFEST.txt。共 17 份交付文档 + 反例 JSON。

---

## 9. 最终状态

- IMPLEMENT-01：NOT_APPROVED
- A/B Merge：NOT_APPROVED
- Source Retire：DENY

**下一步：STOP，等待 ChatGPT 独立复审。**
