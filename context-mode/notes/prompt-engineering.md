# 模块笔记：Prompt Engineering（提示词工程）详细设计

> 分析对象：`~/Projects/context-mode` @ v1.0.169（HEAD `dbec980`，只读分析）。
> 标注约定：**事实**（附 `文件:行` 出处） / **推断** / **未知**。
> 相关 ADR：`docs/adr/0002-tool-description-style.md`、`docs/adr/0003-routing-deny-reasons.md`。
> 配套契约测试：`tests/core/server.test.ts`（tool description style contract，约 `:5268` 起）。

## 0. 一句话与提示词面地图

**事实**：context-mode 的"省 context"目标一半靠工程（沙箱/FTS5/阈值），一半靠提示词——让模型在选型瞬间做出正确路由。提示词共有 6 个互相独立的 surface，各自有不同的注入时机与文案规范：

| # | Surface | 注入时机/载体 | 规范归属 | 出处 |
|---|---|---|---|---|
| P1 | 11 个 `ctx_*` 工具描述 | MCP `tools/list`，模型每次选型时可见 | ADR-0002（WHEN/WHEN NOT/RETURNS/EXAMPLE） | `src/server.ts:1648,2043,2237,2554,3413,3679,3937,4140,4271,4446,4817`；`docs/adr/0002-*.md` |
| P2 | Routing block 系统注入 | SessionStart `additionalContext` + Agent 子代理 prompt 追加 | routing-block 工厂函数，无独立 ADR（ADR-0002 点名它才是 exhortation 该待的地方） | `hooks/routing-block.mjs:16-80`；`hooks/core/routing.mjs:892-916` |
| P3 | 重定向/拒绝文案（deny reason / modify echo） | PreToolUse `deny`/`modify` 即时返回 | ADR-0003（CASE A redirect ≠ CASE B restriction） | `hooks/core/routing.mjs:767-889`；`docs/adr/0003-*.md` |
| P4 | 会话恢复提示词（snapshot + directive + auto-injection） | PreCompact 写快照 → SessionStart(compact/resume) 注入 | snapshot 纯函数 + directive 模板，无 ADR | `src/session/snapshot.ts:475-577`；`hooks/session-directive.mjs:275-500`；`hooks/auto-injection.mjs` |
| P5 | 运行时指针/节流文案（工具返回体里的下一跳指令） | 每次工具返回的正文尾部 | 各工具内联文案，无统一规范 | `src/server.ts:1958-2036,2654-2782,3578,3663` |
| P6 | 静态文档提示词（Skills + `CLAUDE.md`/`SYSTEM.md` 模板） | 安装时写入各客户端配置，模型当项目指令读 | 各文件独立，无契约测试 | `skills/*/SKILL.md`；`configs/claude-code/CLAUDE.md`；`CLAUDE.md` |

**推断**：P1–P3 是"选型层"（决定调哪个工具），P4 是"恢复层"（压缩后别失忆），P5 是"行为层"（拿到返回后下一步做什么），P6 是"兜底层"（hook/MCP 都没覆盖时的静态护栏）。四层缺一不可，但只有 P1/P3 有契约测试锁死，P2/P4/P5/P6 靠 code review 人治。

## 1. P1：工具描述（ADR-0002，最规范的一层）

### 1.1  canonical 结构

**事实**（`docs/adr/0002-tool-description-style.md:Decision`）：所有 `ctx_*` 工具描述 MUST 遵循：

```text
<1-line headline, <= 120 chars, imperative-positive>

WHEN:
  - <positive trigger conditions>

WHEN NOT:
  - <positive disambiguation from sibling tools>

RETURNS:
  <what the agent sees back, 1-3 lines>

EXAMPLE: <one canonical call with realistic params>
```

细则（**事实**，同 ADR `Canonical structure` 7 条）：

