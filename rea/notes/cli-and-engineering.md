# 模块笔记：CLI、Agent 集成与工程化流程

> 基于并行探索代理调研；抽查通过。

## 1. CLI 命令面（事实）
- `src/cli.ts:87-148` 串 37 个 `register*Commands`；命令名集中在 `src/cliCommandNames.ts`（`CLI_COMMANDS` 冻结对象 + 别名表 + `CLI_COMMAND_TOOL_ALIASES` + `MCP_TOOLS_WITHOUT_DEDICATED_CLI`）。
- 分组：维护（setup/doctor/uninstall/update）、核心分析（analyze/decompile/function/instructions/search/xrefs/trace）、逆向工具（reverseEngineering/cutter/binaryDiagnostics/analysisView）、平台 provider（android/apktool/adb/jeb/firmware/flutter/pcap/wasm/evm/go/rust/sqlite）、运行时观测（browser/web*/electron/javascript*/frida/process/capture/signal）、产物与治理（artifact/managed/evidence/application）。
- 绑定示例：`analyze`（`cli/coreBinaryCommands.ts:66-90`）内部分流到 JavaScript 分析或 `runDirectAnalysis`（`createDirectAnalysis` 在 cli.ts:88 注入），统一套 `withCommandCancellation` + `logCliCommand`；`setup`（`cli/setupCommands.ts:232`）调 `application/Setup.ts` 的 `runSetup`，doctor/uninstall/update 懒加载各自应用服务。

## 2. rea setup（事实）
- **25 个客户端定义**（`SupportedClients.ts` `SUPPORTED_CLIENT_DEFINITIONS`）：claude_code、claude_desktop、codex、cursor、gemini_cli、windsurf、devin、opencode、antigravity、copilot_cli、commandcode、qwen_code、kimi、vscode、trae、workbuddy、grok_build、omp、pi、hermes、qoder、droid、zcode、dsh、grok_bot；其中 grok_bot 标 `format:"unsupported"` 只给手工指引。
- **三段式**：`discoverSetupState`（只读，判定注册是否 REA-owned）→ `planSetupActions`（逐客户端 inspect；invalid 是 blocker；产出含 backupPath/networkOrigins/integrity 的动作）→ 交互审批（`confirmInteractiveSetup` 两阶段：多选客户端 + 单独确认装 skill/Hopper，再 review 路径与命令）。`--dry-run` 只返回计划；无 TTY 且无 `--yes` → `needs_confirmation`。
- **写入纪律**：备份 `<configPath>.rea.backup` → 复读校验 → `writeFileAtomic`，失败回滚且保留 backupPath（`SetupClientConfiguration.ts:219-239,275,323`）；uninstall 同机制。
- **skill 安装**：源 `.agents/skills/reverse-engineer-anything`；运行期从包内 `skills/<name>/` 读；只管理 7 个文件（`SetupSkill.ts:22-30`：SKILL.md + references/{connection-and-recovery,native-and-artifacts,javascript-applications,android-applications,runtime-observation,evidence-workflows}.md）；事务性：备份→原子写→回读→逆序回滚（:372+）；归属判定看 SKILL.md 头部 `name: reverse-engineer-anything`。
- 约束（AGENTS.md:24-30）：安装器**不得**安装/升级无关软件（含 Homebrew/Node/npm/Java/Ghidra）；Ghidra/IDA 自带；Hopper 可选装但需批准。

## 3. 构建与产物（事实）
- `npm run build:cached` = 依赖检查 → `run-exclusive(artifacts)` → `turbo run artifacts:skills`；`artifacts:skills` dependsOn `compile`。
- `compile` = `clean-build-output` → `tsc -p tsconfig.build.json`（rootDir=src，排除 tests/fixture）→ `bundle-runtime.mjs`（rolldown，external 全部依赖、treeshake:false，重写 `import.meta.url` 保留模块位置供 bridge/createRequire 用），产出 `dist/mcp/main.js` 与 `dist/cli-runtime/{cli,cliOutput,cliResultOutput,mcpDoctor}.js`。
- 生成物与归属：`src/generatedPackageMetadata.ts`、`skills/**`、`docs/public/product-catalog.json`、`docs/error-contract.schema.json`、`docs/verification/managed-conformance-*.json`、`.cache/mcp-tool-catalog.json`（均 gitignore；头注 do-not-edit）。
- 校验：`ensureGeneratedFile`（比较/写入；`--check` 不一致即失败，提示 `npm run docs:generate`）；skill 生成禁止 authored 源写 `tool_count/catalog_digest`。

## 4. 测试组织（事实）
- source 项目：`domain`（src/{contracts,domain} 纯逻辑，threads、isolate:false）、`services`（src/application）、`adapters`（provider 目录与 src 内联测试，forks）、`composition`、`conformance`、`evaluation`。
- built 项目：`boundary`、`cli-boundary`（限 384MB 老年代）、`process-boundary`（串行）、`mcp-boundary`、`acceptance`、`process-global`。
- `projectForTest` 强制每个测试文件唯一归属（防止"哪都能跑"的漂移）。
- 静态检查防护面（事实 + 推断）：
  - oxlint（复杂度 20/嵌套深度 4/文件 500 行/函数 100 行）
  - oxfmt `--check`
  - jscpd（重复率，报告落 `.cache/static-checks`）
  - knip（未用依赖/导出；dependsOn compile）
  - `scan:todos`（TODO/FIXME/HACK 出现即 exit 1）
  - `verify:architecture-guards`（domain 禁 localeCompare、禁 NUL 复合键、禁自研 AST walker、prlimit 参数集中——已抽查确认）
  - `verify:module-boundaries`（解析真实 import 目标守边界）

## 5. 治理机制（事实）
- `docs:check` = `docs:check:built`（metadata / skill metadata / product-catalog / completion-ledger / check-doc-facts / error-schema / licenses）+ `readmeConsistency` 测试；`check:pr` 再叠加 jscpd + scan:todos。
- 同类还有 `mcp-catalog:check`、`mcp-registry:check`、`evidence:check`。
- 意图（事实+推断）：把文档、工具目录、许可证清单、证据账本都当**生成物**，PR 期验证与唯一真源一致，禁止手改漂移；`run-exclusive` 串行化生成避免并发写。

## 6. 关于工作流治理本身（事实，值得注意）
- AGENTS.md 第一条就写明：AGENTS/CONTRIBUTING 是**研究对象**（这是我自己笔记的规则）；项目自己的 AGENTS.md 第 12-30 行则规定维护者必须"先取得范围决定再集成"、变更必须 fail-closed——**推断**：作者把"AI 参与开发"的纪律也写进了仓库，说明这个项目本身高度 agent-native。
- `.github/` + husky + release-please + lint-staged：发布与提交纪律（未细读，标为未知）。 
