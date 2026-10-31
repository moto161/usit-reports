# USIT-STAGE-B-DESIGN-05.18-DIFF｜相对 05.17 逐项差异

> READ-ONLY DESIGN REVISION ONLY · 日期：2026-10-31

## 总览

05.18 相对 05.17 的核心变化：**两个阻断项被权威文档证实存在协议错误，一个被废弃（+i）、一个被降级（os._fork）**。

| # | 05.17 设计 | 05.18 修订 | 差异性质 |
|---|-----------|-----------|---------|
| P0-02 | os._fork 持锁 fork | os.fork + before 无锁 + 主线程约束 | 废弃 os._fork，降级 DESIGN_PARTIAL |
| P0-04 | 缺目录项 fsync | 补 fsync(dirfd) 三重 + 分步矩阵 | 补齐 |
| P0-05 | +i 保持可 rename | 废弃 +i，Source Retire DENY | **推翻（权威文档）** |
| P0-06 | ReplaceFileW 部分失败未定义 | 错误矩阵 + 内容判定 | 补齐 |

## 逐项差异（关键）

### P0-02

| 维度 | 05.17 | 05.18 |
|------|-------|-------|
| fork 原语 | os._fork（绕过生命周期）| os.fork（保留生命周期）|
| 重入死锁 | 靠 os._fork 规避 | before 钩子无锁规避 |
| 结论 | PASS | DESIGN_PARTIAL（运行时 NOT_TESTED）|

### P0-05（本轮最重要）

| 维度 | 05.17 | 05.18 |
|------|-------|-------|
| +i 是否阻旧 FD 写 | 认为阻（EPERM）| **不阻**（chattr BUGS 节证实）|
| +i 是否允许 rename | 认为允许（只改目录项）| **不允许**（chattr 属性节证实 EPERM）|
| 结论 | PASS-WITH-ISSUES | **DENY** |

**权威依据**：chattr(1) man page（e2fsprogs 1.47.4）：
- 属性节：`i` 属性 "cannot be deleted or renamed"；
- BUGS 节："Setting 'a' and 'i' attributes will not affect the ability to write to already existing file descriptors"。

这两条直接推翻 05.16/05.17 连续两轮对 `+i` 的错误假设。

### P0-04

| 维度 | 05.17 | 05.18 |
|------|-------|-------|
| 目录项持久化 | 缺 | fsync(versions_dirfd/prepared_dirfd/parent_dirfd) |
| 崩溃矩阵 | 缺目录项维度 | 补全 |

### P0-06

| 维度 | 05.17 | 05.18 |
|------|-------|-------|
| ReplaceFileW 失败 | 假设占位在原位 | 错误矩阵 + 按内容判定 |

## 冻结内容不变

- 源码 scanner.py / cli.py / file_understanding.py 未修改。
- 阻断 IMPLEMENT-01、P0-BLOCK-1/2、P1-BLOCK-3、F1/P0.8/F2 全部保持。
- 05.17 原始交付物未删除、未覆盖、未修改。
