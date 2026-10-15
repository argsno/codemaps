# 关键结论与证据出处（sqlite / FTS5）

> 组织方式：按主题列出本目录所有笔记里的承重结论及其 `文件:行` 出处（相对 `/Users/argsno/Projects/sqlite`，省略 `ext/fts5/` 前缀的行号均属该目录）。
> 核验状态：标注 ★ 的条目在写作时经原文 grep/sed 抽查核对；其余来自源码通读与子代理调研，未逐条二次核对（冲突处已在 `questions.md` 标 [存疑]）。

## A. 索引格式与存储

| 结论 | 出处 | 核验 |
|---|---|---|
| 全部索引装在 `%_data(id INTEGER PRIMARY KEY, block BLOB)`，6 类记录 | `fts5_index.c:77-118, 6904-6912` | ★ |
| `%_idx(segid, term, pgno, PRIMARY KEY(segid,term))` WITHOUT ROWID | `fts5_index.c:6908-6911` | ★ |
| rowid 位域编码 `segid(16)\|dli(1)\|height(5)\|pgno(31)`；`FTS5_AVERAGES_ROWID=1`、`FTS5_STRUCTURE_ROWID=10`、`FTS5_MAX_PGNO=0x7FFFFF00` | `fts5_index.c:269-304` | ★ |
| 页头 4B：首 rowid 偏移 + szLeaf；页脚 pgidx 为 term 偏移 delta-varint | `fts5_index.c:171-204, 621-625, 822-826` | |
| term 前缀压缩（每页首 term 全量）；跨页首 rowid 写绝对值 | `fts5_index.c:128-170, 190-204, 4483-4559` | |
| 无显式 free space，打包到满即 flush；poslist 按 varint 边界切页 | `fts5_index.c:4498-4504, 4572-4574, 4599-4627` | |
| doclist = 首 rowid 绝对值 + {delta, poslist}* | `fts5_index.c:144-151` | |
| 删除不重写旧数据（写 `nPos==0` 标记），merge 时 annihilation | `fts5_index.c:4873-4874, 5739-5750` | |
| detail 三档枚举 `FULL=0/NONE=1/COLUMNS=2` | `fts5Int.h:285-287`；解析 `fts5_config.c:404-416` | ★ |
| poslist 尺寸字段最低位是删除位；full 的 collist 编码 `0x01+列号` 与 `+2` 增量 | `fts5_index.c:1873-1881, 4895-4896`；`fts5_buffer.c:167-205, 236-251` | |
| structure record 格式（levels/segments/nMerge/nWriteCounter；V2 magic） | `fts5_index.c:88-118, 59-75, 1429-1479` | |
| structure 的一致性校验 | `fts5_index.c:1125-1228` | |
| dlidx：b-tree，记录每页首 rowid；仅跨 ≥4 空页才建；阈值 `FTS5_MIN_DLIDX_SIZE=4` | `fts5_index.c:49, 205-232, 4278-4289, 4377-4435` | ★（49 行） |
| tombstone 页格式与槽位定位（仅 contentless_delete=1） | `fts5_index.c:234-266, 8166-8182, 7873+` | |

## B. 写路径 / 内存缓冲 / 合并

