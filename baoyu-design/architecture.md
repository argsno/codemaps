# baoyu-design 架构与主流程

> 证据出处 `文件:行`；标记 **事实/推断/未知**。核心问题：一次「帮我设计一个 X」从 agent 进入 skill 到用户看到结果的完整链路。

## 1. 分层总览

```
┌─ 用户 ────────────────────────────────────────────────────────────────┐
│  "设计一个冥想 App 的设置页" / "做个 PPT" / "把这个 .fig 变成设计系统"     │
└──────────────────┬───────────────────────────────────────────────────┘
                   ▼
┌─ 编排层（技能入口，Markdown 提示词）────────────────────────────────────┐
│ SKILL.md            5 步编排：读方法论→识 harness→载 skill→问问题→建/验  │
│ project-types.json  13 类项目的机器可读路由表                            │
└──────┬────────────────────┬──────────────────────┬───────────────────┘
       ▼                    ▼                      ▼
┌─ 方法论层 ─────────┐ ┌─ 专项技能层 ────────┐ ┌─ harness 适配层 ──────┐
│ system-prompt.md   │ │ built-in-skills/    │ │ references/           │
│ 323 行 craft 标准  │ │ ~50 个按需加载的 .md │ │ claude.md / cursor.md │
│ （唯一事实来源）    │ │ 牌（deck/mobile/…）  │ │ / codex.md 工具映射   │
└────────────────────┘ └─────────────────────┘ └───────────────────────┘
       ▼                    ▼                      ▼
┌─ 脚手架层（无构建，复制即用）─────────────────────────────────────────┐
│ starter-components/：deck-stage.js、design-canvas.jsx、animations-v3、  │
│ 设备/社交外壳、image-slot、three-d-stage、doc-page、watercolor-kit…     │
└──────────────────────────────────────────────────────────────────────┘
       ▼
┌─ 本地工具层（agents/，把云端工具移植为本地 CLI）──────────────────────┐
│ import-design-system.mjs │ compile/check-design-system.mjs │ record-asset│
│ import-figma.mjs（离线 .fig）│ build-preview.mjs │ gen-pptx/ │ gen-video/ │
└──────────────────────────────────────────────────────────────────────┘
       ▼
┌─ 交付物 ──────────────────────────────────────────────────────────────┐
│ designs/<project>/：HTML 页面/原型（React+Babel 多文件）、_d_meta.json、  │
│ _ds/<slug>/（设计系统快照）、imgs/；导出物：.pptx / .mp4 / PDF / 单文件   │
└──────────────────────────────────────────────────────────────────────┘
```

**依赖方向（事实）**：上层只以相对路径引用下层；`agents/` 的工具**从不被内联进交付物**，只在 agent 构建过程中被调用（`import-design-system.mjs` 只写 `_ds/` 与 `_d_meta.json`，`system-prompt.md:32`；`record-asset.mjs` 只写 `_d_meta.json`，`record-asset.mjs:26-28`）。

## 2. 主流程时序（入口 → 结果）

以「为某 App 设计设置页，并绑定已有设计系统」为典型路径：

```
用户请求
  │
  ├─1. Agent 读到 SKILL.md（由 skill description 自动激活或 /baoyu-design）
  │     └─ 指令：先读 system-prompt.md，再按 harness 读 references/<h>.md      [SKILL.md:29-40]
  │
  ├─2. Harness 识别：看自己有哪些工具（AskUserQuestion+SendUserFile→Claude Code 等）[system-prompt.md:10-14]
  │
  ├─3. 路由：project-types.json 13 类 or 关键词分派 built-in-skills/*.md        [SKILL.md:42-60]
  │     默认（普通设计）→ hi-fi-design.md + interactive-prototype.md
  │
  ├─4. 追问（必须）：目的/视觉方向/参考物/变体数；探索已有设计系统：
  │     glob designs/*/_ds_manifest.json → multiSelect 问用哪个                [system-prompt.md:19,154-179]
  │     并问保存在哪（默认 designs/<slug>/）
  │
  ├─5. 建项目文件夹 + 设计系统绑定（若选了 DS）：
  │     node agents/import-design-system.mjs <dsDir> <projectDir> [--primary]
  │     → 拷贝 @import 闭包 CSS + url() 资产 + bundle/manifest 到 _ds/<slug>/
  │     → 生成 _ds/<slug>/_ds_prompt.md（binding 约束+接线代码+token 白名单） [import-design-system.mjs:170-200]
  │     → 合并写 _d_meta.json（designSystems[]+primaryDesignSystem）         [import-design-system.mjs:146-168]
  │     → agent 读 _ds_prompt.md 并当作「视觉契约」执行                        [use-design-system.md 步骤5]
  │
  ├─6. 产出交付物：复制 starter-components/<file>；写 HTML（React+Babel 多文件
  │     JSX 为默认工作格式，HTTP 下预览）                                      [system-prompt.md:52-97]
  │
  ├─7. 记录资产：node agents/record-asset.mjs <projectDir> "<file>"
  │     → 写入 _d_meta.json assets{name:{versions:[{path,status,…}]}}
  │     （无 DS 的项目也会被创建 _d_meta.json）                                 [record-asset.mjs 头注释]
  │
  ├─8. 预览与验证：python3 -m http.server 4311 --directory designs
  │     → http://localhost:4311/<project>/<file>.html
  │     → harness 浏览器工具截图/控制台检查（claude.md 的 vision probe 机制）
  │     → 修错重验；把「运行中的预览」当交付的一部分呈现给用户                  [system-prompt.md:181-187]
  │
  └─9. 用户审阅后：record-asset --status approved|changes-requested
        （版本迭代：新文件→新 version；同一文件→原地更新）                      [record-asset.mjs]
```

