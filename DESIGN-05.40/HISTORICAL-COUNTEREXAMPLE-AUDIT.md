# HISTORICAL-COUNTEREXAMPLE-AUDIT.md — 历史反例审计与累计统计（05.40）

证据等级：VERIFIED_STATIC（由 JSON 逐条机器核验）
目标：保留 05.39 统计口径，重新核验 05.33–05.39 历史反例，并叠加 05.40 新增。

---

## 1. 逐版本原始条目（05.33–05.40）

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
| **合计** | | **350** | **350** | **0** | **0** |

---

## 2. 历史唯一 ID 与唯一缺陷（05.33–05.39）

- **RAW_ENTRY_TOTAL（05.33–05.39）= 275**（60+40+30+30+30+30+55）。
- **UNIQUE_ID_TOTAL（历史，05.33–05.39）= 275**。
- **UNIQUE_DEFECT_TOTAL（历史）= 273**（D1 掩码缺陷 3 ID 归 1，减 2）。

### 缺陷聚类（继承 05.39 口径）

| 缺陷 | 关联 case_id | 聚类理由 | 归并数 |
|---|---|---|---|
| D1 掩码 0x7FFE 遗漏 bit15 | B35-MASK-001, B35-MASK-002, B36-MASK-002 | 同一掩码缺陷 | 2（3 ID → 1 缺陷） |
| D2 清理 _cleanup_done 提前置位 | B36-CLEANUP-001 | 独立缺陷 | 0 |

**UNIQUE_DEFECT_TOTAL（历史）= 275 − 2 = 273。**

---

## 3. 05.40 累计统计（权威，可复算）

```
RAW_ENTRY_TOTAL: 350         (275 历史 + 75 本轮)
UNIQUE_ID_TOTAL: 350
UNIQUE_DEFECT_TOTAL: 348     (273 历史 + 75 本轮，本轮无跨缺陷归并)
DUPLICATE_ID: 0
INVALID: 0
VALID: 350
PASS: 347                    (272 历史 + 75 本轮)
FAIL: 2                      (历史 B35-MASK-001/002 修正结论，原始 status 保留)
NOT_PROVEN: 1                (05.34 的 1 条)
NOT_TESTED_STATIC: 0
NOT_TESTED_RUNTIME: 350
```

> 说明：05.39 的累计口径为 RAW=275 / 唯一缺陷=273。05.40 叠加 75 条后，RAW=350，唯一缺陷=348（本轮 75 条无跨缺陷归并，唯一缺陷增量 = 唯一 ID 增量 = 75）。

---

## 4. 历史状态分布（05.33–05.39 原始标注口径，唯一）

| 指标 | 值 |
|---|---|
| PASS | 272（05.33–05.39：217 + 55） |
| FAIL | 2（05.36 的 2 条，原始标 PASS 但修正为 FAIL） |
| NOT_PROVEN | 1（05.34 的 1 条） |
| NOT_TESTED_STATIC | 0 |
| NOT_TESTED_RUNTIME | 275 |

> 注意：B35-MASK-001/002 的**原始 status 字段标 PASS（错误）**，修正结论（FAIL）单独记录在审计轨迹，**不覆盖原始 status**。

---

## 5. 历史错误结论审计轨迹（保留原始 + 修正）

| 原始 ID | 原始 status | 修正结论 | 修正理由 | 证据 |
|---|---|---|---|---|
| B35-MASK-001 | PASS（错误） | FAIL（历史缺陷） | 0x7FFE 遗漏 bit15 | 05.36 EXECUTION-REPORT §C |
| B35-MASK-002 | PASS（错误） | FAIL（历史缺陷） | 同上 | 同上 |

---

## 6. 本轮统计口径一致性核验

- 05.40 主文档、EXECUTION-REPORT、COUNTEREXAMPLE-RESULTS、本审计文档四处统计必须一致：RAW=350 / 唯一ID=350 / 唯一缺陷=348 / PASS=347 / FAIL=2 / NOT_PROVEN=1 / NOT_TESTED_RUNTIME=350。
- 统计数字由 COUNTEREXAMPLES.json（05.40，75 条）实际数据计算，非手工填写。

---

## 证据来源

- 历史反例 JSON：DESIGN-05.33~05.39
- 本轮反例 JSON：COUNTEREXAMPLES.json（05.40，75 条）
- 证明等级：VERIFIED_STATIC + NOT_TESTED_RUNTIME
