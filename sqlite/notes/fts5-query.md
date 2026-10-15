# 模块笔记：FTS5 查询层（fts5_expr.c + fts5parse.y）

规模：`fts5_expr.c` 3286 行 + `fts5parse.y` 193 行（Lemon 文法，构建期生成 `fts5parse.c/h`）。
职责：把 `MATCH '…'` 字符串变成一棵表达式树，并以**拉取式迭代器**按 rowid 有序产出匹配行 + 该行各短语的位置列表。
（行号基于 2026-10-15 工作树；两处与常见命名的出入已在文末标注。）

## 1. MATCH 查询语法

### 1.1 词法（`fts5ExprGetToken` `expr.c:204-266`）

- **[事实]** 空白：空格/Tab/换行/回车（`:197-199, 213`）。
- **[事实]** 单字符 token（`:217-228`）：`(` `)` `{` `}` `:` `,` `+` `*` `-` `^` 与 EOF。
- **[事实]** 双引号串 `"..."`，内部 `""` 转义；未闭合报 `unterminated string`（`:230-246`）。
- **[事实]** 裸词（bareword）：由 `sqlite3Fts5IsBareword` 定义的字符集连续组成（`buffer.c:313-330`）；**只有 `OR`/`NOT`/`AND` 三个整词是保留字**（`:257-259`），其余一律是 `FTS5_STRING`。
- **[事实]** 关键点：裸词与引号串在词法上都是 STRING；`the quick brown` 只有**引号形式**才能构成多 token 短语（裸词不含空格）。

### 1.2 语法与优先级（`fts5parse.y:76-197`）

**[事实]** 优先级（低→高，Lemon 声明顺序）：`OR < AND < NOT < TERM < COLON`（`parse.y:76-80`）。测试里的规范化结果佐证：`a AND b OR c` → `("a" AND "b") OR "c"`，`a OR b AND c` → `"a" OR ("b" AND "c")`，`a AND b NOT c` → `"a" AND ("b" NOT "c")`（`test/fts5eb.test:46-52`）。

**[事实]** 产生式一览：

| 语法 | 位置 | 语义 |
|---|---|---|
| `expr AND/OR/NOT expr` | `parse.y:115-123` | 建 AND/OR/NOT 节点 |
| `colset : ( expr )`、`( expr )` | `:125-129` | 列过滤 / 括号 |
| `exprlist ::= cnearset \| exprlist cnearset` | `:132-135` | **隐式 AND**（仅对相邻 nearset 生效） |
| `cnearset ::= nearset \| colset : nearset` | `:137-143` | 列过滤短形式 |
| `nearest ::= phrase \| ^ phrase \| STRING ( nearphrases [, int] )` | `:151-160` | 短语 / 列锚定 `^` / `NEAR(...)` |
| `phrase ::= STRING star_opt \| phrase + STRING star_opt` | `:185-190` | `+` 把多个"段"拼成一个短语（段内继续分词） |
| `star_opt ::= * \| ε` | `:195-197` | 前缀标记，只作用于其前面那个 STRING |
| `colset ::= { cols } \| -{ cols } \| STRING \| -STRING` | `:97-113` | 列集合（含求反） |

