# 模块笔记：gen-pptx（HTML deck → PPTX 本地导出器）

> 源码：`skills/baoyu-design/agents/gen-pptx/`（TS + esbuild + Playwright + vendored PptxGenJS），构建产物 `dist/cli.mjs`（Node 侧）+ `dist/capture.iife.js`（注入页面的捕获 bundle）。
> 核心思想（README）：**不解析 HTML——在真浏览器里渲染，再把结果翻译成 PowerPoint**。

## 1. 构建形态与流水线

- **事实**：两个 esbuild 产物（`package.json:11-17`）：`build:cli`（`src/cli.ts` → `dist/cli.mjs`，`--alias:pptxgenjs=./vendor/pptxgenjs/src/pptxgen.ts`；playwright/jszip 外部化）与 `build:page`（`src/browser/entry.ts` → IIFE `dist/capture.iife.js`）。注入器按 `import.meta.url` 并列读取（`src/orchestrator/inject.ts:19-27`）。
- **事实**：CLI 参数 `--url/--config/--out`；URL 强制 http(s)（`src/cli.ts:41-44`）；config 文件或 stdin（`--config -`）；deviceScaleFactor：screenshots=2 / editable=1（`cli.ts:98-101`）；preflight（node≥18、playwright、chromium）失败 → exit 1 + 一次性安装提示（`cli.ts:53-76`）；用法错误 → exit 64（`cli.ts:22-26`）。
- **事实**：编排（`src/orchestrator/run.ts:28`）：清洗输入（`core/sanitize.ts`）→ 注入捕获 bundle → screenshots 模式锁 viewport 到幻灯片尺寸 + 双 rAF → `setup()` → 逐页处理：
  - editable：逐页 `captureEditable` → 收集哈希/媒体失败/动画告警 → `buildEditablePptx`（期间回调 `resolveMedia` 批量解析媒体）→ `validate`（`run.ts:70-124`）。
  - screenshots：页内切页+稳定，像素抓取在 Node 侧（`page.screenshot`），djb2 取 dataURL 中段 4096 字符做相邻页判重（`run.ts:125-152`）。
- **超时纪律**：所有跨页调用有硬超时（setup 15s；capture `15000 + delay`）（`inject.ts:44-56`）；错误文案必须含 "timed out" 以便给重试提示（`inject.ts:41-43`、`errors.ts:13-20`）。

## 2. 浏览器内捕获（in-page）

- **事实**：`window.__genpptx` 只暴露 4 个方法：`setup(input)`、`captureEditable(spec, fontSwaps)`、`captureScreenshot(spec)`、`resolveMedia(refs)`（`browser/entry.ts:13-27`）；Node 与页面全部走 `page.evaluate`（JSON 可序列化，`types.ts:1-3`）；切页用页内 `new Function(spec.showJs)()`（`capture-editable.ts:35-41`）。
- **setup() 预处理**（`browser/setup.ts:21-298`）：隐藏 chrome（`35-41`）→ 字体替换（本地 `@font-face{src:local()}` + 探针 face；Google Fonts 走 fetch CSS2 改写 font-family，支持回退无字重 URL）（`47-121`）→ **撤销 transform 缩放**（给 `resetTransformSelector` 元素设 `noscale` + 显式 px 宽高 + `transform:none`，shadow root 内优先测 `.canvas`，存 `resetRect`）（`124-141`）→ `document.fonts.ready` 与 8s 竞速 + 本地探针 500ms（`143-167`）→ 字体存在性 canvas 测宽校验（`173-258`）→ 演讲备注（deck-stage `data-speaker-notes` 优先，回退 `#speaker-notes` JSON）（`262-294`）→ 统计 `[data-anim]` 数（`296`）。

## 3. editable 模式：DOM → JSON 树 → PptxGenJS

