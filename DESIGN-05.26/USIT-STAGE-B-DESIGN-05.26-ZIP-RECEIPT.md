# USIT-STAGE-B-DESIGN-05.26 ZIP 外部回执（ZIP-RECEIPT）

> 本文件**不纳入 DELIVERY.zip**，记录 ZIP 自身哈希与完整解压核验结果，避免自引用循环。

## 1. ZIP 自身

| 文件 | 字节数 | SHA-256 |
|------|--------|---------|
| USIT-STAGE-B-DESIGN-05.26-DELIVERY.zip | 25927 | 6c6e046a323e511ba8fdba0459b6b6ffccf6b2855ebf5b47b4936e3f5d7bb4fb |

## 2. ZIP 内 10 文件最终哈希与 CRC（真实解压后重算）

| 文件 | 字节数 | SHA-256 | CRC-32 |
|------|--------|---------|--------|
| USIT-STAGE-B-DESIGN-05.26.md | 37101 | 48b0ff427fffd0a8844ee7d8293d2b734891edc29ddb988b679f44a856a47d7c | 4756e982 |
| USIT-STAGE-B-DESIGN-05.26-ISSUES.md | 4611 | 5c4f713f835716c4d54c9c0d249e757279c0a49abb020fcc163f0415dcfca29d | 22b216bc |
| USIT-STAGE-B-DESIGN-05.26-DIFF.md | 2375 | 5ea037a3c48715c368675ae8235c8ada127c09cef7ffb506873c94866f54e67b | bc354fac |
| USIT-STAGE-B-DESIGN-05.26-CROSS-REVIEW.md | 1783 | a514136ee7b5a64ff99e6679b1e4ef108d5237612965b540019092b371d2de5c | e9ab1508 |
| USIT-STAGE-B-DESIGN-05.26-EXECUTION-REPORT.md | 2959 | 3329ba998a4c6fa8cd677ff4f8396cfd08caf3f207fa0469955ecf78770c9d88 | 3f4f7a1e |
| USIT-STAGE-B-DESIGN-05.26-HASH-MANIFEST.md | 1882 | ed4409b12c138a0bdbd8d981cd07f65d42f665f198da636f90de75b4b86b8787 | 76a1fbbe |
| USIT-STAGE-B-DESIGN-05.26-HASH-RECEIPT.md | 1912 | 0fb0afe32f6dd28c0d9d748e32fdb36409020e93cf79e385b5121448b2fce171 | b2cc0cdf |
| usit_source_baseline_05.26_pre.txt | 2465 | cf30b936e07495cc2f27943c2c124895ea3f7fc1c06bb22c003dcfe0473a139d | cb1f63ce |
| usit_source_baseline_05.26_post.txt | 2466 | c4407f6ffb6a15c9ec1118147b64fdd251230a4d6bc5a5d56075c23de5f2fc43 | f647b842 |
| usit_source_baseline_05.26_diff.txt | 885 | e08884291b82dc0be8ffce4e3e950076d6594d814aba5f84302b60017e44383e | 4f557f33 |

## 3. 核验结论

- ZIP 独立解压到全新空目录 `_verify/`；
- 10 个文件逐字节与源文件比对 **全部 MATCH**；
- CRC-32 由 unzip 校验通过；
- 文件数量、名称、字节数一致，无缺失/额外/重复；
- 无路径穿越、绝对路径、大小写冲突、符号链接、重复条目、目录条目、敏感信息、临时文件、旧版本。

**ZIP 独立解压核验：PASS。**

核验时间：2026-10-31（GMT+8）
核验机器：Windows（Python 3.12.15）
解压工具：Info-ZIP unzip；哈希：GNU sha256sum；比对：GNU cmp
