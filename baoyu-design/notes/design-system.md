# 模块笔记：设计系统工具链（authoring / check / import / prompt）

> 相关文件：`agents/compile-design-system.mjs`(381) / `check-design-system.mjs`(110) / `lib/ds-core.mjs`(750) / `lib/ds-prompt.mjs`(290) / `import-design-system.mjs`(274) / `lib/asset-store.mjs`+`record-asset.mjs`；文档 `built-in-skills/design-system-authoring-guide.md`、`use-design-system.md`、`design-system-preview.md`。

## 1. 「设计系统」的磁盘形制（事实）

- 平铺在 `designs/` 下，**根标记**区分（不靠目录名）：DS 根有 `_ds_manifest.json`；普通项目根有 `_d_meta.json`，消费副本在 `<project>/_ds/<slug>/`（同名 manifest 深一层）→ `glob designs/*/_ds_manifest.json` 只命中 authored 系统（`use-design-system.md:11-20`）。
- 目录约定（`design-system-authoring-guide.md:43-55`）：`styles.css`（或 index/globals/global/main/theme/app/tokens.css，只写 `@import`）→ `tokens/` 一文件一关注点 → `components/<group>/<Name>.jsx|tsx` + 同名 `.d.ts`（有 `.d.ts` 才算组件模块）+ `<Name>.prompt.md` → `ui_kits/<product>/index.html` → `guidelines/`、`assets/`。
- 卡片：每个目录一张 HTML，首几行 `<!-- @dsCard group=… viewport=… subtitle=… name=… -->`（`ds-core.mjs:597-616`）；卡片里用普通 `<script src=".../_ds_bundle.js">` + `type="text/babel"` 挂载。
- 起始点（seed）：组件写在 `.d.ts` JSDoc `@startingPoint section=…`；屏幕写在 HTML 首行注释（`ds-core.mjs:648-688`）。
- 生成物：`_ds_bundle.js`、`_ds_manifest.json`、`_adherence.oxlintrc.json`（compile 产出）（`compile-design-system.mjs:365-367`）；`preview.html`（build-preview 产出，见 notes/build-preview.md）。
- DS 根还带 `SKILL.md`（可下载为 Agent Skill 的便携件；在本仓库不注册为 skill）（`use-design-system.md:20`）。

### `_ds_manifest.json`（事实）
`namespace, components[{name,sourcePath,kind?,props?[{name,values?|type,default?}]}], startingPoints[{name,path,previewPath,kind,section,subtitle,viewport}], cards[], globalCssPaths[], tokens[{name,value,kind,definedIn,annotation?}], themes[], fonts[], brandFonts[{family,status:'no-face',tokens,path}], source:'spa'`（`compile-design-system.mjs:248-261`、`ds-core.mjs:728-747`）。

### `_ds_bundle.js`（事实）
首行 `/* @ds-bundle: {format:3, namespace, components, sourceHashes{path:sha256[:12]}, inlinedExternals, unexposedExports} */`；主体 IIFE：`window.<NS> = window.<NS> || {}` + 私有 `__ds_scope` + `__errors[]`，每源文件一个 `try{(()=>{…})()}catch{…push}`，末尾把暴露名挂到 namespace（`compile-design-system.mjs:207-244`）。

### `_adherence.oxlintrc.json`（事实）
advisory 级 oxlint 规则：禁裸 hex、禁裸 px、每组件 props 白名单、枚举值白名单、禁直接 import 组件内部；`x-omelette.{components,tokens,tokenKinds,fontFamilies}`（`compile-design-system.mjs:270-338`）。

## 2. compile-design-system.mjs（唯一写步骤）

- **事实**：明确"只有本脚本写；解析器与检查器不写"（`compile-design-system.mjs:2-15`）。发现/解析全部委托 `buildModel(projectDir)`（`348-354`）。
- **事实**：转译器 = **vendored Babel 在 Node 里跑**（`require('./vendor/babel.min.js')`，UMD→CJS；缺失给 curl 提示并 exit 1）（`23-39`）。preset：`['react',{runtime:'classic'}]`（.ts/.tsx 加 typescript）（`97-112`）。自定义 `makeStripPlugin`：删 export、删 react 导入、相对兄弟导入改为 `const {X} = __ds_scope` 解构（`47-95`）。
- **事实**：bundle 内块顺序 = 相对 import 图 **DFS 后序**（依赖先于使用者；`/x/y` 视为根相对）（`119-166`）；单文件转译失败不致命——替换为 `throw new Error('transpile failed: …')` 进 `__errors`（`189-193`）。
- **事实**：暴露规则：`.d.ts` 背书的组件优先占名；非模块文件的 PascalCase 导出额外暴露（`219-232`）；同名非模块实现被过滤（`194-199`）。
- **事实**：stdout 汇总（组件数/cards/starting points/token 直方图）；参数错误 exit 64、目录不存在/缺 Babel exit 1。

