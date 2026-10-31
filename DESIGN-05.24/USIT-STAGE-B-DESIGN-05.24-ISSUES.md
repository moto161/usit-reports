# USIT-STAGE-B-DESIGN-05.24 问题清单（ISSUES）

> 依据：DESIGN-05.23 独立复审结论 FAIL
> 状态：本轮定点修复 B24-01 ~ B24-06

## 1. 缺陷追踪（B24-01 ~ B24-06）

| # | 缺陷 | 严重级 | 05.24 修订位置 | 状态 | 剩余风险 |
|---|------|--------|---------------|------|---------|
| B24-01 | 伪代码与异常路径不一致；回调责任未唯一化；回调异常传播假设 | P0 | §1 全文 | 闭合（协议层）| 回调异常传播 NOT_TESTED |
| B24-02 | Epoch 作用域定义不清；PID/Epoch 单独性未评估 | P0 | §2 全文 | 闭合（协议层）| 跨进程唯一 NOT_TESTED |
| B24-03 | S1~S16 矩阵被"代表性覆盖"未逐项展开 | P0 | §3 全文 | 闭合 | 硬件/FS 边界 NOT_VERIFIED |
| B24-04 | S13-4"读到旧值即继续 S12"不安全推断 | P0 | §4 全文 | 闭合 | 硬件/FS 边界 NOT_VERIFIED |
| B24-05 | ZIP 核验缺真实输出与路径穿越/敏感信息检查 | P1 | §6.2 + RECEIPT | 待执行 | 环境无法解压则 NOT_VERIFIED |
| B24-06 | 源码基线仅"25 文件一致"缺完整清单与 diff | P1 | §6.3 + 3 份基线文件 | 待执行 | — |

## 2. 未关闭 / 遗留问题

| 项 | 状态 | 说明 |
|----|------|------|
| 多线程 fork 运行时行为 | NOT_TESTED | 本机 `hasattr(os,'fork')=False` |
| 回调异常传播 / 子进程异常处理 | NOT_TESTED | 无运行时证据 |
| Epoch 跨进程唯一（PID 重用/多代际/子再 fork）| NOT_TESTED | 协议层证明，运行未验证 |
| 第三方 C 扩展直接 fork/open/close/dup | UNSUPPORTED/DENY | at-fork 钩子无法覆盖 |
| 嵌入 Python / 子解释器 fork | UNSUPPORTED/DENY | 无法确保 Gate 覆盖 |
| posix_spawn / multiprocessing spawn | UNSUPPORTED/DENY | 不经过 Gate/Registry |
| 设备写缓存 / 特定文件系统持久化行为 | NOT_VERIFIED | 需运行时确认 |
| Windows 目录项断电持久性 | NOT_VERIFIED | 无权威 API 完整语义 |
| Source Retire | DENY | 保持阻断 |

## 3. 阻断状态（保持）

P0-BLOCK-1、P0-BLOCK-2、P1-BLOCK-3、F1、P0.8、F2 保持阻断；IMPLEMENT-01 NOT_APPROVED；A/B 不合并。
