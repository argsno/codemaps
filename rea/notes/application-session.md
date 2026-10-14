# 模块笔记：Application 层（会话、路由、记录、工作流）

规模：`src/application/` 319 个 TS 文件。它是 CLI 与 MCP **共享**的工作流层。

## 1. 端口合同（事实，src/application/AnalysisProvider.ts）
- 一个分析 provider 要提供：`identity()`、`capabilities()`（CapabilityDescriptor：operation + effects(7 项，如 mutatesArtifact/mayWriteFilesystem/changesPermissions) + cachePolicy + limitations）、`inspectAvailability()`、`inspectTargetSupport(target)`、`resolveAnalysisProfile(target)`、`createClient(target, profile, context)`。
- `AnalysisClient.execute(operation, params, options)` → `Result<AnalysisExecution, AnalysisError>`；`AnalysisExecution` 含 result/rawResult/provider/limitations/locations/subject/relationshipCoverage。
- 计时/取消：`ExecutionOptions.signal`（AbortSignal 贯穿全程）、`progress`（进度回调 → MCP progress 通知）。

## 2. BinarySession：唯一活跃目标（事实，BinarySession.ts 988 行）
职责与不变量：
- **单一活跃绑定** `SessionBinding = {target, client, profile, route, runId, nativeSourceAdmitted}`。
- **目标切换串行化**：`#serialize()` 生成 transition generation；调用注册时带 generation，切换时 `#drainCalls(admittedThrough)` 只等待"本次切换之前已受理"的调用（:807-845）。
- **失败回滚**：切换失败 → `#failedOpen` → `#restore` 用同一 route 重新 `createClient` + `health`；恢复失败则把两个错误合成 `cleanupIncomplete`（:601-621）。
- **保留快速路径**（针对 #1719 bug）：同一 target+profile 重开时先探 `operationHealthSnapshot()`；只有 `idle|busy` 才算活着（`exited/unreachable/unknown/not_started` 都不是证据），否则走完整重建（:590-620 注释 + :869-... `#retainedClientLive`）。
- **外部变异**（`withAdmittedExternalMutation`）：先 `invalidateSnapshot()`，执行操作，然后**退役目标**并清记录——理由写在注释里："协议回复不能证明后台写入者已停止"；Ghidra 活跃时直接拒绝（其临时分析库的编辑会丢）。
- **关闭语义**：`close()` 会清空会话记录；`closeWithSnapshot(path, overwrite)` 在同一个生命周期锁里"排水 → 导出快照 → 写文件 → 关闭"，返回 `SavedAnalysisSnapshot{path,bytes,primitive_entries,workflow_entries,evidence_records}`。
- **调用执行**（`#execute`，:760-840）：等待 transition → `prepareSessionExecution`（能力/绑定校验 + 快照命中）→ 命中则回放 → 否则 `client.execute` → `commitExecutionProfile` + `bindExecutionTarget`（把结果绑定目标身份）→ callers/callees 无覆盖信息时补 `relationshipCoverage: unknown` → 运行时可用性观察 → 若可缓存：`createEvidence` + `recordEvidence` + `recordSnapshot`。
- **运行时可用性**：provider 抛 ProviderAdapterError/Hopper*Error 时按 provider 粒度标记其所有 operation 不可用，并通知 `onAvailabilityChanged` listener（:860-905）。

## 3. 路由与选择（事实）
- `AnalysisProviderRegistry`（见 architecture.md）：构造期校验 id 唯一、capability provenance 与 identity 一致、operation 不重复；`select()` 三档来源，多个可用 → `ambiguous`（**无回退**）；`candidates()` 给出完整候选状态（availability + targetSupport + capabilities + selected）供 agent 解释。
- `SessionProviderRouter`：`selectable(registry, auxiliaryProviders)`；校验深/辅 operation **不相交**；`route = {identity, capabilities, profile, binding, createClient}`；无深 provider 时用 `emptyClient`（health 正常、其余报 `AnalysisCapabilityUnavailableError`）。
- `CompositeProvider`：把辅助 provider 集合与绑定 provider 合成一个 client，operation 分发到声明它的 provider。
- `LazyAnalysisProvider`：辅助 provider 惰性加载（构造期零 I/O，失败降级为工具级不可用）。

