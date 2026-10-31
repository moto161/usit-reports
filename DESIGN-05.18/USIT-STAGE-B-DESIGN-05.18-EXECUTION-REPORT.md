# USIT-STAGE-B-DESIGN-05.18-EXECUTION-REPORT｜实际执行报告

> 工单：USIT-STAGE-B-DESIGN-05.18
> 执行时间：2026-10-31（GMT+8）
> 执行环境：本地 Windows 工作区（bash + PowerShell + git + WebSearch/FetchURL 查证权威文档）
> 工作目录：`C:\Users\Administrator\WorkBuddy\2026-10-15-22-09-47\usit_p0`
> 输出目录：`C:\Users\Administrator\WorkBuddy\2026-10-15-22-09-47\usit_p0_05.18_reports`（项目外）
> 执行模式：**READ-ONLY DESIGN REVISION ONLY**

## 一、实际读取的历史文件清单

- DESIGN-05.11～05.17 主设计、ISSUES、DIFF、CROSS-REVIEW、EXECUTION-REPORT、HASH-MANIFEST、HASH-RECEIPT
- DESIGN-05.17 ZIP（本地 `usit_p0_05.17_reports/USIT-STAGE-B-DESIGN-05.17-交付物.zip`，22860 字节）
- ChatGPT 05.17 独立复审报告（两个阻断项：os._fork 未证明运行时安全；+i 保持可 rename 未核实）

**权威文档查证（本轮新增，关键）**：
- **chattr(1) man page**（man7.org，e2fsprogs 1.47.4）—— 证实 `+i` 属性 "cannot be deleted or renamed"，且 BUGS 节证实 "will not affect the ability to write to already existing file descriptors"。
- **Python os 模块文档**（docs.python.org）—— 证实 `os.fork()` 触发 fork 生命周期钩子，`os._fork()` 不触发。

## 二、缺陷追踪表

见主文档 §0（四行，含 05.17 失败原因、05.18 修订、反例、验收状态）。

## 三、每项修改位置、证明、反例、结论

| # | 修改位置 | 核心结论 | 关键反例 |
|---|---------|---------|---------|
| P0-02 | §1 | DESIGN_PARTIAL | os._fork 后执行 Python 不安全 → 改 os.fork |
| P0-04 | §2 | DESIGN_PROVEN（POSIX）| 目录项未 fsync 丢失 → 补 fsync(dirfd) |
| P0-05 | §3 | **DENY** | +i 不阻旧 FD / +i 阻 rename（权威文档）|
| P0-06 | §4 | DESIGN_PARTIAL | ReplaceFileW 部分失败 → 内容判定 |

## 四、不变量/链路/时间线数量

- 不变量：**35 条**（I-01~I-35）
- 端到端链路：**15 条**（L1~L15）
- 时间线：**57 条**（F12 + G15 + H15 + J15）

## 五、未解决问题及优先级

见 ISSUES 文件 U-1~U-8。**U-1（Source Retire 无法证明）和 U-2（os._fork 运行时）为 P0 级**。

## 六、交付文件清单、字节数、SHA-256

见 HASH-MANIFEST 与最终答复。

## 七、ZIP 独立解压和逐字节核验

见最终答复（Expand-Archive + sha256sum，全部 MATCH）。

## 八、执行边界确认

- 源码修改：**NO**（仅在项目外 `usit_p0_05.18_reports` 创建 Markdown + ZIP，未触碰 usit_p0 工作区任何文件）。
- 构建：**NOT_EXECUTED**。
- 运行时测试：**NOT_EXECUTED**。
- 故障注入：**NOT_EXECUTED**。
- 真实隔离/删除/归档/清理：**NOT_EXECUTED**。
- IMPLEMENT-01：**NOT_EXECUTED**。

## 九、冻结文件哈希（不变证据）

- `usit/scanner.py` = `58ba5f25483429ce7482ae1d852deabce6ea0490f2e106c6955c6ff5323b45e3`
- `usit/cli.py` = `879a3735838e183f41dd08054095b514ffc0e81e2b43776b75a77e25d138b8d9`
- `usit/file_understanding.py` = `71995a09da04427c028282cfeba17c6b12ebdbcae97796adc2998af7c455ce5b`

## 十、每项验收结果

| 项 | 结果 |
|----|------|
| P0-02 | DESIGN_PARTIAL |
| P0-04 | DESIGN_PROVEN（POSIX）/ NOT_VERIFIED（Windows）|
| P0-05 | **DENY** |
| P0-06 | DESIGN_PARTIAL |
| 恶意同 UID / 特权 -i / Windows 持久化 | NOT_VERIFIED |
| CPython fork 运行时 / chattr 语义 | NOT_TESTED |

## 十一、最终结论

**DESIGN-05.18 = FAIL（整体）**。

依据：
1. **P0-05 是阻断项，本轮诚实结论为 DENY**——chattr(1) 权威文档证实 `+i` 既不阻止旧 FD 写（BUGS 节）、又阻止 rename（属性节），05.16/05.17 连续两轮的 `+i` 方案错误。Source Retire 在本地文件系统上无法证明安全，必须 DENY，不能 PASS。
2. **P0-02 降级为 DESIGN_PARTIAL**——`os._fork()` 绕过 CPython 生命周期后执行 Python 无法证明安全，05.17 的 PASS 不成立；05.18 改回 `os.fork()`，但多线程 fork 运行时仍 NOT_TESTED。
3. P0-04/P0-06 补强到位，但 Windows 持久性 NOT_VERIFIED。

**由于 P0-05 阻断项无法闭合（DENY），整体不得 PASS。IMPLEMENT-01 继续 STOP，等待 ChatGPT 独立复审。**

---

## 附：本轮的关键技术发现（供复审参考）

本轮通过查证 **chattr(1) 官方 man page**，发现并纠正了贯穿 05.16/05.17 两轮的核心错误：

| 错误假设 | 权威事实 | 来源 |
|---------|---------|------|
| "`+i` 让旧 FD 写返回 EPERM" | `+i` **不影响**已 open FD 的写 | chattr(1) BUGS AND LIMITATIONS |
| "`+i` 保持下可 rename（只改目录项）" | `+i` 下文件 **cannot be deleted or renamed** | chattr(1) ATTRIBUTES, `i` |

这意味着：**本地 Linux 文件系统（以及 macOS/Windows）不存在能同时"阻止旧 FD 写"和"允许路径移动"的强制原语**，因此 Source Retire 的"强制不可变边界"在纯本地环境下**无法建立**，必须 DENY，或引入跨信任边界的独立特权服务（NOT_IMPLEMENTED）。
