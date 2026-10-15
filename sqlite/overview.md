# SQLite · FTS5 总览（overview）

> 分析目标：`/Users/argsno/Projects/sqlite` —— SQLite 官方源码树（VERSION: `3.55.0`）。
> 本次聚焦：**FTS5 全文检索模块**的原理、设计、实现与核心逻辑（分析时间：2026-10-15，主干提交 `4226522153`，全程只读）。
> 标注约定：**[事实]** 可直接核对（附 `文件:行` 或命令）；**[推断]** 由实现推导；**[未知]** 未查证，留白不猜。

## 1. 这是什么

- **SQLite**：嵌入式、无服务器、单文件的关系数据库引擎，C 语言实现，源码属公有领域（[事实] `AGENTS.md:11-14`：*"The source is public domain — no copyright or license header should ever be added"*）。官方版本控制用 Fossil（`AGENTS.md:19-21`），本工作区是一个 git 工作树。
- **FTS5**：SQLite 的**全文检索模块**，以虚表（virtual table）形式提供 `CREATE VIRTUAL TABLE ... USING fts5(...)`，支持布尔查询（AND/OR/NOT）、短语、前缀、NEAR、列过滤、BM25 排序、highlight/snippet 等。它位于 `ext/fts5/`，共约 **2.9 万行 C 代码**（`wc -l ext/fts5/*.c ext/fts5/*.h`）——是 SQLite 仓库里最大的单个扩展。
- FTS5 的前身 FTS3/FTS4 同在仓库里（`ext/fts3/`），两者共享"不可变段 + 自底向上构建"的索引思想，但 FTS5 重写了索引格式与查询求值器。对照阅读参考 `ext/fts3/README.content`、`README.tokenizers` 等。

## 2. 受众与阅读价值

- 想在 SQL 层之外理解"全文检索到底怎么工作"的人：FTS5 是教科书级的**倒排索引 + LSM 式分层合并**工业实现，且规模可控（核心就 `fts5_index.c` + `fts5_expr.c` 两个文件）。
- 想给 SQLite 写扩展的人：分词器（tokenizer）与辅助函数（aux function）的插件协议在 `ext/fts5/fts5.h` 里有完整文档注释。
- 想动手改 FTS5 的人：本目录 `architecture.md` + `notes/` 提供到 `文件:行` 的导航。

## 3. 技术栈与工程形态

| 项 | 内容 | 出处 |
|---|---|---|
| 语言 | C（C89 风格，`sqlite3` 内部约定） | 源码 |
| 构建 | autosetup（非 GNU Autoconf） | `AGENTS.md` Build 节 |
| 语法解析 | **Lemon** 解析器生成器（构建期生成 `fts5parse.c/h`，仓库不存生成物） | `main.mk:1775-1780` |
| 测试 | Tcl 脚本经 `testfixture` 解释器运行，FTS5 测试 148 个 `.test` + `fts5_common.tcl` | `ext/fts5/test/`；`AGENTS.md` Testing 节 |
| 工具 | 6 个 Tcl 脚本（amalgamation 打包、性能测试、索引转储等） | `ext/fts5/tool/` |
| 打包 | `tool/mkfts5c.tcl` 把整个模块拼成单文件 `fts5.c` | `main.mk:1782-1784` |

## 4. 如何运行（构建与测试）

官方指引（[事实] `AGENTS.md` 的 Build/Testing 节，命令原文照录）：

```bash
./configure --dev          # 调试构建（autosetup）
make sqlite3               # 命令行 shell
make sqlite3.c             # amalgamation 单文件
make testfixture           # 测试解释器
make tclextension-install  # 装 TCL 扩展，跑测试前需要

# 测试（在构建目录执行）
./testfixture test/main.test   # 单个测试
test/testrunner.tcl            # 快速套件
test/testrunner.tcl fts5%      # 只跑 FTS5（148 个测试文件）
grep '!' testrunner.log        # 查失败
make devtest                   # 最快的代表性子集
```

**[事实]** FTS5 由编译期开关 `SQLITE_ENABLE_FTS5` 启用（`main.mk:964`；入口挂在 `src/main.c:44,61`），并把它列进构建源清单 `FTS5_SRC`（`main.mk:1758-1773`，其中 `fts5parse.c/h` 由 Lemon 从 `fts5parse.y` 现场生成，见 `main.mk:1775-1780`）。

**[未知]** 本次分析未执行构建与测试（只读分析），因此所有测试的通过情况未知。想验证结论时，`test/testrunner.tcl fts5%` 是最快的核对手段。

## 5. FTS5 代码地图（含规模）

