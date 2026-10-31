# ISSUES.md — 未解决问题清单（05.42）

证据等级：VERIFIED_STATIC + NOT_PROVEN 项
目标：列出 05.42 所有未解决的问题、影响、禁用策略和重新启用条件。

---

## 1. NOT_PROVEN 项（持续禁用）

| 编号 | 问题 | 影响 | 禁用策略 | 重新启用条件 |
|---|---|---|---|---|
| I-01 | 跨进程 WAL 追加原子性 | 跨进程事务不安全 | DENY | 证明跨进程 WAL 追加 + fsync 原子 |
| I-02 | 跨进程 txn_lock / fence_lock | 跨进程串行化不安全 | DENY | 证明跨进程锁 |
| I-03 | 跨重启 WAL 持久化介质 + fsync 原子性 | 崩溃后无法可靠重放 | DENY | 证明持久化介质原子性 |
| I-04 | 跨重启每槽位 generation 单调 | 旧 Token 复活风险 | DENY | 证明每槽位 gen 持久化单调 |
| I-05 | 跨重启 consumed_records 持久化 | 幂等释放失败 | DENY | 证明幂等记录持久化 |
| I-06 | operation 非幂等且无法原子持久化 | 崩溃后不确定 | ISOLATED | 证明 operation 幂等或原子持久化 |
| I-07 | 版本回绕 ABA（无持久化 epoch） | 版本耗尽后无法续期 | DENY | 引入持久化 epoch 并进 CAS |
| I-08 | WAL checksum 无法检测的静默损坏 | 恢复错误 | 依赖底层存储完整性 | 底层存储提供完整性保证 |

---

## 2. 继承历史 NOT_PROVEN/UNKNOWN 约束

| 编号 | 约束 | 状态 |
|---|---|---|
| I-09 | 跨快照身份唯一性 | NOT_PROVEN → 禁用 |
| I-10 | Receipt 创建者认证 | NOT_PROVEN → 未认证 Receipt 禁用 |
| I-11 | Break-glass 审计信任根 | NOT_PROVEN → Break-glass 禁用 |
| I-12 | 锁外写入者完备枚举 | UNKNOWN → 跨进程归零/Fork DENY |
| I-13 | Commit/Receipt 覆盖与事务终态证明 | 不足 → Fork/恢复 DENY |

---

## 3. 本轮已闭合（不再视为问题）

| 项 | 05.41 缺陷 | 05.42 状态 |
|---|---|---|
| 两独立存储冒充原子事务 | mutex+CAS | 闭合（WAL + COMMIT fsync 提交点） |
| 恢复误清零历史 generation | bit=0 判 TRT 领先 | 闭合（重放 WAL，gen 由记录保留） |
| revoke 破坏 COUNT 不变式 | revoke_pending | 闭合（revoke 同事务减 COUNT） |
| 锁提前返回泄漏 | 错误分支不释放锁 | 闭合（try/finally） |
| 幂等释放误返回 FENCED_OUT | token 递增丢失历史 | 闭合（consumed_records） |
| owner/resource 未校验 | 只校验 slot+gen | 闭合（全字段校验） |
| 版本检查依赖 txn.changed | 调用方维护辅助集合 | 闭合（提交入口自动检测） |
| release/slot_free 代际不一致 | 两份实现 | 闭合（唯一 release） |

---

## 4. 明确未做（边界）

- 未执行运行时测试：全部 460 条反例静态推演，NOT_TESTED_RUNTIME。
- 未修改源码：25 个 `usit/*.py` 零修改。
- 未进入实现：IMPLEMENT-01 / A/B Merge / Source Retire 未授权。

---

## 证据来源

- 协议文档：DESIGN-05.42/ 目录
- 历史约束：05.33–05.41 继承
- 证明等级：VERIFIED_STATIC + NOT_PROVEN（I-01～I-08）
