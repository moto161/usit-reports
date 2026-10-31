# USIT-STAGE-B-DESIGN-05.23 问题清单（ISSUES）

> 依据：DESIGN-05.22 独立复审结论 FAIL
> 状态：本轮定点修复 B23-01 ~ B23-06

## 1. 缺陷追踪（B23-01 ~ B23-06）

| # | 缺陷 | 严重级 | 05.23 修订位置 | 状态 | 剩余风险 |
|---|------|--------|---------------|------|---------|
| B23-01 | `finally` 父子两侧重复执行，子进程误释放未持有 `_fork_coord_lock` | P0 | §1.2 / §1.3 / §1.4 | 闭合（协议层）| 第三方 C fork 不可控 |
| B23-02 | `fork_epoch_counter` 锁外递增，epoch 分配与快照登记不同临界区 | P0 | §1.6 / §1.7 | 闭合（协议层）| 第三方 C fork 不可控 |
| B23-03 | S1~S16 崩溃矩阵部分崩溃位置缺失 | P0 | §2.2 | 闭合 | Windows 目录持久化 |
| B23-04 | S13 fsync 失败恢复仅凭 current 重读迁移 | P0 | §2.5 / §2.6 / §2.7 | 闭合 | Windows 目录持久化 |
| B23-05 | ZIP 独立核验需真实执行 | P1 | §6.3 / HASH-RECEIPT | 待执行 | 环境无法解压则 NOT_VERIFIED |
| B23-06 | 源码前后完整基线需真实采集 | P1 | §6.4 / EXECUTION-REPORT | 待执行 | 无法独立采集则 NOT_VERIFIED |

## 2. 未关闭 / 遗留问题

| 项 | 状态 | 说明 |
|----|------|------|
| 多线程 fork 运行时行为 | NOT_TESTED | 本机 `hasattr(os,'fork')=False`，无法运行 fork |
| 第三方 C 扩展直接 fork/open/close/dup | UNSUPPORTED/DENY | at-fork 钩子无法覆盖 |
| 嵌入式 Python / 子解释器 fork | UNSUPPORTED/DENY | 无法确保 Gate 覆盖 |
| `os.posix_spawn()` / multiprocessing spawn | UNSUPPORTED/DENY | 不经过 Gate/Registry |
| Windows 目录项断电持久性 | NOT_VERIFIED | 无权威 API 保证完整语义 |
| Source Retire | DENY | 保持阻断，不解除 |

## 3. 阻断状态（保持）

P0-BLOCK-1、P0-BLOCK-2、P1-BLOCK-3、F1、P0.8、F2 均保持阻断；IMPLEMENT-01 NOT_APPROVED；A/B 不合并。
