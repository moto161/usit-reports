# GLOBAL-LOCK-ORDER.md

## 目标
唯一权威全局锁顺序，消除 coord 与 admit/commit 嵌套矛盾。

## 1. 全局锁清单（盘点真实用途）

| 锁 | 真实用途 | 允许嵌套 |
|---|---|---|
| _gate_state_lock | 保护 Gate 字段写入（FREEZING/OPEN 等） | 否（叶子锁） |
| _admit_lock | 保护准入开关 admit_closed | 否（叶子锁） |
| _commit_lock | 保护事务提交临界区 | 可嵌套业务资源 |
| _identity_alloc_lock | 保护身份序号分配 | 否（叶子锁） |
| _receipt_write_lock | 保护回执写入 | 在 commit 内 |
| _fork_coord_lock | 串行化 fork 准入 | **禁止嵌套任何业务锁** |
| 业务资源锁 | 保护业务数据 | 在 commit 内 |

## 2. 唯一权威锁顺序（禁止其他文档重复定义）

```
全局获取顺序（非嵌套时）：
  _identity_alloc_lock → _gate_state_lock → _admit_lock → _commit_lock → 业务资源锁 → _receipt_write_lock

Fork 协调锁 _fork_coord_lock：
  独立串行化，与上述任何锁【不嵌套】。
  coord 锁作用域内不获取 admit/commit/identity/receipt/业务锁。
  需要读写 Gate 字段时，通过 _gate_state_lock 独立短持有（先释放 coord 再取 gate，或 gate 字段单独用原子变量）。
```

## 3. 各执行路径锁获取顺序

### Fork 协调路径
```
_fork_coord_lock（唯一持有者协调线程）
  └─（不嵌套任何业务锁）
     置 Gate 用原子/独立 _gate_state_lock（短持有，不进入 coord 嵌套）
     置 admit_closed 用 _admit_lock（短持有，先退出 coord 或先于 coord）
```
> 裁决：coord 与 admit/commit **不嵌套**。Gate 字段采用原子写或独立 gate 锁，避免 coord 内嵌套 admit。

### 事务提交路径
```
_commit_lock → 业务资源锁 → _receipt_write_lock
```

### 身份分配路径
```
_identity_alloc_lock（单锁）
```

### 恢复/关闭路径
```
无嵌套；各锁独立获取释放
```

### 异常处理路径
```
finally 逆序释放已获取锁
```

## 4. 锁顺序矩阵

| 路径 | 顺序 | 嵌套 | 权威依据 |
|---|---|---|---|
| Fork 协调 | coord（独立） | 否 | 本文件 §3 |
| 事务提交 | commit→业务→receipt | 是（固定顺序） | 本文件 §3 |
| 身份分配 | identity | 否 | 本文件 §3 |
| 回执写入 | commit→receipt | 是 | 本文件 §3 |
| Gate 写 | gate_state | 否 | 本文件 §3 |

## 5. 硬性规则

1. coord 与 admit/commit 不允许嵌套，任何伪代码不得出现。
2. 不通过超时/重试/异常捕获掩盖死锁。
3. 锁不可用/损坏/状态未知 → 安全降级（拒绝操作，进入安全终态）。
4. 本文件是唯一权威；其他文档引用本文件，不得重复定义冲突规则。

## 6. 死锁环检查

Wait-for Graph 详见 LOCK-WAIT-FOR-GRAPH.md。结论：无未解释环。

## 7. 证据等级
锁顺序协议 DESIGN_PROVEN（逻辑）；真实运行时 NOT_TESTED。
