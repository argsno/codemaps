# 模块笔记：CLI / 启动引导 / 自愈 / 构建测试（src/cli.ts, start.mjs, scripts/*）

## 1. CLI 命令（src/cli.ts 分发在 cli.ts:226-261）

**事实**：
| 命令 | 作用 |
|---|---|
| （无参数） | `import("./server.js")` 直接起 MCP stdio 服务（cli.ts:258-261） |
| `index <path>` | 文件/目录索引进 FTS5（flags: --source/--project/--max-depth/--max-files/--ext/--include/--exclude/--no-gitignore/--follow-symlinks；执行 Read deny 策略） |
| `search <query...>` | 搜当前项目知识库（--project/--source/--limit 默认 3/--type code\|prose） |
| `doctor` | 诊断：存储目录、runtime、Linux Node<22.5 SIGSEGV 闸、语言覆盖、起活服务实测、hook 配置与脚本、插件注册、陈旧 .mcp.json、FTS5 探针、本地 vs npm 版本；有 criticalFails 则 exit 1 |
| `upgrade [--platform <id>]` | marketplace git 同步 → 浅克隆 GitHub → npm install+build → 覆盖 pluginRoot → 杀兄弟 MCP 进程 → registry 升级 → 自愈 → hook 配置/权限 → 重跑 doctor |
| `hook <platform> <event>` | 分发对应 hook 脚本（HOOK_MAP 覆盖 11 平台）；未知组合 fail-open exit 0；fd2 重定向丢弃（防宿主误判） |
| `insight` | 打开 https://context-mode.com/insight |
| `statusline` | 转发给 bin/statusline.mjs（candidates: pluginRoot / marketplace clone / registry 路径） |

环境变量：`CONTEXT_MODE_DIR` 覆盖存储根（必须绝对路径，空值忽略）。

## 2. start.mjs 引导流程（616 行，MCP 宿主入口）

**事实**（顺序执行）：
1. `chdir(自身目录)`；解析 `CLAUDE_CONFIG_DIR`（支持 `~`）。
2. 设 `CLAUDE_PROJECT_DIR` + `CONTEXT_MODE_PROJECT_DIR` = 原始 cwd——**但当 cwd 命中插件缓存正则时跳过**（否则 /ctx-upgrade 重启会把会话根扎进插件目录，污染所有下游）。
3. **Linux：Bun re-exec**（找 BUN_INSTALL/~/.bun/bin/bun/...）：绕开 better-sqlite3 在 Node/V8 下的 SIGSEGV（#564）；父进程代理 stdin，转发 EOF + SIGTERM→SIGKILL 升级（#862 防孤儿）。
4. **自愈**（best-effort，全在 try/catch）：installed_plugins.json 前向修复到最新版本目录 / 悬空 installPath 反向建 symlink；enabledPlugins、plugin.json mcpServers、陈旧 .mcp.json 清扫（复用 heal-installed-plugins.mjs 单一实现）。
5. 部署全局 SessionStart 缓存自愈 hook 到 `$CLAUDE_CONFIG_DIR/hooks/context-mode-cache-heal.mjs` 并注册（内容变化才重写）。
6. `normalizeHooksOnStartup`（Windows 占位符→绝对路径、可选 Bun runtime 换名；VITEST 下跳过）。
7. `ensure-deps.mjs` 自动跑：缺 better-sqlite3 则安装；否则 `healBetterSqlite3Binding`（ABI 缓存 `better_sqlite3.abi<N>.node` 换名、Node≥22.5 免子进程探测、npm rebuild 兜底、macOS codesign）。
8. **后台 detached npm install**（turndown/turndown-plugin-gfm/@mixmark-io/domino 缺失时）——codex 式安装没有 node_modules 也能跑；Windows 优先 `node npm-cli.js`（#861）。
9. 缺 `cli.bundle.mjs` 时写一个转发到 build/cli.js 的 shim。
10. `healPartialInstallFromMarketplace`（从 marketplace 克隆补缺文件）+ **完整度闸** `assertPluginCacheIntegrity`——失败写 `CONTEXT_MODE_PARTIAL_INSTALL` 错误并 **exit 2**。
11. 启动：有 `server.bundle.mjs` 就 import；否则 dev 路径（无 node_modules 先 npm install，无 build/server.js 先 tsc）。

**事实**：`src/lifecycle.ts` 此外提供运行期守护：30s 轮询父/祖父进程存活（bridge 子进程 1s）、stdin EOF 作为探测触发器、SIGTERM/SIGINT/SIGHUP、孤立 bridge 子进程空闲自杀（#854）。

## 3. 安装自愈体系（scripts/*）

