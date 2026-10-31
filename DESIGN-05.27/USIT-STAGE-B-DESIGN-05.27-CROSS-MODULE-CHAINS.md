# USIT-STAGE-B-DESIGN-05.27 跨模块链路（CROSS-MODULE-CHAINS）

> 独立材料。共 25 条链路（L1~L25），覆盖 Fork、Gate、FD、Candidate、PREPARED、current、事务身份、fsync、恢复、审计回执之间的依赖关系。

| # | 链路 | 依赖模块 | 关键依赖关系 |
|---|------|---------|-------------|
| L1 | 单线程 fork 成功→父返回 | Fork/Gate/Lock | Gate OPEN → freeze → fork → 恢复 Gate |
| L2 | fork 成功→子初始化→业务 | Fork/Gate/FD | 子继承快照 + 重建锁 + FD generation 匹配 |
| L3 | fork 失败→Gate 恢复 | Fork/Gate | except → _close_fork_parent → Gate OPEN/DENIED |
| L4 | 父回调异常→恢复 | Fork/Gate/Lock | finally → _close_fork_parent → 锁释放 |
| L5 | 子初始化失败→os._exit | Fork | 子回调异常 → os._exit(1) |
| L6 | 多线程拒绝 fork | Fork/Gate | 线程判定失败 → Gate DENIED |
| L7 | 非主线程拒绝 | Fork | 主线程检查 → 拒绝 |
| L8 | 连续 fork | Fork/Epoch | epoch 单调 + L0 串行 |
| L9 | 子 FD generation 关闭 | FD/Fork | snapshot 遍历 + generation 匹配 |
| L10 | 第三方 C fork DENY | Fork | at-fork 不覆盖 → DENY |
| L11 | epoch 溢出 DENY | Epoch/Fork | 计数器溢出 → DENY |
| L12 | 随机源降级 | Identity/Fork | os.urandom 失败 → 降级/拒绝 |
| L13 | S1→S4 Candidate 就绪 | Candidate/fsync | O_EXCL → 内容 fsync → 目录 fsync → 校验 |
| L14 | S5→S7 PREPARED 持久 | PREPARED/fsync | 写入 → 内容 fsync → 目录 fsync |
| L15 | S9 五条件校验 | current/事务身份 | 五条件 + 六字段/五谓词 |
| L16 | S12 rename | current/rename | rename 原子性（非持久）|
| L17 | S13 目录持久 | current/fsync | fsync(dirfd) 成功 |
| L18 | S13 失败恢复 | current/恢复 | 全对象核验 → 重试/INDETERMINATE |
| L19 | S14/S15 回执持久 | 回执/fsync | 回执内容 fsync + 目录 fsync |
| L20 | S16 释放锁 | 事务/锁 | 锁释放非持久化 |
| L21 | 身份冲突隔离 | Identity/恢复 | 身份冲突 → QUARANTINED |
| L22 | 目录 fsync 统一恢复 | fsync/恢复 | S3/S7/S13/S15 统一协议 |
| L23 | 15 项证明门槛 | 恢复/安全门 | 删除/重试前 15 项证明 |
| L24 | 交付哈希闭环 | 交付/MANIFEST | MANIFEST→RECEIPT→ZIP |
| L25 | ZIP 独立核验 | 交付/核验 | 解压→CRC32→逐字节比对 |

---

## 统计

**跨模块链路合计：25 条（L1~L25）。** 每条覆盖至少两个模块之间的依赖关系。
