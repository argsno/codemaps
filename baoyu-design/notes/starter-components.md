# 模块笔记：starter-components（27 个脚手架，~21481 行）

> 定位：无构建的 JS/JSX 组件，`cp` 到项目即用；每个文件头部自带 usage 说明（`system-prompt.md:208-209`）。上游带下划线的名字在本仓库统一改成连字符（`animations_v3.jsx` → `animations-v3.jsx`，`system-prompt.md:211-215`）。

## 0. 共同模式（事实）

- **轻量 custom element + 幂等守卫**：`if (customElements.get('x')) return`（`ios-shell.js:47` 等）→ `CSS` 常量串 → `ensureCss()` 注入一次 → `customElements.define`（`ios-shell.js:125`、`chrome-shell.js:106`、`x-shell.js:112`、`social-frames.js:100`…）。
- **属性驱动**：`observedAttributes` + `_build()` 重建；image-only 等开关用**双选择器** `[image-only=""]`/`[image-only="true"]`（注释明确「不要简化」——兼容绑定渲染出的字符串 "false"）（`ios-shell.js:81-104`、`chrome-shell.js:79-89`）。
- **宿主 chrome 标记**：装饰件（状态栏、home indicator、浏览条）带 `data-omelette-chrome`，导出/打印时隐藏（`ios-shell.js:199-218`）。
- **两种 DOM 策略**：舞台类（deck-stage、image-slot、watercolor-kit、doc-page、three-d-stage、data-overlay）用 Shadow DOM；设备/社交外壳刻意用 light DOM（`_build` 重插槽），避免与宿主 DC 模板快照冲突（`ios-shell.js:132`、`chrome-shell.js:107-111` 拒绝在 `<x-dc>` 模板内构建）。
- **导出友好**：尺寸用 CSS 变量钉死避免 flex 挤压（`--ios-outer-w` 等，`ios-shell.js:49-62`）；`data-om-frame-export` 帧可单帧导出 PNG（`x-shell.js:26-32`、`social-frames.js` 提供整套下载工具条）。

## 1. deck-stage.js（~3760 行，幻灯片舞台）

- **事实**：`<deck-stage width height noscale no-rail>` 自定义元素；每页一个直接子 `<section>`；非活动页**隐藏而非卸载**（保状态）（文件头文档 `deck-stage.js:79-81`）。
- **缩放/呈现**：`min(vw/designW, vh/designH)` 的 `transform:scale()` + letterbox（`_fit()`，`deck-stage.js:1706-1732`）；全屏不靠 Fullscreen API，而由宿主 postMessage 通知置 `_presenting`（`1675,1754-1763`）；`noscale` 时 `transform:none` 且关闭动画引擎（`1709-1718, 3211-3215`）——**这是 gen-pptx 抓取 1:1 几何的挂点**。
- **键盘/触摸**：←→↑↓/PgUp/PgDn/Space/Home/End/数字键；R 回第 0 页；触屏点半区翻页且跳过交互元素（`827,1912,1934,154`）。
- **缩略图 rail**：可拖宽（宽度写 localStorage `deck-stage.railWidth`）、静态 clone 缩略图 + MutationObserver 脏检查、多选/shift/cmd 点选、拖拽重排、右键菜单、Skip（`data-deck-skip`）、Delete+确认框；rail 在展示模式/noscale/窄屏/`no-rail` 下抑制（文件头文档 `(g)` 段）。
- **dc-op 协议**：rail 的结构性变更不改 DOM，而是 `dispatchEvent(new CustomEvent('dc-op', {detail: op, bubbles, composed}))`；有宿主 DC runtime 时等 `postMessage({__dc_op_ack:true,applied})` 才解锁输入；纯原生 .html deck 走本地自变更回退（`2769-2864`）。
- **data-anim 构建动画**：效果表 `ANIM_EFFECTS`（`181-260`）与 gen-pptx `core/anim.ts` **共享算法/默认时长**；`_animSteps` 把 trigger=click|with|after 折成步骤并对齐 gen-pptx 的 timing 分组（`3380-3409`）；前进先消费未播 click 步再翻页（派发 `deckstep`）；后退直接"已构建"（`3414-3460`）；reduced-motion 瞬间播放但保留 click 门控（`3107`）。
- **打印**：`@media print` 每页一张纸；`beforeprint` 给所有 slide 打 `data-deck-active` 导出"完成态"（防捕获入场前状态）（`833-859,1453-1471`）。
- **持久化**：deck 自身不存页码；宿主用 URL `?slide=`，组件按 `location.hash` 恢复（文件头文档 `97-99`）。

