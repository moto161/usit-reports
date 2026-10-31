# HISTORICAL-COUNTEREXAMPLE-AUDIT.md — 历史反例审计与缺陷去重（05.39）

证据等级：VERIFIED_STATIC（由 JSON 逐条机器核验）
目标：修复 B38-R04——185/188/218 三个数字矛盾，逐条复算。

---

## 1. 逐版本原始条目（机器核验，05.33–05.38）

| 版本 | 结构键 | 原始条目 | 唯一 ID | 重复 ID | 无效 |
|---|---|---|---|---|---|
| 05.33 | entries | 60 | 60 | 0 | 0 |
| 05.34 | entries | 40 | 40 | 0 | 0 |
| 05.35 | entries | 30 | 30 | 0 | 0 |
| 05.36 | cases | 30 | 30 | 0 | 0 |
| 05.37 | cases | 30 | 30 | 0 | 0 |
| 05.38 | cases | 30 | 30 | 0 | 0 |

---

## 2. 历史唯一 ID 与唯一缺陷

- **RAW_ENTRY_TOTAL（05.33–05.38）= 220**（60+40+30+30+30+30）。
- **UNIQUE_ID_TOTAL（历史）= 220**（跨 6 版本无重复 case_id）。

### 缺陷聚类（唯一缺陷 = 唯一 ID − 归并数）

| 缺陷 | 关联 case_id | 聚类理由 | 归并数 |
|---|---|---|---|
| D1 掩码 0x7FFE 遗漏 bit15 | B35-MASK-001, B35-MASK-002, B36-MASK-002 | 三者 design_result 均指向同一掩码缺陷 | 2（3 ID → 1 缺陷） |
| D2 清理 _cleanup_done 提前置位 | B36-CLEANUP-001 | 独立缺陷 | 0 |

**UNIQUE_DEFECT_TOTAL（历史）= 220 − 2 = 218。**

---

## 3. 05.38 的 185/188/218 矛盾澄清

05.38 文档出现三个数字，来源与正误：

| 数字 | 05.38 出处 | 正误 | 正确值 |
|---|---|---|---|
| 185 | §4 标题「UNIQUE_DEFECT_TOTAL = 185」 | **错误**（笔误，把 05.37 的「审计修正口径 PASS=185」误写成缺陷总数） | 删除 |
| 188 | §4「190 − 2 = 188」 | 正确但口径旧（只算到 05.37 的 190） | 更新为 218（含 05.38） |
| 218 | §6 累计 | 正确（190−2=188 历史缺陷 + 05.38 的 30 = 218） | 保留 |

**05.39 权威口径**：
- RAW_ENTRY_TOTAL = 220
- UNIQUE_ID_TOTAL = 220
- UNIQUE_DEFECT_TOTAL = 218（历史）

（05.39 本轮 +55 条见 §4，累计见 §5。）

---

## 4. 历史状态分布（原始标注口径，唯一）

| 指标 | 值 |
|---|---|
| PASS | 217（60+39+28+30+30+30） |
| FAIL | 2（05.36 的 2 条） |
| NOT_PROVEN | 1（05.34 的 1 条） |
| NOT_TESTED_STATIC | 0 |
| NOT_TESTED_RUNTIME | 220 |

**05.35 的 B35-MASK-001/002 原始标 PASS（错误），但修正记录单独记载，不改变原始 status 字段**（见 §5）。

---

## 5. 历史错误结论的审计轨迹（保留原始 + 修正）

| 原始 ID | 原始 status | 修正结论 | 修正理由 | 证据 |
|---|---|---|---|---|
| B35-MASK-001 | PASS（错误） | FAIL（历史缺陷） | 0x7FFE 遗漏 bit15 | 05.36 EXECUTION-REPORT §C |
| B35-MASK-002 | PASS（错误） | FAIL（历史缺陷） | 同上 | 同上 |

**原则**：原始 JSON `status` 字段不覆盖；修正结论记录在本审计文档，保留完整审计轨迹。

---

## 6. 05.39 累计统计（权威，可复算）

```
RAW_ENTRY_TOTAL: 275         (220 历史 + 55 本轮)
UNIQUE_ID_TOTAL: 275
UNIQUE_DEFECT_TOTAL: 273     (218 历史 + 55 本轮，本轮无跨缺陷归并)
DUPLICATE_ID: 0
INVALID: 0
VALID: 275
PASS: 272                    (217 历史 + 55 本轮)
FAIL: 2
NOT_PROVEN: 1
NOT_TESTED_STATIC: 0
NOT_TESTED_RUNTIME: 275
```

---

## 7. HASH 差异调查（B38-R04 交付完整性）

05.38 的 `COUNTEREXAMPLES.json` 出现 GitHub 18,352 bytes vs HASH 清单 18,949 bytes 的差异。**根因已查明**：

- Git 仓库 `core.autocrlf = true`：checkout 时把 LF 转成 CRLF，工作区文件 = 18,949 bytes（597 个 CRLF）。
- Git blob 存 LF：`git cat-file` blob = 18,352 bytes，实际 SHA256 = `52dba7d8...`。
- 05.38 HASH 清单声明 `19ec5319... 18949`，哈希的是**本地 CRLF 工作区副本**，而非 git 实际推送的 LF blob。

**结论**：05.38 的 HASH 清单 SHA256 与字节数均错误（哈希对象不对）。05.39 修正：**HASH 清单对 git blob（LF 内容）计算哈希**，而非本地 CRLF 工作区文件。详见 EXECUTION-REPORT.md §HASH。

---

## 证据来源

- 反例 JSON：DESIGN-05.33~05.38（Python 逐条核验）
- HASH 差异：git cat-file + sha256sum 比对
- 证明等级：VERIFIED_STATIC
