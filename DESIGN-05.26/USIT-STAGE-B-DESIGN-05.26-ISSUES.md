# USIT-STAGE-B-DESIGN-05.26 问题清单（ISSUES）

> 依据：DESIGN-05.25 独立复审结论 FAIL
> 状态：本轮修复 B26-01 ~ B26-06

## 1. 缺陷追踪（B26-01 ~ B26-06）

| # | 缺陷 | 严重级 | 05.26 修订位置 | 状态 | 剩余风险 |
|---|------|--------|---------------|------|---------|
| B26-01 | `_safe_restore_gate()` fail-open | P0 | §1 全文 | 闭合（协议层）| 运行时 NOT_TESTED |
| B26-02 | 兄弟进程 fork_epoch 碰撞 | P0 | §2 全文 | 闭合（协议层）| 跨重启唯一 NOT_VERIFIED |
| B26-03 | S1~S16 引用旧矩阵 | P0 | §3 全文重写 | 闭合 | 硬件/FS 边界 NOT_VERIFIED |
| B26-04 | Windows 边界未系统复核 | P1 | §4 全文 | 部分闭合 | Windows 持久化 NOT_VERIFIED |
| B26-05 | ZIP 独立核验 | P1 | §6.2 + RECEIPT + ZIP-RECEIPT | 待执行 | 环境无法解压则 NOT_VERIFIED |
| B26-06 | 源码基线 | P1 | §6.3 + 3 份基线文件 | 待执行 | — |

## 2. 未关闭 / 遗留问题

| 项 | 状态 | 说明 |
|----|------|------|
| fail-closed 真实行为 | NOT_TESTED | 本机无 fork |
| 子进程 os._exit(1) | NOT_TESTED | 未运行时验证 |
| 单线程判定对 C 线程 | UNKNOWN | Python 层不可见 |
| 跨重启唯一 | NOT_VERIFIED | 无协调器，不持久 |
| 随机源不可用降级 | NOT_VERIFIED | os.urandom 降级未验证 |
| 恶意复制身份认证 | UNKNOWN | 方案 B 无认证 |
| EINTR/EIO/ENOSPC 恢复 | NOT_VERIFIED | 需运行时确认 |
| 设备写缓存 / 特定 FS | NOT_VERIFIED | 需运行时确认 |
| Windows 部分失败中间态 | NOT_VERIFIED | 官方未规定 |
| Windows 目录持久化 | NOT_VERIFIED | 无完整保证 |
| 第三方 C fork / 嵌入 / 子解释器 / posix_spawn | UNSUPPORTED/DENY | at-fork 钩子无法覆盖 |
| Source Retire | DENY | 保持阻断 |

## 3. 链路与时间线/反例编号（附注）

### 3.1 端到端链路（25 条 L1~L25）

L1 单线程 fork 成功；L2 fork 成功子初始化；L3 fork 失败恢复；L4 after_in_parent 异常；L5 子初始化失败 os._exit；L6 多线程拒绝；L7 非主线程拒绝；L8 连续 fork；L9 子 FD generation 关闭；L10 第三方 C fork DENY；L11 epoch 溢出 DENY；L12 随机源降级；L13 S1→S4；L14 S5→S7；L15 S9 五条件；L16 S12 rename；L17 S13 目录持久；L18 S13 失败恢复；L19 S14/S15 回执；L20 S16 释放锁；L21 身份冲突隔离；L22 目录 fsync 统一；L23 13 项证明门槛；L24 交付哈希闭环；L25 ZIP 独立核验。

### 3.2 身份时间线（15 条 ID1~ID15）

ID1 进程启动；ID2 首次 fork；ID3 子继承；ID4 子再 fork；ID5 兄弟 fork；ID6 PID 重用；ID7 进程重启；ID8 快照消费；ID9 旧快照复用；ID10 FD 槽位复用；ID11 txn 重复提交；ID12 随机源不可用；ID13 恶意复制；ID14 计数器溢出；ID15 快照跨进程传递。

### 3.3 目录 fsync 时间线（15 条 DF1~DF15）

DF1 S13 失败 current 新值全通过；DF2 重试成功；DF3 重试失败；DF4 旧值不推断；DF5 哈希冲突；DF6 txn 冲突；DF7 父版本冲突；DF8 PREPARED 缺失；DF9 仅重读迁移（反例）；DF10 目录可见当持久（反例）；DF11 S3 失败；DF12 S7 失败；DF13 S15 失败；DF14 重试成功直接 DURABLE（反例）；DF15 设备写缓存延迟。

### 3.4 反例（40 个 CID1~CID40）

CID1 子释放 coord；CID2 子释放 gate；CID3 父双释放；CID4 fail-open 吞异常；CID5 子失败返回业务栈；CID6 主线程当单线程；CID7 多线程并发 fork；CID8 at-fork 重复；CID9 依赖回调传播；CID10 跨进程释放；CID11 全局唯一宣称；CID12 PID 当永久唯一；CID13 fork_epoch 当全局唯一；CID14 兄弟碰撞（05.25）；CID15 不同父碰撞；CID16 系统重启复用；CID17 协调器不可用；CID18 随机源弱；CID19 恶意复制；CID20 溢出回绕；CID21 仅重读迁移；CID22 目录可见当持久；CID23 盲目重试 rename；CID24 旧值直接重试 S12；CID25 盲目回滚；CID26 盲目删除 Candidate；CID27 盲目删除 PREPARED；CID28 状态不确定启动新事务；CID29 身份冲突不隔离；CID30 txn 冲突不隔离；CID31 父版本冲突不隔离；CID32 current 与回执混；CID33 文件 fsync 当目录 fsync；CID34 rename 原子当持久；CID35 fsync 重试成功直接 DURABLE；CID36 未核验启动新事务；CID37 Windows 目录持久标 PASS；CID38 Git SHA-1 冒充 SHA-256；CID39 ZIP 只看中央目录；CID40 自审当独立复审。

## 4. 阻断状态（保持）

P0-BLOCK-1、P0-BLOCK-2、P1-BLOCK-3、F1、P0.8、F2 保持阻断；IMPLEMENT-01 NOT_APPROVED；A/B 不合并。
