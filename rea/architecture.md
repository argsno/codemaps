# REA 架构与主流程

> 事实来源：`docs/architecture.mermaid`（官方架构图）、`src/` 源码直读。
> 所有"事实"均附文件出处；推导内容标"推断"。

## 1. 分层与依赖方向（六边形 / Ports & Adapters）

**事实**（AGENTS.md:31；docs/architecture.mermaid）：依赖一律向内——
`domain` ← `contracts` ← provider 适配器 ← `application` 共享工作流 ← `cli`/`server` 次级适配器。

| 层 | 位置 | 职责（事实） |
|---|---|---|
| Domain | `src/domain/`（568 文件，纯函数、无副作用） | Evidence、Result/错误代数、BinaryTarget、快照格式、比较/图算法、ResidualUnknown |
| Contracts | `src/contracts/`（87 文件） | 面向调用者的 Zod 输入/输出 Schema、`ToolContract` 单一事实源、错误 Schema |
| Providers | `src/hopper/` `src/ghidra/` `src/ida/` `src/browser/` `src/android/` `src/dotnet/` `src/evm/` `src/wasm/` `src/native/` … | 每个分析引擎一个适配器，实现统一的 `AnalysisProvider`/`AnalysisClient` 端口 |
| Application | `src/application/`（319 文件） | CLI/MCP 共享工作流：BinarySession、InvestigationRecords、各领域 Service（Browser→WebRuntime→…）、Setup/Doctor |
| Composition | `src/composition/`（33 文件） | **唯一生产装配点**：工厂 + 惰性可选 provider + 生命周期/清理接线 |
| Adapters | `src/cli/`（50）+ `src/server/`（93）+ `src/main/` | 一次性 CLI（incur 框架）与 MCP stdio server 两个交付形态 |
| Process 基座 | `src/process/`（72）+ `src/windows/` | 自有进程组、私有运行目录、超时、清理、所有权元数据 |

**事实**：`src/composition/binary.ts` 自述为 "sole production wiring point"，CLI 与 MCP 共用同一个 `composeBinarySession`，保证两种入口的 provider 选择/生命周期/证据语义完全一致。

## 2. 入口与两条交付路径

**事实**（scripts/rea.mjs；src/main.ts；src/cli.ts）：

```
rea.mjs (bin: rea / rea-agents)
 ├── `mcp` | `--mcp`      → dist/mcp/main.js   → run() → serveStdio(MCP stdio server)
 ├── `mcp doctor`         → 另加 mcpDoctor CLI
 └── 其它参数              → dist/cli-runtime/cli.js → createCli().serve(argv)
```

- MCP 路径（src/main.ts）：`snapshotEnvironment` → `parseConfig`(Zod) → `createBinarySession` → `openInitialTarget`（若有预置目标）→ `startMcpTransport`（登记 createServer 工厂）→ `createShutdown`（SIGINT/SIGTERM/stdin EOF 幂等清理）。
- CLI 路径（src/cli.ts）：构造 incur `Cli` 实例，注册约 40 组命令（setup/analysis/browser/capture/…），命令层直接复用 `application` 工作流与 `composition` 工厂。

## 3. 核心抽象（设计重心）

**事实**：

1. **BinarySession**（`src/application/binary/BinarySession.ts`，988 行）：持有**唯一活跃目标**，CLI/MCP 共用。
   - 目标切换串行化（`#serialize` + transition generation + 调用排水 `#drainCalls`），切换失败会**重建旧目标**而非留下已关停的 client。
   - 所有调用经 `execute` / `withAdmittedAnalysis`（组合操作全生命周期持锁，保证计算期间绑定不被换掉）。
   - `withAdmittedExternalMutation`：外部可变操作（如 rizin 命令）会先使缓存失效、执行后**退役目标**，要求重新 open（防止后备写入者污染分析结果）。
2. **AnalysisProviderRegistry**（`AnalysisProviderRegistry.ts`）：候选发现 + **确定性选择**，"注册顺序绝不影响选择或输出"。
   - 选择来源三档：`request`（调用方指定）/ `environment`（REA_ANALYSIS_PROVIDER）/ `auto-single-candidate`（自动且**仅当恰好一个候选可用**）。
   - 多个可用候选 → `ambiguous` 错误（**没有优先级回退**，这是刻意的 fail-closed 设计）。
   - 每个 provider 的 capabilities 必须与其 identity 一致，重复 operation 直接 `TypeError`。
