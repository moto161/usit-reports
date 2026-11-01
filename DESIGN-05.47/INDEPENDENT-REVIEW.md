# USIT-STAGE-B-DESIGN-05.47 — 独立复审报告

- 审查对象：`moto161/usit-reports` / `DESIGN-05.46`
- 基线：审查时默认分支最新提交 `5837c14b6495d98c7e7e4fb0d5883c6cab830327`
- 审查方式：GitHub 交付文档与原始 JSON 静态独立复核
- 证据等级：STATIC_REVIEW（不是源码运行验证）
- 结论：**FAIL — 不批准 IMPLEMENT-01 / A-B Merge / Source Retire**

## 1. 总结

05.46 不具备进入实现阶段的条件。问题不止是统计错误；事务释放、授权、WAL 恢复、快照不可变性和状态查询均存在未闭合缺陷。05.46 文档中的 VERIFIED_STATIC / PASS 声明不能作为实现准入依据。

本报告只审查已读取的 05.46 交付内容；没有执行源码、构建、并发实验、崩溃注入或真实资源操作，也没有声称完成对历史 598 条反例的逐条复核。

## 2. 阻断级发现

### R47-01 — 反例 JSON 声明统计与实际 cases 数组不符
- 位置：`DESIGN-05.46/COUNTEREXAMPLES.json`
- 声明：RAW=38、UNIQUE_ID=38、UNIQUE_DEFECT=38；revision PASS=32、FAIL=0、NOT_PROVEN=0。
- 实际解析：`cases.length=29`，唯一 ID=29。
- 实际类别：RECOVERY_BARRIER=6、REQUEST_ID_ATOMIC_IDEMPOTENCY=5、RESOURCE_OPERATION_CRASH_RECOVERY=7、WAL_INTEGRITY_AND_CHECKPOINT=5、POST_COMMIT_FAILURE_CLOSURE=6。
- 实际 baseline：FAIL=23、PASS=2、NOT_PROVEN=4。
- 实际 revision：PASS=25、NOT_PROVEN=4。
- 结论：**FAIL**。机器可读数据与声明统计不一致；类别总数也与声明不一致。所有汇总统计必须由同一 JSON 自动生成，禁止手工维护多套数字。

### R47-02 — 资源释放前缺少明确的授权与 Fencing 校验
- 位置：`RESOURCE-OPERATION-CRASH-RECOVERY.md §3`、`REQUEST-ID-ATOMIC-IDEMPOTENCY.md §2.3`
- 事实：新版本 `atomic_release()` 在调用 `operation()` 前未展示 `validate_token`、owner/resource 绑定校验或 Fencing 检查；相关检查只在旧的 `RESOURCE-RELEASE-SERIALIZATION.md` 05.44 伪代码出现。
- 风险：若 `run_txn()` 才做授权，业务副作用已经先发生，属于授权后置。
- 结论：**BLOCKER**。必须将可信身份、token、owner、resource、generation/Fencing 校验置于副作用之前，并证明其与接管操作具有同一串行化边界。

### R47-03 — start_req() 返回值被忽略
- 位置：`REQUEST-ID-ATOMIC-IDEMPOTENCY.md §2.2-2.3`、`RESOURCE-OPERATION-CRASH-RECOVERY.md §3`
- 事实：`start_req()` 定义了持久化失败返回 ABORT，但调用点没有检查返回值，随后继续获取资源锁并可能执行 operation。
- 风险：请求占用失败仍发生业务副作用，直接破坏“先成功占用、后执行”的安全前提。
- 结论：**BLOCKER**。必须检查并验证 OCCUPIED/ABORT/UNKNOWN/CONFLICT 等所有结果；只有确认占用已持久化后才可继续。

### R47-04 — 资源 operation 伪代码含未定义变量及语法错误
- 位置：`RESOURCE-OPERATION-CRASH-RECOVERY.md §3`
- 事实：`rec` 未定义却用于 OP_STARTED、OP_FAILED、OP_RELEASE；`owner_id`、`payload_hash` 未在该函数作用域定义；使用了未定义的 `confirmed_caller_owner`；OP_FAILED 行出现重复条件/冒号，代码结构不合法。
- 风险：协议伪代码无法作为可执行规格；关键失败路径不能被验证。
- 结论：**BLOCKER**。必须提供参数完整、语法可解析、每个 WAL record 显式构造且字段一致的唯一权威伪代码。

