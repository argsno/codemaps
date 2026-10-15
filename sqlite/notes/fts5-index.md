# 模块笔记：FTS5 索引层（fts5_index.c 为核心）

规模：`fts5_index.c` 9590 行 + `fts5_hash.c` 590 + `fts5_varint.c` 344。这里是 FTS5 的心脏：磁盘格式、内存缓冲、合并、游标。
（文件路径省略前缀 `ext/fts5/`；行号基于 2026-10-15 工作树。）

## 1. `%_data` 表：一套 rowid 编码装下 6 类记录

**[事实]** 全部索引放在 `<name>_data(id INTEGER PRIMARY KEY, block BLOB)`（建表 `index.c:6904-6912`），内部有 6 类记录（清单与格式注释 `index.c:77-266`）：
structure record、averages record、segment 叶页、doclist index 页、tombstone 哈希页（仅 `contentless_delete=1`）、以及页脚（属于叶页的一部分）。

**[事实]** rowid 位域编码（`index.c:269-304`）：

```
FTS5_AVERAGES_ROWID  = 1            # 统计记录
FTS5_STRUCTURE_ROWID = 10           # 结构记录
# 段与辅助页：fts5_dri(segid, dlidx, height, pgno)
#   segid(16bit) | dlidx(1bit) | height(5bit) | pgno(31bit)
FTS5_SEGMENT_ROWID(segid,pgno)        = fts5_dri(segid, 0, 0, pgno)
FTS5_DLIDX_ROWID(segid,height,pgno)   = fts5_dri(segid, 1, height, pgno)
FTS5_TOMBSTONE_ROWID(segid,ipg)       = fts5_dri(segid+(1<<16), 0, 0, ipg)   # 高位偏移避开普通段
FTS5_MAX_PGNO = 0x7FFFFF00          # 超出视为 corrupt
```

**[事实]** 另一张影子表 `%_idx`（`segid, term, pgno PRIMARY KEY(segid,term)` WITHOUT ROWID，`index.c:6908-6911`）是**普通的 SQLite b-tree 表**，存"每页首 term（split key）→ 页号"，pgno 字段用 `flag + (page<<1)` 编码 dlidx 标志位（写 `index.c:4314`，读 `index.c:2691-2701, 2733-2737`）。段被删除时按 segid 整体删除（`index.c:1017-1040`）。

**[事实]** 页大小默认 `FTS5_DEFAULT_PAGE_SIZE = 4050`（`config.c:19`），运行期可通过 `pgsz=` 配置改（范围 32..65536，`config.c:928-938`）；读写缓冲额外留 `FTS5_DATA_PADDING=20` / 零填充 `FTS5_DATA_ZERO_PADDING=8` 防越界（`index.c:316-317`）。

## 2. Segment：不可变的"页链"

**[事实]** 每个 segment 用 16-bit `iSegid` 标识（1..2000，`FTS5_MAX_SEGMENT` `Int.h:107`；分配 `fts5AllocateSegid` `index.c:4143-4195`），页号 1..N 连续，本身**没有 interior/root 页**——段就是一条排好序的叶子链（`Fts5StructureSegment.pgnoFirst/pgnoLast` `index.c:407-418`）。"树"的角色由两张辅助索引承担：`%_idx`（term→页）与 dlidx（页→首 rowid）。

### 2.1 叶页物理格式（`index.c:171-204` 注释 + 读写实现）

```
+--------+--------------------------------+-------------------+
| 4B 页头 |  term/doclist 交替数据区        |  pgidx 页脚        |  (+ 尾部零填充)
+--------+--------------------------------+-------------------+
页头（两个大端 u16, fts5PutU16/GetU16 index.c:712-719）：
  [0..1] = 本页首个 rowid 的偏移（仅当该 rowid 出现在本页首 term 之前才非 0；宏 fts5LeafFirstRowidOff index.c:625）
  [2..3] = szLeaf = 页脚起始偏移（fts5LeafIsTermless index.c:621）
页脚 pgidx：每个 term 一个 varint（本 term 相对上一 term 的偏移 delta；第一个即绝对偏移）
           → 支持顺序推进与二分 seek（fts5LeafSeek index.c:2580-2637）
```

