# LOCK-ORDER-PROOF.md

## 目标
锁依赖有向图、全局锁顺序、死锁反例（20 条）与消除规则。

## 1. 全局锁顺序（必须遵守）

```
身份分配锁 (_identity_alloc_lock)
  → 准入锁 (_admit_lock)
    → 提交锁 (_commit_lock)
      → 业务资源锁
        → Receipt 写入锁
  → Fork 协调锁 (_fork_coord_lock)   [独立，不与业务锁嵌套]
```

**规则**：Fork 协调锁与业务锁（admit/commit/业务资源）**不同时持有**，避免交叉死锁。身份分配锁在 fork 前释放，不在 fork 协调锁内持有。

## 2. 锁依赖有向图

```
_identity_alloc_lock ──> _admit_lock ──> _commit_lock ──> 业务资源锁 ──> Receipt锁
                              │
_fork_coord_lock (独立，串行化 fork 准入，不嵌套业务锁)
```

## 3. 死锁反例（20 条）

| # | 反例 | 消除规则 |
|---|---|---|
| 1 | 线程 A 持 admit 等 coord，线程 B 持 coord 等 admit | 禁止 coord 与 admit 嵌套 |
| 2 | coord 与 commit 交叉持有 | 禁止嵌套，短持有 commit |
| 3 | 身份分配锁在 coord 内持有 | fork 前释放身份锁 |
| 4 | 业务资源锁与 commit 顺序相反 | 统一全局顺序 |
| 5 | 条件变量 wait 时持锁等待 | wait 前释放锁，唤醒重取 |
| 6 | 父子共享 OS 锁 | OS 锁子进程重取，不继承 |
| 7 | fork 时持有 admit，子进程沿用 | 子进程重建 admit |
| 8 | 双线程并发 fork | coord 串行化 |
| 9 | fork 与提交并发死锁 | coord 前先等提交完成 |
| 10 | Receipt 锁与 commit 顺序相反 | 统一 commit→Receipt |
| 11 | 身份锁与 coord 循环 | 禁止循环依赖 |
| 12 | 未持锁释放 | 释放前校验所有权 |
| 13 | 跨线程释放 | 禁止，锁归属线程 |
| 14 | 重复释放 | 释放后置空引用 |
| 15 | fork 前持锁子继承死锁 | 子进程重建所有锁 |
| 16 | 条件变量 notify 前未重取锁 | notify 需持关联锁 |
| 17 | 多锁获取部分失败未回滚 | 获取失败回滚已获锁 |
| 18 | 协调锁超时未释放 | finally 保证释放 |
| 19 | 身份分配锁 fork 复制 | 子重建 + 归零偏移 |
| 20 | 业务线程在 DENIED 后继续持锁 | DENIED 前先清空 in_flight |

## 4. 结论
全局锁顺序 + 禁止 coord 与业务锁嵌套 + 子进程重建锁，消除全部 20 条死锁反例。

## 5. 证据等级
锁顺序与死锁消除：DESIGN_PROVEN（逻辑）；真实运行时死锁：NOT_TESTED。
