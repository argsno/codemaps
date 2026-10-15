# 模块笔记：FTS5 分词器体系（fts5_tokenize.c + fts5_unicode2.c）

规模：`fts5_tokenize.c` 1496 行 + `fts5_unicode2.c` 780 行（机器生成表）。
一句话：分词器是**每连接注册、按名字查找、建表时绑定**的插件；写路径与查询路径共用同一个实例与同一套调用入口。

## 1. 总览与调用链

- **[事实]** 注册表在 `Fts5Global.pTok` 链表（`main.c:83`）；实例挂在 `Fts5Config.t`（`Int.h:176-185`）。
- **[事实]** 唯一对外入口 `sqlite3Fts5Tokenize()`（`config.c:801-826`）：按实例是 v1 还是 v2 走 `pApi1->xTokenize` 或 `pApi2->xTokenize`，惰性实例化（`:810-812` 调 `sqlite3Fts5LoadTokenizer`）。
- **[事实]** 内置注册（`sqlite3Fts5TokenizerInit` `tokenize.c:1460-1495`，已核对原文）：
  - **v1 结构** 注册 `unicode61` / `ascii` / `trigram` 三个；
  - **v2 结构** 注册 `porter`（userdata 传 `fts5_api*`，使其能反查其它分词器）。
- **[事实]** 默认分词器 = `unicode61`：没写 `tokenize=` 时 `zName==0` 返回 `pGlobal->pDfltTok`（`main.c:3459-3460`）；`pDfltTok` 设为"第一个注册的模块"，而 unicode61 恰是注册表第一项（`main.c:3286-3290`；`test/fts5unicode.test:42` 专门断言）。
- **[事实]** 实例化：`sqlite3Fts5LoadTokenizer`（`main.c:3533-3576`），第 0 个参数是分词器名，传给 `xCreate` 的是 `azArg+1`；名字不存在报 `no such tokenizer: %s`（`:3543`）。

## 2. 内置分词器行为一览

| tokenizer | 注册类型 | 扫描粒度 | 折叠 | 选项 |
|---|---|---|---|---|
| `unicode61` | v1 | UTF-8 字符 | Unicode 简单大小写折叠 + 可选去变音符 | `categories` / `remove_diacritics` / `tokenchars` / `separators` |
| `ascii` | v1 | 字节 | 仅 `A-Z→a-z`（`tokenize.c:105-112, 162`） | 只支持 `tokenchars`/`separators`（`:82-92`）；≥0x80 字节一律算 token 字符 |
| `porter` | **v2** | 包装另一个分词器 | 由基分词器决定 | 施加 Porter 词干规则；基分词器默认 unicode61 |
| `trigram` | v1 | UTF-8 字符（滑窗 3） | `case_sensitive` 可关；`remove_diacritics` 只支持非 0 | 见 §5 |

## 3. unicode61 的实现

### 3.1 结构（`tokenize.c:238-248`）

`aTokenChar[128]`（ASCII 快查表）、`aFold/nFold`（复用的折叠缓冲）、`eRemoveDiacritic`、`nException/aiException`（非 ASCII 的 tokenchars/separators 例外表，升序 int 数组、二分查找）、`aCategory[32]`（类别位图）。

- **[事实]** 构造顺序有依赖：**先**解析 `categories`（`:383-391`），**再**处理 `remove_diacritics`/`tokenchars`/`separators`（`:393-417`）——因为 `categories` 解析末尾必须调 `sqlite3Fts5UnicodeAscii` 重铺 ASCII 表（`:350`），否则会覆盖后面对 ASCII 的例外改写。
- **[事实]** 默认类别串 `"L* N* Co"`、默认去变音 `FTS5_REMOVE_DIACRITICS_SIMPLE`（`:372-376`）；参数必须成对（`:367-368`）。

### 3.2 字符分类表（生成物）