| 文件 | 行数 | 职责 |
|---|---|---|
| `fts5_index.c` | 9590 | **核心**：磁盘索引格式、segment、合并、游标、前缀索引、tombstone |
| `fts5_main.c` | 3904 | 虚表模块：xCreate/xBestIndex/xFilter/xUpdate、事务、special 命令、扩展 API 分发 |
| `fts5_expr.c` | 3286 | MATCH 表达式解析（含手写词法）与求值（短语/NEAR/AND/OR/NOT 迭代器） |
| `fts5_storage.c` | 1549 | 存储层：`%_content`/`%_docsize`/`%_config` 的 CRUD、external content、contentless |
| `fts5_tokenize.c` | 1496 | 内置分词器：unicode61 / ascii / porter / trigram |
| `fts5_config.c` | 1125 | `CREATE VIRTUAL TABLE` 参数解析、配置读写、统一分词入口 |
| `fts5_unicode2.c` | 780 | 机器生成的 Unicode 表（字符分类、大小写折叠、去变音） |
| `fts5_vocab.c` | 853 | `fts5vocab` 虚表（row/col/instance 三模式，用于查看索引内容） |
| `fts5_aux.c` | 830 | 内建辅助函数：highlight / snippet / bm25 / fts5_get_locale |
| `fts5_hash.c` | 590 | 内存 pending terms 哈希表 |
| `fts5_varint.c` | 344 | 变长整数编解码 |
| `fts5_buffer.c` | 411 | `Fts5Buffer` 二进制缓冲工具 |
| `fts5.h` | 754 | **对外 API**：tokenizer v1/v2、aux function、`fts5_api` 注册接口（含长篇文档注释） |
| `fts5Int.h` | 959 | 内部头文件：全部结构与跨文件函数声明 |
| `fts5parse.y` | 193 | Lemon 文法（MATCH 查询语法） |
| `fts5_tcl.c` / `fts5_test_mi.c` / `fts5_test_tok.c` | 1753/435/484 | 测试支撑（Tcl 扩展、matchinfo 示例、tokenize 调试虚表） |

## 6. 磁盘上有什么：影子表（shadow tables）

`CREATE VIRTUAL TABLE t USING fts5(a, b)` 会在同一个库里生成一组**影子表**（[事实] 名字白名单 `fts5_main.c:3713-3722`）：

| 影子表 | schema | 存什么 |
|---|---|---|
| `t_data` | `id INTEGER PRIMARY KEY, block BLOB` | **全部倒排索引**：段页、doclist 索引页、结构记录（行 10）、统计记录（行 1）等 6 类记录（`fts5_index.c:6904-6912`；记录清单 `fts5_index.c:80-118`） |
| `t_idx` | `segid, term, pgno, PRIMARY KEY(segid,term)`, **WITHOUT ROWID** | "段内每个页的首 term → 页号" 的查找表（辅助 b-tree，`fts5_index.c:6908-6911`） |
| `t_content` | `id INTEGER PRIMARY KEY, c0…cN-1 [, l0…]` | 原文（`content=` 为 external/contentless 时不存在或部分存在） |
| `t_docsize` | `id INTEGER PRIMARY KEY, sz BLOB` | 每行每列的 token 数（`columnsize=0` 时不建） |
| `t_config` | `k PRIMARY KEY, v` **WITHOUT ROWID** | 持久化配置（版本号、automerge、rank 等） |

**[事实]** 一个设计要点：索引数据**不放在 SQLite 的普通 b-tree 里**，而是作为 blob 按自定义的"页面格式"塞进 `t_data` 的行里，行号本身带着"段号+页号"的编码——这是 FTS5 能精确控制 I/O 粒度（定长页、双向迭代、seek）的前提（`fts5_index.c:22-43, 269-304`）。

## 7. 与 FTS3/FTS4 的关系（一句话）

**[事实]** FTS5 索引层自述与 FTS3/4 的三处关键差异（`fts5_index.c:24-41` 原文要点）：
1. 段数据**全部存定长页记录**，单个 doclist 可跨多页，且"不整体读入内存即可双向迭代或 seek"；
2. 跨多页的大 doclist 带 **doclist index**（记录每页首个 rowid），加速 seek 与"大 doclist 合并小 doclist"；
3. 结构记录多带字段，**持久化增量合并（incremental merge）的状态**。

**[事实]** 存储布局差异：FTS3/4 用 `%_segdir` + `%_segments` 两张表（`ext/fts3/fts3.c:718-733`），FTS5 收敛为单张 `%_data` + `%_idx`。

## 8. 本次分析的边界与可靠性说明

- **[事实]** 目标仓库存在一处**未提交的本地改动**：`ext/fts5/fts5_aux.c` 头部注释块的行文调整（`git diff ext/fts5/` 仅注释，无代码逻辑变化）。本笔记基于该工作树状态。
- **[事实]** 本笔记所有结论来自源码通读 + 并行子代理调研 + 抽查核对（关键常量已逐条 grep 验证）；**未运行任何构建与测试**。
- **[未知]** 凡涉及"哪个版本引入某特性"的问题（detail=none、tokenizer v2、contentless_delete、locale 等的引入版本号），源码树内无记录，需查官方发布历史（见 `questions.md`）。

## 9. 从哪开始读（建议路径）

1. `ext/fts5/fts5_index.c:22-318` —— 文件头就是**索引格式规范**，先读完它。
2. `ext/fts5/fts5.h:408-745` —— tokenizer / aux function 的对外契约。
3. `fts5_main.c:37-61`（事务协议）+ `fts5_main.c:3763-3790`（模块方法表）—— 虚表如何接线。
4. 然后按 `architecture.md` 的图进入 `notes/` 的五个模块笔记。
