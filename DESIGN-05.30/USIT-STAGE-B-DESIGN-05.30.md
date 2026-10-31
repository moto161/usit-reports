# USIT-STAGE-B-DESIGN-05.30

## 0. 工单与结论摘要

| 项 | 值 |
|---|---|
| 工单编号 | USIT-STAGE-B-DESIGN-05.30 |
| 前置工单 | DESIGN-05.29 |
| 前置结论 | FAIL |
| 工作类型 | STRICT READ-ONLY ANALYSIS + DESIGN REVISION + INDEPENDENT CROSS-REVIEW |
| 本轮目标 | 修复 05.29 复审确认的四组阻断 + 一致性修复 |
| 整体结论 | PASS-WITH-ISSUES（协议层闭合；Receipt 创建者认证边界 UNKNOWN） |

本轮不是为 05.29 辩护，也不是全量重设计 STAGE-B。仅对四组被点名的阻断协议做**结构性修订**，保留 05.29 已通过复核的部分。

四组核心阻断：
- **B30-01**：DENIED 不可逆性 —— 修复「DENIED → restore 成功 → OPEN」的矛盾。
- **B30-02**：进程树身份结构唯一性 —— 修复「概率分量冒充结构唯一」。
- **B30-03**：Receipt 真实性 —— 修复「四元组冒充创建者认证」。
- **B30-04**：Fork 锁生命周期 —— 修复「获取/所有权/释放/重建不完整」。

---

## 1. 权限与执行边界

本轮为**只读设计修订**。允许：读取 05.28/05.29 交付与源码、静态分析、写伪代码/状态机/证明/反例、生成项目外文档与 ZIP、机器统计与 SHA-256 校验。

严格禁止（全部遵守）：修改源码 / 修改配置与运行数据 / 打补丁 / 构建编译安装 / 真实 `os.fork()` / 真实 POSIX 提交、rename、fsync、flock、fcntl / Source Retire / A/B Merge / IMPLEMENT-01 / 修改历史基线 / 覆盖历史交付物。

平台 `hasattr(os, 'fork') == False`：所有 fork 相关结论均 `NOT_TESTED`，绝不冒充运行时验证。

---

## 2. B30-01｜DENIED 不可逆性

### 2.1 已确认缺陷

05.29 §1.7 伪代码序列：
```
_gate = 'DENIED'
...
_restore_gate()
...
_gate = 'OPEN'
```
违反 DENIED 不可逆。本轮修改**实际状态转换**，而非措辞。

### 2.2 DENIED 语义重新分层（五类错误状态，不再混为单一 DENIED）

| 类别 | 状态 | 是否可逆 | 恢复路径 |
|---|---|---|---|
| 临时操作失败 | FREEZING 超时、DRAINING 中断 | 可逆 | 普通恢复协议 |
| 恢复失败 | RESTORE_FAILED | 部分可逆 | 受控重试（不跨 DENIED） |
| 安全状态无法确定 | OUTCOME_UNKNOWN / DURABILITY_UNKNOWN | 保持不确定态 | 隔离 + 人工 |
| 永久拒绝 | DENIED | **不可逆** | 仅人工授权管理协议 |
| 进程终止 | TERMINATING | 不可逆 | 进程退出 |

### 2.3 强制状态转换规则

- `DENIED → OPEN`：普通协议**禁止**。
- `DENIED → FREEZING / RESTORING`：禁止。
- `DENIED → TERMINATING`：允许，责任主体 = 主线程终止协调器。
- `DENIED → OPEN` 唯一路径 = 独立管理协议 `manual_reenable()`，须：产生审计记录、持有独立授权锁、禁止普通业务线程调用、禁止 finally/回调/清理函数触发。

**任何 finally、异常处理器、回调、清理函数不得将 DENIED 改成 OPEN。**

完整状态机见 `DENIED-STATE-MACHINE.md`，父进程恢复协议见 `PARENT-RESTORE-PROTOCOL.md`。

### 2.4 修复后的父进程恢复核心伪代码（摘要）

```
def _close_fork_parent(self):
    try:
        if self._fork_err is not None:          # fork 系统调用失败
            self._enter_terminal('RESTORE_FAILED', self._fork_err)   # 非 DENIED
            return
        self._restore_gate_after_success()      # 仅当 gate 仍为 RESTORING 才回 OPEN
    except Exception as e:
        self._record_secondary_exception(e)     # 附加异常，不覆盖主异常
        if self._gate == 'DENIED':
            return                               # DENIED 不可逆，绝不动
        self._enter_terminal('RESTORE_FAILED', self._primary_exc or e)
    finally:
        self._release_owned_locks()             # 只释放本线程持有的锁，失败仅记录
```

关键不变量：`_enter_terminal('DENIED')` 之后，任何代码路径都不再写回 OPEN。

### 2.5 异常优先级

主异常（原始业务/Fork 异常）> 恢复异常 > 清理异常。finally 二次异常仅作为附加异常记录，不覆盖主异常，不影响已确定的 Gate 终态。

---

## 3. B30-02｜进程树身份结构唯一性

### 3.1 已确认缺陷

05.29 用 `process_tree_id + parent_instance_id + fork_epoch` 推导严格唯一，但 `parent_instance_id` 含随机分量 → 概率唯一，不能标 STRICT_STRUCTURAL。

### 3.2 修复：结构性分支路径 `branch_path`