- **[事实]** `fts5_unicode2.c` 是机器生成文件（头注 `:15`），生成器 `ext/fts3/unicode/mkunicode.tcl -fts5`（`mkunicode.tcl:917-920`）。
- **[事实]** 三张表：`aFts5UnicodeBlock[]`（按高 16 位分块）、`aFts5UnicodeMap[]`（块内起始低 16 位）、`aFts5UnicodeData[]`（低 5 位=类别号，高 3 位=区间长度）。查表 `sqlite3Fts5UnicodeCategory`（`unicode2.c:737-766`，二分；>=(1<<20) 或查不到返回 0）；类别号 30 特殊表示 `LC`（大小写成对，按奇偶返回 Ll/Lu）。
- **[事实]** `sqlite3Fts5UnicodeCatParse`（`:253-735`）把 `"L*"`/`"Co"` 之类写法展开成 `aCategory[32]` 位图；注意 `aArray[0] = 1`（`:254`）——**类别号 0 恒为 token 字符**。
- **[事实]** `sqlite3Fts5UnicodeAscii`（`:768-779`）把类别结果铺进 128 字节 ASCII 快表，并强制 `aAscii[0]=0`（NUL 永不是 token 字符）。
- **[事实]** 判断 token 字符：`aCategory[分类号] ^ 例外表命中`（异或实现"例外翻转分类"，`tokenize.c:435-440`）。

### 3.3 `remove_diacritics` 三档

- **[事实]** 常量 `FTS5_REMOVE_DIACRITICS_NONE=0 / SIMPLE=1 / COMPLEX=2`（`tokenize.c:251-253`），参数只接受单字符 `0/1/2`（`:395-404`）。
- **[事实]** 实际工作在 `sqlite3Fts5UnicodeFold(c, eRemoveDiacritic)` 里：折叠后调 `fts5_remove_diacritic(ret, eRemoveDiacritic==2)`（`unicode2.c:240-242`）。
- **[事实]** `fts5_remove_diacritic`（`:31-91`）：`aDia[]` 是 `(码点<<3)|区间低位` 的二分键表，`aChar[]` 是替换字符；**带 HIBIT(0x80) 的条目是"复杂映射"**；`remove_diacritics=1` 遇到 HIBIT 条目返回原码点（`:89`）——即 SIMPLE 会跳过需要间接分解的字符。
- **[事实]** 组合式重音（U+0300..U+0331）会被吸收进 token 但不输出（`tokenize.c:515`；`unicode2.c:98-105` 的 `sqlite3Fts5UnicodeIsdiacritic` 两张位掩码表）。

### 3.4 分词主循环（吞吐关键，`fts5UnicodeTokenize` `tokenize.c:442-544`）

- **[事实]** ASCII 走 128 字节表（`:482-488`）；非 ASCII 才 `READ_UTF8` 解码 + 分类判断（`:474-481`）——字节级快路径是主要性能取舍。
- **[事实]** 折叠缓冲不足按 2 倍扩（`:497-509`）；ASCII 大写就地 `+32`（`:526-532`，避免函数调用）；非 ASCII 走 `Fold` + `WRITE_UTF8`（`:515-521`）。
- **[事实]** 回调 `xToken(pCtx, 0, aFold, len, is, ie)` 中 `is/ie` 是**输入串的字节偏移**（`:538`）。
- **[事实]** 非 amalgamation 编译时自带 `READ_UTF8/WRITE_UTF8` 副本（`:184-236`），非法序列替换为 U+FFFD。

## 4. porter stemmer（"包装型"分词器）