1. 顺序 MUST 是 `WHEN -> WHEN NOT -> RETURNS -> EXAMPLE`（肯定选型线索在前，否定消歧在后，audit rubric #2）。
2. Bullets MUST 只用 markdown `- `；数字/`*`/`•` 一律拒绝（含 routing-target 描述里的编号——"每条 bullet 应独立为真，而非有序步骤"）。
3. Section header MUST 大写 + 行首 + 冒号（`WHEN:`）。理由是跨模型分词器一致性：GPT/Gemini/Llama/Claude 只有大写形态能命中同一个 token。
4. 每个 header 下用两空格缩进（contract test 不 assert 缩进数，但全仓库统一两空格）。
5. Section 之间空一行。
6. 每个工具恰一个 `EXAMPLE:`（`ctx_purge` 这种双输入形态 MAY 有两行 EXAMPLE，但必须相邻）。
7. Carve-out 只有 `ctx_purge`：`DESTRUCTIVE`/`SCOPES`/`CONTRACT` 三个额外 header 被 allow-list 放行。

### 1.2 禁用词（forbidden tokens）

**事实**（ADR-0002 `Forbidden tokens` 表 + `tests/core/server.test.ts:5361-5425` 的 8 条 `FORBIDDEN` 规则）：

| Token | 理由 |
|---|---|
| `MANDATORY:`（开头） | 这是 developer-policy 口吻，不是选型线索 |
| `BLOCKED` | 留给 ADR-0003 CASE B（真实安全拒绝），工具描述里没有安全限制可表达 |
| `PREFER X OVER Y` | 把选择框成 tradeoff；改用肯定式 `WHEN:` |
| `Do NOT use/read/pull`、`Never use` | 肯定 beats 否定（rubric #2），消歧用 `WHEN NOT:` 结构表达 |
| `SESSION STATE` 子句 | skill/role 持久化是 routing-block 的事，不进工具描述 |
| `✅`/`❌` emoji bullets | Llama/Gemini 分词不一致 + 负例泄漏（rubric #4，Probe 3 实证） |

**事实**：RFC 2119 情态动词只保留给 **post-call obligation**（调用后的义务），不用于选型：`ctx_upgrade` 的 "you MUST run the returned shell command…" 被显式允许（`src/server.ts:4270-4444`；ADR-0002 `Allowed imperative hierarchy`）。

### 1.3 长度与豁免

**事实**：描述 SHOULD ≤ 1000 字符，硬上限 1500（ADR-0002 `Length`）。豁免 4 个：`ctx_stats`/`ctx_doctor`/`ctx_insight` 是诊断/GUI affordance（one-line 描述，非路由目标）；`ctx_upgrade` 允许 MUST（post-call 义务）。豁免清单同时写在 ADR 和 contract test 的 `EXEMPT_FROM_WHEN` 集合里（`tests/core/server.test.ts:5339-5346`）。

### 1.4 各工具描述的差异化写法（实证门控，非一刀切）

**事实**（ADR-0002 `Context` + `Consequences`）：38 次 A/B × 6 probes（Haiku + Sonnet）证明"越柔和越好"是错的——重 framing 在 `ctx_purge` 上把小模型的 Preserve-parameter-fidelity 从 3/5 提到 5/5（Probe 4），但在 `ctx_execute` 上反而降级选型。所以 PR #683 只重写 6 个漂移明显的工具（`ctx_execute`/`ctx_execute_file`/`ctx_batch_execute`/`ctx_search`/`ctx_index`/`ctx_fetch_and_index`），`ctx_purge` 走 WS2 独立重写并保留 carve-out。

当前各工具描述要点（**事实**，`src/server.ts` 对应段）：

