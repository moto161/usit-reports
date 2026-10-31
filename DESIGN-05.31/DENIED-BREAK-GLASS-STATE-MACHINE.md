# DENIED-BREAK-GLASS-STATE-MACHINE.md

## 目标
统一 DENIED 与紧急恢复语义，消除「绝对永久不可逆」与「允许人工恢复」的矛盾。

## 1. 语义重定义

- **DENIED**：禁止**普通协议自动恢复**。对普通代码路径而言是不可逆终态。
- **Break-glass 紧急恢复**：独立的、受控的管理协议，**不是普通恢复路径**，故与「DENIED 对普通路径不可逆」不矛盾。

关键：**不再使用「绝对永久不可逆」这一绝对化措辞**。改为精确表述：
> DENIED 对普通自动恢复路径不可逆；仅可通过独立的 Break-glass 管理协议解除。

## 2. 状态全集

| 状态 | 含义 | 对普通路径 | 对 Break-glass |
|---|---|---|---|
| OPEN | 正常运行 | 可进 | — |
| FREEZING/FROZEN/RESTORING/RESTORE_FAILED | 协调/恢复中 | 可进可退 | — |
| DENIED | 安全拒绝 | **不可逆** | 可解除 |
| TERMINATING | 进程终止 | 不可逆 | 不可解除 |
| BREAK_GLASS_PENDING | 紧急恢复审批中 | 不可进 | — |
| NEW_EPOCH | 恢复后新安全纪元 | 重新 OPEN | — |

## 3. Break-glass 授权要求

| 项 | 要求 |
|---|---|
| 谁可授权 | 独立管理授权器，非普通业务线程 |
| 双人审批 | **是**（two-person rule） |
| 审计记录 | **不可篡改**（append-only 审计日志 + 哈希链） |
| 新安全 Epoch | **是**（恢复后创建新 Epoch） |
| 旧授权/Receipt/事务 | 全部失效或隔离 |
| 恢复失败 | 回到 DENIED，不进入 OPEN |

## 4. 状态转换（含拒绝条件）

| 转换 | 条件 | 拒绝条件 |
|---|---|---|
| 任意 → DENIED | 安全状态无法确定 | — |
| DENIED → BREAK_GLASS_PENDING | 授权请求受理 | 无授权者 |
| BREAK_GLASS_PENDING → NEW_EPOCH | 双人审批通过 | 审批不通过 |
| BREAK_GLASS_PENDING → DENIED | 审批拒绝/超时 | — |
| NEW_EPOCH → OPEN | 新 Epoch 建立完成 | 新 Epoch 建立失败 |
| DENIED → OPEN（直接） | **禁止** | 任何直接转换 |
| DENIED → FREEZING/RESTORING | **禁止** | 任何 |

## 5. 反例（20 条，见 COUNTEREXAMPLES.json DENIED 类）

关键反例：
1. finally 直接 DENIED→OPEN → 拒绝。
2. 单点恢复函数调用 manual_reenable → 拒绝（需双人）。
3. 无审计的紧急恢复 → 拒绝。
4. 恢复后复用旧 Receipt → 隔离。
5. 恢复失败后继续 OPEN → 拒绝。

## 6. 不变量

- I-BG-1：DENIED 对普通路径不可逆。
- I-BG-2：Break-glass 需双人审批 + 不可篡改审计。
- I-BG-3：恢复后新 Epoch，旧凭证失效。
- I-BG-4：恢复失败回 DENIED。
- I-BG-5：不存在 DENIED→OPEN 直接转换。

## 7. 证据等级
状态机协议：DESIGN_PROVEN（逻辑）；运行时：NOT_TESTED。
