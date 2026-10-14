# 模块笔记：built-in-skills（专项技能提示词库）

## 1. 规模与形状

- **事实**：~50 个 `.md`（`wc -l` 合计 2696 行）。"薄"的只有 5 行（`wireframe.md`、`interactive-prototype.md`、`make-tweakable.md`、`options-stack.md`、`sound-effects.md`、`gemini-image.md`）；"厚"的是完整操作手册（`make-a-deck.md` 200+ 行、`ask-the-user.md` 17KB、`create-design-system.md` 19KB、`data-science.md` 22KB、`social-media-content.md` 11KB、`watercolor-illustration.md` 11.9KB）。
- **事实**：统一 frontmatter：`name` + `description`（`description` 常为「Title\n一句话」两行形态，如 `hi-fi-design.md:1-5`）。薄文件（wireframe/interactive-prototype）就是 frontmatter + 一段指令——说明这些是从上游主机系统里"按需展开"的提示词片段，独立看也成立。
- **事实**：system-prompt.md 末尾有全量清单（`system-prompt.md:277-323`），每个条目 = 何时用 + 文件名；SKILL.md 顶部另有「先读哪个」的路由（`SKILL.md:42-60`）；`project-types.json` 是机器可读第三份。三份由测试对齐（见 engineering 笔记）。

## 2. 路由（13 类项目）

`project-types.json` → skill + starter 映射（`project-types.json`，与 `system-prompt.md:114-127` 表格同构）：

| id | skills | starters |
|---|---|---|
| slides | make-a-deck | deck-stage.js |
| mobile-app-design | mobile-prototype, hi-fi-design, interactive-prototype | ios-frame/android-frame/ios-shell |
| wireframe | wireframe | design-canvas.jsx |
| document / resume | make-a-doc | doc-page.js |
| animation | animated-video, exportable-video | animations-v3.jsx, tweaks-panel.jsx（可选 watercolor-kit） |
| ui-mockups | hi-fi-design, interactive-prototype | design-canvas.jsx, image-slot.js |
| 3d-object | 3d-object | three-d-stage.js |
| research | web-research | data-overlay.js |
| html-email | html-email | —（table-based standalone HTML） |
| color-type-system | create-design-system | design-canvas.jsx |
| diagram | data-visualization | chart-stage.js |
| flier | flier | doc-page.js |

## 3. 重点技能语义（抽样精读）

### make-a-deck.md（最厚、信息密度最高）
- **角色设定**：presentation designer，不是 web 设计（顾问/董事会材料思维）；写 outline 先于写 HTML；标题序列要能单独讲出故事（`make-a-deck.md` 开头段）。
- **静态优先原则**：幻灯片内容用静态 HTML 写（不是 React/数组生成），因为宿主的直接编辑能把用户点改的文字 splice 回源码；文本放叶子元素、重复结构写死（`make-a-deck.md` 第 3 段）。
- **deck-stage 使用约定**：`<deck-stage width height>` + 每页一个 `<section data-label>` 子元素；不要在 section 上设 position/尺寸（组件全权定位）；**wrapper-fill 陷阱**：组件只拉伸 section 不拉伸内部 wrapper，全绝对定位的 wrapper 会坍缩为 0 高导致全出血图丢失 → 必须加 `section[data-label] > *` 的 height:100% 基础规则（`make-a-deck.md` 中段 + `CHANGELOG.md` 2026-06-19 有一条同主题 fix）。
- **data-anim 动画全表**：48 种效果（entrance/exit/emphasis/motion path × 方向族）+ 时序属性（trigger click|with|after、delay/duration/order/repeat/auto-reverse）；默认 trigger=after（自动串联播放）；与 PowerPoint 原生动画 1:1 对应；元素只能挂一个效果、要两个效果就包两层（`make-a-deck.md` "Animations" 节）。
- **PPTX 导出契约**：deck-stage 的 `noscale` 属性让导出器拿到 1:1 几何；`resetTransformSelector:"deck-stage"` 传参约定（`make-a-deck.md` deck-stage 段）。
- **生成图像的分工原则**：表格/流程图/精确数字 → HTML/CSS（可编辑、精确）；生成图像只留给概念场景、吉祥物、hero/分区美术、真正需要的信息图（`make-a-deck.md` 末段 + CHANGELOG 1.1.x 修订史）。
- **排版规范**：1920×1080 下正文不小于 24px、标题 ≥48px；pt↔px 换算 `px = pt × 1.333`；`--type-*`/`--pad-*` CSS 变量兜住整套尺度（`make-a-deck.md` "Planning steps" 第 3 条）。
- **反 AI 味**：明确列出 Claude 常见标题病（下判词、过度戏剧化、"It's not X, It's Y"、制造悬念）→ 标题应是「翻开下一页的引言」而非"演讲者收尾的 punchline"（`make-a-deck.md` "Slide writing guidelines"）。