- `ctx_execute`（`:1647-1952`）：headline + `Think-in-Code — the core philosophy` 段落（700KB 日志读进来 vs 沙箱里算完只回 3KB 的具体算账）+ 47 文件代码实例；`WHEN` 第一句即"derive an answer FROM data"；`intent` 参数说明（>5KB 自动索引，只回章节标题+预览）。
- `ctx_execute_file`（`:2042-2230`）：`FILE_CONTENT` 变量机制 + "KNOW SOMETHING ABOUT vs SEE all of it" 对比 + `WHEN NOT: intend to EDIT → use Read/Edit`（兄弟工具消歧范例）。
- `ctx_index`（`:2236-2438`）：`WHEN` 列 6 类可索引内容（Context7/Skill/MCP docs、API refs、tools/list 输出、README/migration/changelog），`WHEN NOT` 明确把 log/CSV/build 输出赶去 `ctx_execute_file`（"persist vs process" 分工）。
- `ctx_search`（`:2553-2810`）：把 ranking 管线（Porter + trigram → RRF → proximity-rerank → Levenshtein 纠错 → window snippet）写进描述，并列出 session-memory source 标签（`decision`/`error`/`blocker`/`plan`/`user-prompt`/`rejected-approach`/`compaction`）+ throttle 计数器说明。
- `ctx_fetch_and_index`（`:3412-3676`）：TTL 缓存语义（默认 24h、`ttl:0`/`force:true` 强制刷新、14 天清理）+ `requests[]`/`concurrency` 2-8 并行写入 `WHEN`。
- `ctx_batch_execute`（`:3678-3930`）：`WHEN` 第一句"3+ related commands" + gather-and-query 一次往返（`queries` 同回合返回）+ FETCH vs DERIVATION 两阶段划分（"fetch 并行，derivation 仍归代码"）。
- `ctx_purge`（`:4445-4620`）：唯一保留 `DESTRUCTIVE` 开头 + `SCOPES`（per-session vs per-project 二选一）+ `CONTRACT`（`confirm:true` 必填、双 scope 并存报 ambiguous）——Probe 4 实证要求的重 framing。

### 1.5 契约测试怎么锁死

**事实**（`tests/core/server.test.ts:5268-5620`）：静态扫描 `src/server.ts` 每个 `server.registerTool()` 块（正则找 `description:` 到 `inputSchema:` 之间的源码形态），flatten 后 assert：① 11+ 工具全被提取到；② 非豁免工具 MUST 含 `WHEN:`（兼容 `WHEN TO USE:` 过渡别名）；③ 8 条 forbidden token 全缺席；④ section 顺序 MUST 是 WHEN→WHEN NOT→RETURNS→EXAMPLE；⑤ 非 canonical/非 carve-out 的大写 header 一律拒绝；⑥ bullets 只许 `- `。全程无 LLM 调用，每次 commit 跑。

## 2. P2：Routing block（系统注入，exhortation 合法层）

### 2.1 模板结构

**事实**（`hooks/routing-block.mjs:16-80`，工厂 `createRoutingBlock(t, options)`，105 行文件）：

```xml
<context_window_protection>
  <priority_instructions>…Think-in-Code: program the analysis, do not compute it by reading raw data…</priority_instructions>
  <deferred_tool_bootstrap>…（仅 toolSearchBootstrap=true 时）</deferred_tool_bootstrap>
  <tool_selection_hierarchy> 0.MEMORY / 1.GATHER / 2.FOLLOW-UP / 3.PROCESSING </tool_selection_hierarchy>
  <when_not_to_use>…OBSERVE vs PROCESS / Edit vs analyze / WebFetch…</when_not_to_use>
  <file_writing_policy>…ctx_* 与 Bash 子进程不落盘，写文件用原生 Write/Edit…</file_writing_policy>
  <output_constraints><artifact_policy>…产物写文件，只回路径+一句话…</artifact_policy></output_constraints>
  <session_continuity>…skills/roles 是记忆辅助非长期指令，用户最新消息永远优先…</session_continuity>
  <ctx_commands>…ctx stats/doctor/upgrade/purge 四句触发语 → 调哪个 MCP 工具…（子代理省略）</ctx_commands>
</context_window_protection>
```

关键设计点（**事实**）：

