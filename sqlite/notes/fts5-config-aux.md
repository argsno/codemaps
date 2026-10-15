# 模块笔记：建表配置、辅助函数与外围设施（config / aux / vocab / buffer / tools）

覆盖：`fts5_config.c`（1125 行）、`fts5_aux.c`（830 行）、`fts5_vocab.c`（853 行）、`fts5_buffer.c`（411 行）、`ext/fts5/tool/`、`ext/fts5/test/`、以及扩展 API 的注册流程。
（行号基于 2026-10-15 工作树。）

## 1. `CREATE VIRTUAL TABLE … USING fts5(...)` 参数解析

- **[事实]** 入口链：xCreate/xConnect → `fts5InitVtab`（`main.c:393-461`）→ `sqlite3Fts5ConfigParse(config.c:565-718)` → `sqlite3_declare_vtab`（列 = 用户列 + `<表名>` HIDDEN + `rank` HIDDEN，`config.c:755-776`）→ `sqlite3Fts5ConfigLoad`（`:1051-1104`）。
- **[事实]** **建表参数是手写词法/语法**（`fts5ConfigGobbleWord` 等 `config.c:43-130, 447-485`），**不经过 Lemon**；`fts5parse.y` 只服务 MATCH 表达式。
- **[事实]** 解析规则：先取一个词，跳过空白后若是 `=` 则当选项（值再取一词且必须紧跟结束 `:621-628`），否则当列定义（`:602-645`）；空白只认空格（`fts5_iswhitespace` `:30-32`）；dequote 支持 `" ' [ \`` 四种引号、双写转义（`:145-198`）。
- **[事实]** 列定义只接受 `unindexed` 选项（`:487-512`）；列名不得为 `rank`/`rowid`（`:495-499`），表名不得为 `rank`（`:596-599`）。

### 1.1 CREATE 阶段识别的 10 个选项（分派 `fts5ConfigParseSpecial` `config.c:236-430`）

| 选项 | 语义 | 取值/约束 | 默认 |
|---|---|---|---|
| `prefix=` | 前缀索引长度列表（逗号分隔，字符数） | 1..999；最多 31 个（`Int.h:102`） | 无 |
| `tokenize=` | 分词器名 + 参数 | 重复报 `multiple tokenize=…` | 默认 `unicode61` |
| `content=` | 内容表 | `''`=contentless（NONE）或表名（EXTERNAL，存 `db.tbl`） | NORMAL（自动建 `%_content`） |
| `contentless_delete=` | contentless 也允许 DELETE | 0/1；须真是 contentless，且与 `columnsize=0` 不兼容（`:647-664`） | 0 |
| `contentless_unindexed=` | contentless 保留 UNINDEXED 列内容 | 0/1；须 contentless（`:669-677`） | 0 |
| `content_rowid=` | 外部内容表的 rowid 列名 | 任意标识符 | `rowid` |
| `columnsize=` | 是否维护 `%_docsize` | 0/1 | 1 |
| `locale=` | 每列保存 locale（`l<n>` 列 + `fts5_locale()`） | 0/1 | 0 |
| `detail=` | 索引详细度 | `full`/`columns`/`none`（允许**唯一前缀缩写**，歧义报错 `:207-225`） | `full` |
| `tokendata=` | 索引中额外保存 token 原始数据（配合 xQueryToken/xInstToken） | 0/1 | 0 |

**[事实]** 枚举值（已核对）：`FTS5_DETAIL_FULL=0 / NONE=1 / COLUMNS=2`（`Int.h:285-287`）；content 模式 `NORMAL=0/NONE=1/EXTERNAL=2/UNINDEXED=3`（`Int.h:280-283`）。

### 1.2 只能通过特殊 INSERT 修改的 9 个键

`pgsz`、`hashsize`、`automerge`、`usermerge`、`crisismerge`、`deletemerge`、`rank`、`secure-delete`、`insttoken`
——见 `notes/fts5-sql.md` §6/§7（`config.c:920-1046` + `main.c:1812-1826`）。

### 1.3 `Fts5Config` 关键字段（`Int.h:228-272`）

建表得来：`db/pGlobal/zDb/zName`、`nCol/azCol[]/abUnindexed[]`、`nPrefix/aPrefix[]`、`eContent`、`bContentlessDelete/bContentlessUnindexed`、`zContent/zContentRowid`、`bColumnsize/bTokendata/bLocale`、`eDetail`、`zContentExprlist`（由 `fts5ConfigMakeExprlist` 生成 `T.<rowid>,T.c0,…`，`:517-551`）、内嵌 `Fts5TokenizerConfig t`（含 `ePattern`）。`%_config` 载入：`iVersion/iCookie/pgsz/nAutomerge/nCrisisMerge/nUsermerge/nHashSize/zRank/zRankArgs/bSecureDelete/nDeleteMerge/bPrefixInsttoken`。

