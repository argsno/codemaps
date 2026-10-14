# 模块笔记：深层原生引擎 provider（Hopper / Ghidra / IDA）与 bridge

## 0. 三位"深 provider"的位置（事实）
- 只有它们实现 `AnalysisProviderCandidate` 并进 `AnalysisProviderRegistry`（`composition/binary.ts:29-36`）：`HopperProvider`、`GhidraProvider`、`IdaProvider`。
- 选择规则见 architecture.md：请求 > 环境变量 > auto-唯一候选；多候选即 ambiguous。
- 其它工具（objdump/rizin/gdb/cutter）走 `composition/reverseEngineering.ts` 单独装配（`REA_*_COMMAND`），不是深 provider（agent 调研 + package.json scripts 中的 verify:*）。

## 1. Hopper（src/hopper/ 30 文件 + bridge/hopper_bridge.py 1173 行）
- **启动模型**（BridgeLauncher.ts）：
  - 通过 Hopper 官方 CLI：`hopper <loaderArgs> --analysis -Y <bootstrap.py> --executable|--database <target>`（`#launchPreparedTarget`）。
  - REA 写一个 0600 的 bootstrap.py，注入 `REA_SOCKET`（unix socket 路径）、`REA_TOKEN`（随机）、`REA_RUN_ID`、`REA_TARGET_PATH`、`REA_OWNS_PROCESS_LIFETIME`、重连超时，然后 `exec` bridge 脚本（`bridgeBootstrapSource`）。**非成功路径一律删除含 token 的 bootstrap**。
  - **所有权边界写在注释里**：REA 只拥有"短命 launcher helper"的进程组；macOS LaunchServices 建的 Hopper GUI 进程**不声明所有权**；vendor launcher 可能把 Hopper 激活到前台，属于上游约束而非分析失败。
  - macOS：先用 `open --hide --background` 降低激活；校验原生 helper 的发布者（`verifyMacHopperNativeHelper`）；仅当参数无引号/反斜杠/换行时才敢下"授权拒绝发生在 handoff 前"的结论。
  - Linux：实验性 demo lane 用 `verified_linux_demo` + 私有 X11（`unshare --user --map-root-user --mount` 或直跑 python helper），helper 观察 stdin EOF 以在 REA 被杀（含 SIGKILL）时退出。
  - **目标租约**：`acquireHopperTargetLease`——同一目标已在别的 REA 会话打开 → 直接 `HopperStartError`（"Use that REA session or close it before opening this target again."）。
- **协议**（protocol.ts）：NDJSON；消息二选一：`{id, result}` 或 `{id, error{code,message,type}}`，另支持 `event`（progress/diagnostic，progress 校验 completed ≤ total）。Zod `strict` 全量校验；错误分类 remote/authorization/invalid_request/capability_unavailable/bridge_exception。
- **bridge 侧**（bridge/hopper_bridge.py）：在 Hopper 专用 Python 线程执行（注释警告：移到 worker 会死锁 Hopper）；含 `HopperApiFacade`（可测边界）、文档/伪代码/注释/搜索缓存等；遇 `backgroundProcessActive` 时拒绝未完成分析的操作。
- 其它支撑：`HopperRequestQueue`（串行队列）、`HopperResponseStream`（流式）、`HopperDiagnostics`、`HopperMachOImage`（fat 瘦身）、`HopperProcessDiagnostic`、`HopperCleanup`、`HopperMacPublisher`、`LinuxPrivateDisplay*`。
- application 层还有 `MacHopper.ts`/`LinuxHopper.ts`/`HopperDownload.ts`（安装与定位 Hopper）。

