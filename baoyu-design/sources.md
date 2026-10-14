# 关键结论的证据出处

> 格式：结论 → 出处（文件:行 / 命令）。行号以调研时 HEAD `6530033` 为准。

## 1. 定位与总体

| 结论 | 出处 |
|---|---|
| 项目把 Claude Design 打包为可移植 Agent Skill，输出自我包含 HTML | `README.md`（开头段、How it works） |
| 版本 1.3.0 / MIT / 维护者 Jim Liu | `package.json:3`、`LICENSE`、`README.md` Credits |
| 无 build step、无 runtime（技能本体） | `README.md` "How it works" |
| 上游为 claude.ai/design assets（含 sha256）与去混淆工作区 | `references/upstream-sync/provenance.json` |
| 上游同步动作流（copy / preserve-for-merge + 保护清单） | `references/upstream-sync/apply-report.json` |
| 本地 system-prompt 与上游快照差异（工具替换、隐私行移除等） | diff `references/upstream-system-prompt.md` vs `system-prompt.md`（命令：`bash -c 'diff <(sed -n "1,60p" A) <(sed -n "1,60p" B)'`） |

## 2. 核心机制

| 结论 | 出处 |
|---|---|
| SKILL.md 五步编排 + 路由分支 + 分层职责声明 | `SKILL.md:23-65` |
| system-prompt 章节地图与 craft 硬规则（CDN 固定版本、拆分原则、反 AI 味等） | `system-prompt.md:54-59,70-97,256` |
| harness 识别规则与三份映射文档差异 | `system-prompt.md:10-16`；`references/claude.md`、`cursor.md`、`codex.md` |
| vision probe 机制（committed PNG + 子代理单 token 探测） | `references/claude.md` "Vision input probe"；测试 `agents/tests/vision-probe-docs.test.mjs` |
| 设计系统磁盘形制（根标记 `_ds_manifest.json` vs `_d_meta.json`） | `built-in-skills/use-design-system.md:11-20` |
| `_d_meta.json` schema | `use-design-system.md:105-136`；`lib/asset-store.mjs` 头注释 |
| import 拷贝集（@import 闭包 + url() 资产 + bundle/manifest/…） | `agents/import-design-system.mjs:100-131` |
| `_ds_prompt.md` 内容与生成逻辑 | `agents/lib/ds-prompt.mjs:1-18,154-286` |
| record-asset 资产模型/状态生命周期 | `agents/lib/asset-store.mjs` 头注释；`agents/record-asset.mjs` 头注释 |
| HTTP 预览强制与 4311 端口约定 | `system-prompt.md:68,146,185`；`references/claude.md` "Showing files & preview" |
| 直接可编辑性（canonical HTML/叶子元素/写死重复结构） | `system-prompt.md:38-39`；`make-a-deck.md` 第 3 段 |
| project-types 三份路由由测试对齐 | `agents/tests/project-types.test.mjs` |

## 3. 模块实现

| 结论 | 出处 |
|---|---|
| deck-stage：缩放/noscale/rail/dc-op/print/data-anim/隐藏非卸载 | `starter-components/deck-stage.js:1-100`（使用文档）、`1706-1732`、`2769-2864`、`3380-3460` |
| design-canvas：视口持久化/滚轮 1.1×+200ms 突发锁定/--dc-inv-zoom | `design-canvas.jsx:263-303,412-424` |
| animations 三代契约差异（__animStage vs data-om-seek 事件） | `animations.jsx:409-432`；`animations-v3.jsx:500-563,1339`；grep 结果（`__animStage` 仅 v1/gen-video） |
| gen-video seek 实现（setTime + 双 rAF；默认桥 __animStage） | `agents/gen-video/src/orchestrator/driver.ts:49-60`、`run.ts:135,185` |
| export-as-video 文档与当前引擎陈述冲突 | `built-in-skills/export-as-video.md:13,34-42` vs `system-prompt.md:99,223` |
| gen-pptx 双产物构建与注入 | `agents/gen-pptx/package.json:11-17`；`src/orchestrator/inject.ts:19-27` |
| gen-pptx editable：DOM→JSON 树→PptxGenJS；单位 px/96、px×0.75 | `src/browser/capture-editable.ts:121-342`；`src/core/units.ts:4-12`；`src/render/render-node.ts:46-533` |
| gen-pptx 渐变光栅化（无渐变填充 → 带 alpha PNG） | `src/browser/gradient.ts:1-10,135-233` |
| gen-pptx 动画→`<p:timing>`，与 deck-stage 共享时长算法 | `src/render/timing.ts:198-584`；`src/core/anim.ts:317-318` |
| vendored PptxGenJS 两处 patch + 内部契约 | `agents/gen-pptx/vendor/pptxgenjs/VENDOR.md:6-19`；`vendor/pptxgenjs/src/gen-xml.ts:1569`；`core-interfaces.ts:1790-1792` |
| compile：Node 内 vendored Babel、DFS 后序、失败降级 __errors | `agents/compile-design-system.mjs:23-39,97-166,189-193` |
| ds-core：闭包解析/幻影匹配防护/容错解析 | `agents/lib/ds-core.mjs:56-118,124-198,248-313` |
| check：只读、exit 2 条件、--verbose | `agents/check-design-system.mjs:6-13,110` |
| figma：ZIP/kiwi/zstd-inline-WASM/schema 现场编译 | `agents/vendor/fig-materialize.mjs:388-455`；`agents/import-figma.mjs:123,344-645` |
| build-preview：解析器/内联器/沙箱/自适应 | `agents/build-preview.mjs:2-5,575-767,1027-1438,1493-1551` |
| 测试矩阵与 CI | `agents/tests/*.test.mjs`；`.github/workflows/tests.yml`；根 `package.json:6` |
| release-skills 定位与流程 | `.claude/skills/release-skills/SKILL.md`；`git tag`（v1.0.0…v1.3.0） |

## 4. 命令证据（本次调研实际执行）

- `git log --oneline | head -20`、`git tag`：HEAD=6530033，tags v1.0.0–v1.3.0。
- `wc -l`：SKILL.md 61 / system-prompt 323 / built-in-skills 合计 2696 / starter-components 合计 21481 / agents 工具 4985（不含 gen-pptx、gen-video 目录）。
- `find skills/baoyu-design -maxdepth 2 -type d`：目录结构核实。
- 两次 grep：`__animStage|data-om-seek-to-time-frame|data-om-sync-seek`；`data-om-|bridgeGlobal|__ahe`（在 gen-video 目录）——支撑"桥接错位"结论。
- `diff`（bash -c 包裹进程替换）：上游快照 vs 本地 system-prompt 前 60 行。
- 4 个后台探索 agent 的报告（本会话内），其原始行号引用已抽查吻合（anim-v3、export-as-video、gen-video/grep 均二次核验）。