- `priority_instructions`（`:21`）是全仓库唯一允许"每字节都进 conversation memory"这种成本恐吓的地方——ADR-0002 `Consequences` 明确说 `THINK IN CODE`/`MANDATORY routing rules` 这类 trainer 口吻**只许待在这一层**（system-prompt 注入），不许进 P1 工具描述。
- `tool_selection_hierarchy`（`:30-40`）用数字编号（0→3），与 ADR-0002 禁止工具描述里编号形成对照——ADR 原文解释：system-prompt 里的优先级排序是另一类 prompt surface，编号在此合法。
- `when_not_to_use`（`:42-47`） deliberately 小写下划线命名；contract test 注释（`tests/core/server.test.ts:1950-1957`）记录它由 `<forbidden_actions>` 改名而来（PR #683 follow-up，为去掉 forbidding 语气）。
- `session_continuity`（`:59-61`）防"捕获的旧指令绑架新会话"：过去 phrase 不 bind 人，冲突时跟用户。
- `ctx_commands`（`:62-78`）是自然语言触发器表（"ctx stats"/diagnose/update/wipe → 调哪个工具、purge 必须 `confirm:true`、`/clear` 后告诉用户 KB 还在）。

### 2.2 何时注入、注入多少

**事实**：

- SessionStart 每次注入完整 block（含 commands）：`hooks/sessionstart.mjs` 内 `createRoutingBlock(toolNamer)`（默认 `includeCommands=true`）。
- Agent 子代理：`routePreToolUse` 的 Agent 分支把 block **追加**到 subagent prompt 尾（`hooks/core/routing.mjs:892-916`），但 `includeCommands:false`（子代理调不了 stats/doctor/upgrade/purge，#233），且仅 Claude Code（或 platform 为空的默认）额外前置 `toolSearchBootstrap`（DEFERRED 工具先 `ToolSearch(select:ctx_batch_execute,ctx_search,ctx_execute,…)` 加载一次，否则子代理会 stall，#724）。
- 平台适配：工厂接受 `t(bareTool)` namer（`hooks/core/tool-naming.mjs`），同一语义在各平台渲染成各自工具名；静态导出（`ROUTING_BLOCK` 等，`:100-105`）默认 claude-code 命名做向后兼容。
- 节流：Bash/Read/Grep 的通用 nudge 是每 session 一次（`guidanceOnce`，内存 Set + `tmpdir/context-mode-guidance-*` 文件 O_EXCL 原子标记，`:121-144`）；外部 MCP 工具是每 N 次一次（`guidancePeriodic`，默认 10，`:161-194`，env `CONTEXT_MODE_EXTERNAL_MCP_NUDGE_EVERY` 可调 1-100）。动机写在注释里：50+ 次 Jira/Slack/Notion 长会话里 one-shot nudge 会被 compact 冲掉（#567 follow-up）。
- 降噪：输出结构有界的命令（`pwd`/`whoami`/`git status`/`--version` 等，`SAFE_COMMAND_PATTERNS :260-324`）+ 无 shell 连接符（`SHELL_CONTROL_OPERATORS :339`）→ 不 nudge（#463/#470）；另有 opt-in 的 `CONTEXT_MODE_BASH_NUDGE_MIN_COMMAND_BYTES` 短命令豁免（默认 0 即关闭，`:97-108`，#817）。

### 2.3 四份一次性 guidance

**事实**（`hooks/routing-block.mjs:82-96`）：Read/Grep/Bash/ExternalMCP 各一段 `<context_guidance><tip>…`，统一句式为"X stays correct when …; redirect to ctx_* when you intend to PROCESS/filter/count/aggregate"。Read guidance 区分"为 Edit 而读（Read 正确，Edit 要 exact bytes 做 match）vs 为分析而读（走 `ctx_execute_file`）"；Grep guidance 注明 `language:"shell"` 仅当代码匹配宿主 shell（Windows PowerShell vs Unix POSIX）。

