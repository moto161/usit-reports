# TRANSACTION-CRASH-RECOVERY.md — 事务崩溃恢复（05.42 权威）

证据等级：VERIFIED_STATIC（单进程）+ NOT_PROVEN（跨进程/跨重启）
目标：修复 B41-R01——恢复改为重放 WAL，不再用 `bit=0 && generation!=0` 猜 TRT 领先，保证正常历史 generation 永不被清零。

---

## 1. 05.41 恢复算法缺陷

05.41 用 `bit=0 && generation!=0` 判「TRT 领先」，但**正常释放过的槽位**本来就是 `bit=0 && generation>0`（release 递增了 generation 且清除了 bit）。恢复时把这个 generation 清零，会：

- 破坏每槽位代际单调性。
- 使后续复用的 generation 与历史冲突。
- 无法区分「正常空闲槽位」与「未提交事务留下的 TRT」。

**05.42 修正**：恢复**不猜状态字/TRT 谁领先**，而是**重放 WAL**。generation 的每次变化都有 WAL 记录，重放按记录恢复，永不清零。

---

## 2. 恢复算法（重放 WAL）

```python
def recover():
    # 1. 扫描 WAL，按 txn_id 升序
    records = scan_wal()            # 解析所有完整记录
    if wal_corrupted():
        return 'ISOLATED'           # checksum 失败 → 隔离

    # 2. 丢弃所有内存状态字/TRT，从零重建
    state = init_state()
    trt = init_trt()

    # 3. 重放记录
    for rec in records:
        if rec.checksum_invalid():
            return 'ISOLATED'       # 损坏记录 → 隔离，DENY
        if rec.status == 'PREPARE' and not has_commit(rec.txn_id):
            continue                # 未提交，丢弃（不 APPLY）
        if rec.status == 'COMMIT' or rec.status == 'APPLY':
            apply_to_state(state, rec)   # 重放字段变化
            apply_to_trt(trt, rec)       # 重放 generation/owner/resource
    return ('OK', state, trt)
```

**关键**：
- 状态字与 TRT 从零重建，靠 WAL 重放，不依赖崩溃前的现场值。
- generation 的每次变化（ADMIT +1、RELEASE +1）都来自 WAL 记录，重放后与历史完全一致，**永不被清零**。
- PREPARE 无 COMMIT → 未提交，丢弃；COMMIT → 重放 APPLY。

---

## 3. 恢复状态分类（明确区分）

| 状态 | 判定依据 | 恢复动作 |
|---|---|---|
| 正常空闲槽位（bit=0，generation>0） | WAL 中该槽位最后一次记录是 RELEASE（含 COMMIT） | generation 保留，不清零 |
| TRT 领先（未提交） | WAL 中只有 PREPARE 无 COMMIT | 丢弃该事务，不 APPLY |
| 提交完成 | WAL 中 COMMIT + APPLY | 重放，幂等跳过 |
| 提交不确定 | COMMIT fsync 失败 | ISOLATED（见 §5） |
| 记录损坏 | checksum 失败 | ISOLATED，DENY |

**关键修正**：不再用 `bit=0 && generation!=0` 作为「TRT 领先」的判据。正常空闲槽位的 generation 由 WAL 记录保留。

---

## 4. 崩溃矩阵（重放视角）

| WAL 状态 | 恢复结果 |
|---|---|
| 无记录 | 无事务，初始状态 |
| PREPARE（无 COMMIT） | 未提交，丢弃 |
| COMMIT（无 APPLY） | 已提交，重放 APPLY |
| COMMIT + APPLY | 已提交已应用，幂等跳过 |
| 半条记录（checksum 失败） | ISOLATED，DENY |
| fsync 失败（PREPARE 或 COMMIT 未持久化） | 视为未持久化，按未提交处理（若 COMMIT 不确定则 ISOLATED） |

---

## 5. COMMIT fsync 失败（提交不确定）

COMMIT 记录 fsync 返回失败，但实际可能已落盘（响应丢失）。此状态**无法确定**：

- 若重放后发现 COMMIT 已持久化 → 已提交。
- 若未持久化 → 未提交。
- **无法区分时 → ISOLATED**，不自动重放，人工介入。

**不得**：假设 COMMIT fsync 失败即未提交（可能已落盘）；也不得假设即已提交（可能未落盘）。无法证明时 DENY。

---

## 6. 重复恢复 / 恢复者崩溃 / 恢复中断

| 场景 | 处理 |
|---|---|
| 重复恢复 | 重放幂等（按 txn_id + txn_version 去重） |
| 恢复者崩溃 | 重放未完成，下次重扫 WAL 重放（幂等） |
| 恢复中断 | 重扫重放，无中间态对外可见 |

---

## 7. NOT_PROVEN 边界

- 跨重启 WAL 持久化介质 + fsync 原子性：NOT_PROVEN → DENY。
- 跨进程 WAL 共享 + 追加原子性：NOT_PROVEN → DENY。
- checksum 无法检测的静默损坏：NOT_PROVEN → 依赖底层存储完整性。

---

## 证据来源

- 提交协议：TRANSACTION-COMMIT-PROTOCOL.md
- 数据模型：TRANSACTION-DATA-MODEL.md
- 方案选型：WAL-OR-ATOMIC-STORAGE-DESIGN.md
- 证明等级：VERIFIED_STATIC（单进程）+ NOT_PROVEN（跨进程/跨重启）
