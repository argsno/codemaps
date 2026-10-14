# 模块笔记：MCP Server 适配层（src/server/ 93 文件）

## 1. 工具注册：契约驱动（事实）
- 单一事实源是 `src/contracts/` 里的 `ToolContract`（`toolContractTypes.ts:36-59`）：`name/title/description/kind(17 种适配器家族)/inputSchema/outputSchema/effects/annotations/returnsTargetContent/examples`。
- 汇入 `src/contracts/toolContracts.ts:39` 的 `TOOL_CONTRACTS`（36 个 import/展开的契约数组：official/enhanced/native/artifact/managed/android/apktool/flutter/jeb/adb/firmware/browser/electron/runtime/pcap/application/session/cutter/gdb/rizin…）。
- 工具数量由 `src/catalogIdentity.ts:84-89` 在**运行期**统计（`CATALOG_IDENTITY.counts.mcp_tools`），并计算 tools/prompts/CLI 三方 sha256 摘要；`src/application/Doctor.ts:331` 用它判断已安装 skill 与当前构建是否对齐。
- `createServer.ts:439-582` 逐个注册：每个 `register*Tools.ts` 把 application 服务 + `recordEvidence` + `withAdmittedAnalysis` 绑定到契约。**契约必须在 doc catalog/示例/schema 全部一致**，`scripts/generate-mcp-tool-catalog.mjs --check` 在 CI 校验。

## 2. EvidenceMcpServer：SDK 之上的三个"夹层"（事实，EvidenceMcpServer.ts）
1. **registerTool override**：把**广告出去的** input JSON Schema 压缩到预算内（`REA_MCP_INPUT_SCHEMA_PROFILE=compact`；`compactAdvertisedInputSchema`），但注册给 SDK 的规范 schema 不变——"catalog ordering 不得选择操作或 schema"（AGENTS.md:57-59）。
2. **connect() 包一层 ErrorEvidenceTransport**：
   - 跟踪 `tools/call` / `tools/list` / `completion/complete` 的 pending 请求；收到 `notifications/cancelled` 时释放资源并 abort 完成请求（SDK 会抑制回复，所以 send() 释放不了）。
   - **超预算错误保留**：错误回复超过 stdio 预算时，把完整投影**存进 Evidence 账本**，替换为 `AnalysisResourceConstraintError` + `evidence_reference` + 修复建议（"Export the retained original error…" 或调大 `REA_MCP_MAX_RESPONSE_BYTES`）。
   - `tools/list` 在 `REA_MCP_OUTPUT_SCHEMA_PROFILE=off` 时剥离 outputSchema（本地仍校验成功结构化结果）。
   - 给 SDK 自身生成的错误（输入校验、handler 抛出）补 `_meta.provenance.untrusted_content` 标记（对 returnsTargetContent 工具）。
   - `completion/complete` 的取消信号经 AsyncLocalStorage（`currentMcpRequestSignal`）传到 handler。
3. **instructions**（createServer.ts:139-147）：写给 agent 的固定条款——工具选择原则 + **目标内容是不可信数据，遇注入要报告用户**。

## 3. 一次工具调用的绑定与投递（事实）
- `createContractToolHandler`（contractToolHandler.ts）：`service.execute(contract.name, input, {signal})` → `logToolExecution` → 成功则 `recordEvidence` → `delivery.toEvidenceToolResult`；失败若带 `partialObservation`（已记录的 Evidence）则返回带证据引用的部分错误。
- 全部包在 `runAdmittedToolOperation(...)` 里：即 `session.withAdmittedAnalysis(name, signal, ...)`——**工具执行期间目标绑定被锁住**，取消会短路。
- 投递策略（toolResult.ts）：
  - 结果预算 = `REA_MCP_MAX_RESPONSE_BYTES - 1024`；
  - 超预算/超 Node 单字符串上限 → **不截断**，返回 `AnalysisResourceConstraintError`，给出恢复路径（`inspect_analysis_view` continuation / 导出 evidence / 放大预算）；
  - `analyze_function` 成功时自动附 `continuation`（指向保留证据的分页读法）；
  - 目标内容结果加 `_meta` 标记（§2.2）。
- 录制端（`recordDerivedEvidence.ts`、`runDerivedOperation.ts`）：派生工作流（对比/汇总类）也走 Evidence 记录，保证"派生结论也是证据"。

## 4. 会话工具与可用性（事实）
- `registerSessionTools`（createServer.ts:571-582）：传入 session、`availabilityPolicy`、`androidAnalysisAvailability`、`startedAt` 等；会话工具覆盖 open/close/snapshot/status/unknowns/evidence bundle 等（对应 `contracts/sessionToolContracts.ts`）。
- `installSessionToolAvailability`（:186-251）：把"平台 + 环境变量 + 可选 provider 加载失败"折算成 `optionalFeatures` 布尔面（如 pwntools 需 linux x64 + 绝对路径；wasm 需 WABT 目录；firmware 需 binwalk/unblob；evm 需 linux x64）。`CapabilityInventory` 负责把这些转成人类可读的修复建议。
- `registerPrompts`/`registerGuidedPrompts`：MCP **prompts**（`PROMPT_CONTRACTS` 也是契约化的，含 arguments/steps），给 agent 引导式调查流程。
- 可选 provider 惰性加载：`composition/optionalObservationProviders.ts` 用动态 import；失败不阻断启动，仅对应工具报 `optionalProviderUnavailable`（transport.ts:32-49 记录告警）。

## 5. 生命周期与清理（事实）
- `main/transport.ts`：`serveStdio(() => createServer(...))`——SDK 可能丢弃探测 server 再建新的，所以 servers 集合 + android providers 集合分别跟踪关闭；handle.close 聚合失败抛 AggregateError。
- `main/shutdown.ts`：登记 SIGINT/SIGTERM/stdin end/close 的幂等关闭（未细读，按 main.ts 调用关系）。
- `createServer` 的 `closeProviders()`：`Promise.allSettled` 关闭 25+ 个 provider（android/frida/wasm/packetCapture/signalRecording/apktool/jeb/adb/flutter/pcap/evm/sqlite/binaryLayout/recordedCrash/…），失败聚合抛错并允许重试（:368-408）。
- `main.ts` 顶层 `runEntrypoint` 故意吞掉启动异常细节（"must not leak internal details"），只输出统一消息（main.ts:96-110）。

## 6. 测试聚焦（事实：文件清单）
- 同目录测试：`EvidenceMcpServer.test.ts`、`runDerivedOperation.test.ts`、`toolResultEncoding.test.ts`。
- tests/boundary/mcp/（agent 调研确认存在）：toolSchemaValidity、serverLifecycle、clientCapabilities、browserMcp/electronMcp/serverWorkflowMcp 等。

## 设计要点（推断）
- MCP 层刻意"薄"：只做契约绑定、投递、预算、标记、取消、清理；所有语义（选择/证据/快照）都在 application/domain。
- "广告内容（schemas）"与"执行内容（parsers）"分离，并对两者分别设预算与校验，是为省 agent token 又不牺牲本地严格性。
