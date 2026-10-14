# REA (rea-agents) — 项目全貌

> 分析对象：`/Users/argsno/Projects/rea`（git 分支 `main`，HEAD `b11fe63bf`，工作区干净）
> 笔记为只读分析产物，未改动目标仓库任何文件。

## 一句话定位

**事实**：REA = "Reverse Engineer Anything"，是一个 **本地运行的逆向工程工具**：以 **MCP server + CLI** 双形态暴露同一套分析工作流，让 AI agent（或终端用户）对本地目标做静态/动态逆向分析，产出带**证据（Evidence）与不确定性（Unknown）**的结论。
（出处：README.md:6-8、59-61；package.json:5）

## 受众

- **事实**：主要受众是"用 agent 做逆向/借鉴功能"的开发者与研究员（README 的 showcase：DX-Ball 声音平移重建、Notion Electron 剪贴板桥、TH04 弹幕计算）。
- **事实**：也支持纯终端用法（`npx rea-agents@latest analyze-javascript-application ...`）。
- **事实**：仓库配套一个 agent skill（`.agents/skills/reverse-engineer-anything/SKILL.md`），setup 时安装进用户的 agent。

## 技术栈

| 项 | 事实 | 出处 |
|---|---|---|
| 语言 | TypeScript（ESM，`"type": "module"`），target ES2023，`strict` + `exactOptionalPropertyTypes` + `noUncheckedIndexedAccess` | tsconfig.json |
| 运行时 | Node.js `^22.19 \|\| ^24.11 \|\| >=26`，npm@11.16.0 锁定 | package.json:44-47 |
| 构建 | `tsc -p tsconfig.build.json` → `dist/`，再 `bundle-runtime.mjs` 打包运行时；turbo 做任务缓存 | package.json:293 |
| 测试 | Vitest 5（多 project：domain/services/adapters/composition/boundary/cli-boundary/mcp-boundary/process-boundary/acceptance/conformance/evaluation…），另有 Python `unittest` 测 bridge、Stryker 变异测试 | vitest.config.ts、package.json:141 |
| Lint/格式 | oxlint + oxfmt；jscpd 查重；knip 查死代码；架构守卫脚本 `scripts/verify/architecture-guards.mjs` | package.json:129-137, 181 |
| 关键依赖 | `@modelcontextprotocol/server`、`playwright-core`、`@babel/parser`、`@electron/asar`、`@lydell/node-pty`、`evmole`、`zod`、`incur`(CLI 框架) | package.json:314-353 |
| 外部工具（BYO） | Hopper / Ghidra / IDA / JADX / Apktool / Binwalk / Unblob / TShark / WABT / pwntools / adb / JEB / Frida …（REA 不代装，除 Hopper 可选装且需批准） | README.md:160-198；AGENTS.md:24-27 |
| 原生代码 | `bridge/` 下有 Python（Hopper/mitmproxy/pwntools/pwndbg/lldb）、Java（Ghidra/JADX）、Swift（macOS NativeUI/进程令牌）、C++（`native/windows/` Node-API 插件） | package.json `files`、bridge/ 目录 |

## 规模（事实）

- `src/` TypeScript：**1852 个文件 / 413,712 行**；测试文件 **1232 个**（覆盖 src+tests）。
- 分层文件数 Top：`src/domain` 568、`src/application` 319、`src/browser` 124、`src/server` 93、`src/contracts` 87、`src/process` 72、`src/ghidra` 68。
- 仓库体积 133MB（含 .git）。

## 入口

**事实**：npm bin 是 `scripts/rea.mjs`（`rea` 与 `rea-agents` 两个名字），它是一个**分发器**（dispatcher）：
- `rea mcp` / `rea --mcp` → `dist/mcp/main.js`（MCP stdio server，源码 `src/main.ts`）
- `rea mcp doctor` → 额外加载 `dist/cli-runtime/mcpDoctor.js`
- 其他参数 → `dist/cli-runtime/cli.js`（源码 `src/cli.ts`，一次性 CLI 适配器）
- 运行时若 `dist/` 缺失会报 "compiled runtime is missing"。

（出处：package.json:40-43；scripts/rea.mjs 全文）

## 如何运行（面向使用者）

- 安装：`npx rea-agents setup`（注册 MCP + 安装 skill；支持 Claude Code/Codex/Cursor/Gemini CLI 等）或 `npm i -g rea-agents`。
- 使用：agent 侧发 MCP 调用；终端侧 `rea <command>`。
- **面向开发者**：`npm ci` → `npm run build:cached` → `npm run test:local -- PATH...`；PR 全量门禁 `npm run check:pr`（package.json:110-163；AGENTS.md:79-101）。

## 仓库结构（顶层）

- `src/` 运行时代码（分层见 architecture.md）
- `bridge/` 注入到外部工具内部运行的桥接脚本（Hopper Python、Ghidra Java、JADX、mitmproxy、LLDB tracer、Swift UI 等）
- `native/` 原生扩展（`native/windows/` Node-API）
- `tests/` 集成/边界/验收/一致性测试（acceptance、boundary、composition、conformance、evaluation、process-global、fixtures）
- `docs/` VitePress 文档站 + `docs/architecture.mermaid`（官方架构图）+ `docs/specs/`
- `.agents/skills/reverse-engineer-anything/` agent skill 源（构建产物生成到 `skills/`）
- `experiments/` 效率实验、`website/` 官网、`scripts/` 构建/校验脚本（大量 verify:*）
- 工程治理：turbo + release-please + husky/lint-staged + 架构守卫 + 生成文件校验（docs:check:built）

## 产品边界（治理性约束，事实）

- **本地运行**：分析在本地做，不上传目标（README FAQ）；"local-only" 写在 AGENTS.md 首段。
- **BYO 商业工具**：IDA/Hopper/JEB 需用户自备合法授权，REA 不提供授权（README:384-390）。
- **只读优先**：Ghidra 用临时数据库不改字节、IDA 只读不保存（AGENTS.md:21-23）；运行时捕获（process capture、browser、frida）会真实运行目标，须用户权限。
- **法律免责**：DISCLAIMER.md + README Disclaimer 段。

## 信息分类备注

- README 宣称 "180,000 GitHub stars"、"Trendshift badge" 等为**营销性事实**，未独立核实。
- 项目中大量 `verify:*` 脚本说明作者对"真实 provider 验证"有强要求（**事实**：AGENTS.md 要求区分模拟 provider 与真实 provider 的覆盖），但我本次未运行任何 verify 脚本，**未验证**这些能力在真实环境下的表现。
