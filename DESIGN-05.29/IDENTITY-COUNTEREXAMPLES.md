# USIT-STAGE-B-DESIGN-05.29 — 身份反例（IDENTITY-COUNTEREXAMPLES.md）

> B29-02 支撑文档。30 个身份反例，每条含：场景、初始身份、事件顺序、身份变化、预期唯一性、实际风险、是否碰撞、拒绝条件、最终结论。

## 身份反例（CI1~CI30）

### CI1 根进程创建两个子进程（兄弟）
- 场景：根 fork A、B；初始 根 tree_id=T；事件 根 fork A(seq=1)、B(seq=2)；身份变化 A.fork_event_id=(T,1,rand)、B=(T,2,rand)；预期 严格不碰撞；风险 无；碰撞 否；拒绝 无；结论 结构消除。

### CI2 兄弟进程分别创建下一代
- 场景：A fork C、B fork D；事件 C=(T,A.inst,1)、D=(T,B.inst,1)；预期 不碰撞（父 instance 不同）；碰撞 否；结论 结构消除。

### CI3 多代交错 Fork
- 场景：三代交错；事件 各代父 instance 前缀区分；碰撞 否；结论 结构消除。

### CI4 父子同时产生事件
- 场景：父 fork 与子 fork 并发；事件 各自父内 seq 独立；碰撞 否；结论 父内串行 + 分支区分。

### CI5 多线程同时申请身份
- 场景：两线程 fork；事件 父内 seq 需锁串行；碰撞 否（若锁）；拒绝 无锁则 DENY；结论 需父锁串行。

### CI6 父计数器被 fork 复制（05.28 缺陷）
- 场景：fork 复制 seq；事件 子继承 seq 后继续递增，与兄弟重叠；碰撞 **是**（05.28）；结论 本轮改为子只读 fork_epoch、不继承计数器，消除。

### CI7 子进程沿用父 process_instance_id
- 场景：子沿用父 instance_id；事件 父子 instance_id 相同；预期 应不同；碰撞 是（概念混淆）；结论 本轮子重建 instance_id，消除。

### CI8 子进程重建身份失败
- 场景：重建抛错；事件 身份未确认；预期 拒绝业务；结论 os._exit(1)，禁止沿用旧身份。

### CI9 随机源返回错误
- 场景：os.urandom 失败；预期 拒绝；结论 DENY 不降级。

### CI10 随机 ID 意外重复
- 场景：随机碰撞；预期 概率非零；结论 概率性，非严格，需结构分量兜底。

### CI11 PID 重用
- 场景：旧 pid 复用；身份 instance_id 区分；碰撞 否；结论 不依赖 pid。

### CI12 父进程崩溃
- 场景：父崩溃；身份 子 instance_id/tree_id 保留；碰撞 否；结论 树可追溯。

### CI13 子进程脱离原进程树
- 场景：子独立；tree_id 保留；碰撞 否；结论 历史可追溯。

### CI14 进程重启
- 场景：重启换新身份；碰撞 跨重启不承诺；结论 不承诺。

### CI15 快照被重复引用
- 场景：快照复用；snapshot_id 已失效；碰撞 是；结论 拒绝复用。

### CI16 txn_id 与 fork_event_id 混用
- 场景：两身份混用；碰撞 是（语义混淆）；结论 独立命名空间。

### CI17 身份记录损坏
- 场景：身份数据损坏；结论 UNKNOWN，QUARANTINED。

### CI18 旧身份恢复
- 场景：恢复旧身份；碰撞 跨重启；结论 拒绝或标记。

### CI19 外部导入身份
- 场景：导入外部身份；结论 UNKNOWN 无认证。

### CI20 恶意复制身份数据
- 场景：伪造身份；结论 UNKNOWN 不防。

### CI21 父内 seq 无锁并发分配
- 场景：两线程 fork 无锁；seq 冲突；碰撞 是；结论 需父锁串行。

### CI22 fork_epoch 冒充全局计数器
- 场景：fork_epoch 当进程树全局；碰撞 是（05.28）；结论 撤销，仅父内。

### CI23 process_generation 当全局代际
- 场景：generation 当全局；不同分支同代混淆；结论 仅单链语义。

### CI24 revision 与 fork_epoch 混用
- 场景：对象版本与 fork 计数混淆；结论 独立命名空间。

### CI25 snapshot_id 与 fork_event_id 分离
- 场景：两者独立；快照与 fork 脱钩；结论 snapshot_id=fork_event_id。

### CI26 父 instance_id 在 fork 后可变
- 场景：篡改父 instance 引用；追溯断裂；结论 只读不可变。

### CI27 FD 槽位复用不递增 generation
- 场景：槽位重开 generation 不变；旧 FD 误指；结论 递增 generation。

### CI28 崩溃后 seq 未重置
- 场景：持久化 seq 跨重启；旧身份复用；结论 重启 seq 归零 + 新 instance_id。

### CI29 身份字段无生成失败拒绝
- 场景：生成失败静默降级；身份不可靠；结论 显式 DENY。

### CI30 结构唯一与概率唯一混写
- 场景：随机标严格；结论 分列证明，禁止混写。

## 结论

30 个身份反例覆盖兄弟、多代、父子并发、计数器复制、PID 重用、崩溃、重启、伪造等。05.28 的 CI6（计数器复制）与 CI7（沿用 instance_id）是核心缺陷，本轮已结构性消除。协议层 DESIGN_PROVEN，运行时 NOT_TESTED。
