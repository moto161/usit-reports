# DENIED-STATE-MACHINE.md

## 目标
修复 05.29「DENIED → restore 成功 → OPEN」矛盾。确立 DENIED 不可逆终态，五类错误状态分离。

## 1. 状态全集

| 状态 | 进入条件 | 退出条件 | 可逆 | 状态写入者 |
|---|---|---|---|---|
| OPEN | 初始化 / 正常恢复成功 | 收到 Fork 请求 | 是 | Gate 协调器 |
| FREEZING | Fork 准入通过 | 业务归零或超时 | 是 | Gate 协调器 |
| FROZEN | FREEZING 完成 | Fork 执行 | 是 | Gate 协调器 |
| RESTORING | 父进程 fork 成功后恢复 | 恢复成功/失败 | 是 | 父进程协调器 |
| RESTORE_FAILED | 父进程恢复失败 | 受控重试 / 升级 | 部分可逆 | 父进程协调器 |
| TERMINATING | 不可恢复错误 | 进程退出 | 否 | 终止协调器 |
| DENIED | 安全状态无法确定 / 永久拒绝 | 仅人工授权 | **否（永久不可逆）** | 终止协调器 |

## 2. 强制状态转换规则

| 转换 | 允许 | 责任主体 | 说明 |
|---|---|---|---|
| OPEN → FREEZING | 允许 | 准入协调器 | 原子准入 |
| FREEZING → FROZEN | 允许 | 准入协调器 | 业务归零 |
| FREEZING → OPEN（超时回滚） | 允许 | 准入协调器 | 临时失败，未进 DENIED |
| FROZEN → RESTORING | 允许 | 父进程 | fork 成功 |
| RESTORING → OPEN | 允许 | 父进程 | 恢复成功（gate 仍为 RESTORING 才可） |
| RESTORING → RESTORE_FAILED | 允许 | 父进程 | 恢复异常 |
| RESTORE_FAILED → OPEN | 允许（受控） | 父进程 | 重试成功（未进 DENIED） |
| 任意 → DENIED | 允许 | 终止协调器 | 安全状态无法确定 |
| 任意 → TERMINATING | 允许 | 终止协调器 | 不可恢复 |
| **DENIED → OPEN** | **禁止（普通路径）** | — | 任何 finally/回调/清理不得触发 |
| DENIED → FREEZING / RESTORING | 禁止 | — | — |
| DENIED → TERMINATING | 允许 | 终止协调器 | 进程退出 |
| DENIED → OPEN（人工） | 允许（独立管理协议） | 管理授权器 | `manual_reenable()` |

## 3. DENIED 唯一解除路径

`manual_reenable()` 必须满足全部条件：
1. 由独立管理授权器调用，非普通业务线程。
2. 持有独立授权锁 `_manual_auth_lock`。
3. 产生审计记录（操作者、时间、原因、签名）。
4. 不通过任何 finally/异常处理器/回调/清理函数间接触发。
5. 解除后 Gate 回到 OPEN，但审计轨迹不可清除。

## 4. 不变量

- I-DENIED-1：`_gate == 'DENIED'` 后，普通代码路径不得再写回 OPEN。
- I-DENIED-2：任何 finally 二次异常不得改变已确定的 Gate 终态。
- I-DENIED-3：恢复成功只能恢复 RESTORING（或 RESTORE_FAILED 受控重试），不能跨越 DENIED。
- I-DENIED-4：DENIED 与 TERMINATING 均不可逆，区别在于 TERMINATING 必然进程退出。

## 5. 反例与闭合

| # | 反例 | 结论 |
|---|---|---|
| 1 | finally 中 `_gate='OPEN'` 绕过 | 禁止：finally 只允许释放锁，禁止写 Gate |
| 2 | 恢复异常后误判成功 | RESTORE_FAILED 落定，不自动 OPEN |
| 3 | 清理函数重开 DENIED | 清理函数无权写 Gate |
| 4 | 多线程并发写 Gate | Gate 写入仅协调器线程持有 `_gate_write_lock` |
| 5 | DENIED 后收到新 Fork 请求 | 准入检查拒绝，不进入 FREEZING |

## 6. 证据等级

- 状态转换规则：DESIGN_PROVEN（逻辑推演无未闭合反例）。
- 运行时行为：NOT_TESTED（未执行真实 fork）。