## 2. design-canvas.jsx（平移缩放画布）

- **事实**：Figma 式画布；导出 `DesignCanvas/DCSection/DCArtboard/DCPostIt`；状态存 `.design-canvas.state.json` sidecar（读 `fetch`、写 `window.omelette.writeFile`——宿主内可编辑，普通浏览器只读）（`2-22,138-165`）。
- **视口持久化**：key `'dc-viewport:' + location.pathname`（区分同项目多文件），debounce 写 localStorage；`--dc-inv-zoom = 1/scale` 让屏幕恒定尺寸 UI 反向缩放；同时 postMessage 宿主（`263-303`）。
- **交互**：滚轮 1.1×/次且 200ms 突发内锁定 zoom/pan 意图（`WHEEL_ZOOM_STEP/WHEEL_BURST_MS`，`412-424`——对应 CHANGELOG 1.3.0 的"软化滚轮缩放"修复）；背景拖动平移、Safari 捏合、缩放锚点复位、`__dc_present` 演示（`381-523`）。
- **artboard 管理**：section 顺序/隐藏/删除持久化；`srcKey`（artboard id 列表）变化即丢弃旧状态（`178-213,589-645`）。
- **导出**：`dcExport` 内联 computed style + 内联 @font-face + data URL 图，序列化成 SVG→PNG/HTML 下载（`654-763`）。

## 3. 动画引擎三代（重要）

| 引擎 | 定位 | 导出契约 | `?capture` | 桥全局 |
|---|---|---|---|---|
| `animations.jsx`（v1） | 兼容引擎（旧项目） | 事件 + 桥 | ✅ 内建 | **`window.__animStage`**（`setTime/setPlaying/getTime/duration/fps/width/height` + `captureActive`）（`animations.jsx:409-432`） |
| `animations-v2.jsx` | 兼容引擎 | 事件（+ `<VideoSprite>`） | ❌ | 无桥 |
| `animations-v3.jsx`（当前） | 连续合成引擎（唯一的时间线引擎） | 事件 | ❌ | 无桥 |

- **v3 心智模型（事实）**：整段动画 = **一棵树 × 一个 authored 时钟的纯函数**；`OM_SCENES` 是唯一结构来源；只有一个 rAF 循环在 `Stage`；播放模式 loop/times:N；键盘 Space/←/→/0（`animations-v3.jsx:4-16,428-490`）。
- **导出契约（事实）**：根元素 `<svg data-om-exportable-video-with-duration-secs>` + `foreignObject`；内联 @font-face 并打 `data-om-fonts-inlined`；监听 `data-om-seek-to-time-frame`（`detail:{time,sync,playing}`），`sync===true` 时用 `ReactDOM.flushSync` 同步提交并广告 `data-om-sync-seek`；`?`宿主播放条 seek 流带 `playing` 标记 + 400ms 看门狗（`282-340,500-563,606`）。
- **场景/cue**：`ccDerive`（scene 列表→cue 表）、`ccWarp`（播放时间→作者时间；trim/speed 重放作者切片，编排永不截断）、Proxy cue 表引用不存在 section 时返回 NaN + 预览徽标（徽标在导出 svg 之外）（`953-1051`）。
- **可编辑性通道**：`SceneSync/PlaybackSync` 把 `OM_SCENES/OM_PLAYBACK` 原始字符串逐字盖回可导出根属性，宿主编辑后能写回你的文件（`892-938`）。
- **水彩集成**：`watercolor_kit.js` 缺失时降级提示；`WatercolorPainting`（按笔画顺序显现）、`WatercolorStroke`（multiply 混合叠加）、`WatercolorReveal`（预烘焙帧序列，必须 data URL 否则导出空白）（`1159-1330`）。
- **⚠ 桥接错位（事实，已在 questions.md 记录）**：`__animStage` 只在 v1 存在；v2/v3 用 DOM 事件。而本地 `gen-video` CLI 的 seek 只做 `bridgeGlobal.setTime(...)`（`gen-video/src/orchestrator/driver.ts:49-60`，默认 `__animStage`，`run.ts:135`）——它**不派发** `data-om-seek-to-time-frame`。`export-as-video.md:13,34-42` 又以"当前 `animations.jsx` 注册 `window.__animStage`、支持 `?capture`，零配置"为准。因此：**当前主力引擎（v3）的动画在本地用 gen-video 导出时没有文档化路径**（需自加桥或该 CLI 后续适配事件契约）。

