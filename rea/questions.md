# 未解问题清单（rea）

> 状态：分析中持续更新。标注方法：未查=没看源码；未验证=看到代码但没运行确认。

## 数据与规模
- [未验证] MCP 工具的**确切数量**：由 `src/catalogIdentity.ts:84-89` 运行期统计；本次未构建，未取得数字。可取法：`npm ci && npm run build:cached` 后读 `.cache/mcp-tool-catalog.json`，或跑 `npm run mcp-catalog:generate`。
- [未验证] CLI 命令确切数量：`CLI_COMMANDS` 命中 77 个条目（grep），但别名表与 `MCP_TOOLS_WITHOUT_DEDICATED_CLI` 未合并计数。
- [未查] `docs/roadmap.md`：未来计划与能力追踪表。

## 实现细节
- [未查] `bridge/hopper_bridge.py`（1173 行）的 operation 分发表：REA 侧 operation 名 → Hopper API 的完整映射。
- [未查] `GhidraTransport`/`GhidraSocketConnection` 的帧格式与 Windows 传输差异（`GhidraClient.ts` 999 行只读了个大意）。
- [未查] `IdaNativeProtocol.ts`：native 协议与 legacy-1.4 的差异细节。
- [未查] `InvestigationRecords.ts`（144 行）全文；只读了 `EvidenceLedger.ts`。
- [未查] `docs/prompts`（`PROMPT_CONTRACTS`）实际条目与分步内容；`registerPrompts.ts` / `registerGuidedPrompts`。
- [未查] `WebSourceMapFetcher.ts`（1095 行）与 source-map codec 进程的协议。
- [未查] 快照文件的**版本兼容策略**（`analysisSnapshot.ts` 893 行中的 schema 版本/迁移约定）。
- [未查] `experiments/analysis-efficiency`：项目自测 token/时间成本的方法论。
- [未查] `.github/workflows/*`：真实 provider 验证矩阵与 CI 分片策略。

## 未验证的主张
- README 的营销数字（180,000 stars、Trendshift 徽章）未独立核实。
- 所有 `verify:*` 真实 provider lane 均未运行（本次只读分析，无构建）。
- "运行期无跨调用缓存（除会话内快照）"是探索代理的**推断**，未穷尽确认（例如 browser 侧是否有磁盘缓存）。
- Windows 原生插件的真实行为未验证（只看了 TS 侧与 README）。

## 待确认的边界问题
- 会话切换（`session.open` 换目标）时，之前记录的 Evidence 是否跨目标保留？`close()` 清记录、切换失败回滚保留——但"成功切换后旧目标证据"的语义未逐行确认（推测按 `evidenceBundleForTarget` 过滤，见 BinarySessionRecords.exportAnalysisSnapshot 的用法）。
- `REA_MCP_MAX_RESPONSE_BYTES` 的默认值与 `STDIO_DEFAULT_MAX_BUFFER_SIZE` 的对齐方式（toolResult.ts 引用了 SDK 常量，未核对数值）。
- 三档 provider 选择里 `environment` 档在"目标不被默认 provider 支持"时会静默解绑（Registry.select 中 `binding: null` 分支），对使用者的提示路径未读。