## 3. P3：重定向 vs 拒绝（ADR-0003，一个词的 bug）

### 3.1 动机：`blocked` 一词导致 Opus 4.6 投降

**事实**（`docs/adr/0003-routing-deny-reasons.md:Context`）：PR #654 复现——WebFetch deny reason 里秃一个 `"blocked"` 被 Opus 4.6 读成网络/安全限制，agent 放弃改用被重定向的工具，转而向训练数据投降（capitulate）。Sonnet 上两种文案都投降（6/6），Haiku 上原词反而略好——所以修复必须按 CASE 区分，不能全局替换。

### 3.2 CASE A（redirect）vs CASE B（restriction）

**事实**（ADR-0003 `Decision`，2026-05-24 两次 amendment 后的最终版）：

CASE A（支持，只是换个工具走——context Curriculum Vitae 或效率原因）MUST：

- 开头动词 `"redirected to <ctx_tool>"`（肯定路由意图，无否定、无组织 rationale——"HTTP 301 只带 Location，不解释 for SEO efficiency"，Second amendment 原话）；
- MUST 肯定能力：`"<ctx_tool> has full network access"`（第一版 amendment 的 `"NOT a network restriction"` 被 Mert 驳回：秃-NOT 否定恰恰 prime 了它想否定的 frame，ironic process theory；Haiku 上把 capitulation 从 0/6 干到 2/6）；
- MUST 点名替代工具（`Call ctx_fetch_and_index(url, source) now` 式祈使）；
- MUST 以肯定式 retry hint 收尾：`"Retry the same call on a transient DNS error (EAI_AGAIN, ETIMEDOUT, ENETUNREACH)"`；
- MUST NOT 含秃-NOT 否定（`NOT a network restriction`、`Do NOT retry with curl`）、MUST NOT 含组织 rationale 前缀（`for context-window efficiency` 等）；`BLOCKED` 大写秃词保留给 CASE B。

CASE B（真安全/政策拒绝：deny pattern、私有 IP、敏感路径、不支持的沙箱能力）MUST 以 `"denied"`/`"blocked by security policy"` 开头，MUST 引用违反的 pattern/规则，MAY 给安全替代。

### 3.3 四个 CASE A 落点（当前行号）

**事实**（`hooks/core/routing.mjs`，ADR 里的 L707/738/751/804 是修法时行号，当前文件已漂移，以实际内容为准）：

1. curl/wget stdout flood → `modify` 重写为 `echo "context-mode: curl/wget redirected. Call ctx_execute… Or call ctx_fetch_and_index… Both have full network access. Retry…"`（`:767-782`），附 `redirectMeta{type:"bash-redirected",bytesAvoided:8192}` 供 PostToolUse 记账；有文件输出（`-o/-O/>`）+ 静默（curl `-s`/wget `-q`）+ 无 verbose 的才放行（`:731-786`）。
2. 内联 HTTP（`fetch('http`/`requests.get(`/`http.get(`）→ 同样 `modify` + echo（`:794-805`），只剥 heredoc 不剥引号（`-e/-c` 传的代码仍可见）。
3. 构建工具（gradle/maven/sbt）→ `modify`，并把原命令拼进 `ctx_execute(language:"shell", code:"<cmd> 2>&1 | tail -30")` 模板，还教模型把 `tail -30` 换成 `grep -E '(error|warning|FAIL|…)'`（`:810-818`）。
4. WebFetch → 硬 `deny`（`:877-889`），reason 点名 `ctx_fetch_and_index(url,source)` + `ctx_search(queries)` 或一站式 `ctx_execute`，同样 full-network-access + retry hint，附 `redirectMeta{type:"webfetch-redirected",bytesAvoided:16384}`。

