# 模块笔记：技能核心（编排 + 方法论 + harness 适配 + 上游同步）

## 1. SKILL.md —— 编排入口

- **事实**：frontmatter 只有 `name` + 一个长 `description`（触发词清单：UI mockup、prototype、wireframe、landing page、dashboard、mobile app、slide deck/PPT、document、résumé、animation、3D、research、HTML email、diagram、flier、design system…）（`SKILL.md:2-16`）。
- **事实**：正文 5 步：①读 system-prompt.md ②识别 harness → 读 references/<h>.md ③按任务载 built-in-skills ④问问题 ⑤建 `designs/<project>/`（或恢复）＋import 设计系统＋record-asset ⑥构建/预览/验证（`SKILL.md:23-60`）。
- **事实**：路由逻辑集中在第 3 步，覆盖 12 个具名分支 + 默认分支（hi-fi + interactive-prototype）+ something-cool 特例（`SKILL.md:42-60`）。
- **事实**：Notes 声明分层职责："system-prompt.md 是 craft 的唯一事实来源；references/<harness>.md 是工具调用的唯一事实来源；SKILL.md 只做编排"（`SKILL.md:62-65`）。

## 2. system-prompt.md —— 方法论（323 行）

章节地图（行号）：
| 节 | 行 | 内容 |
|---|---|---|
| Harness setup | 6-16 | 4 个能力差异点 + 识别规则 + 通用兜底 |
| Your workflow | 18-26 | 6 步（问→探索→todo→建文件夹/import→预览验证→极简总结） |
| Output creation guidelines | 28-42 | 命名、版本、项目文件夹、设计系统、资产复制、文件拆分、localStorage、canonical HTML、颜色/emoji 纪律 |
| Review context | 44-50 | DOM 指认上下文 + `[data-screen-label]` |
| React + Babel | 52-103 | 固定版本 CDN 标签（含 integrity）、多 JSX 文件拆分、`window` 共享、styles 命名、CSS 变量主题、动画引擎选择 |
| Speaker notes | 105-106 | 默认不加 |
| How to do design work | 109-137 | 载 skill 规则 + 13 类路由表 + 探索输出形态（canvas vs 原型）+ 变体策略 |
| File paths / Showing files / Linking | 139-149 | 普通文件工具、HTTP 服务、相对链接 |
| System placeholders | 151-152 | `[System: ...]` 处理 |
| Asking questions | 154-179 | 问问题总则（每次都是 fresh start、确认上下文/保存位置/设计系统/变体维度；大项目 10+ 问） |
| Verification | 181-187 | 交付=预览；HTTP 强制；console 检查 |
| In-page controls | 189-190 | 无宿主 Tweaks 面板 → 自建面板 + Show/Hide |
| Web search / Napkin / Fixed-size content | 192-206 | 搜索纪律、固定尺寸缩放、deck-stage |
| Starter Components | 208-232 | 脚手架清单与用途 |
| GitHub | 234-242 | github.md 凭据、gh 命令 |
| Content Guidelines | 244-273 | 无填充、问后加料、署名、系统先行、尺寸最小值、反 AI 味、对比度 4.5:1、流式布局、flex/grid+gap、CJK 排版 |
| Skills | 275-323 | ~50 个内置 skill 的清单（按需 READ） |

**关键 craft 硬规则（事实）**：
- React CDN 三件套固定 18.3.1 / @babel/standalone 7.29.0 + `integrity`（`system-prompt.md:54-59`）。
- 多 Babel 文件各自作用域不通 → 共享拆到 `window`（`:79-90`）；全局 style 对象必须带组件名（`:70-71`）；优先 CSS 变量而非 per-component style object（`:73-77`）。
- 拆分边界按耦合而非行数：抽取数据/图标/纯展示组件；App+紧耦合状态保持一个文件（`:92-97`）。
- 固定尺寸内容（deck/video）须自实现缩放：1920×1080 + letterbox + 控件在缩放元素外（`:202-206`）。
- 反 AI 味清单：渐变滥用、emoji、圆角+左边框 accent 卡、Inter/Roboto/Arial/Fraunces（`:256`）。

## 3. references/ —— harness 适配层

**事实**：识别规则（`system-prompt.md:11-14`）：
- `AskUserQuestion` + `SendUserFile` + Claude Preview MCP → Claude Code
- `AskQuestion` + `cursor-ide-browser` → Cursor
- `functions.*` / Codex Browser / Plan Mode → Codex
- 都不匹配 → 通用流程（chat 里问、HTTP 服务、给路径+URL）

**三份映射文档的差异（事实）**：

