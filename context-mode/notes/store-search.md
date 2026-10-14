# 模块笔记：知识库与检索（store.ts / db-base.ts / search/）

## 1. ContentStore 的 SQLite schema（store.ts:463-551）

**事实**：
- `sources` 表：`id, label, chunk_count, code_chunk_count, indexed_at(default datetime('now')), file_path, content_hash`；索引 `idx_sources_label`。（store.ts:465-473, 503）
- **两张 FTS5 虚表**，同 8 列（`title, content` + `source_id/content_type/source_category/session_id/event_id/timestamp` 均 UNINDEXED）：
  - `chunks` → `tokenize='porter unicode61'`（词干匹配主表）
  - `chunks_trigram` → `tokenize='trigram'`（子串匹配副表）
  （store.ts:475-497）
- `vocabulary` 表（`word TEXT PRIMARY KEY`），由 `#extractAndStoreVocabulary` 在索引时灌入（≥3 字符、非停用词、按 `/[^\p{L}\p{N}_-]+/u` 切词）（store.ts:499-501, 1622-1642）
- **没有 SQL trigger**——两表同步靠索引事务里手动双插入（store.ts:1049-1070）；同 label 重索引前先删旧行（574-582）
- 排序权重：`bm25(chunks, 5.0, 1.0)`（标题权重 5 倍）+ `highlight()` 标记命中（585-599）
- schema 迁移：探测 `source_category` 列存在性判定新旧 schema，旧库 DROP+CREATE 重建两表（513-547）；`file_path/content_hash` 用 best-effort `ALTER TABLE` 补（550-551）
- **推断**：`source_category` 声明了但写入恒为 NULL（store.ts:1065-1066），是预留列。

## 2. 分块策略（chunking）

**事实**：
| 类型 | 入口 | 策略 |
|---|---|---|
| Markdown | `#chunkMarkdown`（1646-1763） | H1-H4 标题栈（`" > "` 连接）；遇新标题/`---` 分节；代码围栏整块不拆；标题=标题路径，无则 "Untitled"；`MAX_CHUNK_BYTES=4096`，超限按空行拆并加 `(n)` 后缀 |
| 纯文本 | `#chunkPlainText`（1858-1923） | 默认 20 行/块。策略 A：若空行分段在 3~200 段且每段 <5KB，按段分块（标题取首行 ≤80 字符，否则 `Section N`）；策略 B：固定 20 行 + 2 行重叠（标题 `Lines a-b`）；≤20 行整体一块 `"Output"` |
| JSON | `#chunkJSON`/`#walkJSON`（998-1023, 1925-2022） | 非法 JSON 降级纯文本；递归遍历，标题=JSON 路径（`" > "`）；数组按字节上限分批，batch 标题用 identity 字段（`id,name,title,path,slug,key,label` 优先） |
| 目录 | `indexDirectory`（906-957） | 委托 `store-directory.ts` 有界遍历：maxDepth=5、maxFiles=200（FTS5 膨胀护栏）、固定扩展名白名单、默认排除 node_modules/.git/dist…、尊重 .gitignore（不支持取反）、默认不跟符号链接（环/逃逸风险）；每文件走 `index()` 的安全门 |

超限文本统一走 `#splitOversizedPlainChunk`（码点安全的字节前缀 + 超过 50% 比例才在空白处断行）（1772-1856）。

## 3. 查询管线（核心算法）

