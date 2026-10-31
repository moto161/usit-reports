# USIT-STAGE-B-DESIGN-05.28 设计差异（DIFF）

> 记录 05.28 相对 05.27 的逐项设计变化。

## 1. B28-01 Fork（相对 05.27 §1）

| 05.27 | 05.28 |
|-------|-------|
| `_restore_gate_fail_closed()` 失败后 `_gate='DENIED'`（可能被跳过）| 先无条件 `_gate='DENIED'` 再尝试恢复（赋值序保证）|
| `_close_fork_parent` 抛异常跳锁释放 | 锁释放移至最外层 finally + `_terminate_fail_closed` 兜底 |
| finally 异常覆盖原始异常未处理 | `_last_fork_error` 独立记录，Gate 终态不依赖异常传播 |
| "所有入口执行同一拒绝协议"仅文字 | §1.3 原子准入协议（检查+登记同临界区）|
| 无 TOCTOU 防护 | enter_op 原子准入 + 副作用前二次轻量重检 |

## 2. B28-02 身份（相对 05.27 §2）

| 05.27 | 05.28 |
|-------|-------|
| 父侧全局严格单调 fork_epoch（被推翻）| fork_event_id = 父 instance_id + 父内 seq + 随机后缀 |
| 子继承父计数器继续递增（碰撞源）| 子只读取 fork_event_id 值，各代 seq 独立 |
| 兄弟碰撞"结构性消除"论证不成立 | instance 前缀 + 父内 seq 结构性消除 |
| 进程树唯一"承诺严格" | 明确进程树内 fork_event_id 严格、跨树概率 |

## 3. B28-03 POSIX（相对 05.27 §3）

| 05.27 | 05.28 |
|-------|-------|
| S10/S11 无独立状态 | CURRENT_TMP_CREATED / CONTENT_DURABLE / VERIFIED |
| 16 态状态机 | 20 态（补临时文件三态 + 回执创建态）|
| S14 无幂等协议 | §3.5 回执唯一键 (txn_id,revision) 幂等 |
| EEXIST 删半成品条件模糊 | §3.7 15 项安全门强制 |
| 两次读一致当无并发修改 | §3.6 两次读 + 锁持有同时成立 |

## 4. B28-04 数量（相对 05.27）

| 项 | 05.27 | 05.28 |
|----|-------|-------|
| Fork 异常时间线 | 20 | 25 |
| 身份时间线 | 15 | 20 |
| POSIX 时间线 | 16 | 30 |
| 反例 | 40 | 90 |
| 跨模块链路 | 26 | 30 |
| 不变量 | 50 | 60 |

## 5. 无变化项

- P0-05 Source Retire：DENY（保持）
- IMPLEMENT-01：NOT_APPROVED（保持）
- A/B Merge：NOT_APPROVED（保持）
- Windows ReplaceFileW 1175/1176/1177 语义：保持 05.26/05.27 已纠正结论
