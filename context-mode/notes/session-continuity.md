# 模块笔记：会话连续性（src/session/*, hooks/session*, precompact, stop, userpromptsubmit）

## 1. 文件职责

**事实**：
| 文件 | 职责 |
|---|---|
| session/db.ts (1726) | `SessionDB`（继承 SQLiteBase）：schema、per-project 路径 hash、事件 CRUD（去重 + FIFO 淘汰）、resume 快照、工具计数 |
| session/extract.ts (2960) | 纯函数抽取器：hook stdin + transcript JSONL → `SessionEvent[]`（13 大类），dispatch 在 extract.ts:2710 |
| session/snapshot.ts (577) | 构建 `<session_resume>` XML 目录（每节内嵌可执行的 ctx_search 调用） |
| session/analytics.ts (3085) | 省流/连续性分析引擎（`queryAll`、lifetime 扫描、`formatReport` 叙事渲染） |
| session/purge.ts | 深度清理：按项目或按单会话删所有会话痕迹（保留项目文件） |
| session/error-classifier.ts | 纯分类器：错误 10 桶、命令类型、耗时桶（对齐平台 seed ABI） |
| session/project-attribution.ts | 每个事件的 `project_dir` 归属（带置信度、carry-forward） |
| session/event-emit.ts | 服务端 fire-and-forget 事件（沙箱执行/索引写入/缓存命中） |
| session/persist-tool-calls.ts | tool_calls 计数持久化与恢复（状态栏跨 PID 场景） |
| session/pricing.ts | 约 61 个模型的价目表（model-prices.json）+ `computeCostUsd` |
| session/retrieval-marker.ts | MCP server → 下一次 PostToolUse 的 bytes_retrieved 临时文件握手 |
| hooks/sessionstart.mjs | SessionStart：startup/compact/resume 三分支（清理、规则捕获、路由注入、快照注入） |
| hooks/precompact.mjs | PreCompact：快照 + resume 落库 + compact_count+1 + 事件 |
| hooks/stop.mjs | Stop：`turn_end` + 按 cursor 增量解析 transcript 用量/成本 |
| hooks/userpromptsubmit.mjs | 原始 prompt + prompt 特征，再抽取 user 侧事件（决策/角色/意图/目标/阻塞/数据） |
| hooks/session-helpers.mjs / session-loaders.mjs / session-directive.mjs | stdin/id/路径助手；bundle-first 加载 + 归属插入 + 平台转发；事件分组与 events.md 生成 |

## 2. SessionDB schema（session/db.ts:814-905）

**事实**（4 张表，无 FTS5——检索是 SQL LIKE，FTS5 在 ContentStore 侧）：
- `session_events`：`id, session_id, type, category, priority DEFAULT 2, data, project_dir DEFAULT '', attribution_source DEFAULT 'unknown', attribution_confidence, bytes_avoided, bytes_returned, source_hook, created_at, data_hash`；索引 `(session_id)`、`(session_id,type)`、`(session_id,priority)`、迁移时加 `(session_id,project_dir)`。
- `session_meta`：`session_id PK, project_dir, started_at, last_event_at, event_count, compact_count` + 迁移加 `usage_cursor`（transcript 增量解析游标）。
- `session_resume`：`id PK, session_id UNIQUE, snapshot, event_count, created_at, consumed DEFAULT 0`。
- `tool_calls`：`(session_id, tool) PK, calls, bytes_returned, updated_at`。
- 约束：`MAX_EVENTS_PER_SESSION=1000`、`DEDUP_WINDOW=5`（去重+淘汰+插入同一事务 + withRetry，db.ts:1173-1204）。

## 3. 数据流（hook 事件 → 落库 → 回注）

