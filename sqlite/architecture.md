# FTS5 架构（architecture）

> 目标：给出 FTS5 的**模块划分、关键抽象、依赖方向与四条主数据流**。所有条目按 [事实]/[推断]/[未知] 标注，出处精确到 `文件:行`。
> 源码根：`/Users/argsno/Projects/sqlite/ext/fts5/`（下称简写，例：`index.c` = `fts5_index.c`）。

## 1. 分层与依赖方向

```
                     SQL 客户端（SELECT/INSERT/UPDATE/DELETE, MATCH 表达式）
                                        │
┌───────────────────────────────────────▼────────────────────────────────────────┐
│ ① SQL 集成层  fts5_main.c (3904行)                                              │
│    sqlite3_module(fts5) 虚表实现：xCreate/xConnect/xBestIndex/xFilter/xUpdate/  │
│    xBegin/xSync/xCommit/xRollback/xSavepoint/xRename/xShadowName/xIntegrity…    │
│    游标 Fts5Cursor、rank 函数分派、special INSERT 命令、fts5_api 扩展分发        │
└───────┬─────────────────────────────────────────────────────┬──────────────────┘
        │ 调用                                            调用 │
┌───────▼───────────────────────────┐          ┌──────────────▼──────────────────┐
│ ② 存储层  fts5_storage.c (1549行)  │          │ ④ 查询表达式层 fts5_expr.c(3286) │
│  %_content/%_docsize/%_config 的   │          │  MATCH 串 → Fts5Expr 树 →       │
│  CRUD、external content 取旧值、    │          │  拉取式迭代器（First/Next）       │
│  contentless 规则、totals(avgdl)   │          └──────────────┬──────────────────┘
└───────┬───────────────────────────┘                          │ 每个 term 打开迭代器
        │                                                      │
┌───────▼──────────────────────────────────────────────────────▼──────────────────┐
│ ③ 索引层  fts5_index.c (9590行) —— 核心                                          │
│  pending hash（fts5_hash.c）→ flush 成 segment → 分层 merge → %_data 页          │
│  多段归并游标 Fts5Iter、前缀索引、doclist index、tombstone、结构记录缓存          │
└───────┬─────────────────────────────────────────────────────────────────────────┘
        │ 按 (rowid) 读写 blob
┌───────▼─────────────────────────────────────────────────────────────────────────┐
│ ⑤ 影子表： t_data(id,block)  t_idx(segid,term,pgno)  t_content  t_docsize  t_config│
└─────────────────────────────────────────────────────────────────────────────────┘

横切组件：fts5_config.c（建表参数/配置解析，所有层都读它）
          fts5_tokenize.c + fts5_unicode2.c（分词器插件，写路径与查询路径共用）
          fts5_varint.c / fts5_buffer.c（编码与缓冲基元）
附加模块：fts5_aux.c（highlight/snippet/bm25）  fts5_vocab.c（fts5vocab 虚表）
```

- **[事实]** 依赖方向：`main → storage → index`（写/事务路径）与 `main → expr → index`（查询路径）；`index` 不认识 `expr`/`main`。出处：各文件的函数调用（如 `sqlite3Fts5StorageIndexInsert` `storage.c:1069` 调 `sqlite3Fts5IndexBeginWrite` `index.c:6812`）。
- **[事实]** `fts5_config.c` 是共享底座：`Fts5Config` 被 index/storage/expr/tokenize 全部引用（`fts5Int.h:228-272`）。
- **[推断]** 这个切法的意图：把"格式与合并"（最难变、性能最敏感）与"SQL 语义"（最善变）隔开；`fts5_storage.c` 是较新的拆分（FTS5 早期把存储与索引混在 `index.c`），现在把 `%_content` 一类的行存/配置职责单独成文件。

## 2. 关键抽象（结构体速查）

| 抽象 | 定义处 | 一句话职责 |
|---|---|---|
| `Fts5Global` | `main.c:78` | 每连接全局：模块链表、aux/tokenizer 注册表、游标链表 |
| `Fts5FullTable` | `main.c:142` | vtab 实例：包着 `pConfig + pIndex + pStorage + pSortCsr` |
| `Fts5Cursor` | `main.c:198` | 查询执行状态：计划（MATCH/SCAN/SORTED…）、表达式、绑定参数、auxdata |
| `Fts5Config` | `Int.h:228` | 建表与 `%_config` 的全量配置（列、detail、content 模式、tokenizer、合并参数…） |
| `Fts5Storage` | `storage.c:45` | `%_content/%_docsize/%_config` 的语句缓存与 CRUD、totals（avgdl） |
| `Fts5Index` | `index.c:350` | 索引handle：pending hash、blob 读写语句、结构记录缓存 |
| `Fts5Structure` | `index.c:424` | 内存中的段目录：levels[] × segments[] + 合并中的前缀计数 |
| `Fts5SegWriter` / `Fts5PageWriter` | `index.c:440` | 写段时的页缓冲（term/doclist/pgidx 三个 buffer） |
| `Fts5Iter`（内嵌 `Fts5IndexIter`） | `index.c:656` / `Int.h:423` | 多段归并游标：`{iRowid, pData(poslist), nData, bEof}` |
| `Fts5Expr` / `Fts5ExprNode` | `expr.c:46/74` | 查询语法树；节点唯一虚函数 `xNext(…, iFrom)` |
| `Fts5ExprPhrase` / `Fts5ExprNearset` | `expr.c:122/137` | 短语（必连续）与 NEAR 组（短语间距离约束） |
| `Fts5TokenizerConfig` | `Int.h:176` | tokenizer 实例 + v1/v2 API + 参数 + pattern 能力 |