## 4. 快照缓存（事实，AnalysisSnapshotCache.ts）
- 上限 `SNAPSHOT_CACHE_ENTRY_CEILING = 10_000`（primitive + workflow 合计）。
- 可缓存判定 `isSnapshotCacheable`：非 stateful（health/current_*/goto/list_documents 排除）、非"跟随光标默认地址"类操作（除非显式给 address）、`effects.mutatesArtifact === false`、`cachePolicy !== "live"`、`mayWriteFilesystem===false` 或 `cachePolicy==="snapshot"`、`changesPermissions===false`。
- `referencesTruncated === true` 的结果**不记录**（回放会把截断当完整）。
- 回放时追加 limitation："Loaded from a local REA analysis snapshot; this call did not re-run the provider."，并重绑 subject 的 local_path。
- 失效：`mutatesArtifact` 的能力在 dispatch **之前**失效；`invalidateSnapshotForExternalMutation()` 供外部命令路径调用。
- `workflow_entries`：同样可缓存"组合工作流"结果（由 server 通过 `recordWorkflowSnapshot` 传入），与 primitive 条目分开计数。

## 5. 证据账本与未知注册（事实）
- `BinarySessionRecords`（继承自 BinarySession）：快照 + 写资源（ArtifactResourceScope）协调器；`recordEvidence` 幂等（added/duplicate）；快照/证据变更通过 listener 广播（`onAnalysisSnapshotChanged`）。
- `EvidenceLedger`（498 行）：
  - `record()` 幂等且**冲突内容被拒绝**（`Conflicting evidence record`）；比较时忽略 `name`/`local_path` 等非身份字段（:481-493）。
  - `import()` 支持原子合并 evidence bundle（含 unknown 图校验 `validateResidualUnknownAddition`），返回 `{recordsAdded, unknownsAdded, changed, metadataChanged}`。
  - 未知项：`recordUnknown` 需带 mutation evidence；`updateUnknown` 乐观并发；`listUnknownsPage` 稳定分页（ID 排序 + next_offset）。
- `InvestigationRecords`：把 ledger 与 report/unknown 查询组合给 MCP 的窄端口（`InvestigationRecordPort.ts`）。

## 6. 其它 application 服务（事实：文件清单 + 名称）
- 直接分析（CLI 主路径）：`DirectAnalysis.ts`、`DirectAnalysisDependencies/Cleanup/Status`、`ProviderOperationExecution.ts`、`AnalysisContextQueries.ts`、`CallPathTracing.ts`、`ProcedureAddressResolution.ts`、`XrefsAddressResolution.ts`。
- 安装与维护：`Setup.ts`（+Planning/Recovery/Skill/Host/ClientConfiguration…）、`Uninstall.ts`、`Update*.ts`（含 `UpdateRuntime.release.ts`）、`Doctor*.ts`、`CapabilityInventory.ts`（能力清单 + remediation 文案）、`SupportedClients.ts`（25 个客户端定义）。
- 证据流转：`EvidenceBundleCommands/Files`、`EvidenceInputResolver.ts`、`WorkflowEvidence.ts`、`ReferenceSourceImport*.ts`（把参考源码导入为可分析的 reference source）。
- 重建/分析视图：`ReconstructionReadiness*`、`ReconstructionObligationLedger*`（含 authority/closure/evaluation/diagnostics 多件套）、`FunctionAnalysisEvidence.ts`、`AnalysisDocument.ts`。
- 目标解析：`BinaryTargetResolver.ts`、`AppBundleExecutable.ts`、`MipsElfMetadata.ts`、`MipsElfSharedLoad.ts`。

## 设计要点（推断）
- 这一层把"会话/证据/缓存/选择"这些**跨 provider 的不变量**集中实现，provider 只负责翻译协议；这样 20+ provider 才能共享同一套"证据可信度"语义。
- 大量文件名自带 `.test.ts`（如 `Setup.planning.test.ts`、`EvidenceLedger.*.test.ts`），说明这层是测试重点。