## 2. `detail=` 三档对存储的影响（汇总）

- **[事实]** 写入：`none` 不存 position（只写删除/存在标记，`index.c:5739-5747, 4886-4892`）；`columns/full` 写 `nPos = 字节数*2 + bDel` varint + poslist（`index.c:4893-4898`）。
- **[事实]** 读取：`fts5SegIterLoadNPos`（`index.c:1894-1921`）按 detail 分支；迭代器 none 专用 `fts5SegIterNext_None`（`:2232-2295`）。
- **[事实]** 输出回调 `fts5IterSetOutputCb`（`:3890-3920`）选 `_None/_Nocolset/_ZeroColset/_Col100/_Col/_Full`。
- **[事实]** `%_docsize` 与 detail 无关（只由 `columnsize=` 控制）；`averages` = "总行数 + 每列 token 总数"（`storage.c:645-662`）。
- **[事实]** detail 限制：`detail!=full` 且 contentless 时 `xInst/xPhraseFirst` 一律空（`main.c:2324-2329`）；`ORDER BY rank` 的 sorter 仅 full 缓存 poslist（`main.c:2361`）。
- **[事实]** 完整性校验按 detail 分工：full→(col,offset)、columns→列号、none→(0,0)（`storage.c:1201-1217`）。

## 3. `fts5_aux.c`：内建辅助函数

- **[事实]** 注册：`sqlite3Fts5AuxInit`（`aux.c:805-830`）把 `snippet`(`:812`)、`highlight`(`:813`)、`bm25`(`:814`)、`fts5_get_locale`(`:815`) 逐个 `xCreateFunction`；由 `fts5Init` 在模块注册时调用（`main.c:3821`）。
- **[事实]** `CInstIter`（`aux.c:36-47`）：`fts5CInstIterNext`（`:53-78`）把**同列中相互重叠的 phrase 实例合并成连续区间**（`io<=iEnd` 则扩 `iEnd`）——这正是 highlight/snippet 的 "coalesced phrase instance" 语义；`fts5CInstIterInit`（`:84-103`）用 `xInstCount` 取总数。
- **[事实]** `highlight()`（`:242-296`）：固定 3 参（col, open, close）；忽略 COLOCATED（`:166`）；按 range 过滤、进/出短语时开闭标签（`:169-226`）；`xColumnLocale + xTokenize_v2` 走 locale 感知分词（`:276-282`）；`SQLITE_RANGE` 时结果为空串（`:265-267`）。
- **[事实]** `snippet()`（`:427-593`）：固定 5 参（col, open, close, ellipsis, nToken；nToken 夹到 0..64 `:463`）；句子边界探测 `fts5SentenceFinderCb`（`:337-368`：非 COLOCATED、向前跳空白、遇 `.`/`:` 视为句首）——**全库唯一的 "sentence" 概念，只用于片段起点对齐**；打分 `fts5SnippetScore`（`:370-412`：命中新 phrase 记 1000、已见记 1，片段居中到命中处）；句首给 100/120 加分（`:517-536`）。
- **[事实]** `bm25()`（`:601-760`）：首调用 `fts5Bm25GetData`（`:629-697`）算 IDF 并缓存进 auxdata——`avgdl = nToken/nRow`（`:658-662`）；每个 phrase 用 `xQueryPhrase`+`fts5CountCb` 数命中行数（`:613-622`）；IDF 下限 1e-6（`:682-684`，注释给出标准 BM25 公式与负值问题）。打分（`:709-710, 746-752`）：

```
k1 = 1.2, b = 0.75
score += IDF_i * ( f_i*(k1+1) / ( f_i + k1*(1-b + b*D/avgdl) ) )
返回值 = -1.0 * score          # 负值 → ORDER BY rank 升序 = 最佳匹配优先
```

  `f_i` 用 `xInstCount/xInst` 统计（`:725-734`），第 i 列权重取第 i 个函数参数（默认 1.0，`:731`）。
- **[事实]** `fts5_get_locale()`（`:761-803`）：参数为整数列号，越界 `SQLITE_RANGE`；要求 API ≥4（断言 `:775`）。

## 4. `fts5_vocab.c`：fts5vocab 虚表（查看索引内容）

- **[事实]** `CREATE VIRTUAL TABLE v USING fts5vocab(<fts5表>, '<mode>')`；三种模式与 schema（文件头注释 `:13-37`；`:180-184`）：

