# baoyu-design 项目速览

> 证据出处格式：`文件:行` 或命令。标记体系：**事实**（有出处）/ **推断**（由事实推导）/ **未知**（没弄清）。
> 调研时间：2026-10-14，目标仓库 HEAD = `6530033 chore: release v1.3.0`（`git log --oneline`），版本 1.3.0。

## 一句话定位

**事实**：baoyu-design 是把 Anthropic 的 Claude Design（[claude.ai/design](https://claude.ai/design) 背后的设计引擎）重新打包成「可移植 Agent Skill」的项目，让本地编码 Agent（Claude Code / Cursor / Codex 及任意可写文件的 agent）在编辑器里直接产出**自我包含的 HTML 设计产物**——UI mockup、交互原型、线框、落地页、仪表盘、移动 App、幻灯片、文档/简历、动画、3D、邮件等（`README.md` 开头段；`skills/baoyu-design/SKILL.md:1-21`）。

- **事实**：MIT 许可（`LICENSE`），维护者 Jim Liu 宝玉，明言"独立社区项目、与 Anthropic 无隶属关系"（`README.md` "Credits & license" 段）。
- **事实**：仓库里没有应用源代码；本体 = 提示词（Markdown）+ 无构建的 JS/JSX 脚手架 + 本地 Node 工具链（`README.md` "How it works"：*no build step, no runtime*）。

## 受众

- 想在本地 Agent 里获得 Claude Design 能力的个人/团队；希望交付物是 HTML 而非截图的设计师/产品/工程师。
- **事实**：README 建议配 Claude Opus 4.8 使用；支持的 harness 列表里 Cursor / Claude Code / Codex 有一等公民待遇（专属工具映射文档），其余走通用流程。

## 技术栈（按层次）

| 层 | 技术 | 事实出处 |
|---|---|---|
| 技能本体 | 纯 Markdown 提示词（SKILL.md、system-prompt.md、built-in-skills/*.md） | `README.md` How it works |
| 脚手架 | 原生 JS / JSX（React+Babel 浏览器内转译，无构建） | `system-prompt.md:52-97` |
| 本地工具链 | Node.js ESM（`.mjs`）+ TypeScript CLI ×2（gen-pptx、gen-video） | `skills/baoyu-design/agents/` 目录 |
| 浏览器自动化 | Playwright + headless Chromium（gen-pptx、gen-video） | `README.md` gen_pptx 段；`references/claude.md` |
| PPTX 生成 | PptxGenJS（vendored 进仓库） | `agents/gen-pptx/vendor/pptxgenjs/` |
| 视频编码 | ffmpeg（外部命令） | `references/claude.md` "Exporting to video" |
| 测试 | `node --test`（Node 内置测试器） | `package.json` scripts.test |
| CI | GitHub Actions（Node 22，`npm ci`+`npm test`） | `.github/workflows/tests.yml` |
| 上游同步 | 从 claude.ai/design 反混淆 bundle 中提取 → 按清单同步 | `references/upstream-sync/provenance.json`、`apply-report.json` |

## 如何运行

**作为用户**（把 skill 装进自己的 agent）：

```bash
npx skills add JimLiu/baoyu-design        # 安装（自动探测 agent；-g 全局）
npx skills update baoyu-design            # 更新
```

在 agent 里直接描述设计需求即自动激活；Claude Code 可 `/baoyu-design` 显式触发（`README.md` "Quick start"）。

**预览**：交付物必须走 HTTP（多文件 JSX 原型在 `file://` 下静默失败）：

```bash
python3 -m http.server 4311 --directory designs
```

**本仓库的验证/开发**：

```bash
npm test                                   # node --test agents/tests/*.test.mjs
cd skills/baoyu-design/agents/gen-pptx && npm install && npx playwright install chromium && npm run build
cd skills/baoyu-design/agents/gen-video && npm install && npx playwright install chromium && npm run build
```

（gen-pptx/gen-video 为按需一次性构建；测试脚本见 `package.json:6`。）

## 目录结构与入口

```
baoyu-design/
├── README.md / README.zh-CN.md / CHANGELOG.md / CHANGELOG.zh.md
├── package.json                 # v1.3.0，private，test 脚本
├── assets/                      # README 截图 + PPTX 管线图（svg）
├── .github/workflows/tests.yml  # CI
├── .claude/skills/release-skills/SKILL.md   # 发布流程 skill（用于本仓库发版）
└── skills/baoyu-design/         # ★ 技能本体（会被安装到用户机器）
    ├── SKILL.md                 # 入口：编排整个流程（61 行）
    ├── system-prompt.md         # 设计方法论 & craft 标准（唯一事实来源，323 行）
    ├── project-types.json       # 13 类项目 → skill + starter 路由表
    ├── references/              # claude.md / cursor.md / codex.md（harness 工具映射）
    │   └── upstream-sync/       # 上游快照溯源（provenance/apply-report/原始 JSON）
    ├── built-in-skills/         # ~50 个按需加载的专项提示词（共 2696 行）
    ├── starter-components/      # 27 个脚手架（共 ~21481 行）：deck-stage、design-canvas、设备/社交外壳、动画引擎、水彩…
    └── agents/                  # 本地工具链（把云端工具移植到本地）
        ├── gen-pptx/            # TS CLI：Playwright 渲染 → PptxGenJS 写 PPTX
        ├── gen-video/           # TS CLI：Playwright 逐帧 → ffmpeg 编码
        ├── import-figma.mjs     # 离线解析 .fig（含 vendor/fig-materialize.mjs）
        ├── import-design-system.mjs / compile-design-system.mjs / check-design-system.mjs
        ├── build-preview.mjs    # 设计系统 → 单文件 preview.html（1919 行，最大单文件）
        ├── record-asset.mjs     # 记录交付物资产到 _d_meta.json
        ├── lib/                 # ds-core / ds-prompt / asset-store 共享库
        ├── tests/               # node --test 测试 + fixtures
        └── vendor/              # React/Babel/fflate/fig-materialize 离线依赖
```

**调用链入口**（`SKILL.md` 五步）：读 `system-prompt.md` → 按 harness 读 `references/<harness>.md` → 按任务读 `built-in-skills/*.md` → 用 Ask-Question 工具问清需求 → 建 `designs/<project>/` 并产出/预览/验证。

## 事实 / 推断 / 未知 速记

- **事实**：目录结构、行数、版本、依赖、脚本、CI 均由命令与文件内容直接证实（见 sources.md）。
- **推断**：`skills/baoyu-design/` 设计成「自包含可整体拷贝」的安装单元——SKILL.md 内部引用全部用相对路径，上游 `references/` 快照只做溯源不影响运行。
- **未知（待查）**：
  - 上游提取脚本（把 deobfuscated bundle 拆成 md 的那套）不在本仓库，具体提取流程只能从 `apply-report.json` 反推。
  - `npx skills`（Vercel Labs）如何识别并安装本仓库的具体约定（README 描述为读 `skills/baoyu-design/`）。
  - gen-pptx / gen-video 的完整内部实现细节（下一步深挖）。