3. **SessionProviderRouter**（`SessionProviderRouter.ts`）：把一个已选中的"深"provider 与若干"辅助"provider 组成 **CompositeProvider**；深/辅 operation 集合必须**不相交**（证明编译期检查）。
4. **AnalysisSnapshotCache**（`AnalysisSnapshotCache.ts`）：target+profile+operation+参数 → 结果的不可变缓存，上限 10,000 条。
   - 可缓存性判定 `isSnapshotCacheable`：非 stateful、非跟随光标的查询、`effects.mutatesArtifact === false`、`cachePolicy !== "live"`。
   - 回放结果会附带 limitation："Loaded from a local REA analysis snapshot; this call did not re-run the provider."
   - **分叉安全**：任何 `mutatesArtifact` 的能力在 dispatch 前就 `invalidateSnapshot()`（"failed reply cannot prove a dispatched metadata write rolled back"）。
5. **InvestigationRecords**（`src/application/investigation/`，由 `BinarySessionRecords` 协调）：会话内 **Evidence 账本 + ResidualUnknown 注册表**；`EvidenceLedger` 与 `Unknown` 有明确 owner，记录变更通过 listener 通知 MCP（如 `listChanged`）。
6. **Evidence**（`src/domain/evidence.ts`）：一条观察记录的严格 schema——
   `evidence_id` = 语义投影的 canonical digest（`ev_<sha256>`，解析时重新计算并拒绝被篡改的 ID）；
   字段含 subject（name/digest/format/architecture）、provider、analysis_profile、operation/parameters、raw_result、normalized_result、`confidence ∈ {observed, derived, inferred}`、`authority ∈ {shipped-artifact, controlled-replay, historical-reference, external-service, analyst-inference}`、environment、limitations、relationship_coverage、locations、evidence_links。
7. **结果与错误代数**：全部端口返回 `Result<T, AnalysisError>`（`src/domain/result.ts`）；错误是 tagged union（`*Error` 类），带 `_tag`、`cleanupIncomplete` 标记、可投影为用户可见消息（`analysisErrorProjection.ts`）；**验证在边界做一次**（Zod strictObject），内部信任类型。

## 4. 一次 MCP 工具调用的完整路径（核心逻辑）

以 `analyze_function` 这类"增强工具"为例（**事实**，源码如下）：

```
agent → MCP tools/call <name>
 └─ server/register*Tools  绑定 contracts 中的 ToolContract（name/title/input/output schema/effects/annotations）
     └─ createEvidenceToolHandler (contractToolHandler.ts)
         ├─ runAdmittedToolOperation  ← session.withAdmittedAnalysis()：目标生命周期持锁 + 取消
         ├─ service.execute(operation, input, {signal})   ← application 服务/增强工作流
         │    └─ BinarySession.execute(name, params)
         │        ├─ prepareSessionExecution：检查绑定/能力/可用性；unbound → ProviderSelectionError
         │        ├─ lookupSnapshot(...)  命中则直接回放（附 limitation）
         │        ├─ active.client.execute(...)  → CompositeProvider → 具体 provider client
         │        │     └─ HopperClient/GhidraClient：socket/MCP 传输 → bridge（Python/Java）
         │        ├─ commitExecutionProfile + bindExecutionTarget（把结果绑定到目标身份）
         │        ├─ 运行时可观测性：operationHealthSnapshot → availability 变更通知
         │        └─ createEvidence(...) → recordEvidence + recordSnapshot
         └─ delivery.toEvidenceToolResult(evidence, contract, recorded)
             ├─ 超预算 → 不截断，改为 AnalysisResourceConstraintError + recovery 建议（重试用 continuation / 导出 evidence / 调大 budget）
             └─ returnsTargetContent → 加 _meta.provenance.untrusted_content 标记（防提示注入）
```

