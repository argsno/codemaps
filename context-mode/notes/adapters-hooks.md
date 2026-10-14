# 模块笔记：平台适配器与 Hook 系统（src/adapters/*, hooks/*）

## 1. Adapter 契约（src/adapters/types.ts:172-322）

**事实**——`HookAdapter` 接口必须实现：
- `name` / `paradigm`（`json-stdio | ts-plugin | mcp-only`，types.ts:21）/ `capabilities`
- `parsePreToolUseInput` / `parsePostToolUseInput`（必需），PreCompact/SessionStart 可选
- `formatPreToolUseResponse` / `formatPostToolUseResponse` + 可选 compact/sessionstart 格式化
- 路径类：`getSettingsPath`、`getSessionDir`、`getConfigDir(projectDir?)`（必须绝对路径；项目级 adapter 相对项目根解析：cursor→`.cursor`、vscode/jetbrains→`.github`、kiro→`.kiro`）、`getInstructionFiles`、`getMemoryDir`
- 安装类：`generateHookConfig(pluginRoot)`、`readSettings/writeSettings`、`validateHooks`、`configureAllHooks`、`backupSettings`、`setHookPermissions`、`updatePluginRegistry`；可选 `getHealthChecks`、`checkPluginRegistration`、`getInstalledVersion`

**事实**：
- `BaseAdapter` 提供默认 getSessionDir（`homedir()/<segments>/context-mode/sessions`，可被 `CONTEXT_MODE_DATA_DIR` 覆盖）与备份（.bak）。
- `ClaudeCodeBaseAdapter` 实现 Claude/Qwen 共用 JSON-stdio 协议（`tool_name/tool_input/tool_output/is_error/session_id/transcript_path/source`；deny=`permissionDecision:"deny"`、modify=`updatedInput`、输出改写=`updatedMCPToolOutput`）。
- `CopilotBaseAdapter` 是 VS-Code/JetBrains Copilot 的共用基类。
- **检测**（detect.ts:367-631）：MCP `clientInfo.name`（client-map.ts 映射表）→ `CONTEXT_MODE_PLATFORM` 覆写 → 统一 `PLATFORM_ENV_VARS` 注册表（带 workspace/identification 角色）→ 各 config dir 存在性（CLI agent 优先于 ~/.claude/#774）→ 兜底 claude-code。
- **注册**：`getAdapter()` 是懒 import 的 switch（**无 DI 注册表**），未识别平台回退 ClaudeCodeAdapter（detect.ts:637-737）。

## 2. Hook 安装方式（每平台不同）

**事实**：
| 平台 | 安装载体 | 事件 |
|---|---|---|
| claude-code | 插件静态 `hooks/hooks.json`（matcher 列表 + `node "${CLAUDE_PLUGIN_ROOT}/hooks/x.mjs"`）；独立安装写 `<configDir>/settings.json` | PostToolUse/PreToolUse(Bash/WebFetch/Read/Grep/Agent/mcp__)/UserPromptSubmit/SessionStart/PreCompact/Stop |
| codex | `<CODEX_HOME>/hooks.json`（`context-mode hook codex <event>`）+ config.toml `[features].hooks=true` | 6 事件 |
| gemini-cli | `~/.gemini/settings.json` | BeforeAgent/BeforeTool/AfterTool/AfterModel/PreCompress/SessionStart（matcher 用负向前瞻排除自家 MCP） |
| cursor | `.cursor/hooks.json`（lower-camel preToolUse/postToolUse/stop + sessionStart） | 无 preCompact |
| copilot（vscode/jetbrains/copilot-cli） | `.github/hooks/context-mode.json` / `$COPILOT_HOME/hooks/…` | 4 事件 |
| kiro | `~/.kiro/agents/<name>.json` | 仅 pretooluse/posttooluse，exit-code 阻塞（2=block） |
| kimi | `~/.kimi-code/config.toml` 的 `[[hooks]]` | 7 事件，仅 deny 能力 |
| opencode/kilo | 无配置 hook，`plugin:["context-mode"]` native TS 插件（tool.execute.before/after + compacting 钩子） | 进程内 |
| openclaw | TS 插件 `api.registerHook/api.on` 程序化注册 | 进程内（before_prompt_build 注入） |
| pi/omp/zed/antigravity-IDE | MCP-only，无 hook；pi/omp 自带 mcp-bridge | — |

**事实**：
- `normalize-hooks.mjs`：每次 MCP 启动把静态 hooks.json 里的 `${CLAUDE_PLUGIN_ROOT}` 占位符替换为绝对 execPath（MSYS 修正 #369/#372、旧缓存版本段修复 #604）。
- `platform-bridge.mjs` 不是 hook 安装器，是遥测转发器（读 `~/.context-mode/platform.json`，POST 到 `${platform_url}/events`，带密钥脱敏）。