**事实**：
- `postinstall.mjs`：Linux+Node<22.5+无 Bun **直接硬失败**（SIGSEGV #564）；全局安装启发式（`npm_config_global` 且无 .git）下执行插件/heal 系列；nvm4w Windows junction 修补 + .cmd shim 重写；registry 陈旧 installPath 反向 symlink。
- `heal-better-sqlite3.mjs`：分层修复——(1) 缺包（npm optionalDep 静默跳过 #514）→ `npm install better-sqlite3 --no-optional --no-save`；(2) A 层用 `process.execPath` spawn prebuild-install（绕 PATH/MSVC）；(3) B 层 `npm install`（**故意不用 npm rebuild**，避免 node-gyp 掉坑）；(4) C 层给出可执行的报错建议。Conda 防御（#533）：钉住非 conda python、剥 `CONDA_*`；Windows VS 版本经 vswhere 探测并注入 `npm_config_msvs_version`。
- `heal-installed-plugins.mjs`：HEAL 3（条目版本 ← 缓存 plugin.json）、HEAL 4（enabledPlugins）、`healPluginJsonMcpServers`（占位符 args）、`healMcpJsonArgs`、`healClaudeJsonMcpArgs`（~/.claude.json）、`sweepStaleMcpJson`（只删 .mcp.json，控制在 cacheRoot 内）——start.mjs / postinstall / cli upgrade 三处共用。
- `heal-partial-install.mjs`：`isPartialInstall` 探 start.mjs/package.json/cli.bundle/server.bundle；从 marketplace 补缺文件（O_EXCL/符号链接/realpath 防护，原子重写 args[0]）。
- `plugin-cache-integrity.mjs`：完整性清单 = `package.json files[]` ∪ 从 `scripts.bundle --outfile` 解析出的 6 个 bundle ∪ 冻结旧清单；豁免 soft-fallback bundle。

## 4. 构建与打包

**事实**：
- `tsc`（ES2022/NodeNext，declaration:true）→ `build/`；npm build 顺带 POSIX chmod `build/cli.js`。
- esbuild 产 6 个 bundle：`server.bundle.mjs`（external: better-sqlite3, turndown, turndown-plugin-gfm, @mixmark-io/domino）、`cli.bundle.mjs`（external: better-sqlite3）、`hooks/session-extract|snapshot|db.bundle.mjs`、`hooks/security.bundle.mjs`（`--platform=node --target=node18 --format=esm --minify`）。
- 为什么 external：better-sqlite3 是 ABI 相关的原生插件必须外部解析；turndown/domino 由沙箱子进程运行时 `require.resolve`。
- `assert-bundle.mjs`：禁止模式扫描（esbuild 的 `__require` 垫片：`Dynamic require of`/`__require('node:`/裸 `require('node:`，#511），命中 exit 1。
- `assert-asymmetric-drift.mjs`：断言 `.mcp.json.example` 与 `.claude-plugin/plugin.json` 的 mcpServers.args[0] 都钉 `${CLAUDE_PLUGIN_ROOT}/start.mjs`、skills 指向 "./skills/"、关键文件存在——防两个入口漂移。
- `bundle.yml`：push main 时自动重建并将 6 个 bundle 强制提交（所以仓库里有编译产物）。

## 5. 测试与 CI

**事实**：
- vitest.config.ts：`tests/**/*.test.ts`，testTimeout/hookTimeout 30s，`pool:"forks"`（better-sqlite3 segfault 安全），CI maxWorkers=2、retry=2。
- 目录：tests/{core,adapters,hooks,session,util,cli,security,integration,scripts,analytics,executor,plugins,codex,fixtures}；另有基准脚本 benchmark.ts / use-cases.ts / context-comparison.ts / ecosystem-benchmark.ts / live-benchmark*.ts。
- CI（ci.yml）：三平台矩阵（ubuntu/macos/windows）、Node 22.5、额外装 Python 3.12/Go/Elixir；步骤 `tsc -b --noEmit` → `npm run build`（含两个断言）→ `npm run bundle` → assert-bundle → `vitest run`（容忍 worker teardown 崩溃的 grep）→ `tsx src/cli.ts doctor`（continue-on-error）。
- 其它 workflow：bundle.yml、tier2-e2e-smoke.yml（仅手动，真实 LLM 冒烟，目前只实现了 Pi）、openclaw-e2e.yml、update-stats.yml（6h 一次更新 stats.json）。

## 6. 版本管理

**事实**：`package.json.version` 是唯一来源（1.0.169）；`scripts/version-sync.mjs` 同步 10 个文件（.claude-plugin/plugin.json、marketplace.json、.cursor-plugin、.codex-plugin、.openclaw-plugin×2、openclaw.plugin.json、.pi/extensions/context-mode/package.json、configs/antigravity-cli/plugin.json、configs/copilot-cli/.github/plugin/plugin.json），任何读写失败即 exit 1（#768 不静默跳过）；npm 的 `version` 钩子 `git add` 精确这 10 个文件。
