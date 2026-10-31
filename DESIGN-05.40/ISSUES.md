# ISSUES.md — 未解决问题清单（05.40）

证据等级：VERIFIED_STATIC + NOT_PROVEN 项
目标：列出 05.40 所有未解决的问题、影响、禁用策略和重新启用条件。

---

## 1. NOT_PROVEN 项（持续禁用）

| 编号 | 问题 | 影响 | 禁用策略 | 重新启用条件 |
|---|---|---|---|---|
| I-01 | 跨进程 128 位 HW CAS 原子性 | 跨进程 Token/状态字修改不安全 | DENY（跨进程功能禁用） | 证明共享内存 + CMPXCHG16B + 16 字节对齐的跨进程原子性 |
| I-02 | 跨重启状态字持久化 | 重启后状态/generation 可能回退 | DENY（跨重启恢复禁用） | 证明 128 位状态字 + generation 单调持久化 |
| I-03 | 跨重启 generation 单调性 | 旧 Token 复活风险 | DENY | 证明 generation 持久化单调无回绕 |
| I-04 | 资源端无原子 token 原语 | 无法安全接管释放 | ISOLATED（禁止接管释放） | 资源端提供 resource_atomic_release 原子原语并证明 |
| I-05 | 资源端 token 持久化失败 | 重启后 token 丢失 | DENY | 证明资源端 token 持久化 |
| I-06 | 版本回绕 ABA（无持久化 epoch） | 版本耗尽后无法安全续期 | DENY（耗尽即终止） | 引入持久化 epoch 且纳入 CAS 比较值并证明 |

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

| 项 | 05.39 缺陷 | 05.40 状态 |
|---|---|---|
| Token 身份唯一性 | 32 位 bit 无代际 | 闭合（slot+generation） |
| 64/128 位 CAS 重叠 | 位宽冲突 | 闭合（统一 128 位） |
| 版本耗尽未全入口 | admit/release 绕过检查 | 闭合（field_cas 内统一检查） |
| 资源端两步 TOCTOU | 校验+释放分离 | 闭合（resource_atomic_release 原子原语） |

---

## 4. 明确未做（边界）

- **未执行运行时测试**：全部 350 条反例均为静态推演，NOT_TESTED_RUNTIME。
- **未修改源码**：25 个 `usit/*.py` 零修改。
- **未进入实现**：IMPLEMENT-01 / A/B Merge / Source Retire 均未授权。

---

## 证据来源

- 协议文档：DESIGN-05.40/ 目录
- 历史约束：05.33–05.39 继承
- 证明等级：VERIFIED_STATIC + NOT_PROVEN（I-01～I-06）