| 结论 | 出处 | 核验 |
|---|---|---|
| 写入先入内存 hash（`Fts5Hash`，key=索引号+term） | `fts5_hash.c:28-77, 245-390` | |
| hash 输出前 32 路归并排序，顺序写段 | `fts5_hash.c:447-488` | |
| flush 触发：rowid 回退 / 同 rowid 二次写 / >nHashSize(1MiB) | `fts5_index.c:6812-6834`；`fts5_config.c:23` | |
| 提交落盘在 xSync；xCommit 空操作 | `fts5_main.c:48-61, 2117-2147` | |
| flush 链：IndexSync → fts5IndexFlush → fts5FlushOneHash → automerge → crisismerge → structureWrite | `fts5_index.c:5836-5854, 5627-5831` | |
| 段不可变；merge 重写并删除输入段 | `fts5_index.c:4437-4474, 4909-4928` | |
| level 0=最新；最多 64 层；promote 合并相近大小的段 | `fts5_index.c:57, 1505-1592, 5805-5822` | ★（57 行） |
| automerge 预算 `nWorkUnit(64)*nWork*nLevel`，每层至少 nMin=4 段 | `fts5_index.c:46-47, 5043-5062, 5018-5020` | ★（46-47 行） |
| crisismerge 默认 16（`FTS5_DEFAULT_CRISISMERGE`）；`FTS5_MAX_SEGMENT=2000` | `fts5_config.c:22`；`fts5Int.h:107` | ★ |
| usermerge / optimize 入口与预算（`FTS5_OPT_WORK_UNIT=1000`） | `fts5_index.c:5925-5984, 5856-5923` | ★（1000/64） |
| merge 主循环 k 路归并 + key annihilation + 删输入段 | `fts5_index.c:4855-4938` | |
| 默认值：page 4050 / automerge 4 / usermerge 4 / crisismerge 16 / hashsize 1MiB / deletemerge 10% | `fts5_config.c:19-25` | ★ |
| 前缀索引 = 同段内首字节分区（`FTS5_MAIN_PREFIX='0'`，第 i 个 `'0'+i`）；上限 31 个、长度 1..999 | `fts5_index.c:51, 7017-7031`；`fts5Int.h:102`；`fts5_config.c:271-291` | ★ |
| 前缀查询三条路径（精确前缀索引 / +1 长度索引+主索引归并 / 主索引 16 路归并） | `fts5_index.c:7452-7503, 6713-6805, 6196` | |
| 游标=`Fts5Iter`，多段归并 + hash 伪段；锦标赛树选最小 | `fts5Int.h:423-430`；`fts5_index.c:656-674, 3960-4043, 3075-3249` | |
| 读输出回调按 detail/colset 分 6 种 | `fts5_index.c:3758-3920` | |

## C. 查询表达式

| 结论 | 出处 | 核验 |
|---|---|---|
| 词法：`"`串（`""`转义）与 bareword 都是 STRING；仅 OR/NOT/AND 为保留字 | `fts5_expr.c:204-266`；bareword 表 `fts5_buffer.c:313-330` | |
| 优先级 `OR<AND<NOT<TERM<COLON`；隐式 AND 只作用于 nearset 列表 | `fts5parse.y:76-80, 132-135`；测试 `test/fts5eb.test:46-52` | |
| `^` 列锚定 = 短语首 term 打 bFirst，要求 offset==0 | `fts5_expr.c:1648-1652, 670` | |
| `*` 前缀只作用于短语最后一个 token | `fts5_expr.c:1889-1891, 985` | |
| 空短语 `""` → 恒不匹配节点；表达式深度上限 256 | `fts5_expr.c:20-22, 65-67, 2411-2414` | |
| 节点唯一虚函数 `xNext(…, iFrom)`；lazy pull 协议 First/Next | `fts5_expr.c:81, 1552-1604` | |
| FTS5_TERM 快路径直接借用 poslist 指针 | `fts5_expr.c:1229-1252` | |
| 短语匹配：多路 reader 对齐 `iPos+i`，跨列天然排除；^ 过滤 offset==0 | `fts5_expr.c:605-687` | |
| NEAR：滑动窗口 `iMin=iMax-nTerm-nNear`，就地裁剪 poslist | `fts5_expr.c:743-831, 767-780, 794-801` | |
| AND 最大 rowid 收敛；OR 选最早+去重；NOT=child0 AND NOT child1 | `fts5_expr.c:1332-1427` | |
| detail≠full 时 NearTest 短路（只判存在） | `fts5_expr.c:906-918` | |
| 构造：词法→Lemon→隐式列过滤→Fts5Expr；分词在构造期 | `fts5_expr.c:271-341, 1847-1899, 1732-1799` | |
| `fts5ExprAnd` 合并多条 MATCH；`ClonePhrase` 不重新分词 | `fts5_expr.c:457-491, 1903-1994` | |
| trigram LIKE/GLOB → MATCH：≥3 字符片段成短语，超集+回查 | `fts5_expr.c:364-430` | |
| xBestIndex 编码 idxStr（'M'/'r'/'L'/'G'/'='/<'/'>'），omit=1 | `fts5_main.c:614-771` | |
| ORDER BY rank → SORTED_MATCH → 内部 SQL 排序器 | `fts5_main.c:721-729, 1084-1134` | |
| 位置数组按需物化（`fts5CacheInstArray`）；detail≠full 重新 tokenize 文档 | `fts5_main.c:2382-2454`；`fts5_expr.c:3084-3112` | |

