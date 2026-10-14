# 模块笔记：工程化（测试 / CI / 发布 / 防护）

## 1. 测试矩阵（事实）

**运行方式**：根 `package.json` 的 `"test": "node --test 'skills/baoyu-design/agents/tests/*.test.mjs'"`——全部用 Node 内建 test runner，**零测试依赖**。

| 测试文件 | 覆盖对象 | 要点 |
|---|---|---|
| `helpers.mjs` | 共享设施 | `tmpdir()`（用例级临时目录 + `t.after` 清理）、`runScript`（`spawnSync(process.execPath,…)`，120s 超时）、`makeDsFixture`（造"故意零 issue"的完整设计系统：tokens.css + styles.css（@import 闭包 + @font-face）+ Button.jsx（反向依赖 Zicon.jsx，验证 bundle 依赖序）+ Button.d.ts（@startingPoint/@default）+ Button.prompt.md + 非模块 Badge.jsx + @dsCard 卡 + 屏幕 startingPoint + README） |
| `asset-store.test.mjs` | `lib/asset-store.mjs` 纯单测 | 原地更新 vs inheritFrom 追加 vs no-op；viewport 局部更新保留旧 height；三种删除维度 + 空壳清理；readMeta（缺失/坏 JSON/数组）；bootstrap 不覆盖；writeMeta createdAt 一次 + updatedAt bump |
| `record-asset.test.mjs` | `record-asset.mjs` CLI 契约 | 用法错误统一 exit 64；首记 bootstrap `_d_meta.json`；同 path 更新不新增版本；绝对路径/`./` 归一化；`Card.dc.html→Card`；`--remove` 空操作友好 |
| `compile-design-system.test.mjs` | 编译器 | 三产物存在；manifest 与模型一致；**Zicon 块在 Button 之前**；`new Function('window', bundle)` 在 Node 执行验证导出与空 `__errors`；adherence 规则；namespace 重编译稳定；坏文件降级为 `__errors` 条目 |
| `check-design-system.test.mjs` | 检查器 | 干净 fixture 输出；**写零**（前后目录树相同）；`--verbose` 细节；无 CSS 入口 exit 2 |
| `ds-core.test.mjs` | 解析器 | pascalCase、token 分类（oklch/shadow）、var() 链式/fallback/自引用、导出收集、`.d.ts` 解析、风险全局名、issue 检测、忽略生成物目录（20 例） |
| `ds-prompt.test.mjs` | `_ds_prompt.md` 渲染 | bundle 接线、props 契约、token 白名单、围栏截断平衡 |
| `import-design-system.test.mjs` | 导入器 | 子集复制、`_d_meta.json` 绑定、幂等、**不碰 DS 源树**、绝对路径拒绝 |
| `import-figma.test.mjs` | Figma 导入器 | 对 `fixtures/minimal.fig` 跑 outline/mount/design-system |
| `build-preview.test.mjs` | preview 构建器 | CLI 冒烟 + 自包含断言（无 `href="../styles.css"`） |
| `project-types.test.mjs` | **文档/路由一致性守卫** | 13 类 id 唯一；逐类读对应 `built-in-skills/*.md`（校验 frontmatter、>120 字节）与 starter 文件存在 + 指定 marker 串（slides 的 `data-anim`/`noscale`；animation 的 `OM_SCENES`/`CompositionStage`；html-email 的 `No JavaScript` 等）；断言 system-prompt.md 提到全部 13 类；断言上游快照 >20KB 且含 `ready_for_verification`；**守护本地特性 marker**：deck-stage 必含 `multi-selection`/`drag to reorder`/`_animOnNav`/`data-anim-auto-reverse`/`noscale`；tweaks-panel 必含 `TweakSuggestionBar`/`__edit_mode_chat`；design-canvas 必含 `WHEEL_ZOOM_STEP = Math.log(1.1)`/`WHEEL_BURST_MS = 200` 且不含旧实现 `Math.sign(e.deltaY) * 0.18` |
| `vision-probe-docs.test.mjs` | **纯文档断言** | vision-probe-agent.md 必须只读、单 token（VISION_OK/VISION_UNSUPPORTED）、不读真实截图；claude.md 把图片 Read/截图门控在探针后；探针图必须是 committed PNG（校验 magic bytes）且禁止 writeFileSync；每 session 一次并缓存 |

- **事实**：gen-pptx / gen-video 有各自目录内的独立测试（见 notes/gen-pptx.md §7、notes/gen-video.md §5），**不在根 CI 里**。
- **事实**：无 E2E/浏览器级自动化；`capture-editable`/`setup`/`media-browser`/`render-node` 主体靠人工验证。

## 2. CI（事实）

`.github/workflows/tests.yml`：触发 `pull_request` + `push`；`permissions: contents: read`；单 job `ubuntu-latest` + Node 22：checkout → setup-node →（有 lock 用 `npm ci`，否则 `npm install --package-lock=false`）→ `npm test`。

**推断**：CI 只保证"工具链单元测试 + 文档一致性守卫"两件事；对 skill 本体（提示词）的质量没有自动化门禁——因为那本质上是内容。

## 3. 发布流程（事实）

- `.claude/skills/release-skills/SKILL.md`：**本仓库自带、供本仓库发版用的项目级技能**（不是分发给用户的部分）。通用发布工作流：探测版本文件（package.json/pyproject/Cargo/marketplace/VERSION）→ 扫 `CHANGELOG*.md` 按文件名后缀判语言 → 分析 tag 以来变更（conventional commits）→ 版本决策（flags > BREAKING > feat > patch）→ 生成多语言 changelog（第三方贡献者标 `(by @username)`）→ 按 skill/module 分组提交（`feat(baoyu-design): …`）→ 确认 → release commit + annotated tag + GitHub Release（`gh release create/edit --notes-file`）。
- 本仓库实际使用：版本 1.3.0（package.json）+ 双语 changelog（CHANGELOG.md / CHANGELOG.zh.md）+ tags v1.0.0…v1.3.0（`git tag`）；无 `.releaserc.yml`（走默认分支）；`private:true` 不涉及 npm 发包。
- **事实**：CHANGELOG 的条目风格与这套流程吻合（多语言分节、credits 标注 PR 作者）。

## 4. 防线的几条设计红线（推断）

1. **文档→代码一致性**：project-types 测试确保"JSON 路由表、system-prompt 表格、SKILL.md 分支"三份不漂移。
2. **上游同步防回归**：受保护文件 + marker 断言，防止一次同步把本地能力洗掉（配合 `apply-report.json` 的 preserve-for-merge 机制）。
3. **安全边界**：build-preview 的路径越界阻断（`abs.startsWith(root)`）与 import-design-system 的绝对路径拒绝各有测试（CHANGELOG 1.3.0 记录 V-001 修复）。
4. **AI 能力门控**：vision probe 的"文档层面"也被测试钉住（因为真实探测依赖运行时模型）。