**[事实]** 语义细节：
- 列名大小写不敏感；不存在报 `no such column`（`:2126-2127`）；列集合内部排序去重（`:2063-2075`），求反=全集差（`:2087-2108`）。
- `^` 只允许出现在 nearset 前（语法 `:152`），实现为给短语第 0 个 term 打 `bFirst`（`:1648-1652`），匹配时要求 `FTS5_POS2OFFSET==0`（`:670`）——"必须是该列第一个 token"。
- 前缀 `*` 打在短语**最后一个** token 上（`:1889-1891`），初始化迭代器时传 `FTS5INDEX_QUERY_PREFIX`（`:985`）。
- `""`（无 token）产生 `eType==0` 的**恒不匹配**节点（`:65-67, 2411-2414`）。
- 表达式树深度上限 256（`SQLITE_FTS5_MAX_EXPR_DEPTH` `:20-22`；检查 `:2438-2445`；DEBUG 断言 `:166-184`）。
- `(a OR b) abc` 是语法错误——括号表达式后不能接隐式 AND（测试 `fts5ea.test:67`）。
- `MATCH '*...'` 是 special query（`main.c:1515-1520` → `fts5SpecialMatch` `main.c:1155-1183`），不进入解析器。
- NEAR 关键字必须拼作 `NEAR`（`:1998-2008` 校验）；距离缺省 `FTS5_DEFAULT_NEARDIST=10`（`Int.h:109`；解析 `:2010-2035`，带防溢出）。
- trigram 表上的 LIKE/GLOB 会被改写成 MATCH 表达式（`sqlite3Fts5ExprPattern` `:364-430`，详见 tokenizer 笔记）。

## 2. 核心结构体

```
Fts5Expr          expr.c:46-53    pIndex / pConfig / pRoot / bDesc(倒序) / nPhrase / apExprPhrase
Fts5ExprNode      :74-90          eType(AND/OR/NOT/STRING/TERM；0=恒不匹配) / bEof / bNomatch
                                  / iHeight / xNext(...)唯一虚函数 / iRowid / pNear / nChild apChild[]
Fts5ExprTerm      :108-116        bPrefix / bFirst(^) / pTerm,nQueryTerm,nFullTerm / pIter / pSynonym链
Fts5ExprPhrase    :122-127        pNode(反向) / poslist(当前行的合并位置列表) / nTerm / aTerm[]
Fts5ExprNearset   :137-142        nNear(距离) / pColset(列过滤) / nPhrase / apPhrase[]
Fts5Parse         :151-159        解析期上下文(全局短语登记 apPhrase)
```

**[事实]** 旁路关系：`Fts5Expr → pRoot(树) → pNear → apPhrase → aTerm → pIter(Fts5IndexIter)`；每个短语同时登记在 `Expr.apExprPhrase[]`（供 aux 函数按短语号取位置）。
**[事实]** `FTS5_TERM` 是"单短语·单 term·无同义词·无 ^"的**快路径形态**，构造期由 `fts5ExprAssignXNext`（`:2247-2278`）决定。

## 3. 求值：拉取式迭代器

### 3.1 对外协议（pull 三步）

| 调用 | 位置 | 作用 |
|---|---|---|
| `sqlite3Fts5ExprFirst(expr, pIdx, iFirst, iLast, bDesc)` | `:1552-1584` | 注入索引句柄与方向；`fts5ExprNodeFirst` 初始化整棵树；若首个结果 < iFirst 则前跳；`bNomatch` 循环跳过伪匹配；超 iLast 直接 EOF |
| `sqlite3Fts5ExprNext(expr, iLast)` | `:1592-1604` | 推进到下一个真匹配或 EOF |
| `sqlite3Fts5ExprEof/Rowid` | `:1606/:1610` | 读根节点状态 |
| 位置数据 | `sqlite3Fts5ExprPoslist` `:2958` 等 | 供 main.c 取当前行位置列表 |

**[事实]** 节点唯一虚函数是 `xNext(pExpr, pNode, bFromValid, iFrom)`（`:81`）——**允许"跳到 ≥ iFrom"**，这是与 SQLite 分层模型（rowid 约束、提前终止）配合的关键。

### 3.2 叶子：两种路径