## 3. ds-core.mjs 的 buildModel()（只读解析器）

- **事实**：零写入，仅 `node:fs/path/crypto`（`ds-core.mjs:1-7`）。遍历跳过 `SKIP_DIRS`（node_modules/.git/dist/build/.next/out/coverage/.cache/.turbo/vendor）、点文件、生成物（`17-47`）。
- **globalCssPaths**：文件名启发式找入口（候选名表 `56-73`），未命中回退读已编译 manifest 的 `globalCssPaths` 最后一项；随后总是重新解析 `@import` 后序闭包；跳过远程 import、`seen` 防环（`81-118`）。全失败 → issues "No global CSS entry found"。
- **tokens**：闭包内所有 CSS 抽 `--*: value;` + 紧随 `/* @kind xxx */`；**幻影匹配防护**（值含 `{}` 即跨选择器边界丢弃——修 BEM `--primary:hover{}` 误读）（`124-198`）。分类值优先：`@kind` > classifyByValue（color/spacing/shadow，支持 oklch/lab/hwb）> `other`＋unclassified 提示。`var()` 解析器支持链式+fallback，深度 10 防自引用（`132-144`）。
- **components/props**：`collectExports` 正则按源顺序收集导出（忽略 default）；组件 = 同目录有 `.d.ts` 的 `.jsx/.tsx` 且 PascalCase；全大写导出标 `kind:'constant'`（`210-234,546-563`）；props 来自 regex 级 `.d.ts` 解析（接口 `NAMEProps`/`NAME`、`@default`、字符串字面量联合、一次类型别名跳转、其余 fail-open 成 `type` 字符串）（`248-313`）。
- **issues**（检查器打印用）：孤儿 `.d.ts`、重复组件名、HTML 直接 `<script src="X.jsx">` 引组件、卡片 `text/babel` 顶层绑定撞 `window.*` 风险名（`RISKY_GLOBALS`，Babel 把顶层 const 变 var 写 window）、README 里过期的 namespace 哈希、未分类 token、`@font-face src` 不存在（`361-722`）。
- **namespace**：manifest → bundle 首行 → `PascalCase(name)_<sha256[:6]>`；持久化保证重编译不变（`423-441`；测试钉住）。

## 4. check-design-system.mjs（只读校验器）

- **事实**：只 import buildModel，不写任何文件（这使它可作为只读 subagent 运行）（`check-design-system.mjs:6-13`；测试对比前后目录树 `check-design-system.test.mjs:28-33`）。
- **事实**：输出 = 一行 summary（namespace/组件数/cards/starting points/tokens/fonts；组件列表上限 40）+ `Issues to fix:` 列表或 `No issues — clean. ✅`；`--verbose` 追加 token 直方图、brand fonts、export map（`✓` 暴露 / `·` 未暴露）（`37-105`）。
- **事实**：退出码 `globalCssEntry ? 0 : 2`；用法 64；非目录 1（`110`）。

## 5. import-design-system.mjs（消费侧同步）

- **事实**：`node import-design-system.mjs <dsDir> <projectDir> [--primary]`；要求 dsDir 根有 `_ds_manifest.json`（否则 exit 1 提示先 compile）（`42-48`）。
- **事实**：拷贝集 = ① globalCssPaths（后序）② 这些 CSS 的本地 `url()` 目标 ③ `_ds_bundle.js`/`_ds_manifest.json`/`_adherence.oxlintrc.json`/`README.md`/`SKILL.md` ④ `assets/` 目录（`100-131`）；`url()` 解析跳过远程/data/#、路径越出 DS 根则跳过（`74-93`）。**绝不碰 DS 源树**，不转译。
- **事实**：`_d_meta.json` 合并写：`designSystems[]` 按 slug 幂等覆盖 + `primaryDesignSystem`（首个导入自动抢占，`--primary` 覆盖）（`146-168`）；显示名取 README 首个 H1 → SKILL.md name → slug title-case（`134-144`）。
- **事实**：生成 `_ds/<slug>/_ds_prompt.md`（详见 §7）；stdout 报告 + 接线代码 + startingPoints + warnings（未编译缺 bundle 等）（`170-260`）。
- **事实**：`_d_meta.json` 整体形状（`use-design-system.md:105-136`）：`type/title/prompt/designSystems[]/primaryDesignSystem/startingPoint/assets{}/createdAt/updatedAt`。

