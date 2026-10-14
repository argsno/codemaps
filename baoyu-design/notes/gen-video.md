# 模块笔记：gen-video（HTML 时间线动画 → MP4/WebM/GIF 本地导出器）

> 源码：`skills/baoyu-design/agents/gen-video/`（TS + Playwright + ffmpeg 子进程）。构建产物 `dist/cli.mjs`。
> 核心思想：**用桥逐帧 seek 时间线，每一帧都是真实截图，帧流管道交给 ffmpeg 编码**。

## 1. 流水线与模块职责

- **事实**：`src/cli.ts`（argv→input→驱动导出→单行 JSON）→ `orchestrator/*`：
  - `run.ts`：主编排（等待桥→暂停→解析时长/帧数→CSS 兜底→逐帧 seek+screenshot→喂 ffmpeg→validate）（`run.ts:127-231`）。
  - `driver.ts`：Playwright 封装（launch、`screenshotBuffer`、`seek`、close）（`driver.ts:27-71`）。
  - `encode.ts`：ffmpeg 参数构造 + 背压帧流写入 + stderr 尾部捕获（`encode.ts:19-101`）。
  - `output.ts`/`filename.ts`/`errors.ts`：路径/清洗/错误提示。
  - `validate/validate.ts`：4 类非致命 flag（`validate.ts:16-56`）。
- **事实**：CLI 参数 `--url/--config/--out`；URL 强制 http(s)；preflight 检查 node≥18、playwright、chromium、**ffmpeg**（`ffmpeg -version` 探测），失败 → 一次性安装提示 + exit 1（`cli.ts:64-101`）；用法错误 exit 64（`cli.ts:22-27`）。
- **事实**：`--config -` 读 stdin；`withCaptureParam()` 给 URL 追加 `?capture=1`（值写死 `1`；页面侧把 `"0"/"false"` 视为关）（`cli.ts:53-62,118-119`）。

## 2. 浏览器驱动

- **事实**：动态 `import("playwright")`（故意非顶层，便于 preflight 给友好提示）；`chromium.launch({headless:true})`；context viewport=width×height，`deviceScaleFactor` 默认 2（超采样）；page 默认超时 30s，`goto(waitUntil:"load")`（`driver.ts:27-41`）。
- **事实**：桥等待 `waitForSetup`：8s 内每 50ms 轮询 `window[bridgeGlobal]` 且要求 `typeof setTime === "function"`；再与 `document.fonts.ready` 竞速；从桥读 `duration` 与 `captureActive`（try/catch 防 getter 抛错）（`run.ts:33-77`）。
- **事实**：逐帧 seek：`page.evaluate` 内 `window[g].setTime(time)` + **双 rAF 沉降**（等 React 提交）；桥不存在时静默 no-op（这正是 `duplicate_frames` 的成因之一）（`driver.ts:49-60`）。帧时刻 `tSec = (startMs + i*1000/fps)/1000`（第一帧落在 startMs）（`run.ts:185`）。
- **事实**：截图是整视口 PNG 原始字节（不裁剪）；DPR=2 → PNG 实为 2W×2H；降采样交给 ffmpeg `scale=W:H:flags=lanczos`（`driver.ts:43-46`、`encode.ts:20`）。
- **事实**：capture 未生效时的 CSS 兜底（仅当给了 `hideSelectors`/`resetTransformSelector`）：注入 `<style>`——全局 `transition:none`、hide 选择器 `display:none`、reset 选择器 `transform/box-shadow:none` + 强制 Stage 尺寸、父容器顶对齐+`overflow:visible`、文档/body 背景透明（`run.ts:78-116,160-170`）。参考选择器：v1 starter 自带 `data-stage-chrome` / `data-stage-canvas` 钩子。

## 3. 编码（ffmpeg）

- **事实**：输入段 `-y -f image2pipe -c:v png -r <fps> -i pipe:0`（由输入侧 `-r` 决定时长）。
- **事实**：mp4（默认）= `libx264 -preset slow -crf -pix_fmt yuv420p -movflags +faststart`；webm = `libvpx-vp9 -b:v 0 -crf`；gif = `palettegen=stats_mode=diff` + `paletteuse=dither=bayer:bayer_scale=5`（`encode.ts:24-56`）。
- **事实**：背压处理（write 返回 false 等 drain）；stderr 保留尾部 ≤8000 字符，失败只带最后 3 行；`destroy()` = SIGKILL（`encode.ts:58-101`）。

