# STATE-TRANSITION-AND-CAS-MATRIX.md — 状态转换与 CAS 矩阵（05.40 权威）

证据等级：VERIFIED_STATIC
目标：修复 B39-R02——为所有状态修改操作建立唯一的状态转换矩阵，标注每个操作的 CAS 宽度、前置条件、字段修改和线性化点。

---

## 1. 唯一状态转换表（全文档权威）

| 操作 | 前置条件（提交点原子校验） | 字段修改（field_cas 一次提交） | 线性化点 | 失败返回 |
|---|---|---|---|---|
| admit | DENIED=0 且 GATE=OPEN 且 COUNT<MAX 且 版本未耗尽 且 有空槽位 | COUNT+1, 槽位置位, VERSION+1 | CAS 成功 | GATE_CLOSED / DENIED / OVERFLOW / TOKEN_EXHAUSTED / VERSION_EXHAUSTED_DENY |
| release(token) | 槽位 active 且 generation 匹配 且 COUNT>0 | 槽位清位, COUNT−1, VERSION+1 | CAS 成功 | INVALID_TOKEN / UNDERFLOW / VERSION_EXHAUSTED_DENY |
| token_create | 有空槽位 且 generation 未耗尽 | generation+1（复用槽位时）, VERSION+1 | CAS 成功 | NO_SLOT / GENERATION_EXHAUSTED_DENY |
| token_consume(token) | 槽位 active 且 generation 匹配 | 槽位清位, VERSION+1 | CAS 成功 | INVALID_TOKEN |
| token_revoke(token) | 槽位 active | 槽位清位, VERSION+1 | CAS 成功 | INVALID_TOKEN |
| freeze | GATE=OPEN 且 DENIED=0 | GATE=CLOSED, FREEZING=1, VERSION+1 | CAS 成功 | ALREADY_FROZEN / DENIED |
| unfreeze | FROZEN=1 且 DENIED=0 且 COUNT=0 | FROZEN=0, GATE=OPEN, VERSION+1 | CAS 成功 | DENIED / NOT_FROZEN |
| set_denied | 任意（DENIED 单调） | DENIED=1, GATE=CLOSED, VERSION+1 | CAS 成功 | —（幂等） |
| cleanup_complete | 清理已到可验证终态 | （无状态字字段修改，仅清理状态机） | 见 CLEANUP 协议 | — |
| recovery | 证据齐全且 DENIED 未并发置位 | 按证据重建字段, VERSION+1 | CAS 成功 | DENIED 优先 / 证据不足 DENY |
| takeover | 资源端 Token 单调递增 | 资源端 Token 更新（非状态字） | 资源端 cas_token | TAKEOVER_CONFLICT |

---

## 2. 每个操作的 CAS 宽度（统一 128 位）

所有上表操作**只通过** `field_cas`（128 位）修改状态字。不存在 64 位 CAS 路径。

---

## 3. 状态字非法组合处理

| 组合 | 判定 | 动作 |
|---|---|---|
| DENIED=1 且 GATE=OPEN | 非法 | DENY（DENIED 优先，GATE 强制 CLOSED） |
| FROZEN=1 且 FREEZING=1 | 非法中间态 | DENY |
| COUNT 与 popcount(TOKEN_ACTIVE) 不一致 | 账本损坏 | DENY，不自动重建 |
| 保留位非 0 | 非法 | INVALID_STATE，DENY |

---

## 4. 版本递增的原子归属

- VERSION 字段**只在 field_cas 内部**通过 `field_set_version` 递增，且递增前在提交路径内检查 `old_version == 0xFFFFFFFF` → `VERSION_EXHAUSTED_DENY`。
- 任何入口不得在 field_cas 之外自行 `version+1`（见 VERSION-EXHAUSTION-ALL-ENTRYPOINTS.md）。

---

## 5. 锁顺序 / 无锁证明

- **单进程**：所有操作通过同一 128 位 `field_cas`，无额外锁，线性化点即 CAS 成功点。VERIFIED_STATIC。
- **跨进程**：依赖共享内存 + HW CAS，未证明 → NOT_PROVEN → DENY。
- 无死锁风险：field_cas 是单一原子原语，无多锁嵌套。

---

## 证据来源

- 状态字布局：UNIFIED-ATOMIC-STATE-WORD.md
- 原子访问宽度：ATOMIC-ACCESS-WIDTH-PROTOCOL.md
- 版本耗尽：VERSION-EXHAUSTION-ALL-ENTRYPOINTS.md
- 证明等级：VERIFIED_STATIC（单进程）+ NOT_PROVEN（跨进程/跨重启）
