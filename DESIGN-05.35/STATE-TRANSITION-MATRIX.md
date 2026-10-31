# STATE-TRANSITION-MATRIX.md

## 目标
合法/非法状态组合表 + 状态转换表。

## 1. 合法状态组合

| gate | frozen | DENIED | count | 状态名 | 合法性 |
|---|---|---|---|---|---|
| 0 | 0 | 0 | 任意 | OPEN | 合法 |
| 1 | 0 | 0 | ≥0 | FREEZING | 合法 |
| 1 | 1 | 0 | 0 | FROZEN | 合法 |
| 1 | 1 | 0 | >0 | 非法（frozen 需 count==0） | 非法 |
| 0 | 1 | 0 | 任意 | 非法（frozen 需 gate==1） | 非法 |
| x | x | 1 | 任意 | DENIED | 合法（DENIED 后 gate 视为关闭） |
| 0 | 0 | 1 | 任意 | 非法（DENIED 后 gate 应关闭） | 非法 |

## 2. 状态转换表

| 当前 | 事件 | 下一 | 线性化点 | 前置 |
|---|---|---|---|---|
| OPEN | 准入 | OPEN(count+1) | cas_admit_enter | gate==OPEN 且非 DENIED 且 count<MAX |
| OPEN | 冻结 | FREEZING | cas_freeze | gate==OPEN 且非 DENIED |
| FREEZING | 退出 | FREEZING(count-1) | cas_admit_exit | count>0 |
| FREEZING | 冻结完成 | FROZEN | cas_frozen | count==0 |
| FREEZING | 超时回滚 | OPEN | cas_unfreeze | 非 DENIED |
| FROZEN | fork 成功 | 分叉 | fork 返回 | — |
| FROZEN | fork 失败 | OPEN | cas_unfreeze | 非 DENIED |
| 任意 | 安全拒绝 | DENIED | cas_denied | — |
| DENIED | unfreeze/rollback | 拒绝 | — | DENIED 不可逆 |

## 3. DENIED 不可逆

- DENIED 进入后，`cas_unfreeze`/`cas_freeze`/`cas_frozen`/`cas_admit_enter` 全部因 `DENIED_MASK` 前置而拒绝。
- 唯一离开 DENIED = 独立 Break-glass 管理协议（不在本状态字内，见历史 DENIED-BREAK-GLASS 文档）。
- 普通恢复/父子初始化不得清除 DENIED。

## 4. 并发反例覆盖

见 COUNTEREXAMPLES.json（B35-* 类别），覆盖：FREEZING/FROZEN 竞争、DENIED 与 unfreeze 竞争、CAS 跨字段、count 溢出/下溢、双状态源不一致等。

## 5. 不变量

- I-ST-1：单一状态源 `_admit_state`。
- I-ST-2：field-aware CAS，禁止裸加。
- I-ST-3：DENIED 位一旦置位，任何普通路径不得清除。
- I-ST-4：frozen==1 ⟹ gate==1 且 count==0。
- I-ST-5：DENIED==1 ⟹ gate 语义关闭。

## 6. 证据等级
状态转换 DESIGN_PROVEN（模型级）；运行时 NOT_TESTED。