**事实**：
1. **PostToolUse**（matcher 覆盖 Bash/Read/Write/Edit/Glob/Grep/Todo/Plan/Skill/Agent/mcp__ 等，hooks/hooks.json:6-13）→ `extractEvents`，事件类型约 30 种：`file_read/write/edit`、`git/git_commit`、`error_tool`+`error_resolved`、`task`、`plan_enter/exit/approved/rejected`、`skill`、`subagent`、`mcp/mcp_tool_call`、`decision_question`、`agent_finding`、`worktree(_exit)`、`bash_outcome`、`webfetch_metadata`、`agent_usage`、`constraint_discovered`、`retry_detected` 等（extract.ts:2710-2750 + 各抽取器）。
2. **UserPromptSubmit** → `user_prompt` + `extractUserEvents`（过滤 `<task-notification>`/`<system-reminder>` 类系统消息）。
3. **Stop** → `turn_end` + `extractTranscriptUsageSince(transcript, cursor)`：逐行 JSON.parse，只取 `type==="assistant" && !isSidechain` 且有 model 的行，按 `usage_cursor`（uuid）之后增量累计 token/成本（offset-by-one fallback 防压缩后游标失效）。
4. **SessionStart** → `session_start` 生命周期、settings 快照、CLAUDE.md 规则捕获（`rule/rule_content`）。
5. **服务端**（MCP 进程内）：沙箱执行/索引写入/缓存命中即时发事件（`event-emit.ts`，bytes_avoided/bytes_returned 口径）。
6. **redirect marker 握手**：PreToolUse 写 tmp 标记（rejected/redirect/latency），PostToolUse 读取即删转事件；`bytes_retrieved` 靠 retrieval-marker 临时文件（因为 PostToolUse 对自家 ctx_* 工具不触发）。
7. 落库前 `resolveProjectAttributions` 给每事件算 project_dir + 来源 + 置信度；若存在 `~/.context-mode/platform.json`，事件再经 `maybeForward` 转发到平台（遥测）。
8. **检索**：`searchEvents` LIKE 查询（按 project_dir）；或 SessionStart/PreCompact 生成 `<hash>-events.md` → MCP server `maybeIndexSessionEvents` 索引进 FTS5（source=`session-events`）后删文件 → `ctx_search` BM25 查询。

## 4. 压缩/恢复的promise到底怎么兑现

**事实**：
- **PreCompact**：`buildResumeSnapshot(compactCount+1)`（maxBytes 参数被保留但忽略——**零截断**，snapshot.ts:29-31）→ 节：how_to_search、session_goal、files(top10)、errors、decisions、rules、git、task_state(仅 pending/in-progress)、environment、subagents、skills、roles、intent、recent_user_messages(最后 3 条 ≤400 码点)；每节内嵌 `ctx_search(queries:[…], source:"session-events")` 块 → `session_resume.upsert`（consumed=0）+ `compact_count+1`。
- **SessionStart(compact)**：写 events.md（人读清单）+ `buildSessionDirective("compact",…)`（上次请求、未完成任务、决策、文件、错误、git、plan 状态、"不要让用户重复"）+ auto-injection 行为状态；发 `snapshot-consumed`（bytes_returned=快照长度）与 `resume_completed` 事件。
- **SessionStart(resume)**：新 id 有 live 事件→注入 live；否则 `claimLatestUnconsumedResume` 用 `UPDATE…RETURNING` 原子领取（防重复领取/自我注入，db.ts:1059-1078, 1601；#413）。
- **推断**：整体思想 = 快照只给"目录 + 检索入口"，细节走 FTS5——这就是 README 说的"不把数据倒回上下文"。

## 5. analytics / 定价 / 状态栏

**事实**：
- 数据模型：`ContextSavings / ToolSavingsRow / SandboxIO / McpToolUsageRow / ConversationStats / RuntimeStats / IndexState / FullReport / LifetimeStats / RealBytesStats / MultiAdapterLifetimeStats`（analytics.ts:44-206, 662, 1040, 1626）。
- `getLifetimeStats` 扫描 sessionsDir 下所有 `*.db` + auto-memory markdown（710-809）。
- 定价：`computeCostUsd` 查价目表（hit 不到返回 null，不静默套 Claude 价）；状态栏口径 `pricePerToken()` = `PI_CONTEXT_MODE_PRICE_OUTPUT_PER_TOKEN` env 或默认 $5/1M（2501-2508）。
- 状态栏 `bin/statusline.mjs`：从 stdin JSON 拿 session_id，直接读 SessionDB + `build/session/analytics.js` 的 `getRealBytesStats`/`getMultiAdapterLifetimeStats`；输出一行 ANSI `context-mode ● <kb> this chat · <kb> lifetime · across N tools · X% kept out`；非 TTY/NO_COLOR 去色。PID 遍历（/proc、ps）只是最后手段。
- 服务端 sidecar JSON（`stats-<sessionId>.json`，500ms 节流 + 60s 心跳）为外部仪表盘提供实时数据（server.ts:974-1101）。

## 6. 多写者与可靠性（ADR-0001）

**事实**：
- SessionDB/ContentStore 多写者安全：WAL + busy_timeout 30s + withRetry（100/500/2000ms）；v1.0.128 的锁文件/EXCLUSIVE 模式在 v1.0.130 移除。
- ADR 记录的残余风险：进程层回归时 WAL 可能无界增长；缓解 = 退出钩子 checkpoint + 插入计数触发的 FTS optimize。
- purge：`purgeSession` 支持按项目或按会话清（sessionId 时保留项目级文件），见 server.ts:4432-4438 的 ctx_purge 语义。
