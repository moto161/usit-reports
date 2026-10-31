# USIT-STAGE-B-DESIGN-05.26 哈希清单（HASH-MANIFEST）

> 记录纳入交付的文档与基线文件 SHA-256 与字节数。所有哈希为最终写入后用 `sha256sum` 重算，非 Git Blob SHA-1。
> MANIFEST 自身哈希与 ZIP 哈希见 HASH-RECEIPT / ZIP-RECEIPT。

## 1. 设计文档（前 5 份）

| # | 文件名 | 字节数 | SHA-256 |
|---|--------|--------|---------|
| 1 | USIT-STAGE-B-DESIGN-05.26.md | 37101 | 48b0ff427fffd0a8844ee7d8293d2b734891edc29ddb988b679f44a856a47d7c |
| 2 | USIT-STAGE-B-DESIGN-05.26-ISSUES.md | 4611 | 5c4f713f835716c4d54c9c0d249e757279c0a49abb020fcc163f0415dcfca29d |
| 3 | USIT-STAGE-B-DESIGN-05.26-DIFF.md | 2375 | 5ea037a3c48715c368675ae8235c8ada127c09cef7ffb506873c94866f54e67b |
| 4 | USIT-STAGE-B-DESIGN-05.26-CROSS-REVIEW.md | 1783 | a514136ee7b5a64ff99e6679b1e4ef108d5237612965b540019092b371d2de5c |
| 5 | USIT-STAGE-B-DESIGN-05.26-EXECUTION-REPORT.md | 2959 | 3329ba998a4c6fa8cd677ff4f8396cfd08caf3f207fa0469955ecf78770c9d88 |

## 2. 源码基线文件（3 份）

| # | 文件名 | 字节数 | SHA-256 |
|---|--------|--------|---------|
| 6 | usit_source_baseline_05.26_pre.txt | 2465 | cf30b936e07495cc2f27943c2c124895ea3f7fc1c06bb22c003dcfe0473a139d |
| 7 | usit_source_baseline_05.26_post.txt | 2466 | c4407f6ffb6a15c9ec1118147b64fdd251230a4d6bc5a5d56075c23de5f2fc43 |
| 8 | usit_source_baseline_05.26_diff.txt | 885 | e08884291b82dc0be8ffce4e3e950076d6594d814aba5f84302b60017e44383e |

## 核验命令

```
sha256sum USIT-STAGE-B-DESIGN-05.26.md USIT-STAGE-B-DESIGN-05.26-ISSUES.md \
  USIT-STAGE-B-DESIGN-05.26-DIFF.md USIT-STAGE-B-DESIGN-05.26-CROSS-REVIEW.md \
  USIT-STAGE-B-DESIGN-05.26-EXECUTION-REPORT.md \
  usit_source_baseline_05.26_pre.txt usit_source_baseline_05.26_post.txt \
  usit_source_baseline_05.26_diff.txt
```

核验时间：2026-10-31（GMT+8）
核验工具：GNU sha256sum（Windows 环境）
