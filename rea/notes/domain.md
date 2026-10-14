# 模块笔记：Domain 与 Contracts 层

## 定位（事实）
- `src/domain/` 568 个 TS 文件，是最大的一层；官方架构图标注为 "Pure side-effect-free modules"（docs/architecture.mermaid）。
- 依赖方向：`contracts → domain`，上游（providers/application/adapters）只能向内依赖。
- **推断**：项目把"逆向分析的语义"（证据、目标身份、快照、错误、未知项）铸成可验证的数据类型，而不是把逻辑散在各 provider 里——这是它敢接十几种引擎的前提。

## 结果与错误代数（事实）
- `src/domain/result.ts:1-11`：`Result<T, E> = {ok:true,value} | {ok:false,error}`，配 `ok()`/`err()`。**所有端口都用它**，不抛异常传递预期失败。
- `src/domain/analysisErrorBase.ts`：
  - `AnalysisError` 抽象基类，27 个稳定 tag（`ANALYSIS_ERROR_TAGS`，:4-31），如 `AnalysisCapabilityUnavailableError`、`ProviderAdapterError`、`EvidenceIntegrityError`、`HopperStartError`…；`_tag` 是联合类型，用于安全投影。
  - 携带 `diagnostics`（可对调用者可见的结构化事实）、`capturedOutput`（stdout/stderr+截断标记）、`cleanup`/`cleanupIncomplete`/`cleanupResources`（清理未完成的**显式**状态）、`partialObservation`（失败前已获得的观察）。
  - 三个保留方法：`retainSecondaryFailure`（不覆盖主错误，聚合成 AggregateError）、`resolveCleanup`（仅当 owner 确认重试成功才清标记）、`retainPartialObservation`。
- `src/domain/analysisErrorProjection.ts`（587 行）+ `contracts/errorSchemas.ts`：把内部错误投影成调用者可见结构，并生成 `docs/error-contract.schema.json`（`scripts/generate-error-schema.mjs --check` 校验一致性）。

## Evidence：核心领域模型（事实，src/domain/evidence.ts）
- 一条观察 = 严格 schema（`z.strictObject`），关键字段：`subject`（name/digest.sha256/format/architecture/local_path）、`provider`、`analysis_profile`、`predicate_type`、`operation`、`parameters`、`raw_result`、`normalized_result`、`confidence ∈ {observed,derived,inferred}`、`authority ∈ {shipped-artifact,controlled-replay,historical-reference,external-service,analyst-inference}`、`environment`、`limitations`、`relationship_coverage`、`locations`、`evidence_links`。
- **身份即内容**：`evidence_id = "ev_" + canonical digest(语义投影)`（:198-201），解析时重算并拒绝被篡改的 ID（`authenticatedEvidenceSchema`，:127-138）。语义投影**排除** display 字段（`name`、`local_path`），所以同一观察换路径后 ID 不变（`rebindEvidenceSubjectTargetPath`）。
- 不可变快照：`rememberImmutableEvidence` 用 WeakMap + 深冻结；`createImmutableEvidenceSteps` 提供协作式（可让出）大对象哈希，避免阻塞事件循环（:230-260）。
- subject format 枚举覆盖：hopper/analysis-database/mach-o/elf/pe/dos-mz/dos-com/raw-binary/zip/ipa/apk/msix/appx/hap/hsp/app-pack/asar/dmg/pkg/plist/javascript/source-map/directory/file/unknown/mach-o-universal/javascript-bundle/entitlements（:26-52）。

## 快照与剖面（事实）
- `src/domain/analysisSnapshot.ts`（893 行）：`AnalysisSnapshot = {target, binding(analysis_profile), entries, workflow_entries, evidence_bundle}`；查询键 `analysisQueryId(target, binding, operation, parameters)`；提供 `snapshotMatchesTarget/Profile`、`immutableAnalysisSnapshot`。
- `src/domain/analysisProfile.ts`：`AnalysisProfileCommitment`（provider + parameters + cache_policy 等），`analysisProfilesEqual`；profile 变化即目标绑定/profile 不匹配 → 拒绝复用缓存/快照（`AnalysisSnapshotMismatchError`）。
- `src/domain/binaryTarget.ts`（563 行）：`BinaryTarget` 含 path/sha256/format/architecture/kind(executable|database)/availableArchitectures/executableRole/managed 等；`isMachOFormat`、`isElfMetadataArchitecture` 等判定辅助。

## JSON 与规范化（事实）
- `jsonValue.ts` / `immutableJson.ts`：严格 JSON 值 schema + "冻结快照"概念（用于零拷贝共享大 payload）；`jsonValueValidationSteps` 可协作式校验。
- `canonicalDigest.ts` + `canonicalize` 依赖：所有摘要都是 canonical JSON 的 sha256（Evidence ID、catalog digest、快照键都靠这个）。
- `digests.ts`：`prefixedDigestSchema("ev")` 之类的带前缀摘要 schema；`unicodeCodePointOrder.ts`：统一排序原语。

## 其它领域能力（事实：文件清单）
调用路径（`callPath.ts` 408）、函数对比（`functionComparison*`）、工件对比（`artifactComparison.ts` 523）、浏览器场景 diff（`browserScenarioDiff.ts` 606）、网页捕获 diff（`webCaptureDiff.ts` 770）、web bundle 分析（`webBundleAnalyzer*` 1000+）、参考源图（`referenceSourceGraph.ts` 598）、重建覆盖/义务账本（`reconstructionCoverage*`、`reconstructionObligationLedgerSchemas.ts`、`reconstructionVerification.ts`）、一致性信任门（`conformanceTrustGate.ts`）、静态/运行时相关性（`staticRuntimeCorrelation.ts`）、行为变更（`changedBehavior.ts`）、残余未知（`residualUnknown.ts` 414）、Hopper 值解析（`hopperValues.ts` 585）等。

## ResidualUnknown（残余未知项，事实来源：EvidenceLedger 用法）
- 有 `unknown_id`、`revision`、`revision_digest`、`status`、`severity`、`domain`、`resolution.disposition`（含 `verified`）。
- **乐观并发**：`updateUnknown` 要求 `expected_revision` 匹配，否则 `revision-conflict`（EvidenceLedger.ts:243-256）。
- 未知项必须挂在**变异证据**上（`recordUnknown(input, mutationEvidence)`），并校验其引用的 evidence 存在（`validateResidualUnknownAddition`）。
- 设计意图（推断）：把"没弄清楚的部分"作为一等公民持久化，避免"未知被当成不存在"（AGENTS.md:41-44 明确要求）。

## Domain 专属架构守卫（事实，scripts/verify/architecture-guards.mjs）
- 禁 `src/domain/` 使用 `localeCompare`（改用 `compareUnicodeCodePoints`，:63-65）。
- 禁 NUL 连接自由文本/变长字段作复合键（:69-74）。
- 禁自研 AST child walker（必须用 `traverseJavaScriptAst/childNodes`，:86-89）。
- prlimit 参数向量集中到 `prlimitArgumentVector`（:93-100）。
- 固定元数反引号 key + NUL 审计规则（:116-122）。
→ **推断**：这些守卫都是"曾经踩过的坑"的沉淀，说明作者把领域层当作可长期依赖的语义核心来保护。
