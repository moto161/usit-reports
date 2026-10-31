# CRASH-RECOVERY-PROTOCOL.md — 崩溃恢复协议（05.37）

证据等级：VERIFIED_STATIC（含 NOT_PROVEN 项，如实标注）
CAS 恢复协议见 RECOVERY-CAS-PROTOCOL.md（权威）；本文件定义恢复的证据来源、状态机与决策矩阵。

---

## 1. 05.36 缺陷定位

05.36 的恢复伪代码存在：
1. `allowed`、`denied_evidence` 来源不明确（未定义布尔变量）。
2. 通过普通 `commit(state)` 提交状态，无 CAS、无版本校验，可能覆盖并发更新。

05.37 修复：明确证据清单、恢复状态机、带版本校验的 CAS（见 RECOVERY-CAS-PROTOCOL.md）。

---

## 2. 恢复输入证据清单

每项证据必须注明：来源、采集时间、版本、可信等级。

| 证据 | 来源 | 采集时间 | 版本 | 可信等级 |
|---|---|---|---|---|
| 状态字当前值 | 原子状态字 load() | 恢复开始 | 状态版本号 | 直接（内存） |
| 状态版本号 | 状态字 version 字段 | 恢复开始 | — | 直接 |
| DENIED 证据 | 状态字 DENIED 位 + 持久化审计日志 | 恢复开始 | 审计链版本 | 高（若审计根可信） |
| 持久化事务记录 | 磁盘 Receipt/Commit 日志 | 恢复扫描 | 日志序列号 | 高（落盘） |
| 旧写入者清单 | 进程表 + 租约 + 心跳 | 恢复开始 | 租约版本 | 中（可能 UNKNOWN） |
| 锁外写入者枚举 | WRITE-ENTRY 覆盖清单 | 恢复开始 | 清单版本 | UNKNOWN（未完备） |

---

## 3. 恢复状态机

| 状态 | 语义 | 进入条件 | 退出到 |
|---|---|---|---|
| RECOVERY_NOT_STARTED | 未开始 | 初始/崩溃后 | SCANNING |
| SCANNING | 采集证据 | 恢复开始 | VALIDATING |
| VALIDATING | 校验证据一致性 | 证据采集完成 | REBUILDING / DENY |
| REBUILDING | CAS 重建状态字 | 证据充分且一致 | COMPLETED / DENY |
| COMPLETED | 恢复成功 | CAS 成功 | （终态） |
| DENY | 证据不足/冲突/不一致 | 校验失败 | （终态，保持隔离） |

**关键**：`allowed` 只能由 VALIDATING 阶段的已验证条件计算（见 §5），不得使用未定义布尔变量。

---

## 4. 崩溃类型与恢复决策矩阵

| 崩溃类型 | 状态字影响 | 事务影响 | 恢复策略 |
|---|---|---|---|
| 线程异常（捕获） | 不变 | 该线程事务半途 | 线程级清理（finally + 幂等清理） |
| 线程取消 | 不变 | 同上 | 同上 |
| 进程正常退出 | 进程内状态字消失 | 事务已/未提交需判定 | 无共享介质则无需恢复；有则走 §5 |
| 进程崩溃 | 进程内消失，共享介质存活 | 遗留事务不明 | §5/§6，无法证明则 DENY |
| 系统崩溃/断电 | 内存全失，磁盘证据部分落盘 | 提交可能未 fsync | §6 事务决策树，仅磁盘证据 |
| 快照回滚/磁盘恢复 | 内存重载，磁盘回旧点 | 回滚点后事务丢失 | §6，身份防回滚约束 |

---

## 5. `allowed` 的计算（消除未定义变量）

`allowed`（允许重建/归零）必须由以下**全部**条件同时满足才为 True：

```python
def compute_allowed(evidence):
    # 每个条件都是可验证的，不存在未定义布尔变量
    if evidence.shared_medium and not evidence.all_old_writers_excluded():
        return False   # 旧写入者未排除
    if evidence.has_unknown_transactions():
        return False   # 存在 UNKNOWN 事务
    if evidence.state_version_mismatch():
        return False   # 状态版本不一致
    if evidence.denied_evidence and not evidence.denied_verified():
        return False   # DENIED 证据未验证
    if evidence.conflicting_records():
        return False   # 证据冲突
    return True
```

`denied_evidence` 的读取、验证、提交必须防止 TOCTOU（见 RECOVERY-CAS-PROTOCOL.md §5）。

---

## 6. 事务恢复决策树

```
事务状态（按磁盘证据）？
├─ COMMITTED_CONFIRMED（落盘确认 + 创建者认证闭合）
│    → 安全，不重放，不计入 count
├─ COMMITTED_UNCONFIRMED（提交已写无确认）
│    → 状态不明 → 隔离，不自动重放
├─ IN_FLIGHT（进行中未提交）
│    → 无活跃写入者且可安全回滚 → 回滚；否则隔离
├─ QUARANTINED（未认证 Receipt）
│    → 保持隔离，不参与成功恢复
└─ UNKNOWN（证据缺失/损坏）
     → ISOLATED，禁止自动重试或放行
```

**强制规则**：共享状态计数器不得仅凭进程退出重置为 0；旧业务/旧写入者/异步任务/遗留事务未全部排除时，禁止将计数重建为 0；无法确认 Commit/Receipt 是否完成时，不自动重放。

---

## 7. 恢复失败 / 中断 / 恢复者崩溃 / 重复恢复

- **恢复失败**：保持 DENY/隔离，不恢复 OPEN。
- **恢复中断**：状态保持 VALIDATING 或 DENY，下次恢复从 SCANNING 重新开始（幂等）。
- **恢复者崩溃**：恢复操作本身无副作用（CAS 未提交则无变化），下次恢复重做，幂等。
- **重复恢复**：CAS 协议保证幂等——若状态已被其他恢复者/线程重建，版本号变化使重复恢复的 CAS 失败，重读最新状态。

---

## 8. NOT_PROVEN 边界（如实标注）

- 锁外写入者完备枚举：UNKNOWN → 跨进程共享场景下「旧写入者全部排除」证明不完整 → 共享归零路径 NOT_PROVEN → Fork DENY。
- 跨快照身份唯一性：NOT_PROVEN（继承）。
- Receipt 创建者认证：NOT_PROVEN（继承）。
- Break-glass 审计信任根：NOT_PROVEN（继承）。

---

## 证据来源

- 恢复 CAS 协议：RECOVERY-CAS-PROTOCOL.md
- 状态字作用域：ATOMIC-STATE-WORD.md §8（05.36，继承）
- 证明等级：VERIFIED_STATIC + NOT_PROVEN（旧写入者完备枚举）
