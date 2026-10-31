# USIT-STAGE-B-DESIGN-05.11｜设计差异清单（DIFF）

> 阶段 B · DESIGN-05.10 → DESIGN-05.11 逐项差异
> 本清单是**设计文档差异**，不是源码差异，也不伪装成源码差异。
> 工单明确：不推倒重做 05.10，仅修复六项缺口，执行顺序 P0-01 → P0-02 → P0-03 → P0-04 → P0-05 → P0-06 → 交叉审查。

---

## 0. 总体差异概览

| 维度 | DESIGN-05.10 | DESIGN-05.11 |
|------|-------------|--------------|
| POSIX 非 OFD 降级 | "创建独立锁文件"（无锁原语）| flock 原语 + 无锁则 DENY |
| fork 假设 | 假设子进程 Registry 为空 | 承认复制，register_at_fork 清理 |
| 锁文件身份 | 未定义 TOCTOU | 三层身份 + 句柄 vs 路径比较 |
| Candidate 提交 | O_EXCL 误当 rename 保护 | 无覆盖提交原语（renameat2/link）|
| 事务恢复 | 仅凭 SHA-256/revision 推断 | transaction_id/revision 全量绑定 |
| Journal 隔离 | 幂等跳过 + 未 fsync 目录 | 副本重校验 + 每次 rename 后 fsync 目录 |
| 终态快照 | 只存摘要字段 | 完整规范化终态内容 |
| 删除归因 | 不分 ALREADY_ABSENT | tombstone + 三态归因 |

---

## 1. P0-01 差异

| 子项 | 05.10 | 05.11 |
|------|-------|-------|
| 降级方案 | "独立锁文件"（概念模糊）| flock 原语，明确载体≠原语 |
| 无锁处理 | 未定义 | 平台无 flock 亦不可用 → DENY |
| 锁区间 | 未定义 | flock 锁整个文件，无区间 |
| 平台标识 | posix-nofd | posix-nofd-flock（更明确）|

---

## 2. P0-02 差异

| 子项 | 05.10 | 05.11 |
|------|-------|-------|
| fork 后 Registry | 假设为空 | 承认复制，after_in_child 清空+作废 |
| ProcessMutex | 未提 fork | fork 后重建，避免 locked 锁继承 |
| 锁 FD 处理 | 未明确 | 子进程 close 不调 LOCK_UN |
| 持锁 fork | 未定义 | 临界区内 fork DENY |
| threading.Lock | 隐含自动释放 | 明确不自动释放，try/finally 保证 |

---

## 3. P0-03 差异（新增）

05.10 未专门定义锁文件身份/TOCTOU，05.11 新增 §3：三层身份区分、生命周期操作规则、句柄 vs 路径对象比较、Windows 独占打开与区间锁职责区分。

---

## 4. P0-04 差异

| 子项 | 05.10 | 05.11 |
|------|-------|-------|
| 无覆盖提交 | 依赖 O_EXCL | renameat2/link 兜底 + CREATE_NEW 探针 |
| 事务归属验证 | SHA-256 + revision | transaction_id + revision + 摘要全量匹配 |
| 恢复组合 | 8 种 | 补齐 9 种（含"摘要相同 txn_id 不同"）|

---

## 5. P0-05 差异

| 子项 | 05.10 | 05.11 |
|------|-------|-------|
| 隔离副本存在 | 幂等跳过 | 重新校验哈希+身份+原文件状态 |
| 目录 fsync | 仅 fsync 文件 | 每次 rename 后 fsync 父目录 |
| 复制中变化 | 未定义 | 复制前后哈希比对，变化 DENY |
| 伪代码契约 | 部分 | 全部列输入/输出/失败/持久化责任 |

---

## 6. P0-06 差异

| 子项 | 05.10 | 05.11 |
|------|-------|-------|
| 快照内容 | 摘要字段 | 完整规范化终态 Journal 内容 |
| 删除方式 | 直接删除 | tombstone（rename→unlink）|
| 归因状态 | 无 ALREADY_ABSENT | 三态区分 |
| 状态机 | PENDING→DELETE_INTENT→REMOVED→CLEANED | 增加 TOMBSTONED/ALREADY_ABSENT/INDETERMINATE |

---

## 7. 未变化部分（明确不改）

- P1-02 / P1-03：状态保持（NOT_TESTED / 原状态）。
- P1-01：设计级闭合（P0-03），运行时 NOT_VERIFIED，不整体标 PASS。
- 三个阻断项、F1/P0.8/F2、IMPLEMENT-01：维持阻断。

---

## 8. 结论

DESIGN-05.11 相对 05.10 的差异全部落在六项 P0 缺口，均为**设计契约修订**，不涉及任何源码、测试、配置、运行时数据、A/B 工作区或冻结基线的变更。
