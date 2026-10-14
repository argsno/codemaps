# 未解问题清单

> 随分析更新。状态：🟡未解 / 🔵已澄清 / ⚪不必再查

## 待解

1. 🟡 **README 口径 vs 代码口径**：README/package.json 说 "11 languages"，`ctx_execute` 的 language 枚举是 12 个（javascript, typescript, python, shell, ruby, go, rust, php, perl, r, elixir, csharp，server.ts:1694-1707）。究竟哪个是设计口径？（不影响功能，只是文档漂移）
2. 🟡 **`getRoutingInstructions()` 无调用点**：src/adapters/* 里定义了从 `configs/<platform>/AGENTS.md|SYSTEM.md` 读路由文案的方法，但 src/ 内没找到消费者。推断是 doctor/upgrade 或外部工具用，未证实。
3. 🟡 **`getDBSizeBytes()` 无调用方**（store.ts:1599-1605）——预留的 DB 体积上限 enforcement？未实现？
4. 🟡 **FTS5 `source_category` 列恒写 NULL**（store.ts:1065-1066）——预留的分类字段，用途未知。
5. 🟡 **遥测服务端**：`platform.json`/`platform-bridge.mjs` 指向的 `${platform_url}/events` 与 context-mode.com/insight 的后端（platform.context-mode.com）不在本仓库——闭源部分，无法审计。
6. 🟡 **tier2 e2e smoke**（真实 LLM 冒烟）目前只实现了 Pi，claude-code/opencode 是占位（.github/workflows/tier2-e2e-smoke.yml）——其他平台的真实端到端验证缺失。
7. 🟡 **项目开发方式**：2232 次提交、大量 "Issue #NNN" 注释、issue 编号已到 #860+，且近期提交大量由 CI 自动重建 bundle。是否为 AI 重度参与开发（CONTRIBUTING/CLAUDE.md 的工作流）值得再从 CONTRIBUTING.md 深挖（本次未细读 19KB 的 CONTRIBUTING.md）。
8. 🟡 **stats.json 用户数**（682.1k+）由 update-stats.yml 每 6 小时自动更新，数据来源是外部统计，无法在本仓库核实。

## 已澄清

- 🔵 `build/` 目录在 repo 中不存在的原因：tsc 产物不提交；npm 发布走 `prepublishOnly` 重建；CI `bundle.yml` 只强制提交 6 个 `*.bundle.mjs`。
- 🔵 git log 最前面一堆 "ci: update install stats" 是自动提交，不是真实开发活动；真实开发从 2026-02-23 的 v0.4.0 提交开始。
- 🔵 平台数量差异：README 说 17 客户端 + OpenClaw 网关；`PlatformId` 有 19 个取值（含 `unknown`，且 kilo/omp 等被计入）。**事实**：docs/platform-support.md 列 18 列（17 客户端 + OMP 列在最后……实际表格 18 列）。口径不完全一致，以代码 PlatformId 为准。

## 中途插入的问题（本对话产生）

（暂无）
