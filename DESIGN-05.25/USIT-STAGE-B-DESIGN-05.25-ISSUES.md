# USIT-STAGE-B-DESIGN-05.25 问题清单（ISSUES）

> 依据：DESIGN-05.24 独立复审结论 FAIL
> 状态：本轮定点修复 B25-01 ~ B25-06

## 1. 缺陷追踪（B25-01 ~ B25-06）

| # | 缺陷 | 严重级 | 05.25 修订位置 | 状态 | 剩余风险 |
|---|------|--------|---------------|------|---------|
| B25-01 | restore_gate 抛异常致锁未释放；子初始化失败返回业务栈；主线程误当单线程；并发 fork 矛盾；at-fork 与手动重复 | P0 | §1 全文 | 闭合（协议层）| 运行时 NOT_TESTED |
| B25-02 | 全局唯一身份宣称 | P0 | §2 全文 | 闭合（协议层）| 跨重启唯一 NOT_VERIFIED |
| B25-03 | S12-G 矛盾推断 | P0 | §3 全文 | 闭合 | 硬件/FS 边界 NOT_VERIFIED |
| B25-04 | 目录 fsync 状态映射不统一 | P0 | §4 全文 | 闭合 | 硬件/FS 边界 NOT_VERIFIED |
| B25-05 | ZIP 核验需真实二进制 + 外部回执 | P1 | §6.2 + RECEIPT + ZIP-RECEIPT | 待执行 | 环境无法解压则 NOT_VERIFIED |
| B25-06 | 源码基线需完整清单 + 机器 diff + Git HEAD | P1 | §6.3 + 3 份基线文件 | 待执行 | — |

## 2. 未关闭 / 遗留问题

| 项 | 状态 | 说明 |
|----|------|------|
| 单线程判定对第三方 C 线程 | UNKNOWN | `threading.active_count()==1` 不覆盖 C 线程 |
| 子进程 os._exit(1) 真实行为 | NOT_TESTED | 本机无 fork |
| 随机源不可用降级行为 | NOT_VERIFIED | os.urandom 降级未验证 |
| 跨重启身份唯一性 | NOT_VERIFIED | 无协调器，不持久 |
| 恶意复制身份认证 | UNKNOWN | 方案 B 无认证，不防伪造 |
| 设备写缓存 / 特定 FS 持久化 | NOT_VERIFIED | 需运行时确认 |
| Windows 目录项断电持久性 | NOT_VERIFIED | 无权威 API 完整语义 |
| 第三方 C fork / 嵌入 / 子解释器 / posix_spawn | UNSUPPORTED/DENY | at-fork 钩子无法覆盖 |
| Source Retire | DENY | 保持阻断 |

## 3. 阻断状态（保持）

P0-BLOCK-1、P0-BLOCK-2、P1-BLOCK-3、F1、P0.8、F2 保持阻断；IMPLEMENT-01 NOT_APPROVED；A/B 不合并。