- **序列化**（`capture-editable.ts:121-342`）：产出 `SlideNode{tag,rect,style,children,text?,href?,imageUrl?,svg?,gradient?,gradientRadius?,untransformedRect?,liIndex?,anim?}`；剪枝 display:none / 零尺寸；style 白名单键（`browser/dom-style.ts:8-63`）+ 颜色归一 + fontFamily 三级解析（swap→generic→探针）；`#text` 用 `Range.getBoundingClientRect()` 取文本自身盒（`300-334`）。
- **语义细节**：`<a href>`、非 disc 列表的 1-based `liIndex`（`139-148`）、旋转前 AABB `untransformedRect`（`150-157`）；`data-anim` 解析 + 三类告警（嵌套/隐藏/根上禁用）（`161-186`）；IMG/OBJECT/CANVAS/内联 SVG 处理（SVG 克隆后绝对化 `<image href>` 存 `outerHTML`）（`188-215`）；shadow `<image-slot>` 提升为 imageUrl 并避免双重绘制（`219-277`）；背景图/渐变提取 + 圆角半径（`278-297`）。
- **单位**：`px/96=inch`、`px×0.75=pt`（`core/units.ts:4-12`）；坐标先减 slide 原点并 clamp（`render/context.ts:61-71`）。
- **渲染映射**（`render/render-node.ts:46-533`）：
  - 透明度组效果 → 祖先连乘下传（`:46-49,108`）。
  - 文本叶 → `addText`（fit:"shrink"、transparency、lineSpacingMultiple、宽度补 3%/高度补 4px、wrap 判定）（`:61-99`）。
  - 背景/统一边框/阴影 → 单个 `addShape`（hex+transparency；圆角 ≥0.49 且近方形 → ellipse，否则 roundRect+rectRadius）（`:130-198`）；非统一边框拆 4 rect 并按旋转绕中心转（`:200-247`）。
  - 图片/画布/SVG/背景图 → `addImage`（cover/contain、固有尺寸直用避免裁切漂移、rounding、rotate、transparency）（`:249-295`）。
  - 列表快捷路径：`detectList` 仅限样式一致的简单 ul/ol 且**无 data-anim**；bullet 映射 decimal→numberType、circle→`25CB`、square→`25A0`（`render/list.ts:13-36,55-70`，`render-node.ts:297-362`）。
  - 行内 run 合并（`render/text-runs.ts`）：有盒装饰/媒体/动画子树都拒绝合并（`:84-123`）；多 run 按 `\n` 拆视觉行再逐段发 `{text,options,breakLine}` 数组（`render-node.ts:445-511`）。
  - 子节点排序：仅当存在 positioned 且非 0 z-index 才按 z 排序（`:513-533`）。
- **渐变光栅化**（`browser/gradient.ts`）：PptxGenJS 无渐变 → 页内把 linear/radial 渐变（含 alpha、双位置色标、`to <side|corner>`、圆角裁剪）画到 canvas 转带 alpha PNG，最长边 1280px；conic/repeating 不支持 → 回退首色标（`gradient.ts:1-10,135-233`；`render-node.ts:122-128`）。
- **媒体缓存**：键 = `imageUrl 原文 | svg:<fnv1a36>:<w>x<h> | grad:<hash>:<W>x<H>:r<radius>`（`render/media-cache.ts:10-30`）；跨所有 slide 去重收集 refs → 一次性 `resolveMedia`（6 路并发池；同源带 credentials 跨源 omit；SVG 2× 光栅化上限 2048）（`media-browser.ts:8,14-26,133-164,166-190`）。

## 4. 动画：`data-anim` → 原生 `<p:timing>`

- **解析**（`core/anim.ts:166-314`）：效果名必须命中 `DURATION_DEFAULT` 表（否则跳过导出为静态）；trigger ∈ click|with|after（默认 **after**）；delay/duration 整型 clamp；appear/disappear 忽略 duration；方向族（fly/wipe 四边、float 上下、split 等 horizontal|vertical、box 系 in|out、strips 四角）非法值回退+告警；rotate/scale 空操作丢弃；path 必需 `data-anim-path`（`L`/`C` 子集，重基到 0,0，≤32 点）；repeat 2..100；auto-reverse 仅 spin/grow/shrink/path；`effectiveDurationMs = dur × repeat ×(autoRev?2:1)`（`:317-318`，deck-stage 的 `_animSteps` 内联同一算法 → 时轴一致性）。
- **形状绑定**：渲染每个形状前写 `objectName = anim-s<slide>-n<index>-k<k>`，成功后按 vendored 库的 `cNvPr id = _slideObjects.length + 2` 计算 spid（`render/context.ts:46-58`，契约由 `test/vendor.test.ts` 守护）。
- **timing XML**（`render/timing.ts:198-584`）：tmRoot→seq→mainSeq→（每 click 组一个 par）→ 每动画一个 par(start offset)→ 每形状一个 effect par；preset 表按 LibreOffice/ONLYOFFICE 校验（`:64-130`）；行为合成 = `setVis` + `animEffect`(filter) + `flyAnim`(ppt_x/ppt_y) + `animMotion` + spin/grow/pulse/teeter 缩放旋转行为（`:289-444`）；**多形状元素的模式滤镜（wheel/blinds/…）只在主形状播，其余退化 fade**（`:281-288`）；repeat/autoRev 落点：单行为效果写主行为 cTn，复合效果用内层 wrapper par；autoRev path 把返回段烘进几何（`:24-58,431-437`）。
- **事实**：`buildEditablePptx` 丢弃 spids 空的 entry 并产出 `animHidden` 告警（`render/build-editable.ts:84-105`）。

