# USIT-STAGE-B-DESIGN-05.29 — 回执归属与幂等协议（POSIX-RECEIPT-OWNERSHIP.md）

> B29-03 支撑文档。重建回执归属证明与幂等协议（P03）。

## 1. 撤销 05.28 的不足

05.28 用 `(txn_id, revision)` 唯一键 + "匹配则幂等成功"，但未证明"EEXIST 后发现的回执文件一定属于本次事务"。本轮重建归属证明。

## 2. 回执唯一键（四元组）

```
receipt_key = (txn_id, revision, current_inode_dev, current_sha256)
```

- `txn_id`：事务身份。
- `revision`：对象版本。
- `current_inode_dev`：current 的 inode/dev 对（POSIX stat st_dev+st_ino）。
- `current_sha256`：current 内容 SHA-256。

四者共同标识"本次事务提交的 current 对象"，**任一不匹配即不属于本次事务**。

## 3. 回执对象归属证明

**不能因为文件名、txn_id 字符串、内容部分匹配就认定文件属于本次事务。**

EEXIST 后必须核验：

1. 对象身份（inode/dev 与记录一致）。
2. 对象类型（regular file）。
3. 文件内容（完整，无短写）。
4. 事务归属（四元组全匹配）。
5. revision 匹配。
6. 对象未被其他事务引用。
7. 无外部并发修改痕迹。
8. 操作结果不处于 UNKNOWN。
9. 删除/覆盖不破坏已提交数据。

辅助元数据（创建时间、文件大小）**不能单独**作为所有权证明。

## 4. EEXIST 处理协议

- 读取已存在回执 → 核验完整四元组 + 文件类型 + 内容完整。
- 四元组全匹配 → **幂等成功**（本次提交已记录，跳过写入）。
- 任一不匹配 → **QUARANTINED**（可能是其他事务残留或外部伪造）。
- 无法确认创建者/归属 → **QUARANTINED**，禁止删除或覆盖。

## 5. 回执幂等协议（S14/S15）

| 步骤 | 处理 |
|------|------|
| 回执唯一键 | (txn_id, revision, inode_dev, sha256) 四元组 |
| txn_id/revision 绑定 | 回执内容含二者，写入时校验 |
| current 对象身份绑定 | 回执含 current inode/dev |
| current 内容哈希绑定 | 回执含 current SHA-256 |
| parent_revision 绑定 | 回执含父版本 |
| 排他创建 | open(O_CREAT\|O_EXCL) |
| EEXIST | 核验四元组，匹配幂等/不匹配 QUARANTINED |
| 短写 | 续写直到完成，失败删半成品（身份确认后）|
| fsync(fd) 失败 | 重读核验后重试 |
| fsync(dirfd) 失败 | 重读核验后重试，不盲目 |
| 回执半成品识别 | 内容不完整 + 身份匹配 |
| 回执重复创建 | 幂等跳过 |
| 回执与 current 不一致 | QUARANTINED |
| 多事务同 revision | revision 冲突 → QUARANTINED |
| 回执被外部替换 | 身份不符 → QUARANTINED |
| 回执被复制/伪造 | 无认证，UNKNOWN |
| 恢复进程无法确认创建者 | QUARANTINED |
| 已提交但回执未完成 | 补回执（AUDIT_PENDING）|

## 6. 禁止覆盖写入

恢复时一律**排他创建 + 核验**，不直接 open(O_TRUNC) 覆盖。禁止普通覆盖作为默认恢复手段。

## 7. 结论

回执归属由四元组 + 文件类型 + 内容完整性证明；无法确认归属时 QUARANTINED。协议层 DESIGN_PROVEN，运行时 NOT_TESTED。