- **[事实]** 结构 `PorterTokenizer { fts5_tokenizer_v2 tokenizer_v2; Fts5Tokenizer *pTokenizer; char aBuf[FTS5_PORTER_MAX_TOKEN+64] }`（`:554-559`），`FTS5_PORTER_MAX_TOKEN=64`（`:552`）。
- **[事实]** 构造 `fts5PorterCreate`（`:577-619`）：跳过前导 `"porter"` 名（允许 `tokenize="porter porter ..."`）；基分词器默认 `unicode61`（`:586`）；用注册时塞入的 `fts5_api*` 调 `xFindTokenizer_v2` 找基分词器（`:602`，v1 注册的也会被自动补出 v2 壳，所以能包任意分词器）；剩余参数原样传给基分词器 `xCreate`（`:606-611`）。
- **[事实]** 分词 `fts5PorterTokenize`（`:1256-1272`）只是"把回调换成 `fts5PorterCb`，其余透传"给基分词器；**flags/locale 全部透传**。
- **[事实]** `fts5PorterCb`（`:1187-1251`）：长度 `<3` 或 `>64` 直接透传；否则在 `aBuf` 上跑 step 1A→1B→1B2→1C→2→3→4→5a→5b（`:1205-1245`），最后以**原样 `iStart/iEnd`** 回调（`:1247`）——去词干不改变位置信息。
- **[事实]** 规则条件函数 `fts5PorterIsVowel/GobbleVC/_MGt0/_MGt1/_MEq1/_Ostar/_MGt1_and_S_or_T/_Vowel`（`:663-743`）即 Porter 的 \*v\*、\*S/\*T、m>0/m>1/m=1、\*o 条件。
- **[事实]** Step1B/1B2/2/3/4 的代码由 `mkportersteps.tcl` 生成并内联（C 文件里 `GENERATED CODE STARTS/ENDS HERE` 标记 `:746-749, 1164-1167`）；生成器按"后缀倒数第 2 个字符"分组产出 `switch(aBuf[nBuf-2])` 预过滤（`mkportersteps.tcl:91-95, 169-173, 103, 111`），6 种模板覆盖各组合（`:116-165`）。设计动机（脚本头 `:13-21`）：把规则表与 C 代码分离，避免手写上百分支，同时保持"FTS5 只有这几个 .c 文件、便于 amalgamation"的工程约束。
- **[事实]** 内置 porter **对文档与查询做完全相同的 stemming**（四个内置分词器都不看 `FTS5_TOKENIZE_*` flags：`tokenize.c:120, 445, 1352, 1269-1270`）——不存在"只对 query 做 stem"的行为。

## 5. trigram

- **[事实]** 构造 `fts5TriCreate`（`:1293-1344`）：`case_sensitive`（0 → `bFold=1`）；`remove_diacritics` 非 0 → `iFoldParam=2`；`remove_diacritics!=0 且 case_sensitive=1` 直接报错（`:1332-1334`）。
- **[事实]** `fts5TriTokenize`（`:1349-1421`）：32 字节栈缓冲 + 3 元素 `aStart[3]` 维护滑窗；**先读满 3 个字符才进入主循环**（`:1369-1377`）——所以**长度 <3 字符的输入/查询不产生任何 token**（`test/fts5trigram.test:357-363`）。折叠为 0 的字符（被移除的变音符）跳过但不重置窗口（`:1370-1375, 1392-1400`），保证文档侧与查询侧位置一致。
- **[事实]** 语义推演：写侧每 3 字一个 token、位置逐个 +1（`storage.c:459-461`）→ 列内三元组位置连续；查询侧同一串也切成连续三元组 → 短语匹配要求三元组位置相邻，**等价于子串匹配**。例：`SELECT fts5_expr('ABCD','tokenize=trigram')` → `{"abc" + "bcd"}`（`test/fts5trigram.test:41-43`）。
- **[事实]** 前缀查询退化为等值：索引 term 全恰 3 字符，`abc*` ≡ `abc`；`ab*` 永远不产生 token（`:1369-1377`）。`prefix=` 机制对它无意义。
- **[事实]** LIKE/GLOB 优化**支持**但有条件：`sqlite3Fts5TokenizerPattern`（`:1434-1445`）判定——`case_sensitive=0 且 remove_diacritics=0` → LIKE；`case_sensitive=1 且 remove_diacritics=0` → GLOB；其余 NONE。常量 `FTS5_PATTERN_LIKE/GLOB` 与 `SQLITE_INDEX_CONSTRAINT_LIKE/GLOB` **故意数值相等**（`Int.h:289-291` + 断言 `main.c:529-530`）。判定结果存 `pConfig->t.ePattern`（`main.c:3562-3566`），`xBestIndex` 据此把 LIKE/GLOB 变成约束 `'L'/'G'`（`main.c:525-540, 677-685`）。
- **[事实]** 因为要在第一次 xBestIndex 前知道 ePattern，trigram 需要**提前实例化**：`sqlite3Fts5TokenizerPreload`（`:1452-1454`）+ xCreate 里提前加载（`main.c:418-420`）。
- **[事实]** 执行时把模式编译成 MATCH 表达式（`sqlite3Fts5ExprPattern` `expr.c:364-430`）：只有连续字面量 ≥3 字符的片段变成带引号短语（`:347-354, 394`）；`%_`/`*?[` 是切分点；生成的是"超集"，正确性由**未 omit 的 LIKE/GLOB 回查**保证（`test/fts5trigram.test:49-64`）；`detail!=full` 时短语改写为 AND 树（`expr.c:414-420, 2309-2345`）。`case_sensitive=1` 的表只接 GLOB 不接 LIKE（测试 EQP 断言 `fts5trigram.test:190-206`）。