## 5. vendored PptxGenJS（为什么以源码形态 vendored）

- **事实**（`vendor/pptxgenjs/VENDOR.md`）：上游 `version-3.13.0`（MIT）；只打了 2 个 patch——① `gen-xml.ts` 尾部插入 `(slide._timingXml || '')`（`vendor/pptxgenjs/src/gen-xml.ts:1569`）② `core-interfaces.ts` 加 `_timingXml?: string`（`core-interfaces.ts:1790-1792`）。所有动画逻辑在 gen-pptx 自己的 `src/`，patch 只负责"把算好的 timing 字符串拼进 slide XML"。
- **事实**：依赖的库内部契约：`cNvPr/@id = idx+2`、`options.objectName → cNvPr/@name`、`makeXmlSlide` 以 `</p:clrMapOvr></p:sld>` 结尾（patch 点）（VENDOR.md:9-13）；升级流程 = 换源码 → 按锚点重放 patch → `npm test && npm run build`（VENDOR.md:15-19）。
- **事实**：`render/context.ts:6-16` 定义 `PptxSlide` 结构化视图（`_slideObjects/_timingXml/background`）反向读取库内部状态。

## 6. 验证与输出

- **事实**：`validate/validate.ts:18-135`（≤10 类 flag，全部 advisory）：`fonts_timeout`、`font_swap_failed`、`reset_selector_miss`、`slide_size_mismatch`（>2px）、`duplicate_adjacent`（djb2 相同 → showJs 没切页）、`duplicate_majority`、`no_speaker_notes/notes_count_mismatch/notes_uniform_nonempty`、`animations_ignored_screenshots`、`images_failed`。另 3 个由编排层补：`animation_invalid / animation_nested / animation_hidden_target`（`run.ts:100-111`，消息 capped 8 条）。
- **事实**：stdout 单行 JSON：`{ok, file, slides, animations, bytes, flags:[{code,message}], warnings, speakerNotes}`；失败 `{ok:false,error}`；退出码 0/64/1（`cli.ts:103-130`）。
- **事实**：文件名 `safeBasename`（保留 Unicode 字母数字，路径分隔符→`_`，强制 `.pptx`）（`core/filename.ts:4-14`、`orchestrator/output.ts:8-21`）。

## 7. 测试覆盖（`test/*.test.ts`，`node --test`，pretest 打 vendored bundle）

- `core.test.ts`：纯函数（颜色/渐变首色标/透明度/单位/圆角/旋转/阴影/行距/字距/下划线/noWrap/字体解析/safeBasename）。
- `anim.test.ts`：属性语法表驱动（默认值、clamp、方向族校验、repeat/autoRev 矩阵、effectiveDuration、path 解析与 OOXML 换算）。
- `timing.test.ts`：对生成的 XML 做正则结构化断言（骨架、click/with/after 分组、逐效果片段、repeat 落点、全效果 sweep 的 cTn id 唯一 + 标签平衡）。
- `anim-render.test.ts`：动画子树不被折叠/吸收（列表与 run 合并路径的动画阻断）。
- `gradient.test.ts`：渐变解析/单调化。
- `vendor.test.ts`：真跑 `pptx.write()` + unzip 断言 patch 落地与 spid/timing/xml 契约（**升级护栏**）。
- **盲区（事实）**：`capture-editable.ts`/`setup.ts`/`media-browser.ts`/`validate.ts`/`render-node.ts` 主体无自动测试（依赖真实浏览器 DOM）。