| 模式 | schema | 内容 |
|---|---|---|
| `col` | `(term, col, doc, cnt)` | 每 term/列一行：doc=含该 term 的行数，cnt=该列实例总数 |
| `row` | `(term, doc, cnt)` | 全表每 term 一行 |
| `instance` | `(term, doc, col, offset)` | 每 term 实例一行（"看索引内容"） |

- **[事实]** xOpen 用 `SELECT t FROM db.tbl AS t WHERE t.tbl MATCH '*id'` 拿 fts5 表句柄（`:356-387`）；xBestIndex 只支持 `term` 列的范围约束（`:275-334`，EQ=100 成本、支持 ORDER BY term ASC）；xFilter 以 `FTS5INDEX_QUERY_SCAN` 扫索引（`:668, 714`）。
- **[事实]** 计数按 detail 分支：row/col 模式在 full 时逐实例数、columns 时把 poslist 直接当列号、none 时退化（`:552-597`）；instance 模式用 `sqlite3Fts5PoslistNext64` 遍历（`:443-492`），**detail=none 时只给一行且 col/offset 为空**（`:474-483`）。测试见证 `test/fts5vocab.test:52-53, 145`。

## 5. `fts5_buffer.c`：二进制缓冲与编码基元

- **[事实]** `struct Fts5Buffer { u8 *p; int n; int nSpace; }`（`Int.h:331-336`）；增长从 64 字节起**倍增**（`buffer.c:18-35`）；保证 `p[n]` 处有 NUL 终止（`:77-90`）。
- **[事实]** 关键函数：`sqlite3Fts5BufferAppendVarint`（`:42-45`，预留 9 字节）、`Put32/Get32`（大端）、`AppendBlob/String/Printf`、`Zero/Set/Free`、poslist 编解码（`PoslistNext64` `:167-205`、`PoslistReader*` `:212-228`、`PoslistSafeAppend`/`PoslistWriterAppend` `:236-262`）、`sqlite3Fts5IsBareword`（`:313-326`，config/expr 词法共用）。
- **[事实]** `Fts5Termset`（`:329-411`）：512 桶哈希，用于 `detail=none` 时 integrity-check 去重 term；哈希算法**刻意与 `fts5_hash.c` 一致**（注释 `:362-365`）。
- **[推断]** 自建 buffer 的动机：需要"紧凑二进制堆缓冲 + 保证 NUL 终止 + 一次性错误传播（`int *pRc`）"这三件事，直接用 SQLite 的通用分配不划算；且它同时被 `%_data` 页写入、hash doclist、poslist 三条路径复用。

## 6. `ext/fts5/tool/` 与测试设施

### 6.1 tool/（6 个 Tcl 脚本）

| 脚本 | 用途 |
|---|---|
| `mkfts5c.tcl` | 拼装单文件 `fts5.c`（amalgamation）；`sqlite3Fts5*` 符号改 static、`yy/TOKEN` 改名（`:111-113, 126-137`）；由 `main.mk:1782-1784` 调用 |
| `loadfts5.tcl` | 把目录树灌进 fts5 表做性能/规模测试；支持 `-fts4/-porter/-delete/-limit/-automerge/-crisismerge/-prefix/-trans/-hashsize/-detail`（usage `:41-58`；建表与特殊 INSERT `:120-150`） |
| `fts5txt2db.tcl` | 从纯文本构造多列 fts4/fts5 表（按 `-colsize` 切 token；`-detail/-prefix/-trigram/-trans/-repeat`，`-repeat` 可测文件名/内容分离——**这是把内容表放 blob 的推荐用法**） |
| `fts5speed.tcl` | 对已有库跑固定查询集并计时（enron 语料常见词条，`:2-17`） |
| `fts5cost.tcl` | 基准：100 万行普通表 vs 100 万行 fts5 表（每行 100 词）的 rowid/match/prefix 查询计时 |
| `showfts5.tcl` | 用 `fts5_decode(rowid, block)` 解析并打印 `%_data` 的结构记录与段内容；`-nterm` 统计 segment term 数 |

另有 `ext/fts5/extract_api_docs.tcl`（从 `fts5.h` 抽 API 文档）与 `mkportersteps.tcl`（生成 porter 代码，见 tokenizer 笔记）。

### 6.2 test/（148 个 `.test` + `fts5_common.tcl`）