## 4. image-slot.js（可填充图片槽）

- **事实**：头注释即规格；属性 `id/shape/radius/mask/fit/placeholder/src/credit/credit-href`（`image-slot.js:1-89,439`）。
- **持久化**：sidecar `.image-slots.state.json` 同级文件；读 `fetch`、写 `window.omelette.writeFile`（序列化写入防并发互覆，`172-225`）；无宿主时只读。
- **裁剪**：`data-reframe` 模式（拖动平移/角柄缩放/滚轮缩放），偏移按"帧百分比"存储 → 响应式重排/PPTX 导出后裁剪不变；提交派发 `image-slot:reframe`（`618-825,1030`）。
- **署名**：Unsplash 子域（含 `plus.unsplash.com`）无 `credit` 时**渲染错误块而非照片**（fail-closed）；回链自动补 utm；credit 在用户拖入图/导出/打印时隐藏（`94-130,390-410`）。
- **导出友好**：`attachShadow({mode:'open', clonable:true})` —— 让 PPTX 捕获与缩略图能复制 shadow（`496`）。

## 5. tweaks-panel.jsx（宿主耦合的 Tweaks 面板）

- **事实**：它"拥有宿主协议"：监听 `__activate_edit_mode/__deactivate_edit_mode`，发送 `__edit_mode_available/__edit_mode_set_keys/__edit_mode_dismissed`；挂载时先注册监听再 announce（顺序敏感）；`useTweaks` 持久化不落 localStorage 而是**把 edits 发给宿主重写文件里的 `/*EDITMODE-BEGIN*/…/*EDITMODE-END*/` 块**；带 `data-omelette-chrome` 与 `--dc-inv-zoom`（`tweaks-panel.jsx:8-9,59-62,170-195,228-242,281`）。
- **结论**：普通浏览器里打开原文件，面板不会出现也无法保存——所以 system-prompt.md 要求 agent 自建面板 + Show/Hide 开关（`system-prompt.md:189-190`），用 tweaks-panel 时把开关接到自己的 state。

## 6. watercolor-kit.js（确定性水彩）

- **事实**：一幅画 =「确定性 JS 函数」（同 kit/颜料/种子随机 → 同一函数永远同一张图）；`wcRng` 自研种子随机、`WC_PIGMENTS` 颜料表、noise/fBm/EDT/blur 数值工具、`paper(W,H)` 提供 `wash/glaze/ink/hatch/splatter/dryStroke/reserve/…` + `ops/seek(u)/renderUpTo(i,t)`（`watercolor-kit.js:2-7,91-238`）。
- **重放**：`<watercolor-kit>` 元素赋值 `.painting` 后逐笔播放，`play/pause/seek(u)/progress` + `watercolor-done` 事件；`frame()` 单帧 data URL；`bake()` 预烘焙；`layers()` 把画拆成笔画层（供 v3 逐笔驱动）；警告"painting 函数必须模块级定义一次"，否则按源码文本复用缓存会串味（`23-67,935-1170`）。

## 7. 其余

- `doc-page.js`：流式可打印页 + 固定单页文档；拥有纸张几何与打印规则（`system-prompt.md:227`）。
- `three-d-stage.js`：Three.js 查看器（影棚光照、orbit、auto-frame、OBJ/GLB 下载）。Shadow DOM（`three-d-stage.js:150`）。
- `chart-stage.js` / `data-overlay.js`：图表舞台与带来源数据叠加层（data-overlay 96KB，最大）. `data-overlay.js:734` shadow。
- `anim-demo.html`：演示样例。
- 社交/设备外壳族（ios-frame/android-frame/macos-window/browser-window/ios-shell/chrome-shell/file-window/facebook/instagram/instagram-story/linkedin/pinterest/post-card/reddit/social-frames/tiktok/x/youtube）。
- **未知**：`deck-stage-patch.md` / `design-canvas-patch.md` 是本地 patch 说明文档（供上游同步时重放本地改动），未逐行读。