## 6. 自定义 tokenizer 的 C API（v1 / v2）

- **[事实]** `fts5_tokenizer_v2`（`fts5.h:633-652`）：首字段 `iVersion`（当前 2），`xTokenize` 多两个参数 `pLocale/nLocale`（locale 支持）。legacy `fts5_tokenizer`（`fts5.h:659-676`）无版本、无 locale。
- **[事实]** 注册时校验 `iVersion>2` 报错（`main.c:3406-3407`）。**双向自动适配**（本版 main.c 的重点代码）：
  - 注册 v1 → 自动补 v2 版（`pNew->x2 = {fts5VtoVCreate, fts5V2toV1Tokenize, fts5VtoVDelete}`，`main.c:3441-3443`；v2→v1 的 tokenize 丢弃 locale `:3379-3390`）；
  - 注册 v2 → 自动补 v1 版（`main.c:3412-3416`；v1→v2 以 `0,0` 当 locale `:3364-3373`）；
  - 两者都套 `Fts5VtoVTokenizer` 包装实例（结构 `:3303-3308`，构造 `:3314-3342`）。
  - 查找：`xFindTokenizer` / `xFindTokenizer_v2`（`main.c:3504-3528 / 3474-3498`）。
- **[事实]** `xTokenize` 回调协议（`fts5.h:643-650`，文档 `:477-498`）：`pToken` 不保证 NUL 结尾；`iStart/iEnd` 是输入串**字节偏移**；**必须按出现顺序回调**；`tflags` 正常 0，同义词用 `FTS5_TOKEN_COLOCATED`；回调返回非 OK 必须立即放弃并原样返回；`SQLITE_DONE` 被当作成功（内置实现 `tokenize.c:170-171, 542-543`）。
- **[事实]** `FTS5_TOKEN_COLOCATED = 0x0001`（`fts5.h:687`）="与上一个 token 同位置"（同义词），首次调用就带是错误；可连续多个。消费点：写路径 `storage.c:459-461`（同义词不增位置）、查询侧 `expr.c:1751-1767`（挂 `pSynonym` 链）、`xColumnSize` 计数 `main.c:2505-2507`、一致性检查 `storage.c:1197`。
- **[事实]** 拿 `fts5_api*` 的方式：`SELECT fts5(?1)` + `sqlite3_bind_pointer(..., "fts5_api_ptr")`（示例 `fts5_test_mi.c:76-80`、`fts5_tcl.c:99-110`）；然后 `xCreateTokenizer(_v2)` 注册。树内示例：`fts5_tcl.c:1255-1271`。
- **[事实]** 实例销毁契约：`xDelete` 对每次成功的 `xCreate` 恰好调一次（`fts5.h:435-438`）；模块在连接关闭时由 `fts5ModuleDestroy` 调 `xDestroy`（`main.c:3584-3602`）。
- **[事实]** `fts5_api.iVersion=3`（`main.c:3802`）；v2 相关函数**追加在结构体尾部**（`fts5.h:726-743`），旧字段顺序不动——ABI 兼容策略。