要点：
- **"目标内容是数据不是指令"**：server 的 `UNTRUSTED_CONTENT_INSTRUCTIONS` 明确要求 agent 忽略目标内容里的指令（createServer.ts:145-147）。
- **结果不截断**：超过 stdio 预算时返回错误 + 恢复路径，而不是悄悄截断（toolResult.ts）。
- **缓存/证据/快照三者在同一处提交**：只有成功执行且可缓存时才写 Evidence 与快照，保证"缓存 = 曾经真实观察过的证据"。

## 5. 外部引擎接入模型（Provider 进程与桥）

**事实**：
- 每种 GUI/原生引擎都靠 **bridge 脚本注入**到对方进程内：
  - Hopper：`bridge/hopper_bridge.py`（1173 行）在 Hopper 的 Python 线程里跑；REA 用 `hopper --analysis -Y bootstrap.py --executable <target>` 启动，bootstrap 把 `REA_SOCKET/REA_TOKEN/REA_RUN_ID` 注入后 exec bridge（BridgeLauncher.ts:bridgeBootstrapSource）。
  - Ghidra：`bridge/ghidra/ReaGhidraBridge.java`（4878 行，含多个 HeadlessScript/插件类）；IDA：适配外部 `ida-pro-mcp`；JADX/Apktool/Binwalk/Unblob/TShark/sigrok/WABT/EVMole… 类似。
- **传输协议**：Unix domain socket 上的 NDJSON，Zod 严格校验（`src/hopper/protocol.ts`：response/event 两种消息、progress 事件带 completed/total 校验）；错误投影为 `HopperRemoteError`。
- **进程所有权**（AGENTS.md:35-40；src/process/）：自建进程组 + run-token + 私有目录；"PID + 可执行路径不足以证明退出/复用后的所有权"；失败的清理必须**保留所有权**并在下次重试（`#pendingCleanup`、`cleanupIncomplete`）。
- **并发/租约**：Hopper 目标级 lease（`HopperTargetLease`），同一目标已在别的 REA 会话打开时**直接报错**而不是抢。
- **背压/队列**：`HopperRequestQueue`、`HopperResponseStream`（流式响应、进度事件）。

## 6. 数据流总览（推导图）

```
             ┌────────────┐        ┌──────────────┐
 agent ──MCP─► EvidenceMcpServer ──► application 工作流 ──► BinarySession
             └────────────┘        └──────────────┘            │
 terminal ──CLI─► cli/*Commands ───────────────────────────────┤
                                                               ▼
                                     SessionProviderRouter (CompositeProvider)
                                       ├─ 深 provider：Hopper | Ghidra | IDA（注册表确定性选择）
                                       └─ 辅助 provider：browser/observations…（与深 provider 操作集不相交）
                                                               │
                                                               ▼
                                          provider client（socket / MCP / stdio / 直接静态解析）
                                                               │
                                                               ▼
                                          外部引擎或纯本地分析（bridge 脚本 / 自带解析器）
                                                               │
                                                               ▼
                                 Result<AnalysisExecution, AnalysisError>
                                                               │
                              createEvidence（语义 digest）→ InvestigationRecords（Evidence + Unknown）
                                              + AnalysisSnapshotCache（可回放绑定）
                                                               │
                                                               ▼
                                    MCP: structuredContent + 文本 + _meta 标记 / CLI: JSON 输出
```

## 7. 明确的设计原则（writer 的取舍，事实）

- **fail-closed**：不确定就报 unknown/unavailable，绝不猜（AGENTS.md:41-44；`AnalysisProviderRegistry` 无 fallback、`no fallback` 注释）。
- **证据可追溯**：一切结论带 evidence_id、provider 身份、analysis profile、limitations、coverage；能回放就标"来自快照"。
- **一个资源一个 owner**：清理失败保留所有权；不得用下游补救掩盖上游表示缺陷（AGENTS.md:31-38）。
- **性能取向**：先省 agent token，再省分析时间，最后省 CPU/内存（AGENTS.md:60-72）；MCP 有响应预算、分页（`inspect_analysis_view` continuation）、保留证据引用。
- **变更须幂等 + 有备份**：setup/配置变更必须 additive/idempotent/backup（AGENTS.md:24-30）。
