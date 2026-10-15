# 模块笔记：FTS5 SQL 集成层（fts5_main.c + fts5_storage.c）

规模：`fts5_main.c` 3904 行 + `fts5_storage.c` 1549 行。
职责：把 FTS5 做成一个标准的 SQLite 虚表；管理影子表、事务、写路径、rank 列与扩展 API 的分发。
（行号基于 2026-10-15 工作树。）

## 1. 模块注册与虚表方法表

- **[事实]** 模块结构 `fts5Mod`（`main.c:3763-3790`，`iVersion=4`）+ 注册 `sqlite3_create_module_v2(db,"fts5",&fts5Mod,p,fts5ModuleDestroy)`（`main.c:3818`）；入口 `sqlite3Fts5Init`（`:3901`），可加载扩展入口 `sqlite3_fts3_init`/`sqlite3_fts5_init`（`:3878, 3891`）。

| 方法 | 入口（行号） | 要点 |
|---|---|---|
| xCreate | `fts5CreateMethod` `:477` → `fts5InitVtab(1,…)` | 建表：解析参数、开索引/存储、declare vtab |
| xConnect | `fts5ConnectMethod` `:467` → `fts5InitVtab(0,…)` | 连接已有表，不建影子表 |
| 共用 | `fts5InitVtab` `:393-461` | ConfigParse(411) → IndexOpen(425) → StorageOpen(430) → DeclareVtab(437) → ConfigLoad(442)；设 `SQLITE_VTAB_CONSTRAINT_SUPPORT`（仅 content=NORMAL，`:445-447`）与 `SQLITE_VTAB_INNOCUOUS`（`:448-450`） |
| xBestIndex | `fts5BestIndexMethod` `:614-771` | 见 §3 |
| xDisconnect / xDestroy | `:365 / :373` | 后者先 `sqlite3Fts5DropAll`（`:375`） |
| xOpen / xClose | `:784-808 / :875-889` | 游标挂入 `Fts5Global.pCsr` 链表；xOpen 先 `fts5NewTransaction`（`:791`） |
| xFilter / xNext / xEof | `:1450-1650 / :991-1052 / :1656` | 查询执行三件套 |
| xColumn / xRowid | `:3050-3114 / :1686-1698` | 列取值（含 rank 列）；rowid 来自 sorter/%_content/expr |
| xUpdate | `fts5UpdateMethod` `:1941-2112` | 见 §4 |
| xBegin / xSync / xCommit / xRollback | `:2130 / :2117 / :2143 / :2153` | **xCommit 是空操作**；落盘全在 xSync（§5） |
| xSavepoint / xRelease / xRollbackTo | `:3166 / :3183 / :3201` | savepoint 先 flush，否则回滚点无法包住内存数据 |
| xFindFunction | `:3121-3141` | 把表上函数名解析到已注册的 aux 函数（rank/bm25/highlight/snippet…） |
| xRename | `:3146-3154` → `sqlite3Fts5StorageRename`（`storage.c:288`） | 逐个 ALTER 影子表 |
| xShadowName | `:3713-3722` | 白名单 `{config,content,data,docsize,idx}` |
| xIntegrity | `:3729-3761` | `PRAGMA integrity_check` 走 StorageIntegrity(`:3743`) |

- **[事实]** 三个版本号：`sqlite3_module.iVersion=4`（`:3765`）、`fts5_api.iVersion=3`（`:3802`）、`Fts5ExtensionApi.iVersion=4`（`:2833`）。

## 2. 影子表（storage.c）

- **[事实]** 名字白名单 `{ "config","content","data","docsize","idx" }`（`main.c:3714-3716`）。
- **[事实]** `%_data`/`%_idx` 由 `sqlite3Fts5IndexOpen` 建（`index.c:6904-6912`）；`%_content`/`%_docsize`/`%_config` 由 `sqlite3Fts5StorageOpen` 建（`storage.c:343-417`）。