- **[事实]** `FTS5_TERM` 快路径（`fts5ExprNodeNext_TERM` `:1257-1279` / `Test_TERM` `:1229-1252`）：**直接借用** `Fts5IndexIter` 的 poslist 指针（`:1245-1248`），`bNomatch = (poslist.n==0)`（detail≠full 时只看是否存在）。
- **[事实]** `FTS5_STRING` 慢路径（`Next_STRING` `:1164-1226` / `Test_STRING` `:1094-1155`）：
  1. 推进"第一个短语的第一个 term"的迭代器（有同义词时按 `fts5ExprSynonymRowid` 归并 `:1174-1206`）；
  2. `Test_STRING` 把所有 phrase/term 的迭代器**对齐到同一 rowid**（落后的用 `sqlite3Fts5IterNextFrom` 跳，`:1125-1148`；`bDesc` 决定"更大/更小"方向，比较器 `fts5RowidCmp` `:1019-1032`）；
  3. 调 `fts5ExprNearTest`（`:898-945`）验证位置并生成 poslist，写 `bNomatch`。

### 3.3 短语匹配算法（`fts5ExprPhraseIsMatch` `:605-687`）

**[事实]** 对短语内每个 term 建一个 `Fts5PoslistReader`（`:629-649`，有同义词的先用 `fts5ExprSynonymList` `:522-591` 多路归并去重）；然后以第 0 个 term 的当前位置 `iPos` 为基准做多路归并：
- 要求第 i 个 term 含位置 `iPos + i`；不满足就推进该 reader；
- 若某 reader 已**超过** `iPos+i`，把基准前移到 `pPos->iPos - i` 重对齐（`:664`）；
- 命中则 append（因为位置含列号，"跨列相邻"数值上不可能相等 → 短语天然要求**同列连续**，`:670-673`）；
- `bFirst`(^) 过滤：仅当 offset==0 才写入（`:670`）；
- 输出写入 `pPhrase->poslist`，非空即匹配（`:681`）；任何 reader EOF 即不匹配（`:648/:662/:676`）。

### 3.4 NEAR 算法（`fts5ExprNearIsMatch` `:743-831`）

**[事实]** 滑动窗口 + 就地裁剪：
- 每个短语建 `Fts5LookaheadReader`（前瞻 1 个位置，`:689-717`）；把该短语的 `poslist.n` 清零、`pOut` 指回原缓冲（`:775-780`）——注释解释这是安全的：新列表是旧列表子集，不会覆盖未读数据（`:767-774`）。
- 判据：`iMin = iMax - (短语长度 nTerm) - nNear`，要求 `iMin ≤ pos_i ≤ iMax`（`:794-801`）；不满足则前跳，`pos_i > iMax` 时抬高 `iMax`。等价于"短语 i 的起点落在窗口内"，即**短语末 token 与其后短语首 token 的间隔 ≤ nNear**。
- 每命中一次把各短语当前 pos 追加进裁剪后列表（去重，`:806-812`），再推进 lookahead 最小的 reader（`:814-822`）找下一个实例；多实例全部保留供 highlight/snippet/bm25 使用。
- 返回值 = 短语 0 的输出是否非空（`:825-830`）；单短语（nPhrase==1）时跳过整个函数（`:940`）。

### 3.5 detail ≠ full 的短路

**[事实]** `fts5ExprNearTest` 开头（`:906-918`）：非 `detail=full` 时不做任何位置运算，只判定"当前 rowid 上是否有 term 命中"（poslist.n=1）。构造期已拦截：`detail=none` 上的列查询报 `column queries are not supported (detail=none)`（`:2237-2240`）；`detail!=full` 上的短语/NEAR/`^` 报 `phrase/NEAR queries are not supported (detail!=full)`（`:2417-2432`）。

### 3.6 布尔节点（迭代器归并）