- **[事实]** 公共 harness `fts5_common.tcl`：source `test/tester.tcl`；`sqlite3_fts5_may_be_corrupt`；`fts5_test_poslist` 断言工具（`:33-39`）；`foreach_detail_mode {full col none}` 与 `detail_is_full/col/none`（`:462, 480-482`）——**同一批测试在三档 detail 下各跑一遍**。
- **[事实]** 主题分布（代表文件）：基础/语法 `fts5first/simple/query/expr/phrase/near/prefix/colset`；配置 `fts5config/detail/columnsize/tokendata/locale`；内容模式 `fts5content/contentless(1-5)/unindexed/blob`；合并与优化 `fts5merge(1-2)/optimize(1-3)/auto/rank/hash`；索引格式 `fts5doclist/dlidx/onepass/integrity(1-2)/version`；健壮性 `fts5corrupt(1-9,A,B)/fault(1-9,A,B,D-I)/securefault/ubsan/fuzz1/limits/interrupt`；分词 `fts5tokenizer/tok/porter/unicode/umlaut/synonym/bigtok`；SQL 集成 `fts5join/leftjoin/plan/rowid/savepoint/alter/rebuild/update/delete/vocab/aux/cat/bigid`。
- **[事实]** C 侧测试组件随 `SQLITE_TEST` 构建：`fts5_test_mi.c`（自定义 aux 函数 matchinfo 示例 + 取 API 的标准写法 `:71-84`）、`fts5_test_tok.c`（`fts5tokenize` 调试虚表）、`fts5_tcl.c`（Tcl 扩展，注册 `sqlite3_fts5_create_tokenizer` 等一大批命令 `:1714-1747`）。

## 7. 扩展注册流程（插件作者路线）

1. **[事实]** 打开模块（内置或 `.load`）→ `fts5Init`（`main.c:3763+`）建 `Fts5Global`，填 `api.iVersion=3`，按序注册：`sqlite3Fts5IndexInit`（注册 `fts5_decode` UDF 与 `fts5_structure` 虚表）、`sqlite3Fts5ExprInit`、`sqlite3Fts5AuxInit`、`sqlite3Fts5TokenizerInit`、`sqlite3Fts5VocabInit`、以及 SQL 函数 `fts5()`、`fts5_source_id()`、`fts5_locale()`、`fts5_insttoken()`（`main.c:3818-3848`）。
2. **[事实]** 取 API：`SELECT fts5(?1)` + `sqlite3_bind_pointer(..., "fts5_api_ptr", ...)`；实现 `fts5Fts5Func`（`main.c:3608-3619`）；示例 `fts5_test_mi.c:76-80`、`fts5_tcl.c:99-110`。
3. **[事实]** `xCreateFunction`：`fts5CreateAux`（`main.c:3216-3250`）先 `sqlite3_overload_function(db, zName, -1)`，再挂 `Fts5Auxiliary` 到 `pGlobal->pAux` 链表。
4. **[事实]** `xCreateTokenizer(_v2)`：`fts5CreateTokenizer`（`:3424-3452`）/ `fts5CreateTokenizer_v2`（`:3394-3422`）；注册记录进 `Fts5TokenizerModule`（含 v1/v2 包装）。
5. **[事实]** 使用期挂钩：xFindFunction `fts5FindFunctionMethod`（`:3121-3141`）按名字在 `pGlobal->pAux` 找（`fts5FindAuxiliary` `:1190-1199`），命中返回 `fts5ApiCallback` + `Fts5Auxiliary*`；执行时 `fts5ApiCallback`（`:2936-2959`）从**第一个参数（cursor id，即隐藏的表名列）**找回游标再 `fts5ApiInvoke` 调 `xFunc`，传入 `sFts5Api`（`main.c:2832`，iVersion=4）。
6. **[事实]** 生命周期：`sqlite3_create_module_v2(..., fts5ModuleDestroy)`；连接关闭时统一调各 aux/tokenizer 的 `xDestroy`（`main.c:3584-3602`）。

## 8. 三个版本号与文件格式版本（速查）

| 项 | 版本 | 出处 |
|---|---|---|
| `sqlite3_module`（fts5） | 4 | `main.c:3765` |
| `sqlite3_module`（fts5vocab） | 2 | `vocab.c:824` |
| `fts5_api.iVersion` | 3 | `fts5.h:697-698`；赋值 `main.c:3802` |
| `Fts5ExtensionApi.iVersion` | 4 | `fts5.h:350-351`；实例 `main.c:2833` |
| `fts5_tokenizer_v2.iVersion` | 2 | `fts5.h:633-634` |
| `FTS5_CURRENT_VERSION(_SECUREDELETE)` | 4 / 5 | `Int.h:277-278` |

- **[事实]** 文件格式版本写在 `%_config('version')`；打开时校验 4 或 5，否则报 `… run 'rebuild'`（`config.c:1088-1095`；测试 `test/fts5version.test:31-33, 44-48`）；首次 secure-delete 落盘时升到 5（`index.c:5581-5597`）。
- **[未知]** 各功能（detail=none、tokenizer v2、contentless_delete、contentless_unindexed、locale）的**引入版本号**在仓库内无记录，需查官方发布历史。
