# 未解问题清单（sqlite / FTS5）

> 状态：首轮分析（2026-10-15）后建立。标注：**[未查]** 没看源码；**[未验证]** 看到代码但没运行确认；**[存疑]** 源码树内证据冲突或不足。

## 材料缺口（需要官网/发布历史）

- **[存疑]** "distributed b-tree" 这一提法：全仓库 grep `distributed b-tree|distributed btree` **0 命中**。源码里只有 "segment b-tree"（`ext/fts5/fts5_index.c:29`）与 "Doclist indexes are themselves b-trees"（`:207`）。该说法应出自官网 `fts5.html`，本仓库无法佐证。
- **[未查]** 各特性的**引入版本号**：`detail=none`、`detail=columns`、`tokenizer v2`、`contentless_delete`、`contentless_unindexed`、`locale=1`、`secure-delete` 分别从哪个 SQLite 版本开始提供——仓库内无注释，grep 不到 3.x.y 字样。
- **[未查]** 官网版 bm25 公式与参数说明（源码已给出实现：k1=1.2, b=0.75, IDF 下限 1e-6, 返回 -score，见 `fts5_aux.c:669-752`；但官网页面的表述未核对）。
- **[未查]** `automerge/usermerge/crisismerge/deletemerge` 的**官方文档版**解释（仓库内只有常量、取值范围 `fts5_config.c:950-1001` 与实现注释 `fts5_index.c:4982-5080`）。

## 实现细节未逐行读

- **[未查]** `fts5_index.c` 的 secure-delete 全路径（`fts5DoSecureDelete` `:5262-5561`，约 300 行）与 `fts5SecureDeleteOverflow`（`:5164-5256`）：只知"写前擦除旧字节 + 格式升 5"，未核对溢出页与非溢出页的差异。
- **[未查]** tombstone 的读写实现细节：`fts5IndexTombstoneAdd/Rehash/Rebuild`（`:7873/7927/8006/8108`）只读了格式注释（`:234-266`），未逐行验证哈希槽位与页溢出的处理。
- **[存疑]** tombstone 查询侧的函数名与行号：子代理给出 `fts5IndexTombstoneQuery`（`:3324`）与 `fts5MultiIterIsDeleted`（`:3360-3387`）两处提法，本笔记未逐一复核（`notes/fts5-index.md` §8 已标注）。
- **[未查]** `fts5_tcl.c`（1753 行）：只扫了注册命令清单，未读每个命令的语义。
- **[未查]** `fts5_test_tok.c` / `fts5_test_mi.c` 的完整行为（作为自定义 tokenizer / aux function 的样例，值得细读）。
- **[未查]** `fts5_vocab.c` 的 `fts5VocabNextMethod`（约 480-620 行）在 detail=columns 下的完整计数逻辑。
- **[未查]** `fts5_index.c` 的 integrity-check 实现（`sqlite3Fts5IndexIntegrityCheck` `:8669` 起）的逐项校验清单。
- **[未查]** `fts5_index.c` 反向迭代（`fts5SegIterReverseInitPage` `:2078`）的边界条件（页头 rowid 偏移为 0 时）。

## 未验证的主张（未跑测试/未构建）

- **[未验证]** 所有测试的通过情况：本次为只读分析，未执行 `test/testrunner.tcl fts5%`（148 个测试文件）。想验证本笔记的结论，这是最快的复核手段。
- **[未验证]** 性能类注释（如 "16^8 足以合并任意多列表" `fts5_index.c:6740-6748`、"每页数据被重写次数 ≈ log 层数"）的实际数据——未跑 benchmark。
- **[未验证]** `prefix=` 三条查询路径（`fts5_index.c:7452-7503`）在真实数据上的选择行为（哪个前缀长度会命中哪条路）。
- **[未验证]** `pgsz` / `hashsize` / `automerge` 调参对性能与空间的实际影响。
- **[未验证]** `contentless_delete=1` 的 delete-merge 在真实负载下的触发频率与空间回收效果。

## 机制边界问题（想清楚了但没写进正文）

- **[存疑]** `%_idx` 表里 "split key"（页首 term）的具体取法：注释说"比本页前面所有 term 都大、比本页新 term 小或等"（`index.c:4516-4539`），实际实现取"公共前缀+1字节"——极端情况下（如整个页只有一个 term）是否有下界保证未深究。
- **[未查]** `sqlite3Fts5IterNextFrom` 在**没有 dlidx** 的大 doclist 上的实际复杂度（注释说会线性扫页，但未验证页内是否有二分）。
- **[未查]** `tokendata=1` 表上 `nFullTerm` 的截断语义（`fts5QueryTerm` `expr.c:3032-3036`）与 `xQueryToken/xInstToken` 返回值的关系。
- **[未查]** `fts5_locale()` 的完整编码格式（`sqlite3Fts5DecodeLocaleValue` `main.c:1371` 起）与 locale 参与 profile/缓存的方式。
- **[存疑]** 变更 `detail=` 是否需要 rebuild：测试覆盖了 detail 行为（`test/fts5detail.test`），但 "ALTER 改 detail 是否报错/需重建" 没有官方路径，源码里 detail 只在建表时解析（`config.c:404-416`）且不在 `%_config` 持久化键清单里——推断改 detail 只能重建表，未验证。
- **[未查]** `columnsize=0` 时 bm25 的成本（`fts5.h:87-88` 有提示，未验证实测差异）。

## 工作树状态备注

- **[事实]** 目标仓库有一处未提交本地改动：`ext/fts5/fts5_aux.c` 头部注释块行文调整（`git diff` 仅注释）。本笔记结论基于该状态；该改动不影响任何代码逻辑。