候选结构：
```
process_tree_id + branch_path + local_fork_sequence
```
- `process_tree_id`：树根身份，128-bit 随机，跨树防碰撞靠概率 → **PROBABILISTIC**（如实标注）。
- `branch_path`：从根到当前进程的**纯结构性**序号序列，如 `[1,3,2]` = 根第1子 → 该子第3子 → 该子第2子。**不含随机分量**，树内 STRICT_STRUCTURAL。
- `local_fork_sequence`：当前进程分支内单调递增序号，由「身份分配锁」原子分配。

关键结论：**树内 fork_event_id 的严格唯一性由 `(tree_id, branch_path)` 提供；树间唯一性仍依赖 `process_tree_id` 的概率唯一性。** 两者分属不同证据等级，不再混写。

完整身份分层、字段定义、唯一性证明、转换表见 `PROCESS-TREE-IDENTITY.md` 与 `IDENTITY-UNIQUENESS-PROOF.md`。

### 3.3 唯一性等级标注（摘要）

| 字段 | 唯一性来源 | 等级 |
|---|---|---|
| process_tree_id | 128-bit 随机 | PROBABILISTIC |
| branch_path | 结构性序号序列 | STRICT_STRUCTURAL（树内） |
| local_fork_sequence | 原子单调计数 | STRICT_STRUCTURAL（分支内） |
| fork_event_id = tree_id+branch_path | 组合 | 树内 STRICT_STRUCTURAL，树间 PROBABILISTIC |
| process_instance_id | 实例区分 | PROBABILISTIC（随机） |
| snapshot_id | 快照 | PROBABILISTIC |
| txn_id | 事务 | PROBABILISTIC |

---

## 4. B30-03｜Receipt 真实性

### 4.1 已确认缺陷

四元组 `(txn_id, revision, inode_dev, sha256)` 仅能证明**对象匹配**，不能证明创建者真实性、防复制、防伪造、事务归属。

### 4.2 修复：六层证据分离

| 层 | 证据 | 结论 |
|---|---|---|
| A. Content Identity | sha256 | 内容一致 |
| B. Filesystem Object Identity | inode/dev | 对象一致 |
| C. Transaction Association | txn_id/revision | 事务关联（弱） |
| D. Creator Authentication | 签名/HMAC | **本设计无法闭合 → UNKNOWN** |
| E. Durability Evidence | fsync(dirfd) | 持久性（弱） |
| F. Commit Outcome | rename 结果 | 提交结果 |

**A+B 不自动推导 C+D+E+F。** 本设计无可信密钥基础设施，无法证明创建者真实性，故 D 层标 `UNKNOWN`；所有无法认证的回执进入 `QUARANTINED`，禁止删除/覆盖/重试。详见 `RECEIPT-AUTHENTICITY.md`、`RECEIPT-IDEMPOTENCY.md`、`RECEIPT-RECOVERY.md`。

---

## 5. B30-04｜Fork 锁生命周期

### 5.1 已确认缺陷

锁所有权文档与生命周期伪代码不一致；父子「继承锁对象」与「继承锁所有权」未区分；子进程重建时机未证明。

### 5.2 修复要点

- 区分：锁对象内存复制（fork 时逐字节拷贝）/ 底层同步原语状态 / OS 锁 / 线程所有权 / 进程所有权 / 锁重建。
- 子进程**不得**释放父进程持有的锁；父进程释放后子进程内存副本**不同步**。
- 必须在子进程重建的锁：`_admit_lock`、`_fork_coord_lock`、`_commit_lock`、身份分配锁、Receipt 写入锁（均为进程本地锁）。
- 必须重新获取的 OS 锁：POSIX advisory lock（若采用）。
- 重建失败 → 子进程进入安全终态（不继续业务）。

详见 `FORK-LOCK-LIFECYCLE.md`、`FORK-LOCK-OWNERSHIP.md`、`LOCK-ORDER-PROOF.md`。

---

## 6. B30-05｜POSIX 回归复核

05.29 的 rename 不确定态 / advisory lock 边界 / 外部写入者边界未退化。结论：
- rename 操作结果（是否发生）与持久性（是否落盘）**已分离**为双维度。
- `OUTCOME_UNKNOWN` 不盲目重试；`DURABILITY_UNKNOWN` 不直接跳 DURABLE。
- advisory lock 明确为**非强制**，仅约束遵守协议的写入者；锁外写入者边界标 UNKNOWN。
- 回执不再承担提交真实性证明。

详见 `POSIX-REGRESSION.md`。

---

## 7. B30-06｜交叉验证

覆盖 DENIED×Receipt、Fork×事务、父子事务共享、身份×Receipt 认证绑定、多进程恢复竞争等 15 项，产出跨模块调用链、交叉时间线、交叉反例、全局不变量。详见 `CROSS-MODULE-CHAINS.md`、`CROSS-TIMELINES.md`、`COUNTEREXAMPLES.md`、`INVARIANTS.md`。

---

## 8. 差异审查、证据等级、机器统计

- `DESIGN-DIFF.md`：逐项对照 05.29 九项问题。
- 证据等级严格执行 `VERIFIED_FACT / VERIFIED_STATIC / DESIGN_PROVEN / DESIGN_PROPOSAL / STATIC_INFERENCE / NOT_VERIFIED / NOT_TESTED / UNKNOWN / CONFLICT / FAIL`。
- 机器统计与一致性检查结果见 `EXECUTION-REPORT.md`。

---

## 9. 最终状态

- 整体结论：**PASS-WITH-ISSUES**（四组核心协议闭合；Receipt 创建者认证边界为 UNKNOWN，属非核心遗留，已隔离处理）。
- IMPLEMENT-01：NOT_APPROVED
- A/B Merge：NOT_APPROVED
- Source Retire：DENY
- 下一步：STOP，等待 ChatGPT 独立复审。