| 影子表 | schema | 用途 | 备注 |
|---|---|---|---|
| `%_data` | `id INTEGER PRIMARY KEY, block BLOB` | 全部倒排索引 | 见索引笔记 |
| `%_idx` | `segid, term, pgno, PRIMARY KEY(segid,term)` WITHOUT ROWID | (segid,term)→页号 | 段删除时整段清理 |
| `%_content` | `id INTEGER PRIMARY KEY, c0…cN-1 [, l0…]` | 原文 + locale 标签（`locale=1` 时） | external/纯 contentless 时不建 |
| `%_docsize` | `id INTEGER PRIMARY KEY, sz BLOB`（contentless_delete 时加 `origin INTEGER`） | 每列 token 数 varint 数组 | `columnsize=0` 时不建 |
| `%_config` | `k PRIMARY KEY, v` WITHOUT ROWID | 持久化配置（含 `version`） | 建表即写 `version`（`:407-409`） |

- **[事实]** external content（`content='tbl'`）：不建 `%_content`（`storage.c:365-393` 只在 NORMAL/UNINDEXED 建），读写都直接对 `zContent` 表拼 SQL（语句缓存 `storage.c:100-142`）。
- **[事实]** contentless（`content=''`）：无 `%_content`；`columnsize=1` 时 `zContent` 默认指向 `%_docsize` 以便 rowid 扫描/查找（`config.c:682-702`；xFilter scan 分支 `main.c:1623-1642` 依赖它，否则报 "table does not support scanning"）。DELETE 需 `contentless_delete=1`，UPDATE 走专门校验 `fts5ContentlessUpdate`（`main.c:1889-1925`：只有"仅改 UNINDEXED 列"或"contentless_delete 且索引列全改"才允许）。
- **[事实]** `contentless_unindexed=1`：建 `%_content` 但只装 UNINDEXED 列（`config.c:672-702`；`storage.c:372-377, 988-1055`）。
- **[事实]** `locale=1`：`%_content`/外部表的列值以 `fts5_locale()` 包装（`main.c:3648-3693`）。
- **[事实]** external content 且 INSERT rowid=NULL 时，rowid 用 `%_docsize` 的 dummy 行分配（`fts5StorageNewRowid` `storage.c:967-983`）；`%_docsize` 不存在则报 `SQLITE_MISMATCH`（`:959-965`）。

## 3. xBestIndex / xFilter / 游标

### 3.1 xBestIndex（`main.c:614-771`）

- **[事实]** 支持的约束与编码（idxStr，格式注释 `:566-577`）：
  - `MATCH` 表列 → `'M'<iCol>`；`iCol==nCol+1`（rank 列）→ `'r'`；不能用的 MATCH 直接 `SQLITE_CONSTRAINT`（`:657-676`）；
  - `rowid =` → `'='`（只取一个）；`rowid < / <=` → `'<'`；`rowid > / >=` → `'>'`（有 `=` 时不再收 range，`:686-711`）；
  - LIKE/GLOB（tokenizer 支持 pattern 时）→ `'L'/'G'<iCol>`（`fts5UsePatternMatch` `:525-540`）。
  - MATCH 与 `=` 额外 `omit=1`；`needToFreeIdxStr=1`（`:647-648`）。
- **[事实]** ORDER BY：rank 列 + 有 MATCH → `FTS5_BI_ORDER_RANK`（`:721-724`）；rowid → `FTS5_BI_ORDER_ROWID`（`:725-726`；`tokendata=1` 不支持 rowid DESC）；可消费时 `orderByConsumed=1` 并加 `FTS5_BI_ORDER_DESC`（`:728-733`）。
- **[事实]** 成本模型（`:736-767`）：rowid= 且无 MATCH → 25.0（并置 `SQLITE_INDEX_SCAN_UNIQUE`，`fts5SetUniqueFlag` `:503-512` / `fts5SetEstimatedRows` `:514-523`）；有 MATCH：无其他约束 50000、一个 rowid 范围 37500、两个 30000、每多一个 MATCH ×2.5；无 MATCH 扫描：3000000 / 2250000 / 750000。注释解释见 `:542-613`（rank 排序不影响成本，`:601`）。

### 3.2 xFilter（`main.c:1450-1650`）