**[事实]** 数据区里 term 与 doclist 交替成 cell：
- 每页**第一个 term 全量存储**：`varint nTerm + term字节 + doclist`；
- 后续 term 前缀压缩：`varint nPrefix + varint nNew + nNew字节新数据 + doclist`（`index.c:128-170` 注释；写侧 `fts5WriteAppendTerm` `index.c:4483-4559`，前缀计算在 `:4541-4543`）。
- 跨页规则（`index.c:190-204`）：本页首个 rowid 若在首 term 之前，写**绝对值**；每页首 term 与"段首 term"同格式（不带 nPrefix）。
- **没有显式 free space**：页是打包到满——追加 term 前检查 `n + pgidx.n + nTerm + 2 >= pgsz` 就先 flush（`index.c:4498-4504`）；poslist 数据按 varint 边界切页（`fts5WriteAppendPoslistData` `index.c:4599-4627`，`fts5PoslistPrefix` `index.c:5107-5119`）。
- 页落盘：`fts5WriteFlushLeaf` 一次 `fts5DataWrite(FTS5_SEGMENT_ROWID(...))`（`index.c:4437-4474`）；空页只写 4 字节头。

### 2.2 doclist 与删除标记

**[事实]** doclist = `varint 首个 rowid` + `{ varint rowid delta, poslist }*`（`index.c:144-151`）。

**[事实]** 删除不重写旧数据：新段里写 `nPos==0` 的条目（detail=full/columns 时 poslist 尺寸 varint 的最低位是 bDel；detail=none 时用 0x00 字节约定），查询时跳过、**merge 时 annihilation**（`index.c:4873-4874`）。

### 2.3 poslist 编码（按 `detail=` 三档）

**[事实]** 位置值编码为 `(iCol<<32) + iOffset`（`FTS5_POS2COLUMN/POS2OFFSET` `Int.h:364-365`；注意"跨列的相邻位置"数值上不可能相等——短语匹配天然要求同列）。

| detail | poslist 存什么 | 编码细节 |
|---|---|---|
| `full`（默认） | 每列精确 offset | `collist(col0)` + `{ 0x01, varint 列号, collist }*`；collist 内部 offset 增量 `+2` 编码（写 `buffer.c:236-251`，读 `sqlite3Fts5PoslistNext64` `buffer.c:167-205`）；每个 poslist 前的 varint 尺寸 = `字节数*2 + bDel`（`index.c:1873-1881, 4895-4896, 5751-5765`） |
| `columns` | 只存列号序列 | 直接把列号当"位置"写（`hash.c:370-373`）；读时特化路径（≤100 列走 `fts5IterSetOutputs_Col100` `index.c:3822-3860`） |
| `none` | 无 poslist | 只有 rowid delta + 0/1/2 个 `0x00`：`0x00`=删除标记；再一个 `0x00`=nPos 1（"确实含该词"），无则 nPos 0（`index.c:1898-1911, 2232-2295, 5739-5750`） |

**[注意]** 源码里 `FTS5_DETAIL_FULL=0 / NONE=1 / COLUMNS=2`（`Int.h:285-287`，已核对）。

## 3. doclist index（dlidx）：大 doclist 的跳页表

