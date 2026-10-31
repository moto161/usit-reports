# USIT-STAGE-B-DESIGN-05.29 — 进程树身份模型（PROCESS-IDENTITY.md）

> B29-02 支撑文档。撤销 05.28 的身份结论，重新证明进程树身份模型。

## 1. 撤销声明

05.28 的以下结论被撤销：

1. "fork_event_id 进程树内严格唯一"——其"父 instance 前缀 + 父内 seq + 随机后缀"中，instance 前缀继承语义与"各代 seq 独立"矛盾，未闭合。
2. "fork_epoch 父侧全局单调"——被 fork 复制计数器推翻。
3. process_instance_id 继承/重建关系未定义。

本轮重新分层定义并重新证明。

## 2. 身份层次（八层分离）

| 层 | 身份 | 唯一性范围 |
|----|------|-----------|
| A | Process Instance Identity | 进程实例内 |
| B | Process Tree Identity | 进程树内 |
| C | Fork Event Identity | 单次事件 |
| D | Snapshot Identity | 绑定 fork 事件 |
| E | Transaction Identity | 实例/主机（视持久化）|
| F | Parent Relationship | 结构关系 |
| G | Generation Identity | 单链 |
| H | PID Identity | 进程存活期 |

**禁止将八类混为一个 ID。**

## 3. 字段定义

| 字段 | 定义 | 生成主体 | 唯一性范围 | 结构/概率 | 继承 | 重建 | 持久化 | 崩溃恢复 | PID 重用 | 失败降级 |
|------|------|---------|-----------|----------|------|------|--------|---------|---------|---------|
| process_instance_id | 进程实例身份 | 进程启动 os.urandom(16) | 实例内 | 概率 | 否 | 是（子新建）| 否 | 重启换新 | 结构规避 | 随机不可用 DENY |
| process_tree_id | 进程树根身份 | 根进程 os.urandom(16) | 树内 | 概率 | 是（同树）| 否 | 否 | 根重启换新 | 结构 | 随机不可用 DENY |
| process_generation | 代际 | 父 fork | 单链 | 结构 | 继承+1 | 是 | 否 | 不持久 | 结构 | 无关 |
| fork_epoch | 父内 fork 序号 | 父 fork | **父内严格** | 结构 | 只读 | 是 | 否 | 不持久 | 结构 | 溢出 DENY |
| fork_event_id | fork 事件身份 | 父 fork | 树内结构/跨树概率 | 结构+概率 | 是（历史值）| 否 | 否 | 不持久 | 结构+概率 | 随机 DENY |
| snapshot_id | 快照身份 | = fork_event_id | 同 fork_event_id | 同 | 是 | 否 | 否 | 不持久 | 同 | 同 |
| parent_instance_id | 父实例引用 | 子 fork | 引用 | 结构 | 是（历史）| 否 | 否 | 不持久 | 结构 | 无关 |
| parent_snapshot_id | 父快照引用 | 子 fork | 引用 | 结构 | 是 | 否 | 否 | 不持久 | 结构 | 无关 |
| fd_generation | FD 槽位代际 | 槽位分配 | 进程内每槽位 | 结构 | 继承后重建 | 是 | 否 | 不持久 | 结构 | 无关 |
| txn_id | 事务身份 | 事务开始 | 实例（概率）+可持久 | 概率+结构 | 否 | 是 | **是** | 可重放 | 结构 | 随机 DENY |
| revision | 对象版本 | 提交 | 实例内每对象 | 结构 | 否 | 是 | 是 | 可重读 | 结构 | 无关 |
| pid | OS 进程号 | OS | 进程存活期 | 结构（OS）| 否 | 是 | 否 | OS 复用 | 结构 | 无关 |

## 4. process_instance_id 继承与重建（核心）

**结论：fork 出的子进程拥有新的 process_instance_id；process_tree_id 才沿用于同树。**

| 时点 | process_instance_id | process_tree_id |
|------|--------------------|-----------------|
| fork 返回前 | 子内存复制父值（未确认）| 同父 |
| fork 返回后 | **重建新值** | 继承 |
| 子初始化前 | 未确认 | 继承 |
| 子初始化完成 | 新值确认 | 继承 |
| 子初始化失败 | 无效 | — |
| 子再 fork | 作父，再重建 | 继承 |
| 父退出 | 不变 | 保留 |
| 子独立运行 | 独立 | 保留 |
| PID 重用 | 区分 | 区分 |

**继承规则**：
- 允许继承：process_tree_id、parent_instance_id（历史）、parent_snapshot_id、fork_event_id（历史值）。
- 必须重建：process_instance_id、pid、fd_generation、start_identity。
- 保留原值作历史：parent_instance_id、parent_snapshot_id。
- 父子关联：parent_instance_id = 父 instance_id。
- 重建失败 → 禁止业务，os._exit(1)。
- 随机源不可用 → DENY。
- 重建前后混用 → 初始化完成标志阻断（未完成前 enter_op 拒绝）。

## 5. 唯一性分列证明

| 目标 | 标签 | 证明 |
|------|------|------|
| 单实例内 | STRICT_STRUCTURAL | 进程内单调计数器 |
| 同父兄弟 fork | STRICT_STRUCTURAL | 父内 fork_epoch 严格单调 |
| 同树多代 fork | STRICT_STRUCTURAL | tree_id（树）+ 父 instance_id（分支）+ 父内 seq（兄弟）|
| 不同树之间 | PROBABILISTIC | tree_id 128-bit 随机 |
| 同主机不同进程 | PROBABILISTIC | instance/tree 随机 |
| PID 重用 | STRICT_STRUCTURAL | instance_id 独立 pid |
| 进程崩溃 | NOT_GUARANTEED | 不持久 |
| 系统重启 | NOT_GUARANTEED | 不承诺 |
| 随机源失效 | UNKNOWN | DENY 不降级 |
| 身份复制/导入 | UNKNOWN | 无认证 |

**fork_epoch 不再是进程树全局计数器**，它只是父进程内严格序号。

## 6. 结论

身份层次分离、父子继承/重建规则、唯一性分列证明均闭合。协议层 DESIGN_PROVEN；跨主机/跨重启/随机降级 NOT_VERIFIED。
