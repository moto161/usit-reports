# USIT-STAGE-B-DESIGN-05.27 问题清单（ISSUES）

> 依据：DESIGN-05.26 独立复审结论 FAIL
> 状态：本轮修复 B27-01 ~ B27-05

## 1. 缺陷追踪（B27-01 ~ B27-05）

| # | 缺陷 | 严重级 | 05.27 修订位置 | 状态 | 剩余风险 |
|---|------|--------|---------------|------|---------|
| B27-01 | finally 未保证 Gate 恢复；锁释放≠Gate 恢复 | P0 | 主文档 §1 | 闭合（协议层）| 运行时 NOT_TESTED |
| B27-02 | fork_event_id 随机标"实例内唯一"矛盾 | P0 | 主文档 §2 | 闭合（协议层）| 跨重启 NOT_VERIFIED |
| B27-03 | S4 跳态、S2/S3 混、S13 名称、S9 五条件、六字段五谓词、EINTR | P0 | 主文档 §3 | 闭合（协议层）| 硬件/FS NOT_VERIFIED |
| B27-04 | 时间线/反例/链路/不变量未独立成文未统计 | P1 | 4 份独立文档 | 闭合 | — |
| B27-05 | 交付证据与状态一致性 | P1 | §6 + RECEIPT + 基线 | 待执行 | 环境无法解压则 NOT_VERIFIED |

## 2. 未关闭 / 遗留问题

| 项 | 状态 | 说明 |
|----|------|------|
| fail-closed 真实行为 | NOT_TESTED | 本机无 fork |
| 子进程 os._exit(1) | NOT_TESTED | 未运行时验证 |
| 单线程判定对 C 线程 | UNKNOWN | Python 层不可见 |
| 跨重启唯一 | NOT_VERIFIED | 无协调器，不持久 |
| 随机源不可用降级 | NOT_VERIFIED | os.urandom 降级未验证 |
| 恶意复制身份认证 | UNKNOWN | 无认证 |
| EINTR/EIO/ENOSPC 恢复 | NOT_VERIFIED | 需运行时确认 |
| 设备写缓存 / 特定 FS | NOT_VERIFIED | 需运行时确认 |
| Windows 部分失败中间态/目录持久化 | NOT_VERIFIED | 官方未规定 |
| 第三方 C fork / 嵌入 / 子解释器 / posix_spawn | UNSUPPORTED/DENY | at-fork 钩子无法覆盖 |
| Source Retire | DENY | 保持阻断 |

## 3. 阻断状态（保持）

P0-BLOCK-1、P0-BLOCK-2、P1-BLOCK-3、F1、P0.8、F2 保持阻断；IMPLEMENT-01 NOT_APPROVED；A/B 不合并。