- **[事实]** 只对"跨页很多"的 doclist 建（阈值 `FTS5_MIN_DLIDX_SIZE=4` 个无 term 页，`index.c:49, 4278-4289`；否则丢弃）。它记录 doclist 每页的首 rowid，用于 seek（`sqlite3Fts5IterNextFrom` `index.c:7563` → `fts5SegIterNextFrom` `index.c:3169-3217`）与"大 doclist 合并小 doclist"。
- **[事实]** 它自己是一棵 b-tree（`index.c:205-232`）：叶页 `[flags][pgno][first rowid][后续页 delta 或 0x00]`，内部节点同构；高度上限 32（`FTS5_DATA_HEIGHT_B=5`），写 `fts5WriteDlidxAppend` `index.c:4377-4435`，页满向上推成高层（`:4389-4416`）。
- **[事实]** `%_idx` 里对应页的 pgno 低位标志告诉读侧"该页有 dlidx"（`:4314` 写 / `:2734-2736` 读）。

## 4. structure record：整个索引的目录

- **[事实]** 位置：`%_data` rowid=10（`FTS5_STRUCTURE_ROWID`）。格式（`index.c:88-118` 注释；序列化 `fts5StructureWrite` `index.c:1429-1479`）：

```
4B cookie → [V2 magic "\xFF\x00\x00\x01" 仅 contentless_delete=1 表]
varint nLevel → varint nSegment → varint nWriteCounter
for each level: varint nMerge → varint nSeg
  for each segment: varint iSegid, varint pgnoFirst, varint pgnoLast
  (V2 追加: iOrigin1/iOrigin2/nPgTombstone/nEntryTombstone/nEntry)
```

- **[事实]** 它存的是**段目录 + 增量合并状态**（每层"正在合并的前 nMerge 个段"）+ 写计数器（automerge 预算依据）。反序列化带一致性校验（`fts5StructureDecode` `index.c:1125-1228`：nLevel/nSegment 上界、`nTotal<nMerge` 报 corrupt、`pgnoLast<pgnoFirst` 报 corrupt、非最顶层才允许 nMerge>0）。
- **[注意]** 它**不是**"每个 term 的 doclist 摘要"——per-term 跳转信息由 `%_idx`（页首 term→页号）和 dlidx 承担。这是个常见误解，别记反。
- **[事实]** 缓存与失效：`fts5StructureRead` + `PRAGMA <db>.data_version` 比较（`index.c:1316-1334, 1347-1384, 9584-9590`）。
- **[推断]** contentless_delete 的 V2 格式额外记录"每段覆盖的 origin 区间与 tombstone 统计"，是为了在删除时定位"哪些段可能包含该 rowid"（配合第 8 节的 tombstone 机制）。

## 5. pending terms：内存 hash

- **[事实]** 结构：`Fts5Hash`（1024 初始槽，装载因子 2 扩容，`hash.c:28-35, 103, 281-285`）+ `Fts5HashEntry`（`:64-77`）。key 在同一个分配里，格式为 `1 字节索引号（FTS5_MAIN_PREFIX='0' 或 '1'..'31'）+ term`；value 是内存态 doclist（rowid varint + poslist + 4 字节记录上段长度，`:37-63`）。
- **[事实]** 为什么不是 b-tree：写路径的访问模式是"随机 key 的追加 + 周期性整体输出"；hash 提供 O(1) 定位且只追加，结束时一次排序输出（`fts5HashEntrySort` 32 路归并 `hash.c:447-488`）就能**顺序**写段，避免 b-tree 的页平衡写放大（注释与实现 `hash.c:21-25, 447-488`；总体说明 `index.c:22-43`）。
- **[事实]** flush 触发条件（`sqlite3Fts5IndexBeginWrite` `index.c:6812-6834`）：rowid 回退、同一 rowid 的二次写、`nPendingData > nHashSize`（默认 1 MiB，`FTS5_DEFAULT_HASHSIZE` `config.c:23`）。
- **[事实]** hash 还被当成"一个额外的段"参与查询归并（`fts5SegIterHashInit` `index.c:2873-2934`）——所以**未提交的写入立刻可查**。
- **[事实]** poslist 尺寸字段在写入时预留 1 字节、扫描/查询时回填（`fts5HashAddPoslistSize` `hash.c:194-235`）。

