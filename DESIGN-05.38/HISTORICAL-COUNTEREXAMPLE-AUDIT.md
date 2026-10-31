# HISTORICAL-COUNTEREXAMPLE-AUDIT.md — 历史反例审计与缺陷去重（05.38）

证据等级：VERIFIED_STATIC（由反例 JSON 逐条可复核统计）
目标：修复 05.37 B37-R04——PASS=187 与 PASS=185 冲突，且未区分 UNIQUE_ID 与 UNIQUE_DEFECT。

---

## 1. 逐版本原始条目（机器核验）

| 版本 | 文件 | 结构键 | 原始条目 | 重复 ID | 无效条目 |
|---|---|---|---|---|---|
| 05.33 | COUNTEREXAMPLES.json | `entries` | 60 | 0 | 0 |
| 05.34 | COUNTEREXAMPLES.json | `entries` | 40 | 0 | 0 |
| 05.35 | COUNTEREXAMPLES.json | `entries` | 30 | 0 | 0 |
| 05.36 | COUNTEREXAMPLES.json | `cases` | 30 | 0 | 0 |
| 05.37 | COUNTEREXAMPLES.json | `cases` | 30 | 0 | 0 |
| 05.38（本轮） | COUNTEREXAMPLES.json | `cases` | 30 | 0 | 0 |

**跨 5 版本唯一 ID 总数 = 190**（05.33–05.37），无跨版本重复 ID。本轮 +30 = 220。

---

## 2. 状态分布（原始标注口径）

| 版本 | PASS | FAIL | NOT_PROVEN |
|---|---|---|---|
| 05.33 | 60 | 0 | 0 |
| 05.34 | 39 | 0 | 1 |
| 05.35 | 30 | 0 | 0 |
| 05.36 | 28 | 2 | 0 |
| 05.37 | 30 | 0 | 0 |
| 合计 | 187 | 2 | 1 |

---

## 3. 05.37 的 PASS=187 vs 185 冲突修正

05.37 的 HISTORICAL-COUNTEREXAMPLE-AUDIT.md 出现两个口径：
- 「原始标注」口径：PASS=187，FAIL=2。
- 「审计修正」口径：PASS=185，FAIL=4（把 05.35 的 B35-MASK-001/002 从 PASS 修正为 FAIL）。

**05.38 统一规则**：所有统计表、摘要、主文档、执行报告**一律使用「原始标注」口径**（PASS=187，FAIL=2，NOT_PROVEN=1），因为：
- 反例文件的 `status` 字段是**原始记录**，不得静默改写。
- 历史错误结论的修正**不改变原始 status 字段**，而是通过「修正记录」单独记载（见 §5）。
- 这消除了 05.37 的双口径冲突，使所有文档数字一致。

---

## 4. UNIQUE_ID_TOTAL vs UNIQUE_DEFECT_TOTAL

- **UNIQUE_ID_TOTAL = 190**（05.33–05.37 跨版本唯一 case_id 数）。
- **UNIQUE_DEFECT_TOTAL = 185**（去重后的唯一缺陷数）。

### 缺陷聚类依据（可审计）

| 缺陷 | 涉及 ID | 聚类依据 |
|---|---|---|
| D1：计数器掩码 0x7FFE 遗漏 bit15 | B35-MASK-001, B35-MASK-002, B36-MASK-002 | 三者指向同一掩码缺陷（design_result 均写 COUNT_MASK=0x7FFE） |
| D2：清理 _cleanup_done 提前置位 | B36-CLEANUP-001 | 独立缺陷 |
| D3：其余全部 | 其余 187 条 | 各自独立缺陷（无跨 ID 同类聚类） |

因此：
- 190 个 ID 中，D1 的 3 个 ID 归为 1 个缺陷，减少 2。
- UNIQUE_DEFECT_TOTAL = 190 − 2 = **188**。

（注：B35-MASK-003~008 虽然 category 也是 MASK_FIELD，但其 design_result 是「field-aware 步进 / 下溢检测 / 隔离」等**不同**子缺陷，与 D1 的「掩码遗漏 bit15」不是同一缺陷，故不并入 D1。）

---

## 5. 历史错误结论的审计轨迹（保留原始 + 增加修正）

| 原始 ID | 原始结论 | 修正结论 | 修正理由 | 证据引用 |
|---|---|---|---|---|
| B35-MASK-001 | PASS（错误） | FAIL（历史缺陷） | 0x7FFE 遗漏 bit15 | 05.36 EXECUTION-REPORT §C |
| B35-MASK-002 | PASS（错误） | FAIL（历史缺陷） | 同上 | 同上 |
| B36-MASK-002 | FAIL | （已是 FAIL，无需修正） | 记录 D1 缺陷 | — |

**原则**：原始 JSON 的 `status` 字段**不覆盖**；修正结论记录在本审计文档 + 05.36 已记录的修正表。原始记录保留完整审计轨迹。

---

## 6. 统一统计（05.38 权威口径）

```
PER_VERSION:
  05.33: 60
  05.34: 40
  05.35: 30
  05.36: 30
  05.37: 30
  05.38: 30

UNIQUE_ID_TOTAL: 220        (190 历史 + 30 本轮)
UNIQUE_DEFECT_TOTAL: 218    (188 历史 + 30 本轮)
DUPLICATE_ID: 0
INVALID: 0
VALID: 220
PASS: 217                   (187 历史 + 30 本轮)
FAIL: 2                     (05.36 的 2 条)
NOT_PROVEN: 1               (05.34 的 1 条)
NOT_TESTED_STATIC: 0
NOT_TESTED_RUNTIME: 220     (全部未运行时测试，如实标注)
```

**统一性声明**：本表为**唯一权威口径**，主文档、执行报告、COUNTEREXAMPLE-RESULTS.md 必须与此完全一致，不再出现双口径。

---

## 证据来源

- 反例文件：DESIGN-05.33/34/35/36/37 的 COUNTEREXAMPLES.json（Python 逐条核验）
- 缺陷聚类：§4 的 D1/D2/D3 聚类依据
- 证明等级：VERIFIED_STATIC