## 7. flags 的实际影响（`FTS5_TOKENIZE_*`）

- **[事实]** 定义：`QUERY 0x0001 / PREFIX 0x0002 / DOCUMENT 0x0004 / AUX 0x0008`（`fts5.h:679-683`），语义文档 `:446-468`。
- **[事实]** 设置点：写路径 `DOCUMENT`（`storage.c:1115-1116, 578, 917, 1316`）；查询 `QUERY|PREFIX`（`expr.c:1864`）；位置回填也走 `DOCUMENT`（`expr.c:3110`）；扩展函数固定 `AUX`（`main.c:2205-2207, 2538`）。
- **[事实]** **四个内置分词器都不看这些 flag**（只透传/忽略）；flag 是给自定义分词器留的语义提示。FTS5 自己用 `PREFIX` 的方式是：告诉分词器"末尾有星号"，然后把最后一个 token 标成前缀 term（`expr.c:1864, 1889-1891`）。

## 8. 写路径 vs 查询路径的对称性

- **[事实]** 同一实例、同一参数同时服务读写（只建一次，缓存在 `pConfig->t`；读写都过 `sqlite3Fts5Tokenize`）。差异只在 flags、locale、查询串先经 FTS5 语法解析与去引号、以及 `tokendata=1` 表查询侧对词面按 `strlen` 截断（`fts5QueryTerm` `expr.c:3032-3036`）。
- **[事实]** 写路径逐列：`sqlite3Fts5StorageIndexInsert` → `sqlite3Fts5Tokenize(…DOCUMENT…)`（`storage.c:1115-1118`）→ 回调 `fts5StorageInsertCallback`（`:447-463`：截断到 `FTS5_MAX_TOKEN_SIZE=32768`，非 COLOCATED 时列内位置 +1）→ `sqlite3Fts5IndexWrite`。
- **[事实]** `detail≠full` 且需要位置时，读路径会**再次对文档跑 DOCUMENT 分词**（`sqlite3Fts5ExprPopulatePoslists` `expr.c:3084-3112`）——这是"查询期做文档分词"的重要例外。
- **[事实]** 分词算法是**数据库格式的一部分**：`tokenize=` 串存在 `sqlite_master`，之后所有连接必须能实例化出同样的分词器，否则旧库不可读。这是自定义分词器的运维约束。

## 9. 设计取舍

1. **[事实]** 字节级扫描 + ASCII 快表 + 复用缓冲（不每次 malloc）：`unicode61` 的 `aFold`、`porter` 的 `aBuf`、`trigram` 的栈上缓冲。
2. **[事实]** token 是 (指针, 长度) 字节串，不要求 NUL、不要求与原文构成子串关系（porter 就改写内容），但**必须给出输入字节区间**——snippet/highlight/insttoken 的基础。
3. **[事实]** 固定上限换零分配：`FTS5_MAX_TOKEN_SIZE=32768`（`Int.h:95`，各路径截断）；porter 超过 64 字节不去词干（`tokenize.c:552, 1200`）。
4. **[事实]** 前缀索引按**字符数**而非字节数（`sqlite3Fts5IndexCharlenToBytelen` `index.c:6980-6992`），多字节语言下 `prefix=2` 真是"前 2 个字符"。
5. **[事实]** locale 只对 v2 自定义分词器生效：内置三个以 v1 注册，locale 会被包装层丢弃（`main.c:3388`）；`fts5_locale()` 机制（`fts5.h:345-348`、`main.c:1312-1322`）是给 ICU 类分词器留的扩展位。