## 4. 输入契约与输出

- **事实**：`GenVideoInput`（`types.ts:8-35`）：`width/height`（必填）、`duration?`（否则读桥）、`fps`(30)、`deviceScaleFactor`(2)、`startMs`(0)/`endMs`(duration*1000)、`format`(mp4|webm|gif)、`crf`(18)、`captureParam`("capture")、`bridgeGlobal`("__animStage")、`hideSelectors?`、`resetTransformSelector?`、`filename?`。
- **事实**：帧数 `max(1, round(totalMs/1000*fps))`（`run.ts:158`）。
- **事实**：stdout JSON：`{ok,file,frames,fps,duration,width,height,bytes,flags,warnings}`（warnings 恒空）；失败 `{ok:false,error}`；退出码 0/64/1（成功即使带 warning 也是 0）。
- **事实**：4 个实际产出的 flag：`zero_duration`（只渲 1 帧）、`fonts_timeout`、`capture_mode_off`（未 capture 且无兜底）、`duplicate_frames`（仅当帧数 ≥8 且 >85% 帧与前一帧相同——阈值刻意放宽，"讲解型动画每节拍停几秒"是正常的）（`validate.ts:19-53`，djb2 中段 4096 字节哈希 `run.ts:118-124`）。

## 5. 测试（`test/core.test.ts`）

- **事实**：`buildFfmpegArgs`（mp4/webm/gif 差异、crf 透传、最后参数是输出路径）；`validate` 6 例（含"capture 关但有兜底不报警"与"52% 重复帧不报警"的边界）；`safeBasename`（CJK/西里尔保留、路径分隔符替换）；`resolveOutputPath`（扩展名归一）。
- **事实**：`driver.ts`/`run.ts`/`encode.ts` 运行时/preflight 均无单测；根 CI（`node --test agents/tests/*.test.mjs`）**不覆盖** gen-video——需在 `agents/gen-video` 内单独 `npm test`。

## 6. ⚠ 桥接错位（核心发现，已写入 questions.md）

**两套时间线协议并存**：

| 协议 | 提供者 | 消费方 |
|---|---|---|
| `window.__animStage` 桥（`setTime(sec)`/`setPlaying`/`duration` + `?capture`） | **仅 v1 `animations.jsx`**（`animations.jsx:409-432`；`?capture` 语义 `337-407`） | gen-video CLI（默认 bridgeGlobal=`__animStage`） |
| `data-om-seek-to-time-frame` DOM 事件（+ `data-om-sync-seek` 广告） | **v2/v3 引擎**（`animations-v3.jsx:500-563`；`exportable-video.md:12`）与宿主产品导出器 | 宿主产品（claude.ai/design）的 Share→Export→Video |

- **事实**：gen-video 的 seek 只认 `bridge.setTime`，不派发 DOM 事件（`driver.ts:49-60`）；全目录 grep `data-om-` 零命中；默认桥 `__animStage`（`run.ts:135`）。
- **事实**：`export-as-video.md:13` 把 `animations.jsx` 称作"current starter"并承诺零配置；但 system-prompt.md:223 明确 v3 是当前引擎、v1/v2 只服务旧项目；`project-types.json` 的 animation 路由也指向 `animations-v3.jsx`。
- **推断（高置信）**：这是一处真实的版本错位——gen-video 是在 v1 时代为 `__animStage` 写的；v2/v3 改走事件契约后，本地 CLI 未同步适配。后果：**用当前引擎（v3）新做的动画，在本地用 gen-video 导出没有文档化路径**（要么页面自加 `setTime` 桥，要么手工派发事件，要么等 CLI 适配）。
- **未知**：上游宿主导出器如何驱动 seek（事件 detail 含 `frame`/`playing`，超采样在宿主侧？）——事件契约里 `frame` 字段的语义只在 `exportable-video.md:12` 一笔带过。
