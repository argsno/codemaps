# 模块笔记：长尾分析 provider（移动/固件/字节码/调试）

> 基于并行探索代理调研；抽查通过。

## 1. 两层结构（关键事实）
- **只有深层 BSP 才实现 `AnalysisProvider/AnalysisClient`**：Hopper/Ghidra/IDA（+ 少数静态 provider 如 `ArtifactProvider`、`NativeMacOSProvider`、`ManagedStaticProvider`），由 `AnalysisProviderRegistry` 统一发现/选择。
- **长尾每域一个专属 Port**，不参与深 provider 选择：`AndroidAnalysisPort`、`PcapAnalysisPort`、`WasmArtifactPort`、`EvmInterfacePort`、`SqliteDatabasePort`、`BinaryLayoutPort`、`FirmwareAnalysisPort`、`SignalRecordingPort`、`PacketCapturePort`、`JebAnalysisPort` 等，形状统一为 `inspect/execute + inspectAvailability + close`。
- 装配：`composition/<name>.ts` 工厂（构造期零 I/O），在 `createServer.ts:300-366` 注入/创建，:`368-407` 用 `Promise.allSettled` 集中关闭。

## 2. 代表实现（事实）

### android / JADX
- 工具发现：`REA_JADX_MCP_JAR` 必须**绝对路径** + JDK≥17 且含 jdk.compiler + JAR 必需类清单校验（`JadxConfiguration.ts:266-309,367-395,511-619`）。
- 启动：`JadxMcpTransport` 用 `spawnOwnedProviderProcess` 拥有进程（POSIX 组 / Windows Job），SDK 帧上限 8MiB/32MiB；单会话串行 + idle 退役（`JadxProvider.ts:225-279`）。
- 输入/输出：入口与源码 `hashStableFile`；输出 pinned zod，失败归 `AnalysisOutputError`。

### dotnet（纯静态）
- 无进程、无外部工具，纯 TS 解析 PE/CLI 元数据（`ManagedStaticProvider.ts:50-65`）。
- 执行前重算 sha256 与 target 比对，不符即 `EvidenceIntegrityError`（:110-115）——**防 TOCTOU**。

### evm
- 引擎是 bundled evmole（npm/WASM），在 **owned Node worker** 里跑；调用者只需提供 prlimit 路径（默认 `/usr/bin/prlimit`）；从 `/proc/self/limits` 收窄预算（`EvmoleInterfaceProvider.ts:137-231`）。
- 输出 strict zod + 相对字节码的证据校验。

### pwntools（离线 ELF 布局/崩溃）
- 私有临时根写 request/snapshot/reply，`runOwnedCommand` 拥有进程；payload zod 校验 + **重编码字节与快照逐字节比对**（`PwntoolsLayoutProvider.ts:52-65`）；预算与保留退出码在 `PwntoolsRelease.ts:8-19`。

## 3. 纯静态 vs 外部工具（事实）
- **无外部工具**：dotnet、flutter（`FlutterBuildProvider.ts:514` "Pure parsing; the provider is always available."）、go（bundled reader）、sqlite（内置 `node:sqlite`）。
- **caller-supplied 外部工具**：android(JAR/JDK)、pwntools(`REA_PWNTOOLS_PYTHON`)、firmware(`REA_BINWALK_COMMAND`/`REA_UNBLOB_COMMAND` + prlimit)、apktool(`REA_APKTOOL_COMMAND` 或 PATH)、adb(`REA_ADB_PATH`)、jeb(运行中的 `REA_JEB_MCP_URL`)、wasm(`REA_WABT_BIN_DIRECTORY` 三工具)、pcap(tshark/capinfos)、captures/signals(`REA_TSHARK_COMMAND`/`REA_SIGROK_COMMAND` + prlimit)、gdb/rizin/objdump(`REA_*_COMMAND`)、cutter(`REA_CUTTER_BRIDGE_DIR`)、frida(可选 npm `frida` 动态 import)。
- 一致性：README "Requirements and guide" 表与实际配置面基本对应（截图见 overview.md 的对照）。

## 4. availability 判定（事实）
- 模式统一：**平台闸门 + 绝对路径 + 可执行性**，缺一 → `AnalysisCapabilityUnavailableError`；例：pwntools 限 Linux x64（`PwntoolsDecoder.ts:224-238`）、evm 限 Linux x64、firmware 限 Linux、pcap 限 linux/win32、JADX 支持 linux/darwin/win32-x64。
- 汇总投影到 `createServer.ts:186-232` 的 `optionalFeatures`；人类可读修复建议在 `CapabilityInventory.ts:404-483`。

## 5. 测试与 verify（事实）
- 单测与源码同目录（`src/android/*.test.ts`、`src/native/pwntools/*.test.ts`、`src/wasm/`、`src/dotnet/`、`src/adb/`）+ `tests/boundary/providers/*` + `tests/acceptance/*`。
- 真实环境 lane：`verify:apktool/pcap/flutter/adb/jeb/frida/go:binary/wasm:artifact/evm:interface/binary:layout/recorded:crash` + `scripts/verify/{firmware,android,harmony,managed}/`；signal 录制走 CI workflow（`.github/workflows/real-signal-recording.yml`）。
- 方法论（AGENTS.md 明确）：模拟 provider 不能证明真实 provider 行为，`verify:*` 才是真实通道。

## 6. 观察（推断）
- 长尾 provider 的代码结构高度同构（Provider + Configuration + ProviderCapabilities + Release/Pin + *.test.ts），说明作者用同一套模板复制扩展，新增一个目标类型的边际成本主要在"发现/启动/输出校验/可用性"四件事上。
- "不代装外部工具"是硬约束（除 Hopper 可选装且需批准）：这些 provider 的价值是**接入**而非分发。