**事实**：CASE B 文案（`:716,932,953,964,985`）统一为 `"Blocked by security policy: … matches deny pattern …"`，与 CASE A 的 `"redirected"` 开头形成机器可检的对立；contract test 锁 CASE A 四处 MUST 以 redirected 开头、MUST NOT 含秃 `BLOCKED`/`"NOT a network"`/`"Do NOT retry"`/`"for context-window efficiency"`（ADR-0003 `Consequences` 末段）。

### 3.4 一个 formatter 陷阱

**事实**（`hooks/core/formatters.mjs:claude-code.modify`）：Claude Code v2.1.x 对 Bash 的 `allow+updatedInput.command` 改写直接忽略（原命令照跑）——/diagnose Phase 4 强迫 deny probe 验证过。所以 Bash redirect 在 claude-code formatter 里被翻译成 `deny` + 把 echo payload 抽成 `permissionDecisionReason`；而 Agent prompt 注入（`updatedInput.prompt`）走 `allow+updatedInput` 是被 honor 的，必须保留 modify 形态。其他 adapter 各自语义不同，不可照搬。

## 4. P4：会话恢复提示词（snapshot + directive + auto-injection）

### 4.1 快照（compact 时写，`src/session/snapshot.ts`）

**事实**（`:475-577` `buildResumeSnapshot`，纯函数、无 DB/FS 副作用）：事件按 category 分组，每类非空即建一节，**全量组装、无优先级丢弃、无字节预算**（注释原话 "no priority dropping, no byte budget"）。XML 骨架：

```xml
<session_resume events="N" compact_count="K" generated_at="ISO">
  <how_to_search>…run the exact tool call shown under each section. Do NOT ask the user to re-explain… Do NOT invent your own queries — use the ones provided.</how_to_search>
  <session_goal>…（如有）</session_goal>
  <files count>…file (write×2, read×1)… + toolCall…</files>
  <errors>/<decisions>/<rules>/<git>/<task_state>/<environment>/<subagents>/<skills>/<roles>/<intent/>
  <recent_messages>…（永远最后，紧邻下一轮 LLM 输入）</recent_messages>
</session_resume>
```

每节 = 自然语言摘要 + 一个可直接运行的 `toolCall(searchTool, queries)`（`:~50-60` helper）：`ctx_search(queries: ["<80字符截断的原文/文件名+op>…"], source: "session-events")`。query 生成是 `buildQueries` 取前 4 个去重原文（`:~40-50`），文件节只列最近 `MAX_ACTIVE_FILES=10`（`:~30`）。

### 4.2 指令（resume 时读，`hooks/session-directive.mjs`）

**事实**（`:275-500` `buildSessionDirective(source, eventMeta, toolNamer)`）：输出三层 `<session_knowledge source="compact|continue"><session_guide>…14节…</session_guide><session_search>…</session_search>[<continue_from>…]</continue_from>]</session_knowledge>`。

- `session_guide` 14 节顺序固定：Last Request（>300字符截断）→ Pending Tasks（creates×updates 按序配对，已完成/删除/失败的过滤掉，`:296-334`）→ Key Decisions → Files Modified → Unresolved Errors → Git → Project Rules（只留路径后两段）→ MCP Tools Used（计数）→ Subagent Tasks → Skills Used → Environment → Data References → Intent/Role → Plan Mode。
- `session_search`（`:487-491`）三句话：细节在 FTS5（`source:"session-events"`）→ 用 `searchTool(queries:[...], source:"session-events")` 按需取 → `Do NOT call ctx_index() — data is already indexed`（防模型把恢复出来的数据又索引一遍）。
- `continue_from`（`:494-496`）仅当 `lastPrompt && source==="compact"`：`Continue working on the last request. Do NOT ask the user to repeat themselves.`——resume/continue leg 不加（避免自我注入，参 `session/db.ts:1059-1078` 原子领取）。
- 防腐细节：subagent 分"本 leg" vs "earlier session — not current"（leg 边界 = 最后一个 `session_start` 事件，`:10-28`，#780）；plan 状态机 APPROVED-EXECUTED/REJECTED/ACTIVE/COMPLETED 四态 + 对应禁令（"Do NOT re-enter plan mode"，`:458-482`）；Data References 只内联最近 8 条、每条超 150 字符转指针（`DATA_REF_INLINE_MAX=8`/`DATA_REF_ENTRY_MAX=150`，`:35-51`，#840——"内联全部捕获物违反 raw-bytes-stay-out 自家原则"）。