## 6. 消费侧流程与多系统（文档事实）

- 流程（`use-design-system.md` 步骤 1-8）：发现（glob）→ 问（保存位置 + multiSelect 系统 + 可选 seed）→ import → **接线**（React/ReactDOM 先行、每系统一组 `<link>`（闭包顺序）、bundle 用普通 `<script>`、primary 组最后）→ **读 `_ds_prompt.md` 当 binding 视觉契约**（不得自创颜色/type/spacing/组件；token 只许用白名单）→ seed（复制屏幕到项目根并改写链接；组件 seed 直接 render）→ `_d_meta.json` → （可选）对副本跑 checker（预期 "Components: (none)"——副本只有 bundle 没有源码）。
- 多系统：JS 靠 `window.<Namespace>` 天然隔离；**CSS 是共享 scope**，靠 `<link>` 顺序定优（primary 最后）（`use-design-system.md` "Multiple design systems"）。
- 恢复：重开项目先读 `_d_meta.json`；`designSystems` 非空即已绑定，不再问；刷新=重跑 import（幂等）；换 primary=重跑带 `--primary`；移除=删 `_ds/<slug>/`+条目+接线两行（`system-prompt.md:20`、`use-design-system.md` "Resuming"）。

## 7. ds-prompt.mjs（`_ds_prompt.md` 渲染）

- **事实**：纯字符串装配（不读不写）；内容：binding+scope 声明 → bundle-first 接线段（固定 React UMD 标签含 integrity + 全部 `<link>` + bundle `<script>`）→ 页面自身 JSX 的 `text/babel` 示例 → **"text/babel 顶层禁用 window 全局名"清单**（`status/name/open/close/top/self/…`，因 Babel 注入为 classic script 会写 window）→ 源树指针 + 完整指南内联（`<design-system-guide>`）→ 每组件 `*.prompt.md` 摘录（`<ds-prompt-excerpts>`；围栏截断但保持闭合）→ props 契约（`<ds-component-props>`；枚举值完整、`*` 标默认）→ token 白名单（精确 `--*` 列表）(`ds-prompt.mjs:1-18,27-31,154-286`)。
- **事实**：`extractPromptExcerpt` 空白行不计入行数上限，确保围栏示例能被选中；`sampleComponentNames` 优先 PascalCase 有大小写的名字（防 `ICON_NAMES` 这类常量被当用法示例）（`ds-prompt.mjs:33-88`）。

## 8. 资产记录（record-asset.mjs + asset-store.mjs）

- **事实**：模型（`asset-store.mjs` 头注释）：`meta.assets[显示名].versions[] = {path, createdAt, status, subtitle?, viewport?{width,height?}, section?, chatId?}`；status ∈ needs-review(默认)|approved|changes-requested；**同 path 再记录 = 原地更新**（保留 createdAt，status 重写），新 path = 追加；删除后空壳清理。
- **事实**：`readMeta`：缺失→`{}`；**坏 JSON 抛错**（防静默覆盖 orchestrator 写的字段）；`bootstrapMeta` 只补 `type/designSystems/primaryDesignSystem` 不覆盖；`writeMeta` createdAt 只设一次 + updatedAt 每次 bump（`asset-store.mjs`）。
- **事实**：CLI（`record-asset.mjs`）：`<projectDir> <htmlPath>` + `--name/--inherit-from/--subtitle/--status/--width/--height/--section/--chat-id`；`--remove` 按 path（可 scoped `--name`）或按名删；路径归一化为项目相对 POSIX；显示名缺省 = 文件名去 `.dc.html`/.html；exit 64/1；**无 DS 项目也会 bootstrap 出 `_d_meta.json`**。
- **推断**：`_d_meta.json` 是「项目恢复」的锚：设计系统绑定 + 资产索引 + title/prompt，全部服务于"下次打开继续设计"。
