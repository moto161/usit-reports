# TOKEN-CONCURRENCY-MATRIX.md — Token 并发矩阵（05.39）

证据等级：VERIFIED_STATIC
原子事务见 TOKEN-ATOMIC-TRANSACTION.md。

---

## 1. 并发操作矩阵

| 场景 | 涉及操作 | 结果 | 保证 |
|---|---|---|---|
| 两线程同时 release 同一 token | release(token) × 2 | 一个成功，一个 INVALID_TOKEN | active 位图 CAS 原子消费 |
| 两线程同时 admit | admit() × 2 | 各得不同 token（seq+1, seq+2） | token_seq 单调 CAS |
| admit 与 release 并发 | admit + release | 各自独立 token，COUNT 正确 | 128 位 CAS 串行 |
| release 与 freeze 并发 | release + cas_frozen | freeze 需 count==0，release 后 count 变化使 freeze 重试 | 谓词纳入 CAS |
| token 重复 release（顺序） | release(token) → release(token) | 第二次 INVALID_TOKEN | active 位图已清除 |
| token 跨线程 release | release(token) by 非持有者 | 允许（token 是凭证），但只消费一次 | active 位图一次性 |

---

## 2. 竞态时序验证（B38-R07 补足）

05.38 反例 `B38-CAS-005` 未覆盖「两线程同时通过 token 检查」的竞态。05.39 补足：

### 时序：两线程并发消费同一 token

```
T0: active 位图含 token=T, COUNT=1
T1: 线程 A load_word → active 含 T, COUNT=1
T2: 线程 B load_word → active 含 T, COUNT=1
T3: 线程 A cas128(COUNT→0, active 清除 T) 成功
T4: 线程 B cas128(期望旧 w) 失败 → 重读 load_word
T5: 线程 B 重读 → active 不含 T → INVALID_TOKEN
结论：T 只被消费一次，COUNT 只减一次。∎
```

**关键**：`is_active` 检查在 `load_word` 读取的最新状态上，且 `cas128` 校验状态未变。B 的第二次 cas 因 `active` 已被 A 改变而失败，重读后 `is_active` 为 False。

---

## 3. 与 05.38 的差异

| 项 | 05.38 | 05.39 |
|---|---|---|
| token 存储 | 独立 admit_tokens 字典 | 内嵌状态字（active 位图） |
| 一次性消费保证 | `token in dict` 检查（非原子） | active 位图 CAS 原子清除 |
| admit 崩溃窗口 | 存在（COUNT 改但 token 未入账） | 消除（单 CAS 原子） |
| release 并发重复 | 存在（两线程都过检查） | 消除（CAS 原子消费） |

---

## 证据来源

- 原子事务：TOKEN-ATOMIC-TRANSACTION.md
- 证明等级：VERIFIED_STATIC
