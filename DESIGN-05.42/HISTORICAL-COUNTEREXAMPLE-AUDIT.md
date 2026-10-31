# HISTORICAL-COUNTEREXAMPLE-AUDIT.md — 历史反例审计与累计统计（05.42）

证据等级：VERIFIED_STATIC（由 JSON 逐条机器核验）
目标：继承 05.41 统计口径，叠加 05.42 新增 50 条，重新计算累计值。

---

## 1. 逐版本原始条目（05.33–05.42）

| 版本 | 结构键 | 原始条目 | 唯一 ID | 重复 ID | 无效 |
|---|---|---|---|---|---|
| 05.33 | entries | 60 | 60 | 0 | 0 |
| 05.34 | entries | 40 | 40 | 0 | 0 |
| 05.35 | entries | 30 | 30 | 0 | 0 |
| 05.36 | cases | 30 | 30 | 0 | 0 |
| 05.37 | cases | 30 | 30 | 0 | 0 |
| 05.38 | cases | 30 | 30 | 0 | 0 |
| 05.39 | cases | 55 | 55 | 0 | 0 |
| 05.40 | cases | 75 | 75 | 0 | 0 |
| 05.41 | cases | 60 | 60 | 0 | 0 |
| 05.42 | cases | 50 | 50 | 0 | 0 |
| **合计** | | **460** | **460** | **0** | **0** |

---

## 2. 历史唯一 ID 与唯一缺陷（05.33–05.41）

- RAW_ENTRY_TOTAL（05.33–05.41）= 410。
- UNIQUE_ID_TOTAL（05.33–05.41）= 410。
- UNIQUE_DEFECT_TOTAL（05.33–05.41）= 408（D1 掩码缺陷 3 ID 归 1，减 2）。

### 缺陷聚类（继承）

| 缺陷 | 关联 case_id | 归并数 |
|---|---|---|
| D1 掩码 0x7FFE 遗漏 bit15 | B35-MASK-001/002 + B36-MASK-002 | 2（3 ID → 1） |
| D2 清理 _cleanup_done 提前置位 | B36-CLEANUP-001 | 0 |

---

## 3. 05.42 累计统计（权威，可复算）

```
RAW_ENTRY_TOTAL: 460         (410 历史 + 50 本轮)
UNIQUE_ID_TOTAL: 460
UNIQUE_DEFECT_TOTAL: 458     (408 历史 + 50 本轮，本轮无跨缺陷归并)
DUPLICATE_ID: 0
INVALID: 0
VALID: 460
PASS: 457                    (407 历史 + 50 本轮)
FAIL: 2                      (历史 B35-MASK-001/002 修正结论)
NOT_PROVEN: 1                (05.34 的 1 条)
NOT_TESTED_STATIC: 0
NOT_TESTED_RUNTIME: 460
```

---

## 4. 历史状态分布（05.33–05.41 原始标注口径）

| 指标 | 值 |
|---|---|
| PASS | 407 |
| FAIL | 2（原始标 PASS，修正为 FAIL） |
| NOT_PROVEN | 1 |
| NOT_TESTED_RUNTIME | 410 |

> B35-MASK-001/002 原始 status 标 PASS（错误），修正结论 FAIL 单独记录，不覆盖原始 status。

---

## 5. 历史错误结论审计轨迹

| 原始 ID | 原始 status | 修正结论 | 理由 |
|---|---|---|---|
| B35-MASK-001 | PASS（错误） | FAIL | 0x7FFE 遗漏 bit15 |
| B35-MASK-002 | PASS（错误） | FAIL | 同上 |

---

## 6. 统计口径一致性

- 05.42 主文档、EXECUTION-REPORT、COUNTEREXAMPLE-RESULTS、本审计文档四处统计一致：RAW=460 / 唯一ID=460 / 唯一缺陷=458 / PASS=457 / FAIL=2 / NOT_PROVEN=1 / NOT_TESTED_RUNTIME=460。
- 由 COUNTEREXAMPLES.json（05.42，50 条）实际数据计算。

---

## 证据来源

- 历史反例 JSON：DESIGN-05.33~05.41
- 本轮反例 JSON：COUNTEREXAMPLES.json（05.42，50 条）
- 证明等级：VERIFIED_STATIC + NOT_TESTED_RUNTIME