**导出支线（按需）**：
- PPTX：`gen-pptx` CLI（Playwright 渲染 → 可编辑/截图两种模式 → 原生动画树）→ `.pptx`（详见 notes/gen-pptx.md）
- 视频：`gen-video` CLI（Playwright 逐帧 seek `window.__animStage` → ffmpeg）→ `.mp4`（详见 notes/gen-video.md）
- PDF/单文件 HTML/Canva/Figma：对应 built-in-skills（`save-as-pdf.md`、`save-as-standalone-html.md` 等），主要由浏览器打印/导出能力+提示词完成。

## 3. 核心机制逐条

### 3.1 可移植性：同一套 craft 标准 × 各 harness 的薄适配
- **事实**：`system-prompt.md` 把 4 种能力抽象成「能力名」——问用户、预览/show、截图、调试/验证——具体工具查 `references/<harness>.md`；正文不出现具体工具名，除非在身体检查表里（`system-prompt.md:6-16`）。
- **事实**：移植层相对上游快照的改动 = 工具替换 + 流程本地化。例：上游 `ready_for_verification({path})` → 本地「HTTP server + harness 预览」；上游 `write_file asset:` → 本地 `record-asset.mjs`；上游「不要泄露环境技术细节」等隐私行被移除（diff `references/upstream-system-prompt.md` vs `system-prompt.md`）。
- **推断**：这层设计使「新增 harness」成本 = 写一份 md 工具映射表 + 在识别的 3-4 行里加一条，无需动方法论。

### 3.2 按需加载（渐进式披露）的主干
- **事实**：SKILL.md 只做编排，明确「built-in skill 不在上下文里就先 READ 对应文件」（`system-prompt.md:277`）。~50 个专项技能总 2696 行，分层加载避免一次性灌爆上下文。
- **事实**：`project-types.json` 是 13 类项目 → `{skills[], starters[], optionalStarters[]}` 的机器可读路由；system-prompt.md:112-127 有同构的人读表格（双通道，防漂移由 tests/project-types.test.mjs 验证——见 notes/engineering.md）。

### 3.3 设计系统：「绑定即契约」的版本化快照
- **事实**：磁盘记法用根标记文件区分：设计系统根有 `_ds_manifest.json`；普通项目根有 `_d_meta.json`，其 `_ds/<slug>/` 里才是被消费的快照（`use-design-system.md` 开头）。`glob designs/*/_ds_manifest.json` 只会命中「被 authored 的系统」。
- **事实**：导入器复用的解析器 `lib/ds-core.mjs` 是纯只读的（`import-design-system.mjs:8-10`），拷贝集合 = global CSS 的 `@import` 闭包 + 这些 CSS 里 `url()` 引用的本地资产 + bundle/manifest/adherence/README/SKILL + `assets/`（`import-design-system.mjs:100-131`）。
- **事实**：`_ds_prompt.md` 由 `lib/ds-prompt.mjs` 纯字符串装配（不读不写文件），内容含：binding+scope 声明、固定版本 React UMD 标签、全部 `<link>`、bundle `<script>`、整个指南内联、每组件 `*.prompt.md` 摘录、`.d.ts` 提取的组件 props（枚举值完整）、`var(--*)` token 白名单（`ds-prompt.mjs:1-18, 154-286`）。
- **事实**：多系统共存时 JS 靠 `window.<Namespace>` 隔离，CSS 靠 `<link>` 顺序定优（primary 最后加载）；这是 orchestrator（agent）在接线时负责的约定（`use-design-system.md` "Multiple design systems"）。

