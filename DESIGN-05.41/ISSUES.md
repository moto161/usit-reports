# ISSUES.md — 未解决问题清单（05.41）

证据等级：VERIFIED_STATIC + NOT_PROVEN 项
目标：列出 05.41 所有未解决的问题、影响、禁用策略和重新启用条件。

---

## 1. NOT_PROVEN 项（持续禁用）

| 编号 | 问题 | 影响 | 禁用策略 | 重新启用条件 |
|---|---|---|---|---|
| I-01 | 跨进程 txn_lock（COUNT+TRT 原子） | 跨进程 Token 功能不安全 | DENY | 证明跨进程锁 + HW CAS |
| I-02 | 跨进程 fence_lock（资源端串行化） | 跨进程接管释放不安全 | DENY | 证明跨进程 fence_lock |
| I-03 | 跨重启 TRT + 状态字持久化 | 重启后账本可能不一致 | DENY | 证明 TRT+状态字单调持久化 |
| I-04 | 跨重启每槽位 generation 单调 | 旧 Token 复活风险 | DENY | 证明每槽位 gen 持久化单调 |
| I-05 | 资源端 operation 非幂等且无法原子持久化 | 崩溃后不确定 | ISOLATED | 证明 operation 幂等或原子持久化 |
| I-06 | 版本回绕 ABA（无持久化 epoch） | 版本耗尽后无法续期 | DENY | 引入持久化 epoch 并进 CAS |

---

## 2. 继承历史 NOT_PROVEN/UNKNOWN 约束

| 编号 | 约束 | 状态 |
|---|---|---|
| I-07 | 跨快照身份唯一性 | NOT_PROVEN → 禁用 |
| I-08 | Receipt 创建者认证 | NOT_PROVEN → 未认证 Receipt 禁用 |
| I-09 | Break-glass 审计信任根 | NOT_PROVEN → Break-glass 禁用 |
| I-10 | 锁外写入者完备枚举 | UNKNOWN → 跨进程归零/Fork DENY |
| I-11 | Commit/Receipt 覆盖与事务终态证明 | 不足 → Fork/恢复 DENY |

---

## 3. 本轮已闭合（不再视为问题）

| 项 | 05.40 缺陷 | 05.41 状态 |
|---|---|---|
| 全局 generation 误杀他槽位 Token | 全局 TOKEN_GENERATION | 闭合（每槽位独立 generation） |
| 128 位无法容纳每槽位代际 | 内嵌模型 | 闭合（独立 TRT + txn_lock） |
| revoke/consume/release COUNT 歧义 | 未明确 | 闭合（统一操作定义） |
| 版本递增靠注释未执行 | field_cas 注释 | 闭合（commit_txn 内强制检查） |
| 接管与释放锁/CAS 分离 | internal_lock + cas_token | 闭合（统一 fence_lock） |

---

## 4. 明确未做（边界）

- 未执行运行时测试：全部 410 条反例静态推演，NOT_TESTED_RUNTIME。
- 未修改源码：25 个 `usit/*.py` 零修改。
- 未进入实现：IMPLEMENT-01 / A/B Merge / Source Retire 未授权。

---

## 证据来源

- 协议文档：DESIGN-05.41/ 目录
- 历史约束：05.33–05.40 继承
- 证明等级：VERIFIED_STATIC + NOT_PROVEN（I-01～I-06）