### R47-05 — OP_CONFIRM / OP_RELEASE 失败闭环不完整
- 位置：`RESOURCE-OPERATION-CRASH-RECOVERY.md §3-4`
- 事实：OP_CONFIRM fsync 失败直接返回 UNKNOWN，没有明确写入持久化隔离记录或建立可证明的进程内隔离状态；OP_RELEASE 使用非 fsync 的 `wal_append(rec,...)`，且 rec 未定义。
- 风险：业务副作用已执行、确认/释放记录未可靠持久化时，函数 finally 仍退出 in_flight 并释放资源锁；后续状态可能被错误地当作普通已处理请求。
- 结论：**BLOCKER**。UNKNOWN 必须是持久化状态机中的明确终态/待处理态；隔离失败时必须有 fail-closed 机制。所有提交点和释放点必须定义原子性、fsync 与恢复语义。

### R47-06 — 重复 req_id 被无条件报告为 ALREADY_RELEASED
- 位置：`RECOVERY-BARRIER-AND-ADMISSION.md §3`、`REQUEST-ID-ATOMIC-IDEMPOTENCY.md §2.3`
- 事实：只要 `lookup_req_in_global(req_id)` 返回记录，就返回 `ALREADY_PROCESSED ... ALREADY_RELEASED`，没有区分 RESERVED、OPERATION_STARTED、UNKNOWN、ISOLATED、COMMITTED、RELEASED，也没有比较 op/owner/resource/payload_hash。
- 风险：未完成或效果未知的操作被伪报为已释放；同一 req_id 携带不同参数可能被错误当作成功重放。
- 结论：**BLOCKER**。必须按持久化状态返回精确结果，并对同 req_id 不同参数返回冲突/隔离。

### R47-07 — WAL 从头恢复算法忽略关键 operation 记录
- 位置：`WAL-INTEGRITY-AND-CHECKPOINT.md §2.2`
- 事实：`seq_recs` 被构造和排序后没有被使用；循环明确跳过 PREPARE、OP_RESERVATION、OP_CONFIRM、OP_RELEASE；`req_id_index` 与 `operation_state_index` 初始化为空后没有被重建。
- 风险：恢复后丢失 req_id 占用与 operation 状态，幂等保护和非幂等副作用判断失效。
- 结论：**BLOCKER**。必须给出完整、确定的 WAL 状态折叠算法，按明确的 WAL 顺序重建所有索引，并对缺失/重复/乱序记录 fail closed。

### R47-08 — 检查点恢复没有按 wal_position 正确截断重放范围
- 位置：`WAL-INTEGRITY-AND-CHECKPOINT.md §2.2`
- 事实：检查点定义了 `wal_position`，但恢复伪代码未使用该字段；仅对 COMMIT/APPLY 比较 commit_seq，其他记录仍可能被重复处理；与检查点之后的 operation 记录如何重建索引没有定义。
- 风险：检查点之前的请求/operation 记录可能被重复折叠，检查点之后的状态又可能被漏掉或错序。
- 结论：**BLOCKER**。必须以经验证的 WAL 位置/序号边界定义恢复区间，并证明快照、索引、WAL 尾部属于同一一致性点。

### R47-09 — ImmutableSnapshot 并非真正不可变
- 位置：`POST-COMMIT-FAILURE-CLOSURE.md §2.1`
- 事实：`_state`/`_trt` 只是私有命名约定；Python 调用者仍可访问属性；构造函数保留传入的可变对象引用；property 返回原始 dict/list 等可变对象。
- 风险：调用者可通过原引用或 `snapshot.state` 修改已发布快照，所谓冻结没有成立。
- 结论：**BLOCKER**。必须深拷贝并递归冻结/使用不可变数据结构，禁止泄漏可变内部引用，并给出可验证测试。

### R47-10 — query_txn() 的 APPLY 校验仍不完整
- 位置：`POST-COMMIT-FAILURE-CLOSURE.md §2.2`
- 事实：只筛选与 COMMIT 相同 commit_seq 的 APPLY；不同 commit_seq 的 APPLY 被忽略；未要求 COMMIT 与 APPLY 的 snapshot_hash 一致；APPLY 缺失时仍返回 OK。
- 风险：冲突 APPLY 或未完成 APPLY 被误报为 OK。
- 结论：**BLOCKER**。必须扫描该 txn 的全部相关记录，验证唯一性、序号、hash、状态转移；缺少 APPLY 应返回明确的 COMMITTED_PENDING_APPLY，而非 OK。

