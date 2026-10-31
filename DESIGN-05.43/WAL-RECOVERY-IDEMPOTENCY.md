# WAL-RECOVERY-IDEMPOTENCY.md — WAL 恢复严格幂等（05.43 权威）

工单：USIT-STAGE-B-DESIGN-05.43
工作包：B43-01（修复 B42-R01）
证据等级：VERIFIED_STATIC（单进程）+ NOT_PROVEN（跨进程/跨重启）
目标：消除 05.42 恢复算法中「COMMIT 与 APPLY 两条记录被当作两笔状态变更」的重复执行缺陷，使每个 `txn_id` 在恢复中最多产生一次逻辑状态变更。

---

## 1. B42-R01 缺陷（05.42 真实代码，必须修复）

05.42 `TRANSACTION-CRASH-RECOVERY.md §2` 的恢复伪代码：

```python
for rec in records:
    if rec.status == 'PREPARE' and not has_commit(rec.txn_id):
        continue
    if rec.status == 'COMMIT' or rec.status == 'APPLY':
        apply_to_state(state, rec)   # 缺陷：COMMIT 和 APPLY 各 apply 一次
        apply_to_trt(trt, rec)
```

**缺陷性质**：一笔事务在 WAL 中产生 `PREPARE`、`COMMIT`、`APPLY` 三条记录（见 TRANSACTION-COMMIT-PROTOCOL）。恢复遍历时，`COMMIT` 记录触发一次 `apply_to_state + apply_to_trt`，随后的 `APPLY` 记录再触发一次。对一笔 `ADMIT`（`count_delta=+1, generation=+1`）而言，恢复结果是 COUNT +2、generation +2，与真实提交时只 +1 不一致。

**后果**：恢复后的状态字 COUNT、每槽位 generation、VERSION 全部被放大，破坏 COUNT==popcount(TOKEN_ACTIVE) 不变式与代际单调性，旧 Token 可能因 generation 被错误抬升而复活或失效。

---

## 2. 三种 WAL 记录的语义（重新定义）

| 记录类型 | 语义 | 是否触发状态变更 |
|---|---|---|
| `PREPARE` | 事务意图落盘（含全部字段变化 + checksum） | 否，仅用于崩溃窗口判定 |
| `COMMIT` | 事务提交标记，提交点（fsync 成功后事务已提交） | **是——这是唯一的状态变更触发点** |
| `APPLY` | 应用进度标记（可选，用于恢复加速/崩溃定位） | 否，仅标记「状态已发布到缓存」 |

**核心原则**：`COMMIT` 是「事务已提交」的唯一权威记录；`APPLY` 是「该事务的缓存发布已完成」的进度标记，**不是第二笔状态变更**。恢复时，每个 `txn_id` 的状态变更只由「是否存在 COMMIT 记录」决定，且只执行一次。

---

## 3. 恢复算法（唯一权威，严格幂等）

```python
def recover():
    records = scan_wal()                 # 按物理顺序解析，保留 txn_id 分组
    if wal_corrupted():
        return 'ISOLATED'                # checksum 失败 → 隔离

    state = init_state()
    trt = init_trt()

    # 关键：按 txn_id 分组，每个 txn_id 只处理一次
    groups = group_by_txn_id(records)    # {txn_id: [records...]}

    # 已应用集合：真实的去重数据结构，不是文字声称
    applied_txn_ids = set()              # 进程内，恢复期存活

    # 按 txn_id 升序重放，保证代际/版本单调
    for txn_id in sorted(groups.keys()):
        recs = groups[txn_id]
        if txn_id in applied_txn_ids:
            continue                     # 双重保险：已在集合中，跳过

        if any(r.status == 'COMMIT' for r in recs):
            # 该事务已提交：取 COMMIT 记录（含权威字段变化）应用一次
            commit_rec = first_commit(recs)
            apply_to_state(state, commit_rec)
            apply_to_trt(trt, commit_rec)
            applied_txn_ids.add(txn_id)  # 记录已应用，防止重复
        else:
            # 只有 PREPARE（或无 COMMIT）：未提交，丢弃，不 APPLY
            continue

    return ('OK', state, trt)
```

**关键点（逐条对应工单要求）**：

1. **每个 txn_id 只产生一次逻辑状态变更**：循环按 `txn_id` 分组（而非按物理记录逐条），每个 txn_id 最多进入一次 `if any(COMMIT)` 分支。
2. **COMMIT 与 APPLY 不再都是触发点**：只有 `COMMIT` 记录承载状态变更；`APPLY` 记录完全不参与 apply，只作为进度标记。
3. **真实去重集合**：`applied_txn_ids` 是进程内的 Python `set`，恢复期存活，每次 apply 后 `add(txn_id)`，循环入口再查一次，双重保险。
4. **COMMIT 无 APPLY（崩溃于 APPLY 前）**：`first_commit` 找到 COMMIT 记录，正常应用一次——因为 APPLY 本就不是变更触发点，APPLY 缺失不影响状态重建。
5. **COMMIT + APPLY 同时存在（正常完成）**：只取 COMMIT 记录 apply 一次，APPLY 记录被忽略，不会重复增加。