## D. 分词器

| 结论 | 出处 | 核验 |
|---|---|---|
| 内置注册：v1 注册 unicode61/ascii/trigram，v2 注册 porter | `fts5_tokenize.c:1460-1495` | ★ |
| 默认分词器 = unicode61（注册表第一项） | `fts5_main.c:3286-3290, 3459-3460`；测试 `test/fts5unicode.test:42` | |
| 统一入口 `sqlite3Fts5Tokenize`（v1/v2 分派+惰性实例化） | `fts5_config.c:801-826`；`fts5_main.c:3533-3576` | |
| unicode61：ASCII 快表 + UTF-8 分类；类别号 0 恒为 token 字符 | `fts5_tokenize.c:442-544, 435-440`；`fts5_unicode2.c:253-779` | |
| 折叠只支持 1:1；区间表 + offset 表 | `fts5_unicode2.c:117-250, 131-207` | |
| remove_diacritics 三档；SIMPLE 跳过 HIBIT 复杂映射 | `fts5_tokenize.c:251-253, 395-404`；`fts5_unicode2.c:31-91` | |
| porter 是 v2 包装器，默认包 unicode61；64 字节上限透传；生成代码内联 | `fts5_tokenize.c:552-619, 1187-1272, 746-1167`；`mkportersteps.tcl:13-95` | |
| trigram：<3 字符无 token；`abc*`≡`abc`；ePattern 判定 LIKE/GLOB；提前实例化 | `fts5_tokenize.c:1349-1421, 1369-1377, 1434-1454`；`fts5_main.c:418-420, 525-540` | |
| v1/v2 双向自动适配（wrapper + locale 丢弃/补零） | `fts5_main.c:3303-3452` | |
| 回调协议：字节偏移、顺序回调、COLOCATED=0x0001、DONE 视为成功 | `fts5.h:477-498, 643-650, 687` | |
| 内置分词器不看 FTS5_TOKENIZE_* flags | `fts5_tokenize.c:120, 445, 1352, 1269-1270` | |
| `FTS5_MAX_TOKEN_SIZE=32768` 截断；前缀索引按字符数换算字节 | `fts5Int.h:95`；`fts5_storage.c:458`；`fts5_index.c:6955-6976` | |
| tokenize= 串存进 sqlite_master（分词算法是格式的一部分） | `fts5_config.c:298-337`；`fts5_main.c:1847` | |

## E. SQL 集成 / 事务 / 影子表

