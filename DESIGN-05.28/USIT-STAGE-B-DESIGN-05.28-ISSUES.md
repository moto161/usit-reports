# USIT-STAGE-B-DESIGN-05.28 问题清单（ISSUES）

> 记录本轮已修复缺陷、遗留问题与证据边界。

## 1. 已修复缺陷（对照 05.27 独立复审）

| 缺陷 | 严重度 | 修复位置 | 状态 |
|------|--------|---------|------|
| `_restore_gate_fail_closed()` 自身抛异常时 DENIED 赋值不执行 | 高 | §1.4 先 DENIED 后恢复 | 已修复 |
| `_close_fork_parent()` 抛异常跳过锁释放 | 高 | §1.6 最外层 finally | 已修复 |
| finally 清理异常覆盖原始异常 | 中 | §1.4 赋值序保证 | 已修复 |
| 锁释放/Gate恢复/清理缺统一终态 | 中 | §1.5/1.6 | 已修复 |
| 未证明所有入口执行同一拒绝协议 | 高 | §1.3 原子准入 | 已修复 |
| Gate 检查与 in_flight 登记 TOCTOU | 高 | §1.3 同临界区 | 已修复 |
| fork_epoch 被 fork 复制计数器推翻 | 高 | §2.3 instance 前缀 + 父 seq | 已修复 |
| 兄弟/多代碰撞未结构性消除 | 高 | §2.3/2.4 | 已修复 |
| S10/S11 无独立状态 | 中 | §3.2 CURRENT_TMP_* | 已修复 |
| S12/S13 不确定态不完整 | 中 | §3.2/3.4 | 已修复 |
| S14 回执缺幂等与归属 | 中 | §3.5 | 已修复 |
| EEXIST 删半成品条件不足 | 中 | §3.7 | 已修复 |
| 两次读一致≠无并发修改 | 高 | §3.6 | 已修复 |

## 2. 遗留问题（NOT_VERIFIED / UNKNOWN）

| 项 | 证据等级 | 说明 |
|----|---------|------|
| fork 异常路径运行时行为 | NOT_TESTED | 本机无 fork |
| 随机源不可用降级行为 | NOT_VERIFIED | 仅 DESIGN_PROPOSAL |
| 恶意复制身份认证 | UNKNOWN | 无认证机制 |
| 跨主机/跨重启唯一性 | NOT_VERIFIED | 不承诺 |
| EINTR/EIO/ENOSPC 实际行为 | NOT_VERIFIED | 依赖内核/文件系统 |
| 设备写缓存持久边界 | NOT_VERIFIED | DEVICE_OR_STORAGE_LIMIT |
| Windows 目录持久化/崩溃恢复/跨卷 | NOT_VERIFIED | 官方文档不覆盖 |

## 3. 证据边界声明

所有设计结论区分 DESIGN_PROVEN（逻辑论证）/ VERIFIED_STATIC（静态证据）/ NOT_VERIFIED / NOT_TESTED / FAIL。不得将 DESIGN_PROVEN 写成运行时 PASS。