---

## 4. 乱序、重复、缺失、损坏的处理

| 情况 | 处理 |
|---|---|
| 记录物理乱序 | `group_by_txn_id` 后按 `txn_id` 升序重放，不依赖物理顺序 |
| 同一 txn_id 出现多条 COMMIT（重复追加） | `first_commit` 取第一条 COMMIT，其余忽略；`applied_txn_ids` 兜底 |
| 同一 txn_id 的 COMMIT 被重复扫描 | 分组后每个 txn_id 只处理一次，天然去重 |
| 只有 PREPARE 无 COMMIT | 未提交，丢弃，不 apply |
| 只有 COMMIT 无 APPLY | 已提交，apply 一次（APPLY 缺失不影响） |
| 记录 checksum 损坏 | `wal_corrupted()` 捕获 → `ISOLATED`，DENY，不猜测 |
| 半条记录（尾部截断） | 视为 checksum 失败 → `ISOLATED` |

---

## 5. COMMIT+APPLY 双记录重放反例（证明不重复增加）

以一笔 `ADMIT`（count_delta=+1, slot=5, generation 8→9, VERSION v→v+1）为例，WAL 含三条记录：

```
rec1: {txn_id=100, status=PREPARE, count_delta=+1, gen_delta=+1, version_delta=+1}
rec2: {txn_id=100, status=COMMIT,  count_delta=+1, gen_delta=+1, version_delta=+1}
rec3: {txn_id=100, status=APPLY,   count_delta=+1, gen_delta=+1, version_delta=+1}
```

恢复执行（按 §3 算法）：

1. `group_by_txn_id` → `{100: [rec1, rec2, rec3]}`。
2. `txn_id=100`，`any(COMMIT)=True` → 取 `rec2`，apply 一次：COUNT+1、generation 8→9、VERSION+1。
3. `applied_txn_ids={100}`。
4. `rec3` 是 APPLY，**不触发 apply**（算法根本不检查 APPLY）。
5. 结束：COUNT 恰好 +1，generation 恰好 9，VERSION 恰好 +1。

**结论**：COMMIT 与 APPLY 双记录同时存在时，COUNT、generation、VERSION 均只增加一次，与真实提交一致。该反例对应 `B43-WAL-001`（见 COUNTEREXAMPLES.json）。

---

## 6. 恢复期间与普通事务的锁隔离

- 恢复流程在**独占恢复锁**下执行，普通 `admit/release/revoke` 事务在恢复完成前被阻塞（见 LOCK-OWNERSHIP-AND-CALL-GRAPH.md）。
- 恢复不与其他事务并发改写状态，因此 `applied_txn_ids` 集合不存在并发污染。
- 恢复完成后释放恢复锁，普通事务才进入。

---

## 7. 与 05.42 的差异

| 项 | 05.42 | 05.43 |
|---|---|---|
| 触发 apply 的条件 | `status=='COMMIT' or status=='APPLY'` | 仅 `status=='COMMIT'` |
| 遍历单位 | 逐物理记录 | 按 txn_id 分组 |
| 去重机制 | 无（文字声称有，代码未实现） | `applied_txn_ids` 集合 + 分组 |
| COMMIT+APPLY 双记录 | 重复 apply（COUNT/generation 翻倍） | apply 一次 |
| APPLY 记录语义 | 与 COMMIT 并列的第二变更点 | 进度标记，不参与 apply |

---

## 8. 证据来源与证明边界

- 提交时序：TRANSACTION-COMMIT-PROTOCOL（05.42 保留，05.43 见 TRANSACTION-PRECOMMIT-VALIDATION.md）
- 数据模型：TRANSACTION-DATA-MODEL（05.42）
- 恢复锁：LOCK-OWNERSHIP-AND-CALL-GRAPH.md
- 崩溃矩阵：CRASH-AND-CONCURRENCY-MATRIX.md

**证明等级**：
- 单进程、单文件 WAL、恢复独占锁 → **VERIFIED_STATIC**（伪代码逻辑可静态推演成立）。
- 跨进程并发写 WAL、跨重启、fsync 语义、半写页（torn write）→ **NOT_PROVEN**，维持 DENY/ISOLATED。
- `applied_txn_ids` 是恢复期内存集合，**不跨进程、不持久化**；若未来需要跨进程恢复，该集合必须改为持久化的「已应用 txn 集合」，当前不声称支持。