| 结论 | 出处 | 核验 |
|---|---|---|
| 模块 iVersion=4；fts5_api.iVersion=3；ExtensionApi.iVersion=4 | `fts5_main.c:3765, 3802, 2833` | ★（3765/3802/2833） |
| 影子表白名单 config/content/data/docsize/idx | `fts5_main.c:3713-3722` | |
| `%_config` WITHOUT ROWID，建表写 version；合法版本 4/5 | `fts5_storage.c:403-409`；`fts5Int.h:277-278`；`fts5_config.c:1088-1095` | |
| 事务：可能失败的提交工作全在 xSync；xCommit 空操作 | `fts5_main.c:37-61, 2117-2147` | |
| savepoint 先 flush；rollback 丢 hash + 失效缓存 + pgsz=0 | `fts5_main.c:3166-3211, 2153-2160` | |
| special 命令表（delete-all/rebuild/optimize/merge/integrity-check/flush/配置键） | `fts5_main.c:1767-1835` | |
| 配置键取值范围（automerge 0..64、usermerge 2..16、crisismerge 夹取、deletemerge>100→0） | `fts5_config.c:920-1046` | |
| xBestIndex 成本模型（25/50000/…；rowid= 置 UNIQUE） | `fts5_main.c:503-523, 736-767` | |
| xUpdate：REPLACE 语义、nochange 旧值、contentless 限制 | `fts5_main.c:1941-2112, 1889-1925`；`fts5_storage.c:19-44, 475-491` | |
| 内容模式枚举 `NORMAL=0/NONE=1/EXTERNAL=2/UNINDEXED=3` | `fts5Int.h:280-283` | |
| external content：不建 %_content，直接读写外部表 | `fts5_storage.c:100-142, 365-393` | |
| contentless：DELETE 需 contentless_delete=1；rowid 分配靠 %_docsize | `fts5_main.c:2013-2023`；`fts5_storage.c:959-983` | |
| bm25 数据链（docsize/averages/IDF 缓存）；公式 k1=1.2 b=0.75 返回 -score | `fts5_aux.c:601-760`；`fts5_main.c:2174-2188, 2511-2561` | |
| `%_idx` pgno 低位是 dlidx 标志（flag + page<<1） | `fts5_index.c:4314, 2734-2736` | |
| integrity-check 机制（xIntegrity / special 命令） | `fts5_main.c:3729-3761, 1803-1805`；`fts5_storage.c:1248-1382` | |

## F. 文档、构建与测试（仓库层面）

| 结论 | 出处 | 核验 |
|---|---|---|
| 版本 3.55.0 | `VERSION` | ★ |
| 源码公有领域、Fossil 为官方 VCS | `AGENTS.md:11-21` | |
| 构建/测试命令（configure --dev / testrunner fts5%） | `AGENTS.md` Build & Testing 节 | ★（原文） |
| FTS5 由 `SQLITE_ENABLE_FTS5` 启用；源清单 `FTS5_SRC`；Lemon 生成解析器 | `main.mk:964, 1758-1780` | ★（1758-1780） |
| amalgamation 由 `tool/mkfts5c.tcl` 拼装 | `main.mk:1782-1784`；`ext/fts5/tool/mkfts5c.tcl:111-137` | |
| 测试规模：148 个 .test + fts5_common.tcl（三档 detail 跑同一批测试） | `ext/fts5/test/`；`fts5_common.tcl:462, 480-482` | |
| FTS5 设计目标三条（定长页/不整载、dlidx、结构记录含合并状态） | `fts5_index.c:22-43`（原文） | |
| 分词器/aux 对外文档在 `fts5.h` 注释里（仓库无 doc/fts5*.md） | `fts5.h:13-18, 408-745`；`doc/` 无 fts5 文件 | |
| FTS3/4 用 `%_segdir`+`%_segments` 两表（对照） | `ext/fts3/fts3.c:718-733` | |
| 官方文档缺口：distributed b-tree 提法、特性引入版本 | 全库 grep 0 命中 | ★ |

## 附：本目录笔记与来源的对应

- `overview.md` ← F 组 + `wc -l` / `git diff` 命令
- `architecture.md` ← A/E 组 + 各文件调用链
- `notes/fts5-index.md` ← A/B 组
- `notes/fts5-query.md` ← C 组
- `notes/fts5-tokenizer.md` ← D 组
- `notes/fts5-sql.md` ← E 组
- `notes/fts5-config-aux.md` ← D/E/F 组
