# context-mode — 项目总览

> 分析对象：`~/Projects/context-mode`（git HEAD `dbec980`，版本 **1.0.169**，只读分析）
> 标注约定：**事实**（附出处） / **推断** / **未知**

## 一句话定位

**事实**：context-mode 是一个「MCP 服务器 + 多平台客户端插件」，核心目标是**节省 AI 编码代理的 context window**：把原始大数据留在沙箱/知识库里，只把「答案」放进模型上下文；并在对话压缩（compact）后自动恢复会话状态。
（来源：`package.json:5` description；`README.md:32-51`）

## 它解决什么问题（作者自述）

**事实**（`README.md:34-51`）：
1. **Context Saving**：MCP 工具调用会把原始数据（Playwright 快照 56KB、20 个 GitHub issue 59KB、访问日志 45KB）灌进上下文；沙箱工具把 315KB 压到 5.4KB（宣称 98% 降幅）。
2. **Session Continuity**：把文件编辑、git 操作、任务、错误、决策写入 SQLite；压缩后不把原始数据倒回上下文，而是索引进 FTS5，用 BM25 检索按需取回。
3. **Think in Code**：让 LLM「写代码分析」而不是「读数据」，只回传 `console.log` 的结果。
4. **不强制文风**：只管数据去哪，不管模型怎么说话。

## 受众与生态

**事实**：
- 面向使用 Claude Code / Gemini CLI / Codex / Cursor / OpenCode / VS Code Copilot 等客户端的开发者（`package.json:5`；`README.md:42` 提到 17 个受支持客户端）。
- 有商业面：`ctx_insight` 打开托管仪表盘 context-mode.com/insight（`src/server.ts:4814`）；stats.json 显示安装量 682.1k+ users / 633.6k+ npm（`stats.json`）。
- 许可证 Elastic-2.0（`package.json:7`），作者 Mert Koseoğlu（mksglu）。
- GitHub Hacker News 头条、Discord 社区（`README.md:6-7`）。

## 技术栈

**事实**：
| 项 | 内容 | 出处 |
|---|---|---|
| 语言 | TypeScript（`type: module`），源码 `src/` 41,617 行 | package.json:4；wc 统计 |
| 运行时 | Node.js >= 22.5（也支持 Bun，Linux 上优先 Bun） | package.json:123-125；start.mjs:62-67 |
| 包管理 | pnpm@10.23.0（但仓库同时有 bun.lock） | package.json:122 |
| MCP | @modelcontextprotocol/sdk ^1.26 | package.json:106 |
| 存储 | better-sqlite3 ^12（SQLite FTS5） | package.json:107 |
| 参数校验 | zod ^3.25 | package.json:111 |
| HTML→Markdown | turndown + turndown-plugin-gfm + @mixmark-io/domino | package.json:105,109,110 |
| 打包 | esbuild 把 TS 打成 .mjs bundle（server/cli/hooks） | package.json:82-85 |
| 测试 | vitest ^4（`pretest` 先 build） | package.json:93-95,120 |
| 开发运行 | tsx（`npm run dev` → `npx tsx src/server.ts`） | package.json:89 |
| 官网 | Cloudflare Workers（web/worker.js + wrangler.jsonc） | web/ 目录 |

## 如何运行

**事实**（`package.json:81-101` + README）：
- 终端用户安装：`npm install -g context-mode`；Claude Code 走 `/plugin marketplace add mksglu/context-mode`。
- MCP 宿主启动入口：`node ${CLAUDE_PLUGIN_ROOT}/start.mjs`（`.claude-plugin/plugin.json` 的 mcpServers 配置）。
- CLI 入口：bin `context-mode` → `cli.bundle.mjs`（package.json:58-60）。
- 源码开发：`npm run dev`（tsx 直跑 server.ts）、`npm run setup` / `npm run doctor`（tsx cli.ts）、`npm run build`、`npm test`。

## 入口点（顶层）

**事实**：
1. **MCP Server**：`src/server.ts` → 构建产物 `server.bundle.mjs`（stdio JSON-RPC，注册 11 个 `ctx_*` 工具；`src/server.ts:143-146,1647-4846`）
2. **Bootstrap 启动器**：`start.mjs`（616 行，普通 JS，为宿主拉起 server，含自救/修复逻辑；`.claude-plugin/plugin.json`）
3. **CLI**：`src/cli.ts` → `cli.bundle.mjs`（setup/doctor/upgrade/statusline 等子命令）
4. **Hooks**：`hooks/*.mjs`（6 类事件：PreToolUse/PostToolUse/UserPromptSubmit/PreCompact/SessionStart/Stop；`hooks/hooks.json`）
5. **各平台适配器**：`src/adapters/*`（检测平台、写各客户端配置、注册原生插件）
6. **官网/统计**：`web/`、`stats.json`

## 目录结构（顶层）

**事实**：
- `src/` — TypeScript 源码。核心：`server.ts`(4991行)、`cli.ts`(2040)、`store.ts`(2071)、`executor.ts`(785)、`security.ts`(889)、`runtime.ts`(765)、`db-base.ts`(617)、`lifecycle.ts`、`runPool.ts`、`truncate.ts`、`fetch-cache.ts`、`store-directory.ts`、`exit-classify.ts`、`types.ts`；子目录 `adapters/`(17 个客户端适配) `session/`(会话连续性~10k 行) `search/`(检索) `util/`
- `hooks/` — 各宿主的事件 hook 脚本（.mjs，部分是 esbuild 产物 *.bundle.mjs）
- `configs/` — 18 个平台的配置文件模板（antigravity、claude-code、codex、cursor、kimi、kiro、openclaw、pi、qwen-code……）
- `skills/` — context-mode 自带的 8 个 Agent Skills（ctx-stats/doctor/index/search/purge/upgrade/insight + context-mode 本体）
- `scripts/` — 安装自愈（postinstall、heal-better-sqlite3、heal-installed-plugins）、打包断言、版本同步
- `tests/` — vitest 测试（~300 个文件，按模块分目录）
- `web/` — 官网（Cloudflare Worker）与 OG 图生成
- `docs/` — 平台支持文档、adr/（4 篇架构决策记录）、adapters/
- `build/` — tsc 产物（未提交？）；`*.bundle.mjs` — esbuild 产物（已提交，供 npm 分发）
- `.claude-plugin/ .codex-plugin/ .cursor-plugin/ .openclaw-plugin/ .agents/ .pi/` — 各宿主插件清单

## 初看层面的疑点 → 见 questions.md

- git log 最近全是 "ci: update install stats" 自动提交（git log 前 8 条），真实开发历史需要看更早的提交。
- README 说 "11 languages"，但 `ctx_execute` 的 language 枚举有 12 个值（javascript, typescript, python, shell, ruby, go, rust, php, perl, r, elixir, csharp），以代码为准。