**[事实]** 三个版本号是三条独立契约线，别混淆：
`sqlite3_module.iVersion=4`（虚表接口，`main.c:3765`）；`fts5_api.iVersion=3`（tokenizer/aux 注册接口，`main.c:3802`）；`Fts5ExtensionApi.iVersion=4`（辅助函数拿到的东西，`main.c:2833`）。

## 3. 四条主数据流

### 3.1 建表（CREATE VIRTUAL TABLE）

```
xCreate (main.c:477) → fts5InitVtab
  ├─ sqlite3Fts5ConfigParse(config.c:565)   解析列与 10 个选项
  ├─ sqlite3Fts5IndexOpen(index.c:6889)     建 %_data / %_idx，写初始 structure+averages
  ├─ sqlite3Fts5StorageOpen(storage.c:343)  建 %_content / %_docsize / %_config（按模式取舍）
  ├─ sqlite3Fts5ConfigDeclareVtab(config.c:755)  声明列 = 用户列 + 表名隐藏列 + rank 隐藏列
  └─ sqlite3Fts5ConfigLoad(config.c:1051)   从 %_config 读回持久配置
```

### 3.2 写入（INSERT / UPDATE / DELETE）

```
xUpdate (main.c:1941)
  └─ fts5StorageInsert → sqlite3Fts5StorageContentInsert (storage.c:988)   写 %_content
                        → sqlite3Fts5StorageIndexInsert (storage.c:1069)
                             ├─ sqlite3Fts5IndexBeginWrite  (index.c:6812)  切换当前 rowid
                             ├─ 逐列 sqlite3Fts5Tokenize(…DOCUMENT…) (storage.c:1115)
                             │     └─ 回调 sqlite3Fts5IndexWrite (index.c:7004)
                             │           └─ sqlite3Fts5HashWrite (hash.c:245)  ← 只进内存 hash
                             └─ 写 %_docsize（每列 token 数）
提交（关键）：
xSync (main.c:2117) → sqlite3Fts5FlushToDisk (main.c:3156)
                        └─ sqlite3Fts5IndexSync (index.c:6839)
                             └─ fts5IndexFlush (index.c:5836)          ← 真正落盘在这里
                                  ├─ fts5FlushOneHash (index.c:5627)  hash → level-0 段
                                  ├─ fts5IndexAutomerge / Crisismerge (index.c:5043/5064)
                                  └─ fts5StructureWrite (index.c:1429) 更新结构记录
xCommit = 空操作 (main.c:2143)；xRollback 丢弃 hash (index.c:6852)
```

**[事实]** 设计要点：**事务内所有写只进内存 hash**，"可能失败的落盘工作全部放在 xSync"——因为 SQLite 忽略 xCommit 的失败（注释原文 `main.c:48-50`）。事务粒度 = hash 里那批 pending terms。

### 3.3 查询（SELECT … WHERE t MATCH '…'）

```
xBestIndex (main.c:614)    只做计划：MATCH/rowid/LIKE 约束编码进 idxStr，报成本
xFilter (main.c:1450)
  ├─ sqlite3Fts5ExprNew (expr.c:271)     查询串 → Fts5Expr 树（词法+Lemon 语法）
  ├─ sqlite3Fts5ExprFirst (expr.c:1552)  初始化整棵树
  │      └─ fts5ExprNearInitAll (expr.c:957)
  │            └─ 每个 term: sqlite3Fts5IndexQuery (index.c:7422)
  │                  └─ fts5MultiIterNew (index.c:3960)   ← 打开：每个段一个分段迭代器 + 内存 hash 伪段
  │                        多路归并（锦标赛树）产出按 rowid 有序的 (rowid, poslist)
  └─ xNext (main.c:991) → sqlite3Fts5ExprNext (expr.c:1592)
        短语匹配在内存里对齐 poslist；NEAR 用滑动窗口裁剪 poslist
输出：xColumn 取 %_content 原文；rank 列调用 bm25；highlight/snippet 通过 auxdata 拿实例位置
```

