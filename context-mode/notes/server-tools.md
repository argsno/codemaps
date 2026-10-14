# 模块笔记：MCP Server 与 11 个工具（src/server.ts）

> 事实来源以 `文件:行` 标注；无标注的推断写「推断」。

## 1. 工具清单（全部在 server.ts 内注册）

| 工具 | 注册位置 | 作用 | 关键点 |
|---|---|---|---|
| `ctx_execute` | server.ts:1647-1952 | 沙箱跑 12 种语言代码，仅 stdout 入上下文 | 有 `intent` 时 >5KB 走意图检索；>100KB 自动索引返回指针 |
| `ctx_execute_file` | server.ts:2042-2230 | 把文件读进沙箱 `FILE_CONTENT` 再跑代码 | **项目边界围栏**（#852）+ Read deny 策略 + 代码内嵌命令扫描 |
| `ctx_index` | server.ts:2236-2438 | 把文本/文件/目录索引进 FTS5 | 目录模式有 maxDepth=5/maxFiles=200 上限；逐文件走 deny 检查；根符号链接重定向再查 |
| `ctx_search` | server.ts:2553-2810 | 多策略检索（见 store 笔记） | FloodGuard 渐进限流；relevance/timeline 双模式；40KB 输出上限 |
| `ctx_fetch_and_index` | server.ts:3412-3676 | 抓 URL→HTML 转 Markdown→索引 | SSRF 防护 + 24h TTL 缓存 + 并发批量（`requests[]`/`concurrency`） |
| `ctx_batch_execute` | server.ts:3678-3930 | 一次跑多条命令，自动索引，可带 queries 同回合返回 | 并发 1-8（runPool）；query_scope: batch/global |
| `ctx_stats` | server.ts:3936-4138 | 省流/货币统计报告 | 走 analytics 引擎 `queryAll`+`formatReport` |
| `ctx_doctor` | server.ts:4139-4269 | 诊断 | 返回给宿主执行的 shell 命令（`context-mode doctor`） |
| `ctx_upgrade` | server.ts:4270-4444 | 升级 | 返回 shell 命令交给宿主执行 |
| `ctx_purge` | server.ts:4445-4620 | 删除索引/会话数据 | 必须 `confirm:true`；新增 sessionId/scope 精确删除但不破坏项目文件（server.ts:4432-4438） |
| `ctx_insight` | server.ts:4816-4846 | 打开商业仪表盘 | `https://context-mode.com/insight`（server.ts:4814） |

工具描述风格（WHEN / WHEN NOT / RETURNS / EXAMPLE）是 **ADR-0002** 规定的（docs/adr/0002-tool-description-style.md）。

## 2. 共享处理器模式（所有工具一致）

- **注册包装**：monkey-patch `McpServer.registerTool`，注册进 `REGISTERED_CTX_TOOLS`，handler 包 `wrapToolHandler`（in-flight 标记 + 存储错误的降级返回）（server.ts:277-320）。
- **参数健壮性**：由于某些客户端会传字符串化的 JSON/布尔，统一用 `z.preprocess`（`coerceBoolean`/`coerceJsonArray`/`coerceCommandsArray`，server.ts:2503-2543）；布尔禁用 `z.coerce.boolean()`（`Boolean("false")===true` 陷阱，server.ts:2523-2526）。
- **schema 兼容**：`installStrictClientSchemaCompat` 把发出去的 `tools/list` schema 中 `const`→`enum`、删 `additionalProperties`，救 Gemini 严格函数调用（server.ts:353-413）。
- **输出回显**：execute 类工具把「执行的代码/路径」回显在输出最前（`buildExecuteEcho`，server.ts:1523-1527），供用户审计与宿主拦截。
- **错误分类**：非零退出码经 `exit-classify.ts` 判断是否算错误（如 grep 无匹配不算）。
- **统计**：`trackResponse` 每次累加 calls/bytes，节流 500ms 写 sidecar JSON；`trackIndexed` 另计 bytesIndexed（server.ts:885-966）。

## 3. 输出分级（"省 context"三档）

**事实**（server.ts:1979-2036, 1958-1973）：
- `INTENT_SEARCH_THRESHOLD = 5000` 字节：与 intent 结合 → 索引 + 返回「章节标题+首行预览+可搜索词」。
- `LARGE_OUTPUT_THRESHOLD = 102400`：无论有无 intent → 自动索引，返回指针文案。
- 错误输出同样适用（error 时 intent 缺省用 `"errors failures exceptions"`，server.ts:1893-1901）。

## 4. 大批量执行（ctx_batch_execute / runBatchCommands）

