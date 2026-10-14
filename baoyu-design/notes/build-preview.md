# 模块笔记：build-preview.mjs（设计系统 → 单文件 preview.html）

> `agents/build-preview.mjs`，1919 行，全仓最大单文件。把「有 `_ds_manifest.json` 和/或带 `@dsCard` 的 HTML 卡片」的设计系统目录编译成**一个自包含、无 iframe 的交互式 preview.html**（文件头 `2-5`）。
> CLI：`node build-preview.mjs <ds-dir> [--out file] [--title t] [--cdn] [--offline]`；默认输出 `<root>/preview.html`（`1555-1583`）。

## 1. 输入与退化

- **事实**：读 `_ds_manifest.json`（坏 JSON 只 warn）；**manifest 缺失或 cards/startingPoints/templates 全空** → 退化为全树扫描 `scanForCards`（跳过 node_modules/.git/_import/_ds/_sources/dist/build 与点目录；匹配每个 `.html` 前 600 字符里的 `@dsCard` 指令）（`1597-1612,274-307`）。
- **事实**：manifest 有数据时按 `buildCardGroups`（移植自 TS 端）归组去重（`startingPoints[].previewPath` 去重同名卡片）；viewport 支持 `"WxH"` 与对象两形态（`170-216`）。Templates 组在最前，其后按插入顺序（`1617-1655`）。

## 2. 卡片解析与资源内联（关键机制）

- **事实**：`parseCardHtml` 用一个大正则切出 `link/style/script`（`672,770-878`）：
  - 本地 CSS → `processCss` 递归内联（`@import` 递归、`url()` 全量 data URI 化、远程 import 保留为 remoteCss）（`609-662`）。
  - 远程 react/react-dom CDN → 只记 `needsReact` 和版本号；远程 babel CDN → 丢弃（构建期已转译）（`674-680,819-831`）。
  - 本地 `_ds_bundle.js` → 整文件内联进 head（`834-838,1815-1821`）。
  - 本地/内联脚本 → 资源重写；`type=text/babel|jsx` 打 `babel:true`（`839-859`）。
- **事实**：内联器 `makeAssetInliner`：≤3MB 才内联（`MAX_INLINE_ASSET`，超限只 warn）；`resolveRef` 拒绝外部协议且用 `abs.startsWith(root)` **阻断路径越界**（`575-607`——对应 CHANGELOG 1.3.0 的 path-traversal 修复）。
- **事实**：**JSX 字符串里的资产引用**也被处理：`STR_REF_RE` 匹配"相对/根路径+资产扩展名"的裸字符串 → data URI（`682-704`）——这条修复了早期"脚本侧引用资产不内联导致坏图"的问题（CHANGELOG 2026-06-11）。
- **事实**：`srcset` 每个候选分别内联；内联事件 `on*=` 改写成 `data-ds-on*` 属性交给沙箱重绑（`713-767`）。
- **事实**：运行时 `fetch()` 兜底：若卡片脚本含 `fetch(`，把项目内 ≤3MB 的 svg/png/jpg/json/txt/csv 全部收进 payload（总预算 3MB），沙箱的 fetch 代理按路径查表（`230-271,1120-1134,1855-1869`）。

## 3. React / Babel 处理（事实）

- 常量 `react 18.3.1 / @babel/standalone 7.29.0`，`VENDOR_DIR = agents/vendor`（`30-32`）。
- **构建期 Babel**：优先 Node 依赖 `@babel/standalone` → 其次 `vendor/babel.min.js` → 再次下载缓存到 vendor（`new Function("module","exports","window",…)` 手工求值 UMD）（`133-166`）。有 babel 脚本就用 `Babel.transform(code,{presets:[["react"]]})` 全量转译并清 mark（`1674-1698`）；失败则退到"内联 babel.min.js + 运行时逐脚本转换"（`1230-1242,1697-1701,1803-1814`）。
- **React 内联**：只在 `needsReact||usesBundle` 时；默认把 unpkg react/react-dom 下载缓存进 vendor 并内联成 `<script>`，失败回落 CDN 标签（`1783-1800`）。`safeInlineScript` 转义 `</script` 与 `<!--`（`70-72`）。

## 4. 生成页面结构（事实）

- **宿主 CSS**：`HOST_CSS` 模板——暖色 token、左右两栏、248px 左导航（≤1007px 隐藏）、760px 内容列、卡片框/舞台、readme prose 风格（`882-1014`）。
- **每张卡 = 声明式 Shadow DOM**：`<template shadowrootmode="open">` + `.__dsroot` 包裹卡片 body（`1493-1551`）。
- **卡片 CSS 按 shadow 语义改写**：`:root/html → :host`、`body → .__dsroot`；`@font-face` 抽出提升到宿主 `<style>`（shadow 不生效）；`@media/@supports/@layer/@scope` 递归作用域化；CSS 块按 djb2 去重跨卡共享（`47-51,534-565,1704-1735,1883`）。
- **payload**：`{cards:[{id,w,h,base,css:[hash],js:[{code,babel,name}]}],css:{hash:css},assets?}`，`<` 转义为 `\u003c`，塞进 `<script type="application/json" id="ds-data">`（`1865-1894`）。
- **运行时 RUNTIME_JS**（`1027-1438`）：shadow 挂载 + `adoptedStyleSheets`（降级插 style）+ 隐藏滚动条的 BASE_CARD_CSS；**每卡一套 `document/window` Proxy 沙箱**（querySelector 等落到该卡 shadow 根；window 写入只进 `vars` 不泄漏；DOMContentLoaded/load 收集后统一 flush；脚本 `new Function(...,"with(__scope){…}")` 执行并把顶层声明回写 `__vars`）；`data-ds-on*` 重绑真实事件（`1102-1273`）。
- **自适应缩放**：`fit()` 仅按宽度缩放（`min(containerWidth/designW,1)`），高度随 `scrollHeight` 单调增长（上限 `max(h,4000)`）防裁剪；首屏 + MutationObserver + 捕获式 load + ResizeObserver + `document.fonts.ready` + 250/750/1500/3000ms 兜底重测（`1275-1358`）。
- 左导航平滑滚动 + 900ms scroll-spy 抑制 + 组折叠；readme 折叠并重测（`1360-1434`）；内置小型 Markdown 渲染器（`311-457`）。

## 5. 测试（`agents/tests/build-preview.test.mjs`）

- **事实**：① 无参数 → 非 0 退出 + `Usage:`；② 用 `helpers.makeDsFixture` → 先 `compile-design-system.mjs` → 再 `build-preview.mjs --offline`，断言输出存在、`<!doctype html>`、标题、`id="ds-data"`、卡片标题、readme 内容，以及关键自包含断言 `!html.includes('href="../styles.css"')`（`build-preview.test.mjs:12-36`）。
- **事实**：测试用 `--offline` + 纯静态卡片，**不覆盖 React/Babel 与 fetch 代理路径**。

## 6. 观察（推断）

- 这个文件的复杂度（1919 行）本质是**一个迷你打包器 + 沙箱运行时**：构建期做「CSS 作用域化 / JS 转译 / 资源内联 / 载荷序列化」，运行期做「shadow 隔离 / document-window 代理 / 尺寸自适应」。
- 它与 `import-design-system.mjs` 是**同一问题的两种输出**：import 产出"可给项目消费的运行时快照"，build-preview 产出"给人看的自包含审阅页"。