### hi-fi-design.md（默认路径之一）
- 五步流程：问 → 收集设计上下文（拷贝可用组件/样例；找不到就问）→ 文件开头先写假设+推理（像初级设计师对经理）→ 尽早展示 → 用工具验证迭代。
- 铁律：好的 hi-fi 设计不从零开始；"从零 mock 整个产品是 LAST RESORT"；鼓励 3+ 变体跨多个维度、混搭"按套路"与"新颖"；没有图标/资产就放**干净占位符**而不是硬画（真实资产只在图像后端可用时生成，摄影用 `<image-slot>` + 署名库存图）。
- **事实**：多选项并排展示用 `design-canvas.jsx`（每个选项一个可拖动 artboard；宿主产品另加 `design_doc_mode` meta，便携版不需要）（`hi-fi-design.md` 中段）。

### use-design-system.md（消费侧全流程）
见 notes/design-system.md（已展开：发现→问→import→接线→读 _ds_prompt.md 为 binding→seed→_d_meta.json→恢复/刷新/增删/换 primary→多系统共存）。

### 其他值得记录的行为约束
- `wireframe.md`：快速探索；手绘风、低饱和、3-5 个截然不同方案；"如果要用户选一个才能继续，不要用并排铺陈，改用 file-options board"（`wireframe.md`）。
- `interactive-prototype.md`：useState/useEffect 真状态、hover/点击/校验/过渡/多步流。
- `something-cool.md`（20 行）：仅在用户明确"给我看点酷的"时触发；**先问想要什么再动手**（`system-prompt.md:129` 复述）。
- `speaker-notes.md`：演讲者备注格式；默认不加（`system-prompt.md:105-106`）。
- `generate-images.md`：检测图像后端（宿主原生工具优先于 `baoyu-image-gen` CLI）；先落 prompt 文件再生成；输出 `imgs/`+`prompts/`；禁 SVG 冒充栅格图（`CHANGELOG.md` 2026-06-19 条目 + `system-prompt.md:257`）。
- `export-as-pptx-editable.md` / `export-as-pptx-screenshots.md`：两种模式的输入 JSON（mode/width/height/slides/hideSelectors/resetTransformSelector/googleFontImports/fontSwaps/filename）——与本地 CLI 的 `--config` 完全同构（`references/claude.md` "Exporting to PPTX"）。
- `export-as-video.md`：见 notes/gen-video.md 的「桥接错位」一节——文档假定 `animations.jsx` 的 `window.__animStage` + `?capture` 零配置可用。
- `exportable-video.md`：事件契约（`data-om-exportable-video-with-duration-secs` + `data-om-seek-to-time-frame`，v2/v3 starter 已内置）；**宿主产品**的 Share→Export→Video 靠它（`exportable-video.md:1-16`）。
- `animated-video.md`：强制 v3 starter；`window.OM_SCENES` / `window.OM_PLAYBACK` 必须写成主文档内联 `<script>` 字面量（编辑器回写锚点）；时间线是用户可编辑的（trim/speed 重放作者切片，不截断编排）；"拍片"必须用边界 ±0.15s 的 filmstrip 自检（`animated-video.md:7-24`）。
- `import-from-github.md` / `import-from-html.md` / `import-from-figma.md`：把外部源当"设计真值"的三条管线；github 先 `gh api` 浏览再稀疏 sparse-checkout 到项目外 scratch；html 读代码不读截图、抽 token/状态/资产；figma 走本地 `import-figma.mjs`（notes/figma-import.md）。
- `handoff-to-claude-code.md`、`save-as-pdf.md`、`save-as-standalone-html.md`、`send-to-figma.md`、`send-to-canva.md`、`google-slides-safe.md`、`claude-api-in-prototypes.md`（`window.claude.complete`）、`sound-effects.md`（ElevenLabs）、`gemini-image.md`（5 行，旧入口）、`read-pdf.md`、`design-feedback.md`、`data-science.md`/`experiment-workflow.md`、`maps-geography.md`、`social-media-content.md`、`design-components.md`（.dc.html）、`make-tweakable.md`/`tweaks-protocol.md`/`low-level-tweaks-api.md`（宿主协议）、`ask-the-user.md`/`options-stack.md`（结构化决策板）——均为按需加载的独立提示词。

## 4. 观察与推断

- **推断**：薄/厚分化不是质量问题，而是任务复杂度差异——`wireframe.md` 只在 `hi-fi-design` 流程上加一层"低保证/多方案"约束；`make-a-deck.md` 承担了完整领域知识（排版、叙事、导出契约）。
- **推断**：这些 md 与上游快照文件一一对应（`references/upstream-sync/built-in-skills.json`），本地改写集中在受保护清单里的 9 个文件；其余文件基本是上游内容的直出。
- **未知**：`design-system-authoring-guide.md` 只有二手信息（bg 报告引用），本笔记未逐行核对。