事件清单文件（`:77-272` `writeSessionEventsFile`）按 H2 分节写 markdown（`## Active Files/Tasks In Progress/Errors…`）——注释明说这是为 FTS5 chunking 优化的（markdown 标题栈切分，`store.ts:1646-1763`），随后被 `maybeIndexSessionEvents` 索引后删除（见 architecture.md 主流程 C）。

### 4.3 自动注入（预算 500 tokens）

**事实**（`hooks/auto-injection.mjs`，仅 `source==="compact"` 触发）：优先级 P1 Role（`behavioral_directive`，永远第一、截 400 字符）→ P2 Decisions（最新 5 条、超预算降到 3 条、每条截 100 字符）→ P3 Skills（去重、最新 10 个）→ P4 Intent（最新）。硬上限 500 tokens（≈4 chars/token 估算，`estimateTokens`）。输出 `<behavioral_directive>/<rules>/<active_skills>/<session_mode>` 四种小 XML 块。

## 5. P5：运行时指针与节流文案（工具返回体里的下一跳）

**事实**（`src/server.ts`）：

- `indexStdout`（`:1958-1976`，>100KB 输出）：`Indexed N sections (M with code) from: <label>\nUse ctx_search(queries: ["..."]) to query this content. Use source: "<label>" to scope results.`——只给数字+label，不给内容。
- `intentSearch`（`:1982-2036`，>5KB 且带 `intent`）：索引后**只回标题+首行预览**（120 字符截断）+ `Searchable terms: …`（`getDistinctiveTerms` 词汇表提示）+ `Use ctx_search(queries: [...]) to retrieve full content of any section.`；零命中时给"indexed N sections … No sections matched intent …" + 同样 terms + `to explore the indexed content`。
- `ctx_batch_execute`（`:1373-1414`）：`query_scope:"batch"` 越界提示 `output cap reached — use ctx_search(queries: ["…"])`；零命中 `No matching sections found.` + `To search across all indexed sources, use ctx_search(queries: [...]) or call ctx_batch_execute with query_scope:"global"`。
- `ctx_fetch_and_index` 缓存命中（`:3578`）：`Cached: **label** — N sections… You MUST call ctx_search() to answer questions about this content — this cached response contains no content.`（MUST 在此合法：post-call 义务）。
- `ctx_search` 节流计数器（`:2773-2782`）：软 cap 前 `> Throttle: call #N/M in this window. … Prefer ctx_search(queries: [...]) array form … it counts as a single call.`；软 cap 后（60s 窗口第 4 次起，每 query 只给 1 条）`⚠ search call #N/M … Results limited to 1/query. … Batch queries: … or use ctx_batch_execute.`；硬 block（第 8 次起）`BLOCKED: N search calls in Ns. You're flooding context. STOP making individual search calls. Use ctx_batch_execute(commands, queries)…`（`:2654-2663`，`isError:true`）。注意此处的 `BLOCKED` 大写是 FloodGuard 运行时文案，非 P1 工具描述——不在 ADR-0002 禁用范围内（**推断**：contract test 只扫 `description:` 块，扫不到这里）。
- HTML→Markdown 预览截断 `charSafePrefix` 3072 字符（`:3119`）等数字 Year 细节见 server-tools 笔记，不在此复述。

## 6. P6：静态文档提示词（Skills + CLAUDE.md）

**事实**：