**事实**（server.ts:1560-1641；runPool.ts）：
- `concurrency<=1`：串行 + **共享时间预算**，超时后后续命令级联 skip，并标注 `(skipped — batch timeout exceeded)`。
- `concurrency>1`：`runPool`（保序、隔离抛错），每条命令独立超时，超时标注 `(timed out after Nms)`。
- Node 子进程的 FS 读取统计靠**内联 env 前缀**注入 `NODE_OPTIONS=--require <preload>`（因为 executor 的 env denylist 禁止 NODE_OPTIONS 透传），preload 在进程退出时把 `__CM_FS__:<bytes>` 写 stderr，服务端剥离并计数（server.ts:420-435, 1529-1545）。

## 5. 抓网页（ctx_fetch_and_index）

**事实**（server.ts:2838-3410, 3288-3410）：
- 抓取动作在**沙箱子进程**里执行：`buildFetchCode` 生成一段 JS——覆盖 `dns.lookup`/`dns.resolve` 做 SSRF 拦截、手写最多 5 次重定向、逐跳重新校验、50MB 上限、用 turndown+GFM 把 HTML 转 Markdown 写到临时文件。
- `classifyIp` 把 IP 分 block/private/public 三档（server.ts:3244-3286）；`ssrfGuard` 在服务器侧先做一遍 URL 校验（3168-3242）。
- 抓取结果索引进 FTS5（source 前缀 `fetch:`），**缓存即 source 行**：key=`${source}::${url}`，TTL 24h（`FETCH_TTL_MS`，3118），过期静默重取；命中只回提示+估算字节数（3295-3311）。
- 预览用 `charSafePrefix` 截 3072 字符（3119）。
- turndown/domino 是运行时 `require.resolve` 的相对路径依赖（2815-2836），start.mjs 会在缺失时后台 npm install（bootstrap 笔记）。

## 6. 防刷（ctx_search 限流）

**事实**（server.ts:2450-2485, 2654-2679；search/flood-guard.ts:78-110）：
- `FloodGuard`：60s 滚动窗口，按 **agent 会话键**（`currentAttribution().sessionId`）分别计数（#769 修复子代理共享预算问题）。
- 第 >3 次：每查询结果数降到 1（正常上限 min(limit,2)）；第 >8 次：直接拒绝并命令改用 `ctx_batch_execute`。
- 阈值可用 env 覆写：`CONTEXT_MODE_SEARCH_WINDOW_MS` / `..._MAX_RESULTS_AFTER` / `..._BLOCK_AFTER`。
- key 表上限 4096，超限淘汰最旧窗口（fail-open 不误封）。

## 7. 会话归属（attribution）

**事实**（server.ts:448-530）：
- MCP 进程拿不到 `CLAUDE_SESSION_ID`（只有 hook 有），所以从本项目 SessionDB 读「最近一条 session_id」（带缓存）；跨全部 adapter 通用。
- `AsyncLocalStorage` 支持测试/嵌入场景 override projectDir+sessionId（336-342）。
- 所有索引写入携带 `{sessionId}`，用于 ctx_search 的 per-project 过滤（#737，2 步 IN 查询）。

## 8. 兼容与自抑（踩坑史集中区）

- **OpenCode/Kilo 双注册抑制**（#623/#637）：检测到 `opencode.json` 同时有 `plugin:["context-mode"]` 和旧 `mcp.context-mode` 块时，本 MCP 子进程不再注册工具（避免重复），并：① stderr 打一行解释诊断；② 安装显式 `tools/list → {tools:[]}` handler（否则 SDK 回 -32601 误导用户）。server.ts:156-331。
- **空 prompts/resources handler**：防 OpenCode 拿到 -32601 后毒化传输层（#168，344-351）。
- **MCP 就绪哨兵**：`/tmp/context-mode-mcp-ready-<pid>`，30s 刷新 mtime（跨 PID namespace 也能信任），hook 侧 90s 新鲜度窗口（server.ts:4858-4916）。
- **版本提醒**：启动抓 npm 最新版 + 每小时刷新；响应里插入 "outdated" 警示（server.ts:777-841, 892-897）。
- **缓存中途自愈**：第一个工具调用时 `healCacheMidSession()` 修插件缓存（server.ts:843-883）。

## 9. 启动序列（main()）

**事实**（server.ts:4851-4979）：
1. 清理陈旧 DB（tmpdir 死 PID DB、>14 天 content DB、>14 天 sources）
2. 注册 shutdown（关 store、杀背景进程、删 preload、删哨兵）+ SIGINT/SIGTERM graceful（含 stats 强制落盘）
3. `startLifecycleGuard`（父进程死亡看护，lifecycle.ts）
4. `StdioServerTransport` connect → 写哨兵 → 启动哨兵刷新/心跳
5. 适配器检测（`detectPlatform(clientInfo)`）→ 恢复上次的 tool 计数（`restoreSessionStats`）
6. 版本检查 + 60s stats 心跳
7. 顶层：`installStrictClientSchemaCompat()` 必须在全部 registerTool 之后执行；`CONTEXT_MODE_EMBEDDED_PLUGIN_TOOLS=1` 时**不**跑 main（被 OpenCode/Kilo 插件进程内 import）（server.ts:4981-4991）
