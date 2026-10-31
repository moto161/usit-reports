# REVOKE-AND-COMPENSATION-PROTOCOL.md — revoke 与补偿协议（05.42 权威）

证据等级：VERIFIED_STATIC（单进程）+ NOT_PROVEN（跨进程/跨重启）
目标：修复 B41-R02——revoke 改为同事务减 COUNT，消除补偿机制的不确定性。

---

## 1. 05.41 缺陷定位

05.41 的 revoke 清位不减 COUNT，把差值交给 `revoke_pending`，但：

- `revoke_pending` 不在 128 位状态字布局中。
- 持久化位置未定义。
- 未证明与状态字同事务。
- 恢复算法发现 COUNT 不一致时直接 ISOLATED，未真正补偿。

**根因**：把「临时破坏不变式」当作可接受的中间态，但没有为这个中间态设计权威存储。

---

## 2. 05.42 决策：revoke 同事务减 COUNT

**revoke 与 release 在数据层完全等价**：都是「COUNT−1 + 清位 + generation+1 + VERSION+1」。区别仅在**语义意图**（revoke = 撤销身份；release = 正常消费）。

```python
def revoke(slot, gen, owner_id, resource_id):
    # 与 release 相同字段变化，通过 WAL 事务提交
    rec = build_record(op='REVOKE', slot=slot, old_gen=gen,
                       new_gen=gen+1, count_delta=-1,
                       active_bit=-1, version_delta=+1)
    return run_txn(rec)   # PREPARE → COMMIT → APPLY
```

**结果**：revoke 成功后 I-1 严格成立（COUNT 与位图同步变化），无需 `revoke_pending`，无需补偿。

---

## 3. 补偿机制的消除

| 05.41 概念 | 05.42 处理 |
|---|---|
| revoke_pending | **删除** |
| cleanup 补偿 COUNT | **删除**（revoke 已减 COUNT） |
| 补偿前冻结 DENY | 不再需要（I-1 始终成立） |

**cleanup 的职责收窄**：cleanup 只负责**释放资源端**（文件、锁、连接等外部资源），不再负责补偿 COUNT。COUNT 的维护完全由 admit/release/revoke 的事务保证。

---

## 4. revoke 与 release 的语义区分

| 操作 | 数据字段变化 | 语义 | 审计标记 |
|---|---|---|---|
| release | COUNT−1, 清位, gen+1 | 正常消费 | op=RELEASE |
| revoke | COUNT−1, 清位, gen+1 | 撤销身份 | op=REVOKE |

**数据等价，语义不同**。区别仅在于 WAL 记录的 `op` 字段（用于审计），不影响状态不变式。

---

## 5. 并发交错（revoke / release / cleanup / freeze / recovery）

| 交错 | 结果 |
|---|---|
| revoke 与 release 同 slot 并发 | txn_lock 串行，一方成功，另一方 INVALID_TOKEN |
| revoke 与 cleanup 并发 | cleanup 不碰 COUNT，只释放资源端，无冲突 |
| revoke 与 freeze 并发 | revoke 减 COUNT 后 I-1 成立，freeze 正常等待 COUNT=0 |
| revoke 与 recovery 并发 | WAL 重放幂等，revoke 记录只 APPLY 一次 |
| 重复 revoke | 第二次 bit=0 → INVALID_TOKEN（幂等） |

---

## 6. 崩溃恢复

revoke 走 WAL 事务，崩溃恢复重放 revoke 记录（op=REVOKE），APPLY 后 COUNT−1 + 清位 + gen+1 一致。无「补偿未完成」的中间态。

---

## 证据来源

- 状态转换：STATE-INVARIANTS-AND-TRANSITIONS.md
- 提交协议：TRANSACTION-COMMIT-PROTOCOL.md
- 证明等级：VERIFIED_STATIC（单进程）+ NOT_PROVEN（跨进程/跨重启）