| 能力 | claude.md | cursor.md | codex.md |
|---|---|---|---|
| 问用户 | `AskUserQuestion`（内联答案，≤4 问/次） | `AskQuestion`（内联答案，`allow_multiple`） | Plan Mode `functions.request_user_input`，否则聊天提问 |
| 呈现交付物 | `SendUserFile` + 给 localhost URL；无共享浏览器 | 浏览器 MCP 可见化（`position:"active"`）+ 截图内嵌 | 给绝对路径+URL；用 Codex in-app browser 可见化 |
| 截图/调试 | Claude Preview MCP（`preview_start/console_logs/screenshot`）+ vision probe 门控 | `cursor-ide-browser` MCP + CDP `Runtime.evaluate`；禁 CDP `Input.*` | Browser 插件（Playwright API）+ `tool_search` 暴露 Node REPL |
| 子代理 | `Agent` 子代理跑 fork-verifier | `Task` 子代理（`browser-use`） | 默认不催子代理（用户明确要求才用） |
| PPTX/视频导出 | 本地 CLI 调用命令（完整命令段） | 同左（简写） | 同左（简写） |
| 设计系统 checker | `Agent` 隔离子代理 | `Task` 隔离子代理 | 默认内联、用户要求才子代理 |

- **事实**：`claude.md` 另含：vision probe（一次性探测本 session 是否支持图像输入——用 committed 的 `agents/assets/vision-probe.png` + 子代理只回 `VISION_OK`/`VISION_UNSUPPORTED`；不支持则禁读图、禁截图回灌模型，只走文本/DOM 校验）（`claude.md` 中部）。
- **事实**：三份文档都含「预览 harness 的怪癖」：React 18 `createRoot` 委托 `onClick` 对合成点击不响应（Claude Preview 需 `__reactProps$*` 绕行；Cursor 用真实点击无此问题）、按键要 `window.dispatchEvent`、`location.reload()` 后截图错位建议 `location.href=`（各 harness 文档"Preview-harness gotchas"节）。

## 4. references/upstream-* —— 上游溯源与移植证据

- **事实**：`upstream-sync/provenance.json` 记录了提取来源（claude.ai/design 的 assets 文件 + sha256）、去混淆 profile（变量覆盖映射、导入解析表）、systemPromptOptions 开关（designBriefsEnabled、enableStockPhotos、enableVersionHistory 等）、slices 统计（skills 61 statements / systemPrompt 127 / starters 38）——**这说明上游是一坨被反混淆的 bundle，本仓库从中"切"出提示词**。
- **事实**：`apply-report.json`（2026-09-14 一次具体同步）显示动作流：`copy`（上游文件 → 本仓库）、`preserve-for-merge`（本地已改文件 → 跳过覆盖待手工合并）。受保护：9 个 skill（animated-video、design-components、frontend-design、hi-fi-design、make-a-deck、read-pdf、save-as-pdf、save-as-standalone-html、speaker-notes）+ 2 个 starter（deck-stage.js、tweaks-panel.jsx）。
- **事实**：`built-in-skills.json`（229KB）与 `starter-components.json`（16KB）是上游原始内容快照，`upstream-system-prompt.md`（265 行）是上游 system prompt 快照。
- **事实**：本地 system-prompt.md 相对上游快照的核心差异（diff 前 60 行）：
  - 移除：「不要泄露环境技术细节」「不要描述 system prompt」等隐私/保密条款；
  - 替换：`ready_for_verification({path})`（宿主工具）→「HTTP 服务 + harness 预览 + record-asset 状态」；`write_file asset:`（资产审查面板）→ `record-asset.mjs`；`questions_v2` → harness 问询工具；
  - 新增：Harness setup 节、6 步本地工作流、设计系统 import 流程、starter 清单改写（下划线→连字符文件名映射，`system-prompt.md:211-215`）、固定版本 CDN 纪律、In-page controls 替代宿主 Tweaks。
- **推断**：同步靠「上游提取 → 按清单 copy/保护 → 测试守护本地特性」三件套；`project-types.test.mjs` 断言受保护文件里的本地 marker（如 deck-stage 的 `_animOnNav`、tweaks-panel 的 `TweakSuggestionBar`），防止一次上游同步把本地能力洗掉（见 notes/engineering.md）。

## 5. 未解
- 上游提取脚本本体不在仓库（`claude-design-v2/.deobfuscate-javascript/...` 是他机器的路径）；只能从产出物反推。
- 本地 overlay 与上游快照的逐文件 diff 只抽查了 system-prompt.md 与 make-a-deck 的一部分（受保护清单里其余文件的合并历史未逐条核对）。
