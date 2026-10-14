# 未解问题清单

> 随时更新。标记：❓未知 / ⚠️不一致（有冲突证据）/ 🔍待验证。

## 高优先级

1. **⚠️ 动画导出协议错位（v3 引擎 vs 本地 gen-video）**
   - 事实：`animations-v3.jsx`（当前引擎）只监听 DOM 事件 `data-om-seek-to-time-frame`（`animations-v3.jsx:500-563`），不注册任何 seek 桥全局；`gen-video` CLI 只会调用 `window[bridgeGlobal].setTime()`（默认 `__animStage`）（`gen-video/src/orchestrator/driver.ts:49-60`、`run.ts:135`），全目录无 `data-om-` 命中。
   - 冲突证据：`export-as-video.md:13,34-42` 以"当前 `animations.jsx` 注册 `__animStage` + `?capture`，零配置"为准；而 `system-prompt.md:99,223` 与 `project-types.json` 以 v3 为当前引擎。
   - 待查：本地（非宿主）用 gen-video 导出 v3 动画的正确路径是什么？（自加桥？CLI 待适配事件协议批量 seek？）仓库里没有写明。

2. **❓ 上游提取工具链不在仓库**
   - `apply-report.json` 显示提取自 `claude-design-v2/restored/.deobfuscate-javascript/upstream-sync-2026-09-14/extracted`（他机器的路径）；去混淆/切片/同步的脚本本体不在本仓库。
   - 待查：这套脚本是否开源在另一个仓库（JimLiu/claude-design-v2?）；同步频率与触发方式。

3. **❓ `npx skills`（Vercel Labs）安装器的精确约定**
   - README 说它"reads this repo, finds `skills/baoyu-design/`"。安装到 `.claude/skills/`（Claude Code）或 `.agents/skills/`（Cursor/Codex 风格）。
   - 待查：installer 如何枚举 skill（目录扫描规则？SKILL.md frontmatter 要求？）；本仓库的目录布局是否对 installer 有隐性约束（如必须 `skills/<name>/SKILL.md`）。

## 中优先级

4. **🔍 gen-pptx 的浏览器侧主体无自动测试**：`capture-editable.ts`/`setup.ts`/`media-browser.ts`/`render-node.ts`/`validate.ts` 依赖真实浏览器 DOM，只有纯函数层有测试（notes/gen-pptx.md §7）。真实回归风险集中在这些文件。

5. **🔍 gen-video 不在根 CI**：根 `npm test` 只跑 `agents/tests/*.test.mjs`；`agents/gen-video` 与 `agents/gen-pptx` 的测试需目录内单独跑。CI（`.github/workflows/tests.yml`）不覆盖二者——这是有意的（避免 Playwright/Chromium 依赖）还是遗漏？

6. **❓ `minimal.fig` fixture 的构造方式**：测试用到它但没有生成脚本的记载；构造一个合法 kiwi/ZIP .fig 的方法未核实。

7. **❓ 本地 overlay 与上游快照的完整 diff**：受保护清单有 11 个文件（9 skill + 2 starter），我只抽查了 system-prompt.md、make-a-deck 一处、design-canvas 的滚轮 patch。其余（hi-fi-design、read-pdf、save-as-pdf、save-as-standalone-html、speaker-notes、animated-video、design-components、frontend-design、tweaks-panel）的本地上改未逐条核对。

8. **🔍 `deck-stage-patch.md` / `design-canvas-patch.md`**：本地 patch 说明文档（供上游同步重放本地改动用），未逐行读；它们与 `project-types.test.mjs` 的 marker 断言是同一套防线的两半。

9. **❓ 宿主产品（claude.ai/design）与本仓库的其余差异面**：provenance.json 的 systemPromptOptions 里 `enableDesignComponents:false`、`canAuthorDc:false` 等开关说明宿主还有本仓库未打包的能力（.dc.html Design Components 有 skill 文档但开关是关的）。哪些能力被有意裁剪、哪些只是没开关，未核实。

## 低优先级 / 随口记录

10. ❓ `animations-v2.jsx` 的 `<VideoSprite>`（loop 视频音频混合）在 gen-video 里没有对应实现——本地导出是否支持视频混音？（`exportable-video.md:7,13` 提到音频混合属于宿主导出器。）
11. ❓ `data-om-exportable-video-play-start/end/speed` 这几个音频契约属性在本地工具链里无人消费。
12. 🔍 `ask-the-user.md`（17KB）/ `data-science.md`（22KB）等大文件的内部结构未细读——它们对理解「路由质量」有价值。
13. 🔍 `three-d-stage.js`（Three.js 版本来源、OBJ/GLB 导出实现）、`chart-stage.js`、`data-overlay.js`（96KB）细节未展开。
14. ❓ CHANGELOG 里提到但未在本次调研中核实的条目：`gen-pptx` 的 gradient/supercref 边界、`send-to-figma`/`send-to-canva` 的实际实现机制（这俩 skill 只有文档，无本地工具）。
