# 模块笔记：Figma 导入器（import-figma.mjs + vendor/fig-materialize.mjs）

> 定位：把本地 `.fig` 文件**完全离线**解码成参考树 / React 组件 / 设计系统。不需要 Figma 账号、API token 或 MCP（`SKILL.md:52-54`）。
> 规模：`import-figma.mjs` 740 行 + `vendor/fig-materialize.mjs`（大）+ `vendor/fflate.mjs`（同步解压）。

## 1. `.fig` 离线解码原理（事实）

1. **容器判定**：前两字节 `PK` → ZIP；否则按裸 kiwi 容器处理（`import-figma.mjs:123`）。
2. **ZIP 内容**：`*.fig` 字节 = canvas（多个取最后一个）、`*.json` = meta、`images/<hash>` = 图片资产（mime 由 magic 嗅探）（`fig-materialize.mjs:424-437,486-514`，fflate `unzipSync`）。
3. **canvas 头**：8 字节 magic `fig-kiwi`（也接受 fig-jam/fig-deck/fig-make/fig-site/fig-buzz），uint32 版本，之后 `uint32 length + bytes` 的 chunk 序列；用 chunk[0]=schema、chunk[1]=data（`439-455`）。
4. **压缩**：zstd magic（`28 B5 2F FD`）→ **内联 base64 的 ZSTD WASM 解码器**（Node 里 `WebAssembly.instantiate`）；否则 inflate（deflate）（`388-423`）。
5. **schema 解码**：kiwi 二进制 schema → `decodeBinarySchema` → `compileSchema` 用 `new Function` 现场编译成解码器（自带手写 ByteBuffer，kiwi-schema 风格）→ `decodeMessage(dataChunk)`（`11-…,321-379,455`）。**即：从 .fig 里自带 schema 现场生成解码器**，这是"离线解码"的核心。
6. **建树**：按 `guid`（`sessionID:localID`）建 Map，用 `parentIndex.guid` 挂 children，按 `parentIndex.position` 排序；根必须是 `0:0`（`407-475`）。`FigDocument` 提供 `pages()/frames()/getNode/summary/nodeProps/find()/tree`（`552-602`）。

## 2. 子命令（事实）

- `outline <file.fig> [--json]`：枚举组件（变体集 + 独立 SYMBOL）、tokens、样式类、pages/frames（每页最多 20）；输出摘要 + `Next:` 命令提示（`import-figma.mjs:129-147,344-392`）。
- `mount <file.fig> <destDir> [--name/--pages/--force]`：解码成**可浏览参考树** `<destDir>/_fig/<slug>/`（React 模块 + 资产 + node-index.json guid→路径）；重写 README 的 Tools 行 + agent 使用规则附录（`399-481`）。
- `materialize <file.fig> --out <dir> (--components|--frames) [--tokens] [--typography] [--annotate]`：发射 `.jsx/.d.ts` + `assets/` + `fig-assets.css`/`fig-tokens.css`/`fig-typography.css`；报告大小写冲突/未解析名/字体族提示（`197-319,512-548`）。
- `render <file.fig> --frame <guid|name> --out <file.html>`：单文件 HTML 视觉 ground truth，图片全部 data URI 内联（`552-576`）。
- `design-system <file.fig> <destDir>`：全量组件 → `components/` + `tokens/` + 只含 `@import` 的 `styles.css` + stub README；打印"尚未编译/未策展"+ 编译→检查→预览三步（`593-645`）。
- 退出码统一 0/1/64；只写给定目标目录（`28,59-67`）。

## 3. fig-materialize.mjs 的职责（事实）

- 全流程：`.fig bytes → zstd/deflate → kiwi decode → node tree → FigVfs 虚拟文件系统 → React JSX / token CSS / typography CSS / 资产发射`（文件头 `1-8`）。
- **字体权重解码**：Figma 样式名（thin/extra light/light/medium/semi bold/bold/extra bold/black…）→ 100–900 数字权重（`FONT_WEIGHT_PATTERNS`，`3786`）；用于节点 `textStyleProps`（`971-981`）与 styleOverride（`3787-3796`，含 fontVariations、small-caps、上下标）。字体栈 mono/serif 启发式（`964-969`）；**`.fig` 不含字体二进制**——输出明确提示自备 `@font-face`/CDN（`import-figma.mjs:540,634,703`）。
- **资产提取**：位图按 `images/<hash>` → `assets/<hash[:16]>.<ext>`；单张 4MiB/合计 16MiB 预算，超限丢弃并告警；`url()` 抽象成 `.fig-asset-*` 类写进 `fig-assets.css`（`2044-2062,4814-4854,4924`）。矢量：小路径直接内联 `<svg>`；路径总长 >400 或 >3 条 → 独立 `.svg` 资产 + `<img>`（`2205-2319`）。图标字体（FontAwesome/Material）文本节点渲成内联 SVG 字形（`3769-3781,3970-3981`）。
- **变体/Props 模型**：`buildPropModel`（变体轴 + 组件属性 → props）；`synthTextSlots/synthIconSlots` 把重复文案/可替换实例变成 `textN`/`iconN` props（`3542,3705-3768`）。
- **Token/样式**：`collectTokens`（Figma variables → 带 modes 的 CSS 变量；mode 选择器）（`3128,3152,3358`）；引用了但未定义的 `var()` 顶部列 TODO（订阅库别名）（`5055-5068`）；TEXT/EFFECT 共享样式 → `@layer typography` 类（`4977-5024`）。
- **切分与落盘**：`weightMap/pickSplits` 按权重把大组件切子模块（`2121-2201`）；`FigVfs` 管页面→slug 树与 `mountModule` 递归落盘（`2813-3029`）；`emitFigSelection` 汇总 `{files,assets,emitted,missing,fonts,warnings,tokensCss,…}`（`5070-5111`）。

## 4. 测试（`agents/tests/import-figma.test.mjs`）

- **事实**：用 `tests/fixtures/minimal.fig` 真跑 `outline` / `mount` / `design-system` 三个子命令，断言解码与产出（fixture 由构造器生成——具体构造方式未逐行核对）。
- **未知**：fixture 的生成脚本位置与内容；大 `.fig`（如 Chakra UI Kit）的实际吞吐/耗时没有本地基准。

## 5. 边界与洞察（推断）

- **推断**：与上游的关系——claude.ai/design 用 `enableFigmaMcp:true`（provenance.json 的 systemPromptOptions），本仓库把同一能力做成**离线解码**，把"Figma 账号/MCP"这个依赖整个去掉了，这也是 README 宣传点（"no Figma account, API token, or MCP server"）。
- **推断**：`design-system` 子命令的产出是"未策展的原料"（组件/ token 是机翻），必须继续走 authoring 指南的人工策展 + compile + check + preview 四步流水线才算成品。
