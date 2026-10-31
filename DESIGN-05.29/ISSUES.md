# USIT-STAGE-B-DESIGN-05.29 问题与阻断清单（ISSUES.md）

## 1. 已撤销的 05.28 结论

| 05.28 结论 | 复审发现 | 05.29 处置 |
|-----------|---------|-----------|
| Fork 原子准入 DESIGN_PROVEN | in_flight 自锁（F01）| 业务/Fork 计数域分离 |
| 上下文管理器 | `__enter__()` 直调（F02）| 完整 with 协议 |
| 锁释放 finally | 所有权不明（F03）| 锁所有权模型 |
| DENIED 失败关闭 | 可被重开（F04）| 不可逆终态 |
| process_instance_id 继承描述 | 继承/重建冲突 | 重新定义（子重建 instance_id，继承 tree_id）|
| fork_epoch 父内序号 | 多代仍需证明 | 撤销"全局"措辞，仅父内 |
| fork_event_id 结构+随机 | 唯一性需复核 | tree_id+父 instance+seq 组合 |
| commit_lock 全局 | 无法约束锁外写入者 | advisory 边界明确 |
| rename 不确定态 | 结果/持久性混淆 | 双维度状态机 |
| Receipt 幂等 | 半成品归属不足 | 四元组归属证明 |

## 2. 本轮遗留问题（NOT_VERIFIED / UNKNOWN）

| 项 | 证据等级 |
|----|---------|
| fork 异常运行时行为 | NOT_TESTED（本机无 fork）|
| POSIX 提交运行时行为 | NOT_TESTED |
| 锁外写入者防护 | NOT_VERIFIED（advisory 本质）|
| 跨主机/跨重启唯一性 | NOT_VERIFIED（不承诺）|
| 随机源失效降级 | UNKNOWN |
| 恶意复制身份 | UNKNOWN（无认证）|
| EINTR/EIO/ENOSPC 实际行为 | NOT_VERIFIED |
| 设备写缓存持久边界 | NOT_VERIFIED |

## 3. 未闭合项（保持阻断）

- IMPLEMENT-01：NOT_APPROVED（保持）
- Source Retire：DENY（保持）
- A/B Merge：NOT_APPROVED（保持）

## 4. 证据边界声明

严格区分 DESIGN_PROVEN / VERIFIED_STATIC / NOT_VERIFIED / NOT_TESTED / UNKNOWN / FAIL。不将概率唯一写严格唯一，不将 advisory 锁写强制锁，不将文件可见写已持久化。