- `skills/context-mode/SKILL.md`：`MANDATORY RULE`（"Default to context-mode for ALL commands. Only use Bash for guaranteed-small-output"）+ Bash 白名单（mutations/git writes/navigation/echo 等）+ 决策树（whitelist→context-mode→fetch→Playwright→agent-browser→MCP output→execute_file）+ 自动触发器列表（API debugging/log/test/git/data/infra/…任意一条无需询问直接用）。description 字段本身即触发器（含 "page snapshot/browser snapshot/DOM/accessibility tree/Playwright" 等长尾词 + "ANY MCP tool output that may exceed 20 lines"）。
- `skills/ctx-search/SKILL.md`：`Prefer ctx_search MCP tool → Batch all related questions in one queries array → Scope with source → 2-4 technical terms per query → MCP 不可用时 fallback CLI → 空索引时指引 ctx-index` 六步。
- `skills/ctx-index/SKILL.md`：同理的 index-then-search 闭环（`ctx_index(path) → ctx_search`）。
- `CLAUDE.md`（仓库根）与 `configs/claude-code/CLAUDE.md` 内容一致（`diff -q` 无差异）：`# context-mode — MANDATORY routing rules` + `Think in Code — MANDATORY` + `BLOCKED — do NOT attempt`（curl/wget、Inline HTTP、WebFetch 三节）+ `REDIRECTED` + 数字编号 Tool selection（0.MEMORY→5.INDEX）+ 并行 I/O（concurrency 4-8 / gh 限 4）+ artifact 输出规范。

**推断**：P6 的 `MANDATORY`/`BLOCKED`/`Do NOT retry` 与 ADR-0002/0003 在 P1/P3 层禁用的词**字面冲突**，但不违反 ADR——因为 ADR 的管辖域明确只覆盖"工具描述"与"hook deny reason"两个运行时 surface，静态项目指令是第三类 surface。此处是刻意分层（静态指令允许 training 口吻，运行时选型线索必须肯定式），还是历史漂移未同步，须问作者。

## 7. 跨模型设计 rationale（一处地方讲透）

**事实**（ADR-0002 `Cross-LLM rationale` + ADR-0003 `Amendment`）：

- 大写 header + 冒号是唯一跨 Claude/GPT/Gemini/Llama 单-token 命中的形态；小写/混合大小写在各家分词器下碎成不同 token。
- `FORBIDDEN`/`BLOCKED`/`NEVER`/`MANDATORY` 等词撞 Constitutional-AI 式 RLHF 先验（模型把它读成"开发者政策/安全红线"而非"选型建议"）。
- emoji bullets 在 Llama/Gemini 下分词不一致，且 `❌` 类 bullet 是负例泄漏（模型学到"这个例子是错的"而不是"别这么做"）。
- 秃-NOT 否定（`NOT a network restriction`）prime 它想否定的 frame（ironic process theory），Haiku 实测 0/6→2/6 倒退。
- 组织 rationale（`for context-window efficiency`）是元数据不是行动输入，301 只带 Location 不解释 SEO——删掉后动词+目标+能力肯定即完整行动信号。

## 8. 未知与待确认

- **未知**：ADR 引用的 `TOOL-DESCRIPTIONS-AUDIT.md`（§3 audit table、§5 probe evidence、§6 rewrites）与 `GRILL-Q1-VERDICT.md` 在本仓库不存在——38 trials 的原始数据无法审计，上述"实证"只能采信 ADR 转述。
- **未知**：`configs/*/SYSTEM.md` 与 `AGENTS.md` 模板是否与 `CLAUDE.md` 同词（本次只 diff 了根 vs claude-code 的 `CLAUDE.md`）；其余 17 平台模板的 BLOCKED/MANDATORY 残留量未统计。
- **未知**：Agent 子代理 `ToolSearch` bootstrap（#724）在各平台真实 stall 率下降多少——只有 issue 描述，无数字。
- **未知**：`getDistinctiveTerms` 的词汇表算法（P5 intentSearch 的 Searchable terms）在 `store.ts` 的具体实现与质量，本次未展开。