- **[事实]** 公共基础：`bNomatch` 标记"伪匹配"，由 `First/Next` 外层循环跳过（`:1576-1579, 1596-1599`）；`fts5ExprSetEof`（`:1034-1041`）与 `fts5ExprNodeZeroPoslist`（`:1043-1057`，防止 aux 函数读到伪匹配的位置）。
- **[事实]** AND（`Test_AND` `:1332-1382` / `Next_AND` `:1384-1397`）：只推进第 0 个孩子；反复"最大 rowid 收敛"——落后的孩子 `NodeNext(child, 1, iLast)` 跳到 iLast 之后，取到更大值就抬高 iLast 重来（`:1347-1369`）；任一孩子 EOF → 全 AND EOF（`:1362-1365`）。
- **[事实]** OR（`Test_OR`、`Next_OR`）：`Test_OR` 在孩子里挑最靠前的（同 rowid 优先非 `bNomatch`，`:1288-1297`）；`Next_OR` 推进所有"等于旧 rowid"的孩子后重选——天然去重（`:1309-1323`）。
- **[事实]** NOT（`Test_NOT` `:1399-1425` / `Next_NOT` `:1427`）：恰好 2 个孩子，语义 `child0 AND NOT child1`；循环：左在右之后 → 前跳右；相等（或右是伪匹配）→ 排除候选、推进左（`:1410-1416`）；结果继承左孩子。
- **[事实]** n 元化：`fts5ExprAddChildren`（`:2283-2296`）把同型节点**搬平**（`A AND B AND C` 是 3 孩子而非嵌套）并维护 `iHeight`（深度上限检查依据）。

## 4. 构造路径（字符串 → 树）

- **[事实]** `sqlite3Fts5ExprNew(pConfig, bPhraseToAnd, iCol, zExpr, …)` `:271-341`：词法循环喂 Lemon（`sqlite3Fts5ParserAlloc` → `sqlite3Fts5Parser` → Free，`:290-298`；语法文件生成的入口名 `sqlite3Fts5Parser`，`parse.y:42`）；若 `iCol < nCol`，把 LHS 列施加为**整棵树的隐式列过滤**（`:305-313`）；装配 `Fts5Expr`（`:317-329`）。
- **[事实]** 分词发生在构造期：`sqlite3Fts5ParseTerm`（`:1847-1897`）对每个 STRING 调 `sqlite3Fts5Tokenize(… QUERY[|PREFIX])`（`:1864-1868`），回调 `fts5ParseTokenize`（`:1732-1799`）把 token 变成 `Fts5ExprTerm`；同义词（`FTS5_TOKEN_COLOCATED`）挂到 `pSynonym` 链（`:1751-1767`）。
- **[事实]** 一个引号串/裸词 = **一个 `Fts5ExprPhrase`**；分词器回调多少次就追加多少个 term——**短语的切分粒度由 tokenizer 决定**。
- **[事实]** `sqlite3Fts5ExprAnd(pp1, p2)`（`:457-491`）：把多条 MATCH 约束（`t MATCH 'a' AND t MATCH 'b'`）拼成一棵 AND 树并合并短语数组。
- **[事实]** `sqlite3Fts5ExprPattern`（`:364-430`）：LIKE/GLOB → MATCH 的构造路径（trigram 专用；只把连续字面量 ≥3 字符的片段转成短语，`:394`；`detail!=full` 时改写为 AND 树 `:414-422`）。
- **[事实]** `sqlite3Fts5ExprClonePhrase`（`:1903-1994`）：**复用已分好的 term** 重建短语（不重新分词），供 `xQueryPhrase` 使用（`:1951`）。

## 5. xBestIndex → xFilter 的传递（与 SQL 层的接口）

- **[事实]** `fts5BestIndexMethod`（`main.c:614-771`）把可用约束编码为 `idxStr` 字符串：`'M'<iCol>`（MATCH）、`'r'`（rank 列 MATCH）、`'L'/'G'<iCol>`（trigram LIKE/GLOB）、`'='`/`'<'`/`'>'`（rowid 比较），并全部 `omit=1`；ORDER BY rank/rowid 反映到 `idxNum` 与 `orderByConsumed`。
- **[事实]** `fts5FilterMethod`（`main.c:1450-1650`）解码 `idxStr`：`'M'` 分支取 MATCH 文本（处理 `fts5_locale()` 包装与 insttoken 子类型，`main.c:1411-1436, 1505`），以 `'*'` 开头走 special query，否则 `sqlite3Fts5ExprNew` 建树、多 MATCH 用 `sqlite3Fts5ExprAnd` 合并（`main.c:1523-1525`）；随后选计划（`FTS5_PLAN_MATCH` / `SORTED_MATCH` / `ROWID` / `SCAN` / `SOURCE`）并 `fts5CursorFirst`（`main.c:1136`）→ `sqlite3Fts5ExprFirst`。
- **[事实]** 列号语义：用户列 `0..nCol-1`；`iCol==nCol` 是"表名"隐藏列（`t MATCH ?` 的 LHS，不施加列过滤）；`nCol+1` 是 rank 列。

