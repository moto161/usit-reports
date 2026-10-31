# USIT-STAGE-B-DESIGN-05.29 — 身份时间线（IDENTITY-TIMELINES.md）

> B29-02 支撑文档。身份生命周期时间线，每条含：初始状态、事件、身份变化、判定、证据等级。

## 身份时间线（IT1~IT20）

### IT1 根进程启动
- 事件：os.urandom(16) 生成 process_instance_id + process_tree_id；判定：实例内/树内概率唯一；证据：STATIC_INFERENCE。

### IT2 根进程首次 fork（A）
- 事件：根 fork A，fork_epoch=1，A.tree_id=根.tree_id，A.instance_id=新值，A.parent_instance_id=根.instance_id；判定：兄弟区分（seq）；证据：STATIC_INFERENCE。

### IT3 根进程第二次 fork（B）
- 事件：根 fork B，fork_epoch=2；B 与 A 的 fork_event_id 因 seq 不同；判定：结构性不碰撞；证据：STATIC_INFERENCE。

### IT4 A 作为父 fork C
- 事件：A fork C，A 的 seq 从 1 起（独立于根），C.parent_instance_id=A.instance_id；判定：分支区分（父 instance 不同）；证据：STATIC_INFERENCE。

### IT5 B 与 C 身份比较
- 事件：B.fork_event_id=(根.tree_id, 根 seq=2, rand)，C.fork_event_id=(根.tree_id, A.instance_id, A seq=1, rand)；B 与 C 父 instance 不同；判定：不碰撞；证据：STATIC_INFERENCE。

### IT6 子进程重建 instance_id
- 事件：fork 返回后子立即重建 instance_id；判定：父子 instance_id 不同；证据：STATIC_INFERENCE。

### IT7 子进程继承 tree_id
- 事件：子继承 tree_id 不变；判定：同树；证据：STATIC_INFERENCE。

### IT8 子进程记录 parent_instance_id
- 事件：子保存父 instance_id 作历史；判定：可追溯；证据：STATIC_INFERENCE。

### IT9 子进程再 fork（代际 +1）
- 事件：孙 generation=子 generation+1；判定：单链代际；证据：STATIC_INFERENCE。

### IT10 PID 重用
- 事件：旧 pid 复用；instance_id 区分；判定：不依赖 pid；证据：VERIFIED_FACT。

### IT11 进程崩溃
- 事件：崩溃后身份丢失；判定：不持久；证据：STATIC_INFERENCE。

### IT12 系统重启
- 事件：重启换新 instance_id/tree_id；判定：跨重启不承诺；证据：STATIC_INFERENCE。

### IT13 随机源不可用
- 事件：os.urandom 失败；判定：DENY 不降级；证据：UNKNOWN。

### IT14 快照消费失效
- 事件：子消费 snapshot；snapshot_id 失效；判定：不复用；证据：STATIC_INFERENCE。

### IT15 旧快照复用拒绝
- 事件：新 fork 读旧 snapshot；fork_epoch 不匹配；判定：拒绝；证据：STATIC_INFERENCE。

### IT16 FD 槽位复用
- 事件：槽位重开，fd_generation 递增；判定：不混淆；证据：STATIC_INFERENCE。

### IT17 txn_id 持久化重放
- 事件：崩溃后重放 txn_id；判定：可重放校验；证据：STATIC_INFERENCE。

### IT18 revision 跨重启重读
- 事件：重启后重读 revision；判定：可重读；证据：STATIC_INFERENCE。

### IT19 恶意复制身份
- 事件：进程复制 instance_id；判定：无认证不防；证据：UNKNOWN。

### IT20 计数器溢出
- 事件：fork_epoch 溢出；判定：DENY 不回绕；证据：STATIC_INFERENCE。

## 多级 fork 分支推演（IT21~IT30）

### IT21 五代深链
- 事件：根→A→C→F→G，每代 parent_instance_id 指向上代；判定：全程可追溯；证据：STATIC_INFERENCE。

### IT22 兄弟多代交错
- 事件：A、B 各自繁衍，任意两节点分支不同或 seq 不同；判定：不碰撞；证据：STATIC_INFERENCE。

### IT23 父退出后子独立
- 事件：父退出，子 instance_id/tree_id 不变；判定：tree_id 可追溯根；证据：STATIC_INFERENCE。

### IT24 子进程脱离树后
- 事件：子独立运行，tree_id 保留；判定：同树历史保留；证据：STATIC_INFERENCE。

### IT25 身份记录损坏
- 事件：身份数据损坏；判定：UNKNOWN，进入 QUARANTINED；证据：UNKNOWN。

### IT26 旧身份恢复
- 事件：恢复旧身份数据；判定：跨重启不承诺，拒绝或标记；证据：STATIC_INFERENCE。

### IT27 外部导入身份
- 事件：导入外部身份；判定：无认证，UNKNOWN；证据：UNKNOWN。

### IT28 父计数器被 fork 复制
- 事件：fork 复制父 seq（05.28 缺陷）；判定：本轮子只读 fork_epoch 不继承计数器；证据：STATIC_INFERENCE。

### IT29 子进程重建身份失败
- 事件：重建抛错；判定：禁止业务，os._exit(1)；证据：STATIC_INFERENCE。

### IT30 随机 ID 意外重复
- 事件：随机碰撞（概率非零）；判定：概率性，非严格；证据：STATIC_INFERENCE。

## 结论

30 条身份时间线覆盖启动、fork、继承、重建、多代、PID 重用、崩溃、重启、随机失效、伪造等场景。协议层 DESIGN_PROVEN，运行时 NOT_TESTED。