## 6. 合并体系（LSM 式分层）

### 6.1 层级与 promote

- **[事实]** level 0 = 最新（hash flush 出来的最小段）；level 越大越老越大（flush 进 level 0：`fts5FlushOneHash` `index.c:5805-5822`；合并从 level i 输出到 i+1：`fts5IndexMergeLevel` `index.c:4812, 4829-4839`）。层数上限 `FTS5_MAX_LEVEL=64`（`index.c:57`）。
- **[事实]** structure record 里 `aLevel[0].aSeg[0]` 是最旧的段（`index.c:419-423`）。
- **[事实]** promote（`fts5StructurePromote` `index.c:1548-1592` + `fts5StructurePromoteTo` `:1505-1529`）：把"大小相近的段"提到同一层，避免大段被反复重写——这是分层合并逼近 LSM 的关键。

### 6.2 触发时机与预算

| 机制 | 触发 | 预算/阈值 | 出处 |
|---|---|---|---|
| automerge | 每次 hash flush 之后 | `nWorkUnit(64) × nWork × nLevel` 页的合并工作量；每层至少 `nMin = nAutomerge`（默认 4）个段才值得合并 | `index.c:47, 5043-5062, 5018-5020`；`FTS5_WORK_UNIT` |
| crisismerge | 同上（flush 后） | level 0 段数 ≥ `nCrisisMerge`（默认 16）强制逐层合并 | `index.c:5064-5080`；`FTS5_DEFAULT_CRISISMERGE=16` `config.c:22` |
| usermerge | `INSERT INTO t(t) VALUES('merge', N)` | `nMin = nUsermerge`（默认 4）；N<0 时先折叠结构再合并 | `index.c:5961-5984`；`config.c:21` |
| optimize | `INSERT INTO t(t) VALUES('optimize')` | 先把所有段折叠进一个新的最顶层，再从下往上反复合并（每步 `FTS5_OPT_WORK_UNIT=1000` 页），最终只剩 1 个段 | `index.c:5925-5955, 5856-5923, 46` |
| delete merge | 仅 `contentless_delete=1` | 按 tombstone 占比 ≥ `nDeleteMerge`（默认 10%）选择要合并的层 | `index.c:4948-4980`；`config.c:25` |

- **[事实]** 规则：同一时刻只允许一层处于增量 merge（`nMerge>0`，`index.c:1211-1212, 5006-5012`）。
- **[事实]** merge 主循环 `fts5IndexMergeLevel`（`index.c:4855-4898`）：k 路归并输入段（`fts5MultiIterNew(iLvl, nInput)`）→ 新 term 就 `fts5WriteAppendTerm` → 逐 rowid 写 doclist；跳过 `nPos==0` 且（本层最旧或非删除标记）的条目（**key annihilation** `:4873-4874`）；结束后 `fts5DataRemoveSegment` 删输入段并更新结构（`:4909-4928`）。
- **[事实]** `nWriteCounter` 跨过 work unit 数决定 automerge 的 `nWork`——"写得越多，merge 预算越大"（`index.c:5043-5062`）。
- **[推断]** CRUSH vs OPTIMIZE 一句话：CRUSH 是"有预算的日常整理"（可能留下未合并段），OPTIMIZE 是"一次性重写成单段"（最贵，但查询/rank 最快，并顺带清掉删除标记）。

### 6.3 常量表（默认值，均已核对）

