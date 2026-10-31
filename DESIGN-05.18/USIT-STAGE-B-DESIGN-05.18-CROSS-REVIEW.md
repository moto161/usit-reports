# USIT-STAGE-B-DESIGN-05.18-CROSS-REVIEW｜反例审查记录

> READ-ONLY DESIGN REVISION ONLY · 日期：2026-10-31
> 性质：独立反例审查，主动寻找反例，不复述设计作者证明

## 一、本轮两个关键反例（由权威文档证实）

### 反例 1：+i 不阻止旧 FD 写

- **历史假设**（05.16/05.17）：`chattr +i` 让旧可写 FD 的写入返回 EPERM。
- **权威证据**：chattr(1) man page，BUGS AND LIMITATIONS 节——"Setting 'a' and 'i' attributes will not affect the ability to write to already existing file descriptors."
- **结论**：旧 FD 在 +i 后**仍能写**。05.16/05.17 的"EPERM 阻止旧 FD"假设**错误**。

### 反例 2：+i 阻止 rename

- **历史假设**（05.17）："rename 只改目录项，不受 +i 限制"，故 +i 保持下可 rename 退役。
- **权威证据**：chattr(1) man page，ATTRIBUTES 节，`i` 属性——"cannot be deleted or renamed"。
- **结论**：`+i` 下 rename 返回 **EPERM**，退役无法完成。05.17 的"保持 +i rename"方案**错误**。

## 二、四项 P0 反例审查

| # | 反例 | 结论 |
|---|------|------|
| P0-02 | os._fork 后 child 执行 Python（threading 锁未重置）| 已修复：改 os.fork |
| P0-02 | before 钩子持锁重入死锁 | 已修复：before 无锁 |
| P0-02 | 非主线程/子解释器 fork | DENY |
| P0-04 | 目录项未 fsync 丢失 | 已修复：补 fsync(dirfd) |
| P0-05 | +i 不阻旧 FD 写 | 承认 → DENY |
| P0-05 | +i 阻止 rename | 承认 → DENY |
| P0-06 | ReplaceFileW 部分失败中间态 | 已修复：内容判定 + INDETERMINATE |

## 三、35 条不变量反例检验

主文档 §5 列出 I-01~I-35。其中 I-17（+i 不阻旧 FD）、I-18（+i 阻 rename）是**本轮新发现**，由 chattr(1) 权威文档证实，对应具体反例。每条不变量均有反例检验或诚实标注 NOT_VERIFIED。

## 四、15 条端到端链路审查

主文档 §6.1 列出 L1~L15。L5/L6/L7（Source Retire 相关）**全部 DENY**——因为 +i 方案被推翻，本地 FS 无法证明源内容稳定。其余链路 DESIGN_PROVEN 或 DESIGN_PARTIAL。

## 五、57 条时间线审查

P0-02 十二条 + P0-04 十五条 + P0-05 十五条 + P0-06 十五条 = **57 条**。

## 六、审查结论

本轮最重要的成果是**通过权威文档（chattr(1) man page）发现了 05.16/05.17 连续两轮的错误假设**：`chattr +i` 既不阻止旧 FD 写、又阻止 rename。这导致 Source Retire 的 `+i` 方案彻底失效，必须 DENY。

整体 = **FAIL**（P0-05 阻断项无法闭合，诚实结论为 DENY）。
