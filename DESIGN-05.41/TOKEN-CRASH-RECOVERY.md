# TOKEN-CRASH-RECOVERY.md — Token 崩溃恢复（05.41 权威）

证据等级：VERIFIED_STATIC（单进程）+ NOT_PROVEN（跨进程/跨重启）
目标：在独立 TRT 模型下，定义崩溃后 COUNT/位图/TRT 三者一致性的恢复判定。

---

## 1. 恢复的输入证据

| 证据 | 来源 | 可信等级 |
|---|---|---|
| TOKEN_ACTIVE 位图 | 状态字 bit64–79 | 随状态字持久化则可信 |
| COUNT | 状态字 bit1–15 | 同上 |
| TRT[slot].generation | 独立 TRT | 随 TRT 持久化则可信 |
| TRT[slot].owner_id / resource_id | 独立 TRT | 同上 |
| VERSION | 状态字 bit32–63 | 同上 |
| DENIED | 状态字 bit17 | 单调置位 |

---

## 2. 崩溃窗口与恢复（TRT 领先状态字）

提交顺序 = 「先写 TRT，再 CAS 状态字」。状态字 CAS 是唯一提交点。

| 崩溃点 | 状态 | 恢复判定 |
|---|---|---|
| admit：TRT 已写（gen+1，owner 写入），CAS 未提交 | COUNT 未+1，bit 未置位，但 TRT 有「新」记录 | **检测矛盾**：bit=0 但 TRT.gen 与上一持久化值不一致 → 回滚 TRT（恢复 gen 到旧值） |
| admit：CAS 已提交 | COUNT+1 + bit 置位 + TRT 一致 | 事务完成，无需恢复 |
| release：TRT.gen+1 已写，CAS 未提交 | bit 仍置位，但 TRT.gen 已变 | 检测「bit=1 但 TRT.gen 领先」→ 回滚 TRT.gen |
| release：CAS 已提交 | 清位 + COUNT−1 + TRT.gen+1 一致 | 事务完成 |

**核心规则**：状态字 CAS 是提交点。恢复时对比「位图 bit」与「TRT.gen 是否领先于持久化的位图状态」，矛盾则回滚 TRT 到与位图一致的状态。

---

## 3. 恢复算法（txn_lock 内，CAS 保护）

```python
def recover():
    while True:
        txn = begin_txn()
        s = txn.state
        if get_denied(s):
            return 'DENIED'                       # DENIED 优先
        # 检测 TRT 与位图矛盾
        for slot in range(16):
            bit_set = s.token_active & (1 << slot)
            trt_has = txn.trt[slot].generation != 0  # 简化：generation 非 0 视为有记录
            if bit_set and not trt_has:
                return 'ISOLATED'                  # 位图有但 TRT 无 → 损坏，隔离
            if not bit_set and trt_has:
                # TRT 领先（崩溃在 CAS 前）→ 回滚 TRT
                txn.trt[slot].generation = 0
                txn.trt[slot].owner_id = 0
                txn.trt[slot].resource_id = 0
        # 校验 COUNT == popcount(位图)
        if get_count(s) != popcount(s.token_active):
            return 'ISOLATED'                      # COUNT 与位图不一致，不自动重建
        r = commit_txn(txn)
        if r == 'OK':
            return 'RECOVERED'
        if r == 'CAS_FAILED':
            continue
        return r
```

**关键**：恢复只用**持久化的状态字 + TRT**，不用进程内内存。COUNT 与位图不一致时 ISOLATED（不自动重建）；TRT 领先时回滚 TRT。

---

## 4. LEAKED Token 恢复

槽位仍占用（业务未 release），按事务终态判定：

| 业务事务 | 恢复动作 |
|---|---|
| 已提交 | 槽位清位 + COUNT−1（可回收） |
| 未提交 | 槽位清位 + COUNT−1（回滚） |
| 不确定 | ISOLATED，不自动回收，冻结 DENY |

---

## 5. 崩溃类型决策矩阵

| 崩溃类型 | TRT/状态字状态 | 决策 |
|---|---|---|
| 单进程崩溃（无持久化） | 丢失 | DENY |
| 跨进程崩溃（共享内存） | 可能残留 | NOT_PROVEN → DENY |
| 持久化崩溃（跨重启） | 需持久化 | 未证明 → NOT_PROVEN → DENY |
| 进程内线程崩溃 | 状态字 + TRT 仍在 | 按 §2/§3 判定 |

---

## 6. 重复恢复 / 恢复者崩溃 / 恢复中断

| 场景 | 处理 |
|---|---|
| 重复恢复 | 幂等：TRT 已回滚则无变化 |
| 恢复者崩溃 | txn 原子，要么完成要么无变化 |
| 恢复中断 | 重读重算，重试上限 |
| 恢复失败 | DENIED/ISOLATED，不自动放行 |

---

## 7. NOT_PROVEN 边界

- 跨重启 TRT + 状态字持久化：NOT_PROVEN → DENY。
- 跨进程 TRT 共享 + txn_lock：NOT_PROVEN → DENY。
- generation 回绕（无持久化 epoch）：NOT_PROVEN → 耗尽 DENY。

---

## 证据来源

- 原子边界：ATOMIC-TRANSACTION-BOUNDARY.md
- 每槽位代际：TOKEN-IDENTITY-AND-PER-SLOT-GENERATION.md
- 历史约束：无提交证据不自动重放（05.37）
- 证明等级：VERIFIED_STATIC（单进程）+ NOT_PROVEN（跨进程/跨重启）