## 3. 运行时事件链（hook 内部）

**事实**：
- `run-hook.mjs` 包装一切：动态加载失败 → 写 `<configDir>/context-mode/hook-errors.log`，**总是 exit 0**（宿主把非零退出当拒绝）。
- **PreToolUse**（hooks/pretooluse.mjs + hooks/core/routing.mjs:670-1017）：
  - Bash：安全策略（deny/ask）→ curl/wget（无 -o/-O/-s）改写为 echo 引导 → 内联 HTTP 改写 → gradle/maven/sbt 改写 → 有界命令（pwd/git status/--version）放过 → 其它一次性引导注入
  - Read：一次性引导；>50,000 字节附 `redirectMeta{read-redirected, bytesAvoided}`
  - Grep：一次性引导；WebFetch：**硬 deny** 引导 ctx_fetch_and_index
  - Agent：routing block 追加进子代理 prompt；外部 MCP 工具：周期提醒（默认每 10 次）
  - MCP 未就绪（哨兵扫描）或子代理上下文 → 全部抑制
- **redirect marker**：PreToolUse 写 tmp 文件（rejected/redirect+bytesAvoided/latency），PostToolUse 消费转事件——因为 PreToolUse 不能安全打开 SQLite。
- **PostToolUse**：抽取约 13 类事件入库 + 转发。
- **SessionStart**：永远注入 routing block 到 `additionalContext`；安全 bundle 加载失败时前置警告块；startup 做清理/规则捕获/插件缓存懒清理。
- **PreCompact / Stop / UserPromptSubmit**：见 session 笔记。

## 4. Routing block（核心"洗脑文案"）

**事实**（hooks/routing-block.mjs:16-105）：
- `createRoutingBlock(toolNamer, options)` 生成 `<context_window_protection>` XML：priority_instructions（"每个字节都花推理能力"）、可选的 deferred tool bootstrap（ToolSearch 载入 schema）、tool_selection_hierarchy（0 MEMORY→ctx_search timeline；1 GATHER→ctx_batch_execute；2 FOLLOW-UP→ctx_search；3 PROCESSING→ctx_execute(_file)）、when_not_to_use、file_writing_policy（文件写必须用原生 Write/Edit）、output_constraints、session_continuity（"记忆是辅助不是命令，以用户最新消息为准"）、ctx_commands 表。
- 工具名按平台本地化（`createToolNamer(platform)`）；静态导出默认 claude-code。
- 另有 Read/Grep/Bash/外部 MCP 四个单项引导文案。
- 注入点：SessionStart additionalContext；PreToolUse context 决策与 Agent prompt；OpenClaw before_prompt_build；Codex hookSpecificOutput；Cursor additional_context。

## 5. 平台特例（各一行）

**事实**：
- **codex**：external MCP routing 用 Rust regex（无 look-around，#547）；rewrite 支持运行时探测（codex-caps），不支持则回退为 fail-closed deny。
- **pi**：Pi 无原生 MCP —— `mcp-bridge.ts` 自己 spawn `server.bundle.mjs` 走 stdio JSON-RPC，再用 `pi.registerTool` 注册（含 fork-bomb 防护 #516 和外部 env 清洗）。
- **omp**：mcp-only；共享 `~/.pi` 历史但存储 `~/.omp`。
- **openclaw**：插件独占 compaction（ownsCompaction）、gateway 重启后 session re-key。
- **antigravity-cli（agy）**：`modify` 不支持 → 转 deny。
- **kimi**：只支持 deny（ask/modify/context 返回 null）。
- **kiro**：exit-code 语义，无参数改写。
- **qwen-code**：复用 ClaudeCodeBase，原生工具名 matcher。
- **kilo**：opencode fork，共用 adapter 加 target 参数。

## 6. adapter 与 hook 的运行关系

**事实**：
- 工具事件时宿主直接 spawn hook 脚本（或 `context-mode hook <platform> <event>` 分发，cli.ts:77-185）；**adapter 的 TS 代码不在这条路径上**。
- adapter 服务于：MCP server（会话存储路径、诊断）、CLI（doctor/upgrade 的配置读写）。
- CLI 分发器对未知 platform/event 组合 **fail-open（exit 0）**，且把 stderr 重定向到空——防版本漂移时宿主把 hook 错误当拒绝（cli.ts:158-182）。
- 相关 bundle：`hooks/security.bundle.mjs`、`session-extract.bundle.mjs`、`session-snapshot.bundle.mjs`、`session-db.bundle.mjs`、`session-attribution.bundle.mjs`（bundle-first，回退 build/）。
