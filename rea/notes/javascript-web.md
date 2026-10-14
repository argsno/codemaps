# 模块笔记：网页 / JavaScript / Electron 分析线

> 本笔记基于并行探索代理的源码调研（含 file:line 证据），关键处已抽查。

## 1. 子系统构成（事实）
- **`src/browser/`（124 文件）**：四族能力在一个目录里——
  - 被动 CDP 观察：`CdpBrowserProvider.ts`(435)、`CdpConnection.ts`(388)、`CdpEndpoint.ts`(530，loopback 发现)、`CdpPageCapture.ts`(470)、`CdpScreenshot.ts`(163)、按事件类型拆分的 `CdpCaptureEvents*`。
  - 主动 Playwright 场景：`PlaywrightScenarioBrowser.ts`（自建/托管浏览器 + 私有 profile）、`PlaywrightScenarioSession.ts`(440)、`PlaywrightScenarioActions.ts`（**声明式动作**，不执行调用方代码）、`PlaywrightElectronActiveProvider.ts`(495)。
  - 离线抓包：`history/HarCapture.ts`(246)、`SazCapture.ts`(174)、`QlogCapture.ts`(519)、`HistoricalCaptureDecoder.ts`(366)、`CaptureRedaction.ts`。
  - 脚本导出与运行时归因：`assets/PublishWebScripts.ts`(202)、`assets/ScriptCaptureAdapters.ts`、`execution/CdpWebRuntimeProvider.ts`(97)+Sources/Requests、`CdpPreciseCoverage.ts`、`WebSourceMapFetcher.ts`(1095)。
- **`src/inspector/`**：Node/Electron V8 Inspector 被动观察（`V8InspectorProvider.ts` 565），复用 browser 的 `CdpConnection`。
- **`src/javascript/`**：三条子进程线——`analysis/*`（JSON-lines worker：`JavaScriptAnalysisWorkerServer.ts`）、`sourceMaps/*`（一次性 codec 进程 `SourceMapCodecProcess.ts`）、`recovery/*`（外部 Wakaru 二进制，pin 1.14.0）。
- **`src/electron/`**：ASAR/ELF fuse 读取（`ElectronFuseReader.ts` 480、`ElectronFusePath.ts` 307）。
- **`src/artifacts/`**：端口 `ArtifactReader.ts`；实现 `AsarArtifactReader.ts`(729)、`ZipArtifactReader.ts`(426)、`DirectoryArtifactReader.ts`；`artifacts/javascript/*` 只是薄封装。**注意：不是"bundle 反打包"**，"重建"指静态应用图重建（`application/javascript/JavaScriptArtifactReconstruction.ts:68`）。
- **`src/reference/`**：参考源码读取（`ReferenceSourceReader.ts:22`），不跟随符号链接、路径身份反复校验。
- **易混淆点（事实）**：`src/captures/` 是 **pcap（tshark）** 线；HAR/SAZ 在 `browser/history`，应用层服务是 `application/WebNetworkCaptureService.ts`。

## 2. 数据流（事实）
**分析 Electron/JS 应用**：`analyze_javascript_application` → `registerElectronTools.ts:99-118` → `JavaScriptApplicationService` → `JavaScriptArtifactReconstruction.ts:68-119`（扫描 inventory → reader）→ `analyzeAndAdoptJavaScriptArtifactFiles`（694 行）→ **子进程** worker（`@babel/parser` 解析 + 静态/语义分析）→ 图构建（JavaScriptArtifactGraph / JavaScriptSemanticGraph / ElectronBoundary）→ `createOwnedJavaScriptApplicationEvidenceCooperatively`（freeze→validate→hash→`createImmutableEvidenceSteps`）→ Evidence 由 server 记录。

**抓网页脚本**：`inspect_web_page` / `capture_browser_scenario` 产出 Evidence → `export_web_scripts`：先 `parseEvidence` 防篡改（`ScriptCaptureAdapters.ts:58-70`）→ `publishWebScripts`（`SafeOutputTree` 独占写 + manifest + 回滚）。

**读抓包**：`inspect_web_network_capture` → `HistoricalCaptureDecoder`（组合 har/saz/qlog/mitmproxy 适配器）→ HAR 用 lossless-json 保数字字面量 + har-schema 校验；SAZ 用 zip reader（拒绝加密/链接、32 MiB 预算）→ `createEvidence`。

**Electron 主动场景**：`capture_electron_scenario` → `ElectronActiveObservationService`（realpath + 可执行校验）→ `PlaywrightElectronActiveProvider`（60s 启动期限）→ Evidence。

## 3. 分层接口（事实）
- server 只依赖 application 端口：`BrowserObservationPort`(8 方法)、`ElectronObservationPort`、`ElectronActiveObservationPort`、`BrowserScenarioCapturePort`、`WebRuntimePort`、`WebScriptArtifactPort`、`WebSourceLocationPorts`、`JavaScriptRecoveryPort`。
- composition 是唯一 `new` provider 的地方（`composition/browserObservation.ts`、`electronScenario.ts` 等）；5 个可选 provider 用动态 import，失败 → 工具级 `optionalProviderUnavailable`。
- CLI 复用同一 service（`cliBrowserScenarioCommands.ts`、`cliJavaScriptRuntimeCommands.ts`），命令名集中在 `cliCommandNames.ts`。

## 4. 工程决策（事实）
- **资源所有权**：`ArtifactResourceScope` 联合类型持有一切（reader/output-tree/file-handle/private-runtime-root），失败清理挂起到 `close()`；Playwright close 幂等 + 30s 确认；`closeOwnedWebResources.ts` 用 allSettled 报全部失败。
- **进程**：JS 分析在子进程，stdio 控制帧 + 临时文件传源码与结果，逐 job 释放并 `global.gc()`；源码文本用 sha256 对账。
- **超时**：`PlaywrightExecutionBoundary.ts` 统一 deadline+abort；source-map fetch 30s；Electron 启动 60s；Wakaru 1 GiB 地址空间 / 120s。
- **证据**：一律走 `createEvidence`/`createImmutableEvidenceSteps`；取消时保留 partial Evidence。
- **未发现跨调用通用缓存**（推断：确定性优先于复用）；"snapshot" 仅指 reader 内部快照与稳定读。

## 5. 测试与验证（事实）
- browser 内约 40 个 `*.test.ts`；`tests/boundary/browser/` 60+ 文件按面拆分（network/storage/indexed_db/redirectErrors/discovery_authorization…）；`tests/boundary/mcp/{browserMcp,electronMcp,electronFusesMcp,browserScenarioMcp}.test.ts`；假 CDP fixture `tests/fixtures/fakeCdpBrowser*.ts`。
- verify 脚本：`verify-real-browser.mjs`、`verify-real-electron.mjs`（需 `REA_ELECTRON_EXECUTABLE`）、`verify-browser-{scripts,modules,source-maps,runtime,dom-destinations}.mjs`、`verify-{web-network-captures,http3,qlog-captures}.mjs`、`verify/javascript/*`。