### 3.4 合并（merge / optimize）

```
触发点：每次 hash flush 之后（fts5FlushOneHash 末尾 index.c:5827-5829）
  ├─ automerge：按"写页数预算"（nWorkUnit=64 × nWork × nLevel）增量合并一层的段
  ├─ crisismerge：某层段数 ≥ 16（默认）时强制向下压
  └─ 用户命令： INSERT INTO t(t) VALUES('merge', N) → sqlite3Fts5IndexMerge (index.c:5961)
                 INSERT INTO t(t) VALUES('optimize')  → sqlite3Fts5IndexOptimize (index.c:5925)
合并执行：fts5IndexMergeLevel (index.c:4812)
  k 路归并输入段 → 顺序写出新段（term 前缀压缩、doclist/poslist 重编码）→ 删除输入段 → 更新结构
```

**[事实]** 层级语义：level 0 = 最新最小段，level 越大越老越大（`index.c:4812, 4829-4839`）；最多 64 层（`FTS5_MAX_LEVEL` `index.c:57`）。

## 4. 全局不变量（实现里被 enforce / assert 的规则）

- **[事实]** 段不可变：插入只追加新段，旧段只在 merge 时整体重写/删除（`index.c:4437-4474` 写、`index.c:4909-4915` 删）。
- **[事实]** 结构记录里最多允许**一层**处于增量 merge 中（`nMerge>0`），且不能是最顶层（`index.c:1211-1212` 校验、`:5006-5012` 运行时保证）。
- **[事实]** `nLevel ≤ nSegment ≤ FTS5_MAX_SEGMENT(2000)`；`pgnoFirst ≤ pgnoLast ≤ FTS5_MAX_PGNO`（`index.c:1155-1156, 1206-1209`）。
- **[事实]** 页头 `4 ≤ szLeaf ≤ nn`；term 数据不得越过 szLeaf 页脚起点（`index.c:937-941, 1962`）。
- **[事实]** 删除不修改旧数据：只写 `nPos==0` 的删除标记（detail≠none 用 poslist 尺寸最低位），真正消失发生在 merge 的 key annihilation（`index.c:4873-4874`）。
- **[事实]** 结构记录缓存靠 `PRAGMA data_version` 判失效（`index.c:1316-1334, 1386-1391`）。

## 5. 影子表与索引格式速览（细节见 notes/fts5-index.md）

```
t_data 记录 6 类（index.c:80-266）：
  1) structure record   rowid=10      段目录（levels/segments/merge 状态/写计数器）
  2) averages record    rowid=1       总行数 + 每列 token 总数（bm25 的 avgdl 来源）
  3) segment leaves     大正整数 rowid  term 页（页头4B + [term,doclist]* + pgidx 页脚）
  4) doclist indexes    同左（DLI 位区分） 大 doclist 的"每页首 rowid"b-tree
  5) tombstone pages    仅 contentless_delete=1  被删除 rowid 的哈希页
  6) 页脚/pgidx         属于 3 的组成部分
rowid 编码：segid(16bit) | dli(1bit) | height(5bit) | pgno(31bit)   （index.c:285-304）
```

## 6. 一张图：数据在读写两侧的形态

```
写：  SQL 行 ─tokenize→ 若干 term ─→ 内存 hash(term → doclist) ─flush→ 段(page序列) ─merge→ 更少的段
      %_content(原文)  %_docsize(token数)                       t_data                t_data

读：  MATCH 串 ─词法/语法→ 表达式树 ─→ 每个 term 打开多段归并游标 ─→ rowid 流
      短语/NEAR 在内存对齐 poslist ─→ xColumn 回 %_content 取原文 / rank 调 bm25 / 位置给 highlight
```

**[推断]** 整体是"写放大换读简单"的 LSM 结构：写永远顺序追加，读要做多路归并；用分层合并把"每字节被重写的次数"压到近似 O(log 段数)。

## 7. 子模块索引（详细笔记入口）

| 想了解 | 去看 |
|---|---|
| 磁盘格式、hash、合并、游标、前缀索引、tombstone | `notes/fts5-index.md` |
| MATCH 语法、短语/NEAR 算法、迭代器协议 | `notes/fts5-query.md` |
| unicode61/porter/trigram、自定义 tokenizer、折叠表 | `notes/fts5-tokenizer.md` |
| 虚表方法、xUpdate 语义、事务、special 命令、rank/docsize | `notes/fts5-sql.md` |
| 建表选项逐条、aux 函数、vocab、buffer、工具与测试、扩展注册 | `notes/fts5-config-aux.md` |
| 未解问题 | `questions.md` |
| 全部结论的原始出处清单 | `sources.md` |
