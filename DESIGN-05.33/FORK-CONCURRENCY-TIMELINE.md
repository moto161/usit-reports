# FORK-CONCURRENCY-TIMELINE.md

## 目标
并发时序与线性化点，证明冻结生效后无新准入。

## 1. 线性化点唯一性

- 准入线性化点 LP_admit = `CAS(admit_state, [OPEN, count+1])` 成功点。
- 冻结线性化点 LP_freeze = `CAS(admit_state, [OPEN → FREEZING])` 成功点。
- 两者作用于同一原子字，全序化（total order）由 CAS 线性化保证，唯一。

## 2. 并发时序图（关键场景）

### 场景 A：准入与冻结竞争
```
      LP_admit         LP_freeze
         │                │
线程A: ───●─────────────────────────► (准入成功，count=1)
冻结者:       ●────────────────────► (若 LP_freeze 在 LP_admit 后，则等 count 归零)
```
若 LP_admit 先于 LP_freeze，则该业务已准入，冻结等待其退出。
若 LP_freeze 先于 LP_admit，则该业务 CAS 失败拒绝。

### 场景 B：冻结后新请求
```
      LP_freeze
         │
冻结者: ───●──────── wait drain ────────►
新请求:     ● (CAS 失败·拒绝)  ← 冻结后无准入
```

### 场景 C：冻结超时回滚
```
LP_freeze ── timeout ── LP_unfreeze(OPEN)
冻结者: ●──────────────●
旧请求:                        ● (需重新 CAS 准入，不绕过)
```

## 3. 每个时序下允许/禁止结果

| 时序 | 允许 | 禁止 |
|---|---|---|
| LP_admit 先 | 该业务完成，冻结等归零 | 冻结跳过该业务 |
| LP_freeze 先 | 该业务被拒绝 | 该业务在冻结后准入 |
| 冻结等待期间 | 无新准入 | 任何新业务成功进入 |
| 超时回滚后 | 旧请求重新 CAS 准入 | 旧请求绕过重检直接准入 |
| 父子恢复期间 | 子进程独立 admit_state | 父请求进入子进程 |

## 4. 不变量

- I-CT-1：冻结线性化点之后，admit_count 单调不增（直至归零）。
- I-CT-2：不存在 gate==FREEZING 且新准入成功的状态。
- I-CT-3：LP_admit 与 LP_freeze 全序，唯一线性化点。
- I-CT-4：快照时 admit_count==0 且 commit_idle 且 receipt_idle。

## 5. 证明

由 CAS 原子性：LP_admit 与 LP_freeze 必有一个先行。若 LP_freeze 先行，则 gate 已非 OPEN，LP_admit 的 CAS 前置条件不满足 → 拒绝。故冻结后无新准入。I-CT-2 成立。

## 6. 证据等级
线性化点与并发时序：DESIGN_PROVEN（模型级）；真实运行时：NOT_TESTED。
