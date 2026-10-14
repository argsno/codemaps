# 证据出处（sources）

> 分析对象：`/Users/argsno/Projects/rea`（分支 main，HEAD `b11fe63bf`，工作区干净，133MB）。
> 分析日期：2026-10-14。全程只读，未改动目标仓库。命令均在该目录执行。

## 概述与定位
| 结论 | 出处 |
|---|---|
| 产品定位 "Reverse Engineer Anything"，MCP + CLI | README.md:6-8, 59-61；package.json:5 |
| 受众与 showcase（DX-Ball / Notion / TH04） | README.md:217-240 |
| 本地分析不上传 | README.md FAQ "Does REA upload my app?"；AGENTS.md:3 |
| BYO 商业工具（IDA/Hopper/JEB）法律边界 | README.md:384-390 |
| setup 支持 Claude Code/Codex/Cursor/Gemini CLI 等 | README.md:84-87；src/application/SupportedClients.ts |
| Node 版本要求 | package.json:44-47 |
| 规模统计（1852 TS 文件 / 413,712 行 / 1232 测试文件 / bridge 32 文件） | 命令：`find src -name '*.ts' | wc -l` 等 |

## 架构与主流程
| 结论 | 出处 |
|---|---|
| 分层与依赖方向、官方架构图 | docs/architecture.mermaid；AGENTS.md:31 |
| 入口分发（mcp / mcp doctor / CLI） | scripts/rea.mjs（全文） |
| MCP 启动流程（parseConfig → session → transport → shutdown） | src/main.ts:50-95 |
| CLI 命令注册（37 组） | src/cli.ts:87-148 |
| 唯一生产装配点、CLI/MCP 共享会话 | src/composition/binary.ts（文件注释 + :22-36） |
| BinarySession 串行化/排水/失败回滚/外部变异纪律 | src/application/binary/BinarySession.ts:1-31, 187-320, 371-420, 601-621, 760-905 |
| 保留 client 快速路径的判定（idle/busy 才算活） | 同上 :869-895 注释 |
| 确定性 provider 选择、ambiguity 报错、无回退 | src/application/binary/AnalysisProviderRegistry.ts:95-120, 185-300 |
| 深/辅 provider 操作集不相交 | src/application/binary/SessionProviderRouter.ts:39-90 |
| 快照缓存可缓存性判定与上限 10000 | src/application/binary/AnalysisSnapshotCache.ts:20-70, 250-300 |
| mutatesArtifact 先失效再 dispatch | 同上 :66-72 注释 + BinarySession.ts:780-783 |
| 证据/快照/记录协调 | src/application/binary/BinarySessionRecords.ts（全文） |
| EvidenceLedger 幂等/冲突拒绝/原子导入 | src/application/investigation/EvidenceLedger.ts:31-60, 62-140, 481-493 |
| 未知项乐观并发 | 同上 :243-256 |
| 一次工具调用的绑定与投递 | src/server/contractToolHandler.ts:29-90；src/server/toolResult.ts:1-120 |
| 工具执行期间目标绑定持锁 | src/server/analysisAdmission.ts:6-30 |
| 超预算不截断 + 恢复建议 | src/server/toolResult.ts:150-215 |

## 领域模型
| 结论 | 出处 |
|---|---|
| Evidence schema 全字段 | src/domain/evidence.ts:96-140 |
| 语义 ID（canonical digest）与篡改拒绝 | 同上 :180-205 |
| authority/confidence 枚举 | 同上 :82-95 |
| ResidualUnknown 与 merge 校验 | src/domain/evidenceBundle.ts（引用）/ residualUnknown.ts |
| Result 代数 | src/domain/result.ts:1-11 |
| 错误 tag 全表 + cleanup/partial 语义 | src/domain/analysisErrorBase.ts:4-31, 55-100 |
| 架构守卫（domain 约束） | scripts/verify/architecture-guards.mjs:63-100（已抽查） |

## Provider 与进程
| 结论 | 出处 |
|---|---|
| Hopper 启动命令与 bootstrap 注入 | src/hopper/BridgeLauncher.ts:280-330, 218-235 |
| 所有权边界（不含 Hopper GUI 进程） | 同上 :88-100 注释 |
| Hopper NDJSON 协议与事件校验 | src/hopper/protocol.ts:1-80 |
| Hopper bridge 在专用 Python 线程、缓存与 BAD_ADDRESSES | bridge/hopper_bridge.py:1-35 |
| Ghidra 支持矩阵/Windows P0/语言覆盖准入/seeds | src/ghidra/GhidraProvider.ts:63-120, 140-200, 240-300, 355-395 |
| IDA BYO 两模式 + registration_digest 参与 profile | src/ida/IdaProvider.ts:40-105, 118-145 |
| 运行 token 校验与 PID 复用防护 | src/process/ProcessOwnership.ts（多处，见 notes/process-foundation.md 表） |
| Windows 能力三件套 | src/process/WindowsAuthority.ts:29-33 |
| 长尾 provider 模式（Port + inspectAvailability + close） | 见 notes/artifact-providers.md 各 file:line（代理调研） |

## 工程化
| 结论 | 出处 |
|---|---|
| 构建链（tsc + rolldown + 生成物） | package.json:293；scripts/bundle-runtime.mjs:24-36 |
| 生成物所有权与 --check 规则 | scripts/lib/generated-file.mjs:14-37 |
| setup 事务性写入与回滚 | src/application/SetupClientConfiguration.ts:219-239, 275, 323 |
| skill 只管理 7 个文件 | src/application/SetupSkill.ts:22-30（已抽查） |
| 测试项目划分与策略 | scripts/lib/test-projects.mjs；vitest.config.ts:20-90 |
| catalog 身份与摘要 | src/catalogIdentity.ts:20-92 |

## 已执行的验证命令（抽查项）
- `grep -nE "localeCompare|NUL|prlimit|walker" scripts/verify/architecture-guards.mjs` → 确认守卫存在
- `ls native/windows` → README.md + src（原生插件存在）
- `grep -n "Pure parsing" src/flutter/FlutterBuildProvider.ts` → :514
- `ls src/browser` / `ls src/process` → 关键文件存在

## 未执行的验证
- 未运行任何 `npm run verify:*` / 测试 / 构建（只读分析）。
- 未运行目标程序本体（未安装 rea-agents）。