1. **[事实]** 解码 idxStr → 建 `pCsr->pExpr`（多个 MATCH 用 `sqlite3Fts5ExprAnd` 合并 `:1525`）；`'L'/'G'` → `sqlite3Fts5ExprPattern`（`:1549`）；rowid 边界 `:1558-1566`（DESC 时互换 `:1579-1585`）。
2. **[事实]** 计划选择：`pTab->pSortCsr!=0` → `FTS5_PLAN_SOURCE`（排序器内部游标，`:1590-1610`）；有 expr → 解析 rank（`fts5CursorParseRank` `:1253-1288`）→ `FTS5_PLAN_SORTED_MATCH` + `fts5CursorFirstSorted`（`:1615-1617`）或 `FTS5_PLAN_MATCH` + `fts5CursorFirst`（`:1618-1621`）；无 expr → `FTS5_PLAN_ROWID`/`SCAN`（取 storage 语句并绑参执行，`:1623-1643`）。
3. **[事实]** 推进：`fts5NextMethod`（`:991-1052`）→ `fts5CursorReseek`（`:957-980`，处理写事务置的 `FTS5CSR_REQUIRE_RESEEK`）→ `sqlite3Fts5ExprNext`；SORTED_MATCH 走 `fts5SorterNext`（`:891-927`）；SCAN/ROWID 走 `sqlite3_step`（`:1029-1047`）。
4. **[事实]** `fts5SeekCursor`（`:1708-1750`）按当前 rowid 去 `%_content`/外部表取原值（xColumn 用）；丢行报 "missing row"（`:1736-1741`）。

### 3.3 ORDER BY rank 的实现（关键机制）

- **[事实]** `fts5CursorFirstSorted`（`:1084-1134`）向**同一 vtab 发一条内部 SQL**：`SELECT rowid, rank FROM … ORDER BY rank(...) ASC|DESC`（`:1111-1117`），把结果读入 `Fts5Sorter`（结构 `:168-174`）；外层游标按 rank 序返回。内部游标以 `FTS5_PLAN_SOURCE` 运行避免无限递归（`:1590-1610`）。
- **[事实]** SORTED_MATCH 时 rank 列返回的是 poslist blob（供内部排序 SQL 用，`:3077-3078`），排序后 aux 函数改从 sorter 读位置（`:2706-2717`）。

## 4. xUpdate 写路径（`main.c:1941-2112`，参数布局注释 `:1927-1940`）

- **[事实]** 前置：断言事务已开（`:1953`）；必要重载 config（`:1961-1964`）；`fts5TripCursors` 让 MATCH 游标重新定位（`:1969`）；`sqlite3_vtab_on_conflict` 取冲突策略（`:2003-2006`，仅 NORMAL/contentless_delete 用）。
- **[事实]** 特殊 INSERT（apVal[0]=NULL 且表名列非 NULL）：`'delete'` → `fts5SpecialDelete`（`:1837-1848`）；其余 → `fts5SpecialInsert`（§6）。
- **[事实]** DELETE（nArg==1）：contentless 且无 `contentless_delete` 报错（`:2015-2019`），否则 `sqlite3Fts5StorageDelete`。
- **[事实]** INSERT：REPLACE 冲突时先删旧 rowid（`:2047-2050`），再 `fts5StorageInsert` = ContentInsert(`storage.c:988`) + IndexInsert(`storage.c:1069`)。
- **[事实]** UPDATE（`:2054-2104`）：`iOld!=iNew` 且 REPLACE → 删旧+删新+插新（`:2072-2077`）；否则 `FindDeleteRow`(2079) → `ContentInsert`(2081) → `Delete`(2084) → `IndexInsert`(2087)；仅 UNINDEXED 列变化 → 只写 `%_content`（`:2090-2098`）。
- **[事实]** 旧值来源：external content 用 `FTS5_STMT_LOOKUP`（`SELECT … FROM <zContent> WHERE <rowid>=?`，`storage.c:103, 137-142`）；UPDATE 未修改列由 `sqlite3_value_nochange()` 传入，用 `pSavedRow`（LOOKUP2）兜住原值（`storage.c:19-44, 475-491, 1025-1033, 1095-1101`）。
- **[事实]** `sqlite3Fts5StorageDelete`（`storage.c:771-828`）：先 `sqlite3Fts5IndexBeginWrite(..., bDelete=1, ...)`（`:786`）写删除标记，再删 `%_docsize`（`:804-811`）与 `%_content`（`:813-825`）。

## 5. 事务协议（两级：main → storage → index）