| 常量 | 值 | 出处 |
|---|---|---|
| `FTS5_OPT_WORK_UNIT` | 1000 页 | `index.c:46` |
| `FTS5_WORK_UNIT` | 64 页 | `index.c:47` |
| `FTS5_MIN_DLIDX_SIZE` | 4 页 | `index.c:49` |
| `FTS5_MAIN_PREFIX` | `'0'` | `index.c:51` |
| `FTS5_MAX_LEVEL` | 64 | `index.c:57` |
| `FTS5_MAX_SEGMENT` | 2000 | `Int.h:107` |
| `FTS5_MAX_PREFIX_INDEXES` | 31 | `Int.h:102` |
| `FTS5_DEFAULT_PAGE_SIZE/AUTOMERGE/USERMERGE/CRISISMERGE/HASHSIZE` | 4050 / 4 / 4 / 16 / 1 MiB | `config.c:19-23` |
| `FTS5_DEFAULT_DELETE_AUTOMERGE` | 10（%） | `config.c:25` |

## 7. 前缀索引（`prefix=`）

- **[事实]** 上限 31 个、每个长度 1..999（`config.c:271-291`）。**实现不是额外建段**，而是同一个 segment 内用 key 首字节区分索引区：主索引 `FTS5_MAIN_PREFIX='0'`，第 i 个前缀索引首字节 `'0'+i`（`index.c:51, 7017-7031`；hash 排序输出天然按首字节分区，`:6765-6769`）。长度按**字符数**换算为字节数（`sqlite3Fts5IndexCharlenToBytelen` `index.c:6955-6976`）。
- **[事实]** 前缀查询三条路径（`sqlite3Fts5IndexQuery` `index.c:7452-7503`）：
  1. 有"字符数恰好等于前缀"的前缀索引 → 直接查该区（`:7480-7489`）；
  2. 有"字符数 = 前缀长度+1"的索引 → 从该区取"更长"的 doclist，再补上主索引里"恰好等于前缀"的 term 的 doclist，内存归并（`fts5SetupPrefixIter` `:6713-6805`）；
  3. 都没有 → 扫主索引所有以该前缀开头的 term，16 路级联归并（`FTS5_MERGE_NLIST=16` `:6196`；注释"足以合并 16^8 个列表" `:6740-6748`），最后包成普通迭代器。
- **[事实]** 调试开关 `FTS5INDEX_QUERY_TEST_NOIDX`（`Int.h:439`）强制走第 3 条路做交叉校验（`index.c:7462-7467`）。

## 8. tombstone（仅 `contentless_delete=1`）

- **[事实]** contentless 表要支持 DELETE，靠"墓碑"：删除时把 rowid 写进**覆盖它的那些段**的 tombstone 哈希页（`sqlite3Fts5IndexContentlessDelete` `index.c:8166`，段区间判定 `:8176-8182`，`%_docsize` 里的 `origin` 列用于定位）。
- **[事实]** 页格式（`index.c:234-266`）：8 字节页头（key 宽度 4/8、rowid0 标志、保留、大端条目数）；槽位定位 `iPg = rowid % nHashPg`、`iSlot = (rowid/nHashPg) % nSlot`；实现 `fts5IndexTombstoneAddToPage/Rehash/Rebuild/Add`（`:7873, 7927, 8006, 8108`）。
- **[事实]** 查询路径在归并时跳过命中墓碑的 rowid（`index.c:3426-3431`）；结构 V2 记录的统计用于触发 delete merge 真正清掉数据。
- **[未知]** tombstone 查询侧函数的确切命名与行号范围（子代理给出 `fts5IndexTombstoneQuery` `:3324` 与 `fts5MultiIterIsDeleted` `:3360-3387` 两处提法，未逐一复核）。

## 9. 读路径：多段归并游标