## 6. 位置数据、rank 与 aux 函数的位置（分工边界）

- **[事实]** expr 层只产出"哪些 rowid 匹配 + 该行各短语的位置列表"。**bm25/highlight/snippet 都不在 expr 层**（在 `fts5_aux.c`），通过 `Fts5ExtensionApi` 取数据。
- **[事实]** 位置数据两条来源（`main.c`）：`fts5CsrPoslist`（`main.c:2312-2375`）——detail=full 直接读 expr poslist；否则**重新 tokenize 文档正文**回填（`sqlite3Fts5ExprPopulatePoslists` `expr.c:3084-3112`；校验 `CheckPoslists` `:3170`）。
- **[事实]** 实例数组 `fts5CacheInstArray`（`main.c:2382-2454`）把每短语 poslist 归并成 `(iPhrase,iCol,iOff)` 三元组数组——这是"按需物化"的唯一一处，只在 aux 函数真要位置时才建。
- **[事实]** `xQueryPhrase`（`fts5ApiQueryPhrase` `main.c:2862`）= 克隆短语并构造一个新的 MATCH cursor 再跑一次索引查询。
- **[事实]** `ORDER BY rank` 时，main.c 用临时 sorter 表物化 `(rowid, poslist blob)`（`fts5CursorFirstSorted` `main.c:1084-1134`），排序后 `xPhraseFirstColumn` 改从 sorter 读（`main.c:2706-2717`）。

## 7. 设计取舍（为什么是 lazy 迭代器）

**[推断]**（代码无明说，由实现推出）：
1. **天然支持 seek/提前终止**：`xNext(..., iFrom)` 让 rowid 上界/连接顺序约束能直接下推，物化结果集会丢掉这个能力。
2. **内存 O(1)/行**：每节点只保存当前 rowid 与复用的 poslist 缓冲；NEAR 甚至**原地覆盖**旧列表（注释 `:767-774`），不为匹配行保存副本。
3. **与 SQLite 火山模型一致**：上层逐行拉取，不预扫全表；并发写导致 cursor 失效时只需以旧 rowid 重定位（`fts5CursorReseek` `main.c:957-980`）而不是重建结果集。
4. **昂贵物化按需付费**：位置数组只有 aux 函数要时才建；detail≠full 连 poslist 都不产生。
5. **代价**：迭代是"比较+追赶"式（AND 的收敛循环、短语里的基准回退），最坏比物化归并多做常数倍比较；这是用 CPU 换内存与早期终止。

## 8. 与常见命名/记忆的出入（勘误）

- **[事实]** 不存在 `fts5ExprPhraseNext` / `fts5ExprNearAssert`；对应物是 `fts5ExprPhraseIsMatch`（`:605`）与 `fts5ExprNearIsMatch`（`:743`）/`fts5ExprNearTest`（`:898`）。
- **[事实]** 节点层只有 `xNext` 一个函数指针；"First"由 `fts5ExprNodeFirst`（`:1493`）+ `sqlite3Fts5ExprFirst`（`:1552`）承担。
- **[事实]** "句内/最后一句话"类特性**不存在于匹配层**：全库 `sentence` 只出现在 `fts5_aux.c:302-368`（snippet 片段起点对齐），不参与匹配。
- **[事实]** 测试佐证语法行为的权威样例：`test/fts5eb.test:32-56`（优先级/隐式 AND/空短语）、`test/fts5expr.test`、`test/fts5colset.test`。