### 3.4 资产（deliverable）索引：`_d_meta.json` 的资产模型
- **事实**：模型在 `lib/asset-store.mjs`：`meta.assets[显示名].versions[] = {path, createdAt, status, subtitle?, viewport?{width,height?}, section?, chatId?}`；status ∈ needs-review|approved|changes-requested；同 path 再记录 = 原地更新，新 path = 追加新版本（`asset-store.mjs` 头注释与 `recordAssetVersion`）。
- **事实**：`record-asset.mjs` 是唯一入口脚本，支持 `--name/--inherit-from/--subtitle/--width/--height/--section/--status/--chat-id` 与 `--remove`；路径自动归一化为项目相对 POSIX；`--height` 依赖 `--width`；退出码 64/1（`record-asset.mjs` 头注释与参数解析段）。
- **推断**：该文件同时是「项目恢复」的入口——重开项目先读 `_d_meta.json`，据此恢复设计系统绑定而不重问用户（`system-prompt.md:20`）。

### 3.5 预览纪律：一切走 HTTP、`file://` 禁止
- **事实**：所有预览/截图/打开一律 `python3 -m http.server 4311 --directory designs` + `http://localhost:4311/...`；理由：多文件 Babel 原型 `<script type="text/babel" src="...">` 在 `file://` 下静默失败（`system-prompt.md:68,146,185`）。
- **事实**：单一 server 服务整个 `designs/`，所有项目复用（`.claude/launch.json` 的 `designs` 配置约定，`references/claude.md` "Showing files & preview"）。

### 3.6 直接可编辑性（作者面向的产物约束）
- **事实**：产物要「canonical HTML」：显式闭合、双引号属性、不自动闭合；文本放叶子元素；重复结构写死不用数组生成——目的是让宿主编辑器的直接编辑（点选改文案）能把修改 splice 回源码；React/脚本生成的 DOM 会破坏这条路径（`system-prompt.md:38-39`；`make-a-deck.md` 同主题长段落）。
- **推断**：这条约束与上游 Claude Design 的网络产品编辑体验对齐，是「本地产物也能被 agent 精确编辑」的工程基础。

## 4. 数据/文件流的落盘形态

```
designs/
├── my-project/                    # 一个「普通项目」
│   ├── Settings Page.html         # 交付物（可多文件 + .jsx）
│   ├── _d_meta.json               # 项目元数据：designSystems[]、assets{}、title/prompt…
│   └── _ds/
│       └── fluent2-design-system/ # 设计系统的版本化快照（含 _ds_prompt.md）
│           ├── _ds_manifest.json
│           ├── _ds_bundle.js
│           ├── _ds_prompt.md
│           └── …（CSS 闭包 + 字体/图片资产）
└── fluent2-design-system/         # 一个「设计系统」本体（authoring 产出）
    ├── _ds_manifest.json          # 根标记
    ├── ui_kits/  components/  tokens/  preview/  …
    └── preview.html               # build-preview.mjs 产出的单文件预览
```

**关键守恒**：快照自包含——项目重开后不依赖 DS 源目录也能运行；重新 import 即拉更新（`use-design-system.md` "Resuming… Refresh"）。

## 5. 与上游的关系（移植策略）

- **事实**：`references/upstream-sync/` 记录 2026-09-14 快照的溯源：源为 claude.ai/design 的 assets（`assets-proxy.anthropic.com/design/v2/...`，含 sha256），提取自一个 `claude-design-v2` 反混淆工作目录；`apply-report.json` 显示两类动作：`copy`（直接同步）与 `preserve-for-merge`（本地有改动的文件跳过覆盖，稍后手工合并）——受保护清单含 `make-a-deck.md`、`hi-fi-design.md`、`deck-stage.js`、`tweaks-panel.jsx` 等 9 个 skill + 2 个 starter。
- **推断**：本地价值增量（data-anim→PPTX、本地 gen-pptx/gen-video、便携 import/export、vision probe 等）就沉淀在「受保护文件」与 `agents/` 里；上游同步只更新未受保护的内容。

## 6. 未知 / 待深挖
- gen-pptx / gen-video / build-preview / import-figma 的内部细节 → notes/ 分模块笔记（进行中）。
- `npx skills`（Vercel）安装器如何枚举 skill 目录的精确约定 → questions.md。
- 上游提取脚本（deobfuscated bundle → md/JSON）本体不在仓库 → questions.md。