- **[事实]** 公开对象极小：`struct Fts5IndexIter { i64 iRowid; const u8 *pData; int nData; u8 bEof; }`（`Int.h:423-430`）；实现实体 `Fts5Iter` 内嵌 `base`（`index.c:656-674`）。
- **[事实]** 一次查询打开**所有相关段的段迭代器 + 一个 hash 伪段**（`fts5MultiIterNew` `index.c:3960-4043`），用"锦标赛树"（`aFirst[]` + `fts5MultiIterDoCompare` `:3075` + `fts5MultiIterAdvanced` `:3233-3249`）维护最小 rowid。
- **[事实]** 推进：`sqlite3Fts5IterNext` → `fts5MultiIterNext`（`index.c:3396-3434`），按段选 `xNext`（普通 `:2305` / none `:2232` / reverse `:2198`），并跳过删除与墓碑条目。
- **[事实]** 输出回调按 detail/列过滤器选择（`fts5IterSetOutputCb` `:3890-3920`）：`_None` / `_Nocolset` / `_ZeroColset` / `_Col100` / `_Col` / `_Full`——列过滤在**索引层**就把 poslist 裁剪掉。
- **[事实]** Seek：`sqlite3Fts5IterNextFrom(iter, iMatch)` 借助 dlidx 直接跳到目标页（`index.c:7563`）。
- **[事实]** 反向扫描（`ORDER BY rowid DESC` 等）：`FTS5_SEGITER_REVERSE`（`index.c:613-614`），`fts5SegIterReverseInitPage` 先把页内每个 rowid 的 offset 记进 `aRowidOffset[]` 再倒着走（`:2078`）。

## 10. 写路径 & 事务衔接（与 storage 的边界）

- **[事实]** 入口链：`sqlite3Fts5StorageIndexInsert`（`storage.c:1069`）→ `sqlite3Fts5IndexBeginWrite(idx, 0, rowid)`（`storage.c:1084`；`index.c:6812`）→ 逐列 tokenize 回调 `sqlite3Fts5IndexWrite`（`storage.c:1115-1118`；`index.c:7004`）→ `sqlite3Fts5HashWrite`（`hash.c:245`，主索引+各前缀索引）。
- **[事实]** 提交：`xSync → sqlite3Fts5FlushToDisk (main.c:3156) → sqlite3Fts5StorageSync (storage.c:1501: 先回写 averages) → sqlite3Fts5IndexSync (index.c:6839) → fts5IndexFlush (index.c:5836) → fts5FlushOneHash + automerge + crisismerge + structureWrite`。
- **[事实]** 错误保持：flush 出错记入 `p->flushRc` 复用（`index.c:5838-5852`）；`sqlite3Fts5IndexRollback` 关 blob 句柄、丢 hash、失效结构缓存（`index.c:6852-6857`）。
- **[事实]** blob I/O：读用 `sqlite3_blob_open/reopen`（`index.c:847-874`），写/删用预编译语句 `pWriter/pDeleter` 等（`index.c:372-380`）；`sqlite3_blob_write` 还被用来**就地改结构记录头 4 字节 cookie**（`sqlite3Fts5IndexSetCookie` `:7818-7836`）。

## 11. 设计取舍与不变式（汇总）

1. **[事实]** 段不可变 + 只追加：插入永不改写旧 doclist，merge 才重写（`:4437-4474` vs `:4786-4938`）→ 低写放大、高读成本，用分层 merge 调和。
2. **[事实]** 删除不重写旧数据（只写标记），merge 时 annihilation（`:4873-4874`）。`secure-delete` 是唯一例外：它直接在段上做字节级编辑（`fts5DoSecureDelete` `:5262-5561`），并把文件格式版本升到 5（`Int.h:277-278`）。
3. **[事实]** 内存 hash 而非 b-tree（见 §5）。
4. **[事实]** `detail` 三档 = 空间 ↔ 短语/位置精度 的取舍；rank 精度与 highlight 依赖它。
5. **[事实]** 明确不变式（§architecture.md 第 4 节列出，均可在 `index.c` 1125-1228 / 1155-1156 / 1206-1212 / 937-941 / 1962 等处核对）。
6. **[推断]** `%_data` 单表 + 自定义页格式的设计动机：把"索引的自定义 I/O 粒度"（定长页、双向扫、seek 不整载）与"SQLite 的事务/回滚"解耦——blob 行天然受 SQLite 事务保护，FTS5 不需自己实现 WAL。