- **[事实]** 协议注释在 `main.c:37-61`：**可能失败的提交工作必须放在 xSync**——因为 SQLite 忽略 xCommit 的错误；调试状态机 `:68-71, 285-332`。
- **[事实]** 流水：`xBegin`(`:2130`) → `fts5NewTransaction`(`:773-779`)（若已有本表游标则空转，否则 Storage/Index Reset 检查 data version）；`xSync`(`:2117`) → `sqlite3Fts5FlushToDisk`(`:3156`) → `sqlite3Fts5StorageSync`(`storage.c:1501`：先回写 averages，再 `sqlite3Fts5IndexSync` `index.c:6839` → `fts5IndexFlush`)；`xCommit` 空转(`:2143`)；`xRollback`(`:2153`) → 丢 hash、失效缓存、`pConfig->pgsz=0` 强制重读配置(`:2158`)。
- **[事实]** savepoint：`xSavepoint` 先 flush（否则回滚点包不住内存数据），成功后记 `iSavepoint+1`（`:3166-3176`）；`xRelease` 在最内层之外再 flush（`:3183-3194`）；`xRollbackTo` 丢内存数据 + `pgsz=0`（`:3201-3211`）。
- **[事实]** 另有 `'flush'` 特殊命令直接触发同一 flush（`:1810-1811`）。

## 6. 特殊控制命令（`fts5SpecialInsert` `main.c:1767-1835`）

写法：`INSERT INTO t(t[, rank]) VALUES('命令' [, 参数])`（文档注释 `:1752-1765`）。**本版本没有 `fts5RunConfigure`**（旧名），全部在此分派：

| 命令 | 行为 | 位置 |
|---|---|---|
| `delete-all` | 仅 contentless/external；清 `%_data/%_idx[/%_docsize]` 并重建 structure | `:1777-1787`；`storage.c:833-867` |
| `rebuild` | 非 contentless；DeleteAll 后重扫内容表重新索引 | `:1788-1797`；`storage.c:869-944` |
| `optimize` | 全索引并成一段 | `:1798-1799`；`index.c:5925-5955` |
| `merge=N` | 用户合并 | `:1800-1802`；`index.c:5961+` |
| `integrity-check` | 校验（external 也可带参做 checksum） | `:1803-1805`；`storage.c:1248-1382` |
| `prefix-index` | 仅 SQLITE_DEBUG | `:1807-1809` |
| `flush` | 立即落盘 | `:1810-1811` |
| `automerge=` `crisismerge=` `usermerge=` `secure-delete=` `deletemerge=` `pgsz=` `hashsize=` `rank=` `insttoken=` | 先 flush + 载入 config，再 `sqlite3Fts5ConfigSetValue`（内存）成功后持久化到 `%_config` | `:1812-1826`；`config.c:920-1046` |
| `'delete'`（单独分支） | `INSERT INTO t(t,rowid,…) VALUES('delete',rowid,…)` | `:1837-1848` |

- **[事实]** special query 是另一类：`MATCH '*…'` 支持 `'*reads'`（返回 `%_data` 读计数）与 `'*id'`（cursor id），`fts5SpecialMatch`（`:1155-1183`）由 xFilter 在 `zText[0]=='*'` 时调用（`:1515-1520`）。

## 7. `%_config` 机制

- **[事实]** 键的持久化清单（`sqlite3Fts5ConfigSetValue` `config.c:920-1046`）：`pgsz`(32..65536)、`hashsize`、`automerge`(0..64；1 视为默认 4)、`usermerge`(2..16)、`crisismerge`(≤1 取默认 16；≥2000 夹到 1999)、`deletemerge`(>100 → 0=关闭)、`rank`（`Bareword( args )` 格式，`sqlite3Fts5ConfigParseRank` `config.c:861-918`）、`secure-delete`（非 0 即开，文件格式版本升 5）、`insttoken`；不识别的键报错（`:1042-1043` + `main.c:1820-1823`）。
- **[事实]** 读回：`sqlite3Fts5ConfigLoad`（`config.c:1051-1104`）：默认值 → 逐行 SetValue → 校验 `version ∈ {4,5}`（不合法报 `invalid fts5 file format … run 'rebuild'`）→ 记 iCookie。
- **[事实]** 写与 cookie：`REPLACE INTO %_config VALUES(?,?)`（`storage.c:114`）；`sqlite3Fts5StorageConfigValue` 另外递增 cookie 并经 `sqlite3Fts5IndexSetCookie` 直接改结构记录头 4 字节（`index.c:7818-7836`）。
- **[事实]** 载入时机：InitVtab（`main.c:442`）、每次 xFilter（`:1587`）、配置类命令后（`:1815, 1831`）、Rollback 后重取 pgsz（`:1961-1964`）。