## 2. Ghidra（src/ghidra/ 68 文件 + bridge/ghidra/*.java 4878 行）
- **定位**：headless、**临时数据库**、不改可执行字节、不控制 GUI（AGENTS.md:21-23 明文授权边界）。
- `GhidraProvider.ts`（395 行）：
  - `inspectAvailability()` 来自 `inspectGhidraInstallation`（安装/Java 校验），Windows 还要过 `inspectWindowsNativeReadiness`（原生插件的 NTFS/DACL/Job 能力）。
  - `inspectTargetSupport()`：支持 x86/x86_64/arm/arm64/mips/6502/riscv/powerpc 的 executable；**universal Mach-O 需先瘦身**；PSP Allegrex 只在 Linux x64；Windows P0 只收原生 PE（x86/x86_64）+ 显式 raw 6502，且需 executableRole ∈ {application, shared-library} 且 managed === false。
  - `resolveAnalysisProfile()`：语言覆盖需通过 `admitGhidraLanguageOverride` 对着**已安装 Ghidra 的语言目录**做准入校验；再合并 extensions 与 `REA_GHIDRA_SEED_FILE` 的 seeds（可复现分析的"种子"）。
- `GhidraClient.ts`（999 行）+ `GhidraLauncher/JavaLaunch/JavaProperties/SessionRoot/SessionValues/SnapshotCache/Transport(RequestQueue/ResponseRouter/ResponseBuffer/SocketConnection)`：Unix/Windows 传输、会话根目录（临时库）、快照缓存、请求队列、响应路由。
- bridge Java 侧有多个角色：`ReaGhidraBridge`（inventory/decompiler/references/CFG）、`ReaGhidraPrepareCom`、`ReaGhidraApplySeeds`、`ReaGhidraAnalysisOptions`、`ReaGhidraNoReturnFix`、`ReaGhidraMipsSoftFloat`、`ReaGhidraPrepareRaw`、`ReaGhidraExtensions`/`BundleActivator`（package.json files 列表）。
- 支持矩阵文档：docs/ghidra-{dos,mips,ppc,riscv,psp,nativeaot}.md、docs/windows-ghidra-p0.md；verify 脚本按架构各有一条（package.json:186-204）。
- **外部变更纪律**：Ghidra 活跃时禁止 `withAdmittedExternalMutation`（BinarySession.ts:383-390），因为其"临时库"的编辑会被丢弃。

## 3. IDA（src/ida/ 33 文件）
- **BYO + 适配上游 MCP**：`REA_IDA_MCP_CONFIG` 指向一份上游 IDA-MCP 注册 JSON；两种模式：attached（连 GUI 的 MCP）与 headless（由 REA 监管数据库），协议 native 或 legacy-1.4（IdaProvider.ts:76-100）。
- availability：headless 还需私有运行根可用（Windows 需原生 DACL 支持），否则 `unavailable`。
- target 限制：executable；MIPS/RISC-V/PowerPC 无验证通道；raw-binary 不接受（提示改用 Ghidra）；不支持打开分析数据库（要原始输入）。
- profile 里记录 `registration_digest`（canonicalize 配置的 sha256）——**配置本身参与身份**。
- 别处有 `IdaWorkspace/WorkspaceOwnership/WorkspaceThinning`、`IdaCallGraph`、`IdaDiagnosticRedactor`（诊断脱敏）、`IdaBusyHealth`。

## 4. 其他原生/调试工具（事实：composition/reverseEngineering.ts + contracts）
- objdump（GNU objdump 命令 profile）、rizin（静态命令 + **持久调试会话** `RizinDebugSessionManager`）、gdb（`GdbSessionManager`，GDB/MI 持久会话）、cutter（`CutterBridgeClient/Service` + bridge/cutter/rea_bridge.py，需要 `REA_CUTTER_BRIDGE_DIR`）。
- 这些不被注册表选择，而是各自 MCP 工具族；会话类（gdb/rizin）有自己的生命周期与「外部变异 → 退役目标」协同（createServer.ts:481-500 传入 session）。

## 5. 所有权与清理（贯穿三者，事实）
- 见 architecture.md §5 与 notes/process-foundation.md：run token（`REA_PROCESS_RUN_ID`）、进程组、start identity（Linux starttime / Darwin 原生 helper）、清理失败保留 owner 待重试。
- bridge 产物（bootstrap.py、session root、socket）都在私有目录（owner-only），关闭时统一清理。
