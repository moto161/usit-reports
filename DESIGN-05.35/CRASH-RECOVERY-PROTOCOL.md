# CRASH-RECOVERY-PROTOCOL.md

## 目标
崩溃恢复协议：通过磁盘证据发现遗留记录、识别事务归属、完成隔离。

## 1. 崩溃场景

| 场景 | 覆盖 |
|---|---|
| 进程崩溃/强制终止 | 进程级恢复 |
| 系统崩溃/重启 | 持久化 + 磁盘证据 |
| 线程异常/取消 | 线程 finally（不覆盖进程崩溃） |

## 2. 共享计数器与持久化事务状态的崩溃发现

| 状态 | 崩溃后发现 | 处理 |
|---|---|---|
| 内存 admit_count | 随进程消失 | 恢复进程重读原子字（重置为 0 或从持久化恢复） |
| 持久化事务（半成品） | 扫描事务目录 | 半成品隔离 |
| 持久化回执（半成品） | 扫描回执目录 | 半成品隔离 |
| current 文件（rename 不确定） | 检查 inode/哈希 | 不确定态隔离 |
| 遗留锁 | 进程退出自动释放 OS 锁 | 无需人工 |

## 3. 识别事务归属

- 事务归属 = `(process_tree_id, branch_path, txn_id)`。
- 恢复进程通过磁盘上的事务元数据识别归属。
- 无法确认归属 → 隔离（QUARANTINED），不自动重试。

## 4. 无法确认提交的处理

- 无法确认遗留事务是否已提交 → **不自动重试、不放行**，进入隔离。
- 仅当磁盘证据（回执完整 + 可认证）确认提交成功时才可判成功（但 Receipt 认证 NOT_PROVEN，故实际隔离）。

## 5. 恢复操作的可信依据、执行权限、失败终态

| 项 | 定义 |
|---|---|
| 可信依据 | 仅磁盘证据（回执、事务元数据、文件 inode/哈希），非进程内变量 |
| 执行权限 | 恢复协调器，受安全策略约束 |
| 失败终态 | 隔离（QUARANTINED），不恢复 OPEN，不继续 Fork |

## 6. 崩溃恢复伪代码

```python
def _crash_recover(self):
    # 仅依赖磁盘证据
    for txn in scan_transaction_dir():
        if txn.is_half_written():
            quarantine(txn)                 # 半成品隔离
        elif txn.unknown_outcome():
            quarantine(txn)                 # 不确定隔离
        elif txn.authenticated_success():   # 需 Receipt 认证（NOT_PROVEN）
            quarantine(txn)                 # 认证未建立 → 隔离
        else:
            quarantine(txn)                 # 默认隔离

    for receipt in scan_receipt_dir():
        if not receipt.authenticated():     # 认证 NOT_PROVEN
            quarantine(receipt)             # 隔离

    # 无法确认的任何状态不自动重试/放行
```

## 7. 不变量

- I-CR-1：崩溃恢复仅依赖磁盘证据。
- I-CR-2：无法确认提交的事务不自动重试/放行。
- I-CR-3：隔离对象不自动清理。
- I-CR-4：清理幂等，不覆盖 DENIED。

## 8. 证据等级
崩溃恢复协议 DESIGN_PROVEN（逻辑）；真实崩溃恢复 NOT_TESTED。
