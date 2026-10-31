# USIT-STAGE-B-DESIGN-05.24 与 05.23 差异（DIFF）

> 仅列 05.23 → 05.24 实质性差异，不重述未改动内容。

## 1. P0-02｜Fork 回调协议（B24-01）

**05.23**：伪代码手动调用 `_after_in_parent`/`_after_in_child`，同时提及 `os.register_at_fork()`，回调责任未唯一化；隐含假设回调异常可被 try/finally 捕获。

**05.24**：
- **决策：手动接管，不注册 at-fork 回调**。`fork_entry()` 是唯一受控入口，所有回调动作显式唯一，杜绝"回调+手动"双路径重复恢复；
- 不依赖"回调异常被捕获"假设——父进程异常全在 fork_entry 同一 try/except/finally 结构内处理，子进程异常本地处理（记录+DENY），不跨 fork 回传；
- 补齐 9 路径异常清理表（含"回调部分执行后异常""子进程初始化失败"）。

## 2. P0-02｜Epoch 作用域（B24-02）

**05.23**：Epoch 作用域（线程/进程/跨进程）未定义；未评估 PID/Epoch 单独性。

**05.24**：
- 明确 **Epoch 仅进程内唯一**，跨进程由 `process_generation + pid + fork_epoch + fd_generation` 组合标识；
- 明确 PID 单独不足（PID 重用）、Epoch 单独不足（跨进程各自 epoch=1 碰撞）；
- 明确子进程 fork_epoch 从继承值继续（不重置 0、不写父状态）；进程代际在子进程 +1；
- 补齐 PID 重用、进程退出、子再 fork、连续 fork、失败消耗、溢出等边界。

## 3. P0-04｜崩溃矩阵重做（B24-03）

**05.23**：S1~S16 矩阵部分行用"代表性覆盖"，未逐项展开。

**05.24**：S1~S16 **每步独立表格**，逐项覆盖 A~N 十四类位置（含 M=目录项可见但持久性无法确认、N=并发/旧事务残留），不适用位置明确"不适用及理由"，无留空。

## 4. P0-04｜目录 fsync 统一恢复（B24-04）

**05.23**：S13-4"读到旧 current 就继续 S12"不安全。

**05.24**：删除该推断，改为 §4 统一恢复协议——读到旧值**不推断 rename 未发生**，继续核验身份/哈希判断真实状态；S3/S7/S13/S15 目录 fsync 失败统一采用"全对象核验→重试→失败 INDETERMINATE/冲突 QUARANTINED"。

## 5. 交付（B24-05 / B24-06）

- **B24-05**：ZIP 核验增加路径穿越、大小写冲突、目录条目、敏感信息、临时文件、旧版本检查；ZIP 哈希对应最终上传 GitHub 二进制。
- **B24-06**：新增 3 份基线文件（pre/post/diff）作为正式交付物，不只报告"25 文件一致"。

## 6. 数量升级

- 不变量 45 → **50**；链路 20 → **25**；时间线 40 → **50**；反例 30 → **40**。
- 新增证据类型标注（AUTHORITATIVE_SPEC / OFFICIAL_RUNTIME_DOC / OS_DOCUMENTATION / FILESYSTEM_DOCUMENTATION / DEVICE_OR_STORAGE_LIMIT / DESIGN_DERIVATION / NOT_VERIFIED / NOT_TESTED）。

## 7. 术语统一

七态状态名、S1~S16 编号、锁级别 L0~L3、`process_generation + pid + fork_epoch + fd_generation` 组合标识在全文一致。
