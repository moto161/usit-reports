# USIT-STAGE-B-DESIGN-05.48 — 定点修订工单

- 前置审查：DESIGN-05.47/INDEPENDENT-REVIEW.md
- 修订对象：DESIGN-05.46
- 执行模式：READ-ONLY ANALYSIS + DESIGN REVISION
- 目标：逐项关闭 R47-01～R47-12，建立唯一、可复核的单进程事务安全协议。
- **本轮不实现源码，不修改 `usit/*.py`，不构建、不部署、不执行真实资源操作。**
- **不得将静态推演写成运行时 PASS。**

## 一、执行要求

1. 先完整读取 DESIGN-05.47 独立审查报告及 DESIGN-05.46 全部相关协议文件；不得只读摘要。
2. 对 R47-01～R47-12 建立逐项追踪矩阵：审查发现 → 修订位置 → 设计依据 → 新增反例 → 验证方式 → 状态。
3. 不允许仅修改文字或把 FAIL 改成 PASS。每项必须提供完整伪代码、状态转移、持久化边界、失败路径和反例推演。
4. 发生冲突时，以本轮新建立的唯一权威协议为准；旧版文档明确标注 SUPERSEDED/HISTORICAL，不得继续保留为并列权威。
5. 对不能证明的能力继续 NOT_PROVEN，并保持 DENY/ISOLATED；禁止为了通过验收而放宽安全策略。

## 二、必须修复的十二项

- R47-01：重建 COUNTEREXAMPLES.json。raw/unique/category/baseline/revision/历史累计全部由同一数据源自动计算；脚本断言数组长度、唯一 ID、分类和状态统计一致。禁止手工填写统计摘要。
- R47-02：统一 atomic_release。可信调用者身份、token、owner、resource、generation/Fencing 校验必须在 operation 前完成，并与资源接管处于同一明确串行化边界。
- R47-03：start_req 返回值必须逐分支检查；只有持久化占用成功才能执行副作用。明确 ABORT、UNKNOWN、CONFLICT、OCCUPIED 的语义。
- R47-04：修复所有未定义变量、未定义 WAL record、错误参数、语法错误。每个记录必须显式构造，字段 schema 固定，所有函数参数与作用域完整。
- R47-05：OP_STARTED/OP_CONFIRM/COMMIT/APPLY/OP_RELEASE 的写入顺序、fsync、提交点、失败处理和恢复语义必须闭合。任何不确定结果不得被释放成普通成功或普通失败。
- R47-06：同 req_id 必须校验 op/owner/resource/payload_hash，并根据真实持久化状态返回；不得将 UNKNOWN/ISOLATED/COMMITTED_PENDING_APPLY 伪报为 ALREADY_RELEASED。
- R47-07：重写 WAL 从头恢复算法，完整重建 state、TRT、req_id_index、operation_state_index；明确每类 WAL 记录的折叠规则、顺序、重复/缺失/损坏处理。
- R47-08：重写 checkpoint 恢复算法，明确 wal_position 与 commit_seq 的权威关系、快照与索引的一致性校验、检查点后记录范围；不得重复应用检查点前记录。
- R47-09：实现真正不可变快照设计：输入防御性深拷贝、递归不可变表示、不得泄漏可变引用；列出 Python 对象访问模型下的限制与验证方法。
- R47-10：query_txn 必须检查 txn 全部 COMMIT/APPLY 记录的唯一性、序号、snapshot_hash、checksum、状态转移；明确无 APPLY 时返回 COMMITTED_PENDING_APPLY。
- R47-11：建立全局锁层级与每个函数的 acquire/release 区间表，覆盖 barrier_cond、global_request_lock、resource_lock、txn_guard、WAL、checkpoint；证明所有路径一致且不存在锁顺序环。
- R47-12：统一跨文档权威关系。更新或废止 05.44/05.45 遗留协议，生成跨文档一致性自动检查，不得只核对标题和术语。

## 三、强制交付文件

在 `DESIGN-05.48/` 下交付：
1. `USIT-STAGE-B-DESIGN-05.48.md` — 主文档、最终结论、边界。
2. `ATOMIC-RELEASE-PROTOCOL.md` — 唯一权威释放协议与完整伪代码。
3. `LOCK-HIERARCHY-AND-WAIT-FOR-GRAPH.md` — 锁层级、获取区间、等待图与死锁反例。
4. `WAL-RECOVERY-AND-CHECKPOINT.md` — 从头恢复、检查点恢复、记录折叠规则。
5. `IMMUTABLE-SNAPSHOT-AND-QUERY.md` — 快照不可变性、query_txn 状态语义。
6. `COUNTEREXAMPLES.json` — 完整反例数据，逐条包含唯一 ID、baseline_status、revision_status、证据引用、verification_method。
7. `COUNTEREXAMPLE-RESULTS.md` — 由脚本生成的统计结果。
8. `CROSS-DOC-CONSISTENCY.md` — 跨文档自动校验结果。
9. `VALIDATION-SCRIPTS/` — 统计校验与文档一致性校验脚本及运行说明。
10. `ISSUES.md` — 未解决项、影响、DENY/ISOLATED 策略。
11. `EXECUTION-REPORT.md` — 实际执行报告。
12. `HASH-MANIFEST.txt` — Git blob SHA256 与字节数清单。

## 四、验收标准

- A：十二项 R47 发现逐项有证据闭合；不能闭合的明确标记 NOT_PROVEN，不得标 PASS。
- B：JSON 实际解析结果与所有摘要统计完全一致；脚本重复运行结果稳定。
- C：所有伪代码语法可解析；静态类型/名称检查无未定义符号；伪代码中的每个返回值均被调用方处理。
- D：授权与 Fencing 在副作用前；请求占用失败绝不执行副作用。
- E：崩溃矩阵覆盖每个持久化边界前后；非幂等副作用在结果未知时不自动重放。
- F：WAL 与 checkpoint 恢复可逐记录推演，索引完整重建，无双入口、无重复折叠。
- G：快照不可变性与 query_txn 所有状态均有正反例。
- H：锁顺序表覆盖全部调用路径，等待图不存在环；若依赖运行时行为才能证明，标记 NOT_PROVEN。
- I：跨文档校验脚本实际运行，退出码与输出纳入报告。
- J：SHA256 清单与 Git blob 内容、字节数一致。

## 五、禁止事项

- 禁止修改任何项目源码、配置、数据库或运行数据。
- 禁止执行 IMPLEMENT-01、A/B Merge、Source Retire。
- 禁止删除或覆盖历史证据；只新增修订并明确版本关系。
- 禁止把静态推演称为并发测试、崩溃测试、运行时验证。
- 禁止仅因文档齐全、脚本通过或统计一致就宣称事务协议安全；逻辑审查仍需独立复核。

## 六、执行报告固定格式

必须报告：实际读取文件清单；R47-01～R47-12 每项 PASS/FAIL/NOT_PROVEN 与证据；新增/修改文件；脚本命令、退出码、实际输出；静态推演覆盖范围；未解决问题；源码是否零修改；运行时测试是否未执行；Git commit SHA；最终结论。

**最终状态只能是 PASS-WITH-ISSUES 或 FAIL。即使 PASS-WITH-ISSUES，也不代表批准 IMPLEMENT-01；必须等待 ChatGPT 独立复审。**