## 8. auxdata / docsize / rank 的数据链

- **[事实]** auxdata：`Fts5Auxdata`（`main.c:270-275`）挂在 `Fts5Cursor.pAuxdata`；`fts5ApiSetAuxdata`（`:2566-2599`）按函数指针找/建节点，替换时调旧 `xDelete`；`fts5ApiGetAuxdata`（`:2601-2619`）。生命周期 = 一次 MATCH 查询的游标。
- **[事实]** docsize 写：`fts5StorageInsertDocsize`（`storage.c:741-766`）→ `FTS5_STMT_REPLACE_DOCSIZE`；读：`sqlite3Fts5StorageDocsize`（`:1437-1464`）解 varint 数组（`fts5StorageDecodeSizeArray` `:664-675`），坏数据报 `FTS5_CORRUPT`。
- **[事实]** `xColumnSize`（`main.c:2511-2561`）：`columnsize=1` 读 `%_docsize`；否则现场 tokenize 内容表计数。
- **[事实]** rank 数据链（以 bm25 为例）：`xColumnSize(-1)` 得行长 D（`fts5_aux.c:739`）；`xRowCount`/`xColumnTotalSize`（`main.c:2184-2188 / 2174-2182`）读 `%_data` averages 记录（`index.c:7773-7791`）；每查询的 BM25 常量经 auxdata 缓存（`fts5_aux.c:629-697`）。默认 rank 名 `"bm25"`（`Int.h:110`；回退逻辑 `main.c:1279-1285`）。

## 9. 设计取舍

1. **[事实]** 影子表普遍用 `INTEGER PRIMARY KEY`：`%_data.id` 成为 rowid 别名，索引层可以直接 `sqlite3_blob_open(..., iRowid, ...)` 按行号打开单条记录（`index.c:847-874`），并可用 `sqlite3_blob_write` 就地改 cookie——rowid 是"稠密整数空间"，宏在 segid/页号上计算行号（`index.c:271-304`）。
2. **[事实]** 纯映射表才用 `WITHOUT ROWID` + 自然主键：`%_idx`（`index.c:6908-6911`）与 `%_config`（`storage.c:403-405`），省一层 rowid 存储。
3. **[推断]** `%_data` 单表装下全部索引（对比 FTS3/4 的多表）让新格式特性（V2 结构、tombstone 页）不需要改 schema；所有索引写统一受 SQLite 事务/回滚日志保护。
4. **[事实]** 两级存储拆分：storage 管行存/配置/totals，index 管段/合并/flush；提交顺序"内容先落、索引最后 flush"由 `sqlite3Fts5StorageSync` → `sqlite3Fts5IndexSync` 的串行保证。

## 10. 关键函数清单（速查）

`fts5InitVtab` `main.c:393` · `fts5BestIndexMethod` `:614` · `fts5FilterMethod` `:1450` · `fts5CursorFirst` `:1136` · `fts5CursorFirstSorted` `:1084` · `fts5SorterNext` `:891` · `fts5CursorReseek` `:957` · `fts5NextMethod` `:991` · `fts5UpdateMethod` `:1941` · `fts5SpecialInsert` `:1767` · `fts5SyncMethod` `:2117` · `fts5ShadowName` `:3713` · `fts5IntegrityMethod` `:3729` · `fts5CreateAux` `:3216` · `fts5CreateTokenizer_v2` `:3396` / `fts5CreateTokenizer` `:3426` · `sqlite3Fts5LoadTokenizer` `:3533`
`fts5StorageGetStmt` `storage.c:82` · `sqlite3Fts5DropAll` `:250` · `sqlite3Fts5StorageOpen` `:343` · `fts5StorageInsertCallback` `:447` · `sqlite3Fts5StorageDelete` `:771` · `sqlite3Fts5StorageRebuild` `:869` · `sqlite3Fts5StorageIndexInsert` `:1069` · `sqlite3Fts5StorageIntegrity` `:1248` · `sqlite3Fts5StorageSync` `:1501` · `sqlite3Fts5StorageConfigValue` `:1522`
