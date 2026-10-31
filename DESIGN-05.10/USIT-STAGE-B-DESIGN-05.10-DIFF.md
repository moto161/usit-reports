# USIT-STAGE-B-DESIGN-05.10｜设计差异清单（DIFF）

> 阶段 B · DESIGN-05.9 → DESIGN-05.10 逐项差异
> 本清单是**设计文档差异**，不是源码差异，也不伪装成源码差异。
> 工单明确：本轮不推倒重做 05.9，仅修复四项 P0，执行顺序 P0-01 → P0-02 → P0-03 → P0-04 → 交叉审查。

---

## 0. 总体差异概览

| 维度 | DESIGN-05.9 | DESIGN-05.10 |
|------|-------------|--------------|
| POSIX 锁机制 | 传统 fcntl 进程关联记录锁 + `F_GETLK` 查自身 | **OFD 锁（平台支持时）+ 独立锁文件（降级）+ ProcessMutex** |
| 自身锁归属验证 | 依赖 `F_GETLK` 返回 PID | **放弃该假设**，改用"获取事实 + FD 打开 + 身份未变"逻辑推导 |
| 进程内线程互斥 | 缺失 | 新增 `ProcessMutex` |
| Windows 自身锁查询 | 隐含等价 F_GETLK | 明确**无**该接口，靠 dwShareMode=0 独占打开间接保证 |
| Journal 隔离 | `_atomic_replace(src, None)` 伪接口 | 改为 `rename src -> src.corrupt` 七步协议 |
| CAS | revision 比较逻辑无效 | `expected_revision` CAS 契约，绑定授权锁 |
| 清理状态机 | 两套表达（CLAIMED/… vs DELETE_INTENT/REMOVED）| 统一单一状态机 |
| 清理审计 | 只存路径/哈希/状态字符串 | 终态快照（完整不可变摘要）|
| 审计存储定位 | 隐含"独立可靠存储" | 明确同 FS 同故障域，不宣称独立可靠 |

---

## 1. P0-01 差异（锁协议）

| 子项 | 05.9 | 05.10 |
|------|------|-------|
| 锁类型 | fcntl 进程关联记录锁 | OFD 锁（`F_OFD_SETLK`），不支持时降级独立锁文件 |
| 自身锁查询 | `F_GETLK` 返回 PID 判断自身持有 | 禁止，`assert_lock_owned` 四层验证（registry / 句柄 / 互斥事实 / 路径身份）|
| 线程互斥 | 无 | ProcessMutex（threading.Lock 按 auth_id）|
| fork | 未明确 | 子进程禁止继承父 Lease，须重新 acquire |
| FD 所有权 | 未明确 | LockHandle 独占，仅 release 关闭，禁止 dup |
| Windows 归属查询 | 隐含等价 | 明确无 F_GETLK 等价物，独占打开间接保证（NOT_VERIFIED）|

---

## 2. P0-02 差异（Candidate 提交与恢复）

| 子项 | 05.9 | 05.10 |
|------|------|-------|
| Candidate 创建 | 未明确排他 | O_EXCL / CREATE_NEW 排他 + 随机命名 + 符号链接防护 |
| 提交持久化 | 顺序不完整 | 十步协议，含 POSIX 父目录 fsync |
| 候选原件 | 可能移动丢失 | 复制（非移动），保留到终态 |
| 恢复矩阵 | 有"重放或 ABORT"二义 | 每组合唯一动作，自动重放提交定为唯一策略 |

---

## 3. P0-03 差异（CAS 与 Journal 隔离）

| 子项 | 05.9 | 05.10 |
|------|------|-------|
| CAS 绑定 | 无有效互斥 | 绑定授权锁，读-比较-写全程持锁 |
| revision 比较 | 逻辑恒真/恒假风险 | expected_revision CAS 契约，失败不覆盖不重试 |
| 隔离伪接口 | `_atomic_replace(src, None)` | `rename src -> src.corrupt`，七步可恢复协议 |
| 临时文件 | 未明确 | 唯一命名 + O_EXCL/CREATE_NEW + 遗留识别 |
| 平台写入顺序 | 仅提及 | POSIX 写→fsync→rename→fsync目录；Windows 写→Flush→MoveFileEx→断电 NOT_VERIFIED |

---

## 4. P0-04 差异（清理归因与终态审计）

| 子项 | 05.9 | 05.10 |
|------|------|-------|
| 审计存储 | 隐含独立可靠 | 同 FS 同故障域，明确限制范围 |
| 审计内容 | 路径/哈希/状态字符串 | 终态快照（TerminalStateSnapshot 完整摘要）|
| 删除顺序 | 先删后记 | DELETE_INTENT 先行，分对象五步 |
| 对象不存在 | 未定义证明 | 必须证明满足删除目标，否则 DENY |
| 状态机 | 两套表达 | 单一 PENDING→DELETE_INTENT→REMOVED→CLEANED |

---

## 5. 未变化部分（明确不改）

- P1-01 / P1-02 / P1-03：状态保持（NOT_VERIFIED / NOT_TESTED / 原状态）。
- 三个阻断项（P0-BLOCK-1、P0-BLOCK-2、P1-BLOCK-3）：未解决。
- F1 / P0.8 / F2：不解除。
- IMPLEMENT-01：继续 STOP。

---

## 6. 结论

DESIGN-05.10 相对 05.9 的差异全部落在四项 P0 的锁协议、提交/恢复、CAS/隔离、清理归因四个层面，均为**设计契约修订**，不涉及任何源码、测试、配置、运行时数据、A/B 工作区或冻结基线的变更。