**事实**（store.ts:90-125, 1122-1405）：
1. `sanitizeQuery`：剥掉 FTS5 元字符 `' " ( ) { } [ ] * : ^ ~`、丢弃 `AND/OR/NOT/NEAR` 操作词、词典去重、停用词过滤（全停用词则不过滤），每个词加双引号后 `" "`（AND）或 `" OR "`（OR）连接——**FTS5 语法注入防御**。
2. `sanitizeTrigramQuery`：同样清洗 + 要求 ≥3 字符（否则 trigram 查询返回空）。
3. **RRF 融合**（#rrfSearch，1244-1284）：porter 与 trigram **总是并行**执行（OR 模式），各取 `max(limit*2,10)` 条，按 `${source}::${title}` 去重，分数 `Σ 1/(60+rank+1)`——"trigram fallback" 是误称，实际是融合双匹配器。
4. **邻近度重排**（1288-1336）：仅对 top-limit 生效；标题命中加成（code 0.6 / prose 0.3）× 词覆盖率 + `1/(1+最小跨度/长度)` + `0.5*min(1, 相邻对数/4)`（30 字符间隔）。
5. **模糊纠正**（1195-1240）：仅当 RRF 零结果；词长 ≥3，用 vocabulary 表按长度窗口 (±容差) 拉候选，Levenshtein 容差按词长 1/2/3；纠正成功则**整条查询重跑 RRF**；256 条进程内 LRU 缓存。
6. **过滤器**：`source` 子串匹配 `LIKE '%..%' ESCAPE '\'`（用于 `ctx_batch_execute` 时用 exact 变体）；`contentType` code/prose 有专用预编译语句；session 过滤 `#makeSessionFilter`（保留旧 `session_id=''` + 允许集，池放大到 `max(limit*8,40)`）。
7. **搜索增强包装**：`searchWithFallback`（1340-1397）负责 porter→RRF→fuzzy 主链。
8. 文件型来源**自动过期**：每次搜索对 `file_path` 非空的 source 比较 mtime + SHA-256，变了就重索引（#refreshStaleSources，1415-1459）。

## 4. 会话过滤与 timeline

**事实**（search/unified.ts:64-176）：
- `relevance`（默认）：只查 ContentStore。
- `timeline`：合并 ContentStore + 历史 SessionDB 事件 + auto-memory markdown，按时间排序。
- `#737` per-project 隔离：shared-DB 模式下用 SessionDB 把 `project_dir` 转成 session_id 允许集，再对 chunks 做 IN 过滤（server.ts:2697-2702）。

## 5. DB 生命周期与多写者

**事实**（db-base.ts；store.ts:179-261, 409-452；ADR-0001）：
- 路径：`<configDir>/context-mode/content/<sha256(projectDir)[:16]>.db`（darwin/win32 大小写折叠；一次性迁移旧命名含 -wal/-shm）（session/db.ts:430-479；server.ts:686-697）。
- WAL + `synchronous=NORMAL` + `mmap_size=256MB` + `busy_timeout=30000`；**多写者契约**（v1.0.130 移除了 EXCLUSIVE/锁文件）；`withRetry` 对 SQLITE_BUSY 重试 3 次（100/500/2000ms）；损坏 → rename 或删除重建；退出钩子 `wal_checkpoint(TRUNCATE)`；每 50 次插入 + close 时 FTS optimize。
- 清理：死 PID 的 tmpdir DB；>14 天 content DB；WAL 非空超 1h 的僵尸库；>14 天 sources（`ctx_search` 首页会报 stale 计数）。
- 上限常量：chunk 4096B、空段策略 5000B、标题 80 字符、fetch 预览 3072 字符、ctx_search 输出 40KB、snippet 1500 字符。（`getDBSizeBytes` 无调用方——未知用途）

## 6. 其它

**事实**：
- `fetch-cache.ts`：缓存 key = `${source}::${url}`；TTL 24h 存在 sources.indexed_at，过期由通用清理器兜底（13 行小文件，逻辑主要在 server.ts）。
- `truncate.ts`：UTF-8 安全前缀（字节二分 + 代理对退避）、`charSafePrefix`（保证 JSON 合法）、`truncateJSON`、`escapeXML`。实际消费者只有 snapshot.ts 与 server.ts（store.ts 自建了 `#byteCappedPrefix`）。
- `flood-guard.ts`：见 server-tools 笔记第 6 节。
