# CRASH-RECOVERY-PROTOCOL.md — 崩溃恢复协议（05.38 修订）

证据等级：VERIFIED_STATIC（含 NOT_PROVEN 项，如实标注）
恢复 CAS 见 RECOVERY-CAS-PROTOCOL.md（权威）；版本原子性见 RECOVERY-VERSION-ATOMICITY.md。

---

## 1. 与 05.37 的差异

05.37 的恢复依赖「独立版本 + 无条件 commit」。05.38 统一为「状态+版本合并 64 位原子字 + cas_state_version」，并明确证据新鲜度、重试上限、DENIED 单调语义。

---

## 2. 恢复输入证据清单（含来源/时间/版本/可信等级）

| 证据 | 来源 | 采集时间 | 版本 | 可信等级 |
|---|---|---|---|---|
| 最新状态字 + 版本 | load_state_version() | 每次 CAS 循环 | 原子版本号 | 直接 |
| DENIED 持久化标志 | 审计日志 | 恢复扫描 | 审计链版本 | 高（若审计根可信） |
| 事务记录 | 磁盘 Receipt/Commit 日志 | 恢复扫描 | 日志序列号 | 高（落盘） |
| 旧写入者清单 | 进程表 + 租约 + 心跳 | 恢复扫描 | 租约版本 | 中（可能 UNKNOWN） |
| 锁外写入者枚举 | WRITE-ENTRY 清单 | 恢复扫描 | 清单版本 | UNKNOWN（未完备） |

---

## 3. 恢复状态机

| 状态 | 语义 | 进入 | 退出到 |
|---|---|---|---|
| NOT_STARTED | 未开始 | 初始/崩溃 | SCANNING |
| SCANNING | 采集证据 | 恢复开始 | VALIDATING |
| VALIDATING | 校验证据 | 采集完成 | REBUILDING / DENY |
| REBUILDING | cas_state_version | 证据充分一致 | COMPLETED / CONFLICT / DENY |
| COMPLETED | 成功 | CAS 成功 | 终态 |
| DENY | 证据不足/冲突 | 校验失败 | 终态（隔离） |

---

## 4. 崩溃类型与恢复决策矩阵（继承 05.37，补充版本维度）

| 崩溃类型 | 状态字 | 版本 | 事务 | 策略 |
|---|---|---|---|---|
| 线程异常 | 不变 | 不变 | 该线程半途 | 线程级清理 |
| 线程取消 | 不变 | 不变 | 同上 | 同上 |
| 进程正常退出 | 进程内消失 | 消失 | 需判定 | 无共享介质则无需恢复 |
| 进程崩溃 | 共享介质存活 | 存活（若共享） | 遗留不明 | §5/§6，无法证明则 DENY |
| 系统崩溃/断电 | 内存失，磁盘部分落盘 | 内存版本失 | 提交可能未 fsync | §6，仅磁盘证据 |
| 快照回滚 | 内存重载 | 版本回绕 | 回滚点后丢失 | 版本回绕 NOT_PROVEN → DENY |

---

## 5. compute_allowed（可验证条件，无未定义变量）

```python
def compute_allowed(e):
    if e.shared_medium and not e.all_old_writers_excluded(): return False
    if e.has_unknown_transactions(): return False
    if e.state_version_mismatch(): return False
    if e.denied_evidence and not e.denied_verified(): return False
    if e.conflicting_records(): return False
    if e.version_atomicity_not_proven(): return False   # 跨进程/持久化未证明
    return True
```

---

## 6. 事务恢复决策树（继承，不变）

```
COMMITTED_CONFIRMED → 安全，不重放，不计入 count
COMMITTED_UNCONFIRMED → 隔离，不重放
IN_FLIGHT → 可安全回滚则回滚，否则隔离
QUARANTINED → 隔离
UNKNOWN → ISOLATED，不重试不放行
```

**强制**：共享计数器不得仅凭进程退出重置为 0；旧写入者/异步任务/遗留事务未排除时禁止重建为 0；无法确认 Commit/Receipt 不自动重放。

---

## 7. NOT_PROVEN 边界（如实标注）

- 跨进程共享状态字 64 位 CAS 原子性：NOT_PROVEN（未证明目标平台 HW 支持）。
- 版本回绕 ABA 防护（无持久化 epoch）：NOT_PROVEN。
- 跨重启版本持久化一致性：NOT_PROVEN。
- 锁外写入者完备枚举：UNKNOWN。

在这些条件下，跨进程恢复、跨重启恢复 DENY；仅单进程内恢复可证明。

---

## 证据来源

- 恢复 CAS：RECOVERY-CAS-PROTOCOL.md
- 版本原子性：RECOVERY-VERSION-ATOMICITY.md
- 证明等级：VERIFIED_STATIC + NOT_PROVEN