### R47-11 — 锁顺序声明与跨文档权威协议不一致
- 位置：`RECOVERY-BARRIER-AND-ADMISSION.md §4`、`REQUEST-ID-ATOMIC-IDEMPOTENCY.md §5`、`RESOURCE-RELEASE-SERIALIZATION.md（05.44 文档）`
- 事实：05.46 声称 barrier_cond → global_request_lock → resource_lock → txn_guard；保留的 05.44 资源释放权威伪代码却先持 resource_lock，再调用 start_req/run_txn。05.46 也未证明 WAL 内部锁与上述锁的实际嵌套顺序。
- 风险：不同入口按不同顺序持锁时可能形成环路；仅列出一张锁顺序图不能证明实际调用链遵守它。
- 结论：**BLOCKER**。建立唯一锁层级表，逐函数标出 acquire/release 区间；所有路径必须符合全序，并做等待图/死锁反例推演。

### R47-12 — 文档声称“跨文档一致”但权威版本仍有旧协议
- 位置：`CROSS-DOC-CONSISTENCY.md` 与 `RESOURCE-RELEASE-SERIALIZATION.md`
- 事实：跨文档一致性报告只核对列出的字段与名称；资源释放文档仍标为 05.44 权威，伪代码采用不同锁顺序与旧流程。
- 风险：执行者无法判断冲突时哪个协议优先，05.46 的“唯一权威规则”不成立。
- 结论：**FAIL**。必须明确废止/替换旧权威文档，或在 05.46 中完整迁移并标记旧版仅供历史对照。

## 3. 统计口径的进一步矛盾

除 R47-01 外，`COUNTEREXAMPLE-RESULTS.md` 还存在：
- §3 叙述称 2 条 NOT_PROVEN；
- §4 实际列出 3 条 NOT_PROVEN；
- JSON 实际有 4 条 NOT_PROVEN（含 B46-RES-007）。
- 历史状态 524+14+7=545；按文档所称本轮新增 32 PASS、0 FAIL、0 NOT_PROVEN，累计状态应为 577，而非文档表格合计 580。
- JSON 实际新增 25 PASS、0 FAIL、4 NOT_PROVEN；不能与历史统计直接相加后仍沿用旧累计数。

## 4. 复审结论

**DESIGN-05.46：FAIL。**

- 不批准 IMPLEMENT-01。
- 不批准 A/B Merge。
- 不批准 Source Retire。
- 不将静态文档审查标记为运行时验证。
- 保持跨进程/跨重启能力 DENY/ISOLATED。
- 不修改 `usit/*.py` 源码，不执行真实资源操作。

## 5. DESIGN-05.48 必须完成的准入条件

05.47 作为独立审查记录；下一轮修订应至少完成：
1. 修正并自动生成反例 JSON、分类统计、状态统计、历史累计统计；所有断言必须由脚本验证。
2. 重写唯一的 `atomic_release()` 伪代码：可信身份与 Fencing 前置、req_id 原子占用、每个状态明确持久化、所有返回值检查、UNKNOWN/ISOLATED 不得伪报成功。
3. 统一锁层级和真实 acquire/release 区间，覆盖 recovery、request、resource、txn、WAL、checkpoint 全部路径。
4. 重写 WAL 从头恢复与检查点恢复算法，证明 COMMIT/APPLY、operation 状态、req_id 索引的完整重建。
5. 修复真正不可变快照与 query_txn 的唯一性/hash/待 APPLY 语义。
6. 为每项修复新增可复现反例；静态推演与运行时测试分别报告，不得混为一谈。
7. 交付跨文档一致性检查脚本、反例校验脚本、完整执行报告与 Git blob SHA256 manifest。

## 6. 范围声明

本审查以 GitHub 中实际读取的 05.46 文件为证据。未审查未读取的历史 05.33～05.45 全部原始反例，也未读取/运行当前 USIT 源码。因此历史 560 条统计及源码真实行为均不在本报告的独立确认范围内。
