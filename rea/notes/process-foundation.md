# 模块笔记：进程与资源所有权基座（process / windows / config / evaluation）

> 基于并行探索代理调研；抽查通过。

## 1. `src/process/` 核心原语（事实）

| 原语 | 位置 | 解决的问题 |
|---|---|---|
| `OwnedProcessGroup` + 清理计划 | `ProcessOwnership.ts:16-33,240,409,907` | 声明"我拥有哪些进程"：runId + leaderPid + pgid + expectedCommand/expectedParentPid；清理前逐成员 token 校验，任一失败 fail-closed |
| `verifyNoTokenOwnedProcesses` | `:301` | 清理后复查"没有活进程还带我的 run token" |
| `ProviderProcessSupervisor` | `ProviderProcess.ts:273` | 生命周期监督：TERM→KILL 分级、有界诊断；stop 未完成保留 owner 供重试 |
| `runOwnedCommand` | `OwnedCommand.ts:55` | 短命令：deadline + 输出预算 + 独立 cleanup，失败带 cleanupResult/resources |
| 共享绝对 deadline | `ProviderDeadline.ts:31` | 单调时钟 deadline + caller signal；区分 cancelled/timeout |
| 私有根 | `PrivateRuntimeRoot.ts:29,45` | POSIX `mkdtemp` owner-only；Windows 需原生 DACL 证明，否则拒绝 |
| 有界读/诊断 | `BoundedFileBytes.ts:4`、`ExecFileBoundedOutput.ts:11`、`ProviderDiagnosticRedaction.ts` | 溢出即 undefined/头尾保留；凭证脱敏 |
| host 观测 | `ProcessOwnershipObservation.ts:220,450` | `ps` 列表、`/proc/<pid>/environ`、身份批读、`process.kill(-pgid)` |
| 清理重试 | `ArtifactResourceScope.ts:52`、`ArtifactAnalysisResources.ts:35` | 失败的 owner 留在 pending，下次 close/操作前重试 |

## 2. 所有权模型（核心设计，事实）
- **run token** = 注入子进程环境变量 `REA_PROCESS_RUN_ID`（`ProviderProcess.ts:184` 附近；capture 侧 `capture/ProcessCaptureEnvironment.ts:23`）。
- **校验**：读回进程环境（批量优先，逐进程回退）；可读但不等于 runId ⇒ `run-token-mismatch`；不可读 ⇒ `environment-unreadable`（**不跳过**，作为证据记录）。
- **PID 复用防护四层**：
  1. token 读取前后各取一次 start identity，变化即失败（`ProcessOwnership.ts:731-783,1034-1064`）；
  2. Linux 用 `stat` 第 22 字段 `starttime`（`ProcessOwnershipObservation.ts:196-209`），Darwin 走原生 helper（`DarwinProcessRunTokenReader.ts`）；
  3. `captureBaseline` 记录 launch 前 pid→identity，扫描时丢弃 identity 未变者（`:329-364`）；
  4. 单 PID 信号前再校验 start identity（`:530`）。
- **退场竞态**：要求"新快照证明 PID 已消失"才能把 mismatch 归因于退出（`:361-407`）；只有"PID 不存在"能消解不可读（`:816-857`）。
- 设计哲学（AGENTS.md:35-40 明文）：**PID + 可执行路径不足以证明所有权**；`unverified` 是证据不是失败；不把 OS 原文当授权。

## 3. Windows 宿主边界（事实）
- 加载器只加载包内 artifact（校验 packageVersion / Node-API ABI / PE x64 / SHA-256；`WindowsNativeLoader.ts:96+`），结果 Zod 严格校验；缓存但可重试失败。
- 能力面三件套（`WindowsAuthority.ts:29-33`）：`job_object_process_ownership`、`private_runtime_dacl`、`reparse_safe_path_admission`——明确声明"不由 taskkill/chmod/符号链接推断"。
- 原生 addon（`native/windows/src/`，C++）单入口 `call(operation, args)`；句柄用 `napi_type_tag_object`+wrap，**伪造/序列化句柄拿不到权限**。
- 为什么必须原生：CREATE_SUSPENDED + `PROC_THREAD_ATTRIBUTE_JOB_LIST` 原子入 Job（`process.cc:115,162`）、`KILL_ON_JOB_CLOSE`、breakaway 检查、内存限额消息；NTFS reparse/卷序列号+文件 ID admission、创建时装私有 DACL。
- 重试纪律：只有 `win32Code ∈ {32,33}` 且 `runtimeAllocation: "not-started"` 才算可重试（`WindowsPrivateRuntime.ts:29-33`）。

## 4. 配置面（事实）
- `config/environment.ts:22-76` 单一 Zod schema；错误只回 setting+constraint，**不回值**（:92-102）；跨字段规则在 `:124-148`。
- 主要变量：`REA_ANALYSIS_PROVIDER`、`REA_MCP_MAX_RESPONSE_BYTES`、`REA_MCP_INPUT_SCHEMA_PROFILE`、`REA_MCP_OUTPUT_SCHEMA_PROFILE`、`REA_IDA_MCP_CONFIG`、`REA_GHIDRA_*`（超时/语言/compilerSpec/禁用分析器/seed/nativeaot）、`REA_HOPPER_*`、`REA_LOG_LEVEL`、`REA_REFERENCE_SECRET_PATTERNS_JSON`、`GHIDRA_INSTALL_DIR`、`JAVA_HOME`；路径类强制绝对路径。
- `snapshotEnvironment`：**冻结快照**（含 Windows 大小写不敏感键的去重语义），在 main/server/cli 入口统一冻结——防止运行中环境漂移。
- **推断**：配置分层——全局门（parseConfig）+ 各 adapter 自读（如 `REA_JADX_MCP_JAR`、`REA_EVM_PRLIMIT_COMMAND`、`REA_DARWIN_SWIFTC_PATH`）；前者管跨域规则，后者管局部工具发现。

## 5. evaluation（事实）
- Codex CLI 作为外部 agent 的**发布前评估**：解析 JSONL 转录里的 MCP 调用（`CodexAgentEval.ts:11-47`），用封闭 fixture 断言（`KnownAnswerEvaluation.ts:8-34`，要求结论有 Evidence 支撑），门槛 `agentEvaluationPassed`（`AgentEvaluationReport.ts:18-35`）。
- 驱动：`scripts/verify-agent-experience.mjs`（隔离 HOME/CODEX_HOME，产物落 `~/.cache/rea-agent-evaluations`）。

## 6. 测试验证（事实）
- vitest 项目：`process-boundary` = `tests/boundary/process/**`（**串行** `fileParallelism:false`，共享宿主采样）；`process-global` = `tests/process-global/**`；runner `tests/processOwnershipRunner.ts` 在 worker 结束回收原生 host。
- 真跑：`tests/boundary/process/ownedCommand.test.ts` 真起进程验 deadline；另有 providerProcess、processEvidence.host、darwinProcessRunTokenReader 等约 40 文件。
- 假宿主单测：`src/process/ProcessOwnership.{validation,identity,observation,runId}` 注入 `ProcessOwnershipHost` 覆盖 token mismatch / 身份变化 / unreadable。
- Windows：mock native call 只验"证据充分性"；真实验证在 `scripts/verify-windows-native*.mjs` 与 CI。
