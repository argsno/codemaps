# 关键结论的证据出处（sources.md）

> 格式：结论 → 出处（文件:行 / 命令）。仅列高价值结论。

## 定位与统计

| 结论 | 出处 |
|---|---|
| v1.0.169 / Elastic-2.0 / Node>=22.5 / pnpm | package.json:3,7,123-125 |
| bin=cli.bundle.mjs；MCP 入口 start.mjs | package.json:58-60；.claude-plugin/plugin.json（mcpServers） |
| 6 类 hook 事件注册 | hooks/hooks.json（PostToolUse/PreToolUse/UserPromptSubmit/SessionStart/PreCompact/Stop） |
| 11 个 MCP 工具 | server.ts:1648,2043,2237,2554,3413,3679,3937,4140,4271,4446,4817 |
| src 总行数 41,617 | `find src -name '*.ts' | xargs wc -l` |
| git：2232 提交，2026-02-23 起 | `git rev-list --count HEAD`；`git log --reverse` |
| 18 平台支持矩阵（按 hook 语义分 3 类） | docs/platform-support.md:1-20 |

## 核心机制

| 结论 | 出处 |
|---|---|
| 输出分级阈值 5KB/100KB | server.ts:1979-1980 |
| intentSearch 只回标题+预览+可搜索词 | server.ts:1982-2036 |
| JS/TS 插桩 __CM_FS__/__CM_NET__ | server.ts:1761-1819,1829-1842 |
| FloodGuard 60s 窗口，3/8 两档 | server.ts:2450-2485；search/flood-guard.ts:78-110 |
| RRF K=60 双匹配器融合 | store.ts:1244-1284 |
| 邻近度重排/模糊纠正 | store.ts:1288-1336,1195-1240 |
| 分块：markdown 标题栈/纯文本 20 行/JSON 路径 | store.ts:1646-1763,1858-1923,1925-2022 |
| FTS5 双表 porter/trigram，无 trigger | store.ts:475-497,1049-1070 |
| 项目边界包含判定（#852） | security.ts:686-783；server.ts:1184-1209 |
| 链式命令/子 shell 拆分防绕过 | security.ts:174-314 |
| env denylist（约 90 项） | executor.ts:573-745 |
| 100MB 输出硬上限、进程组击杀 | executor.ts:510-547,196-208 |
| PreToolUse 重定向逻辑（curl/wget/WebFetch/大 Read） | hooks/core/routing.mjs:670-1017 |
| redirect marker 文件握手 | hooks/pretooluse.mjs:175-221；hooks/posttooluse.mjs:62-179 |
| routing block 文案与注入点 | hooks/routing-block.mjs:16-105 |
| PreCompact→快照→resume 原子领取 | hooks/precompact.mjs:33-99；session/db.ts:1059-1078,1601；session/snapshot.ts |
| SessionDB 四表 schema | session/db.ts:814-905 |
| 事件类型约 30 种（posttooluse 链） | session/extract.ts:2710-2750 |
| 状态栏 sidecar JSON 500ms 节流 | server.ts:974-1101；bin/statusline.mjs |
| WAL 多写者 ADR | docs/adr/0001-sessiondb-multi-writer.md |
| start.mjs：Bun re-exec(#564)/自愈/ensure-deps/完整度闸 exit 2 | start.mjs:62-135,137-225,291-426,463-465,540-598,600-616 |
| postinstall Linux 硬失败（Node<22.5） | scripts/postinstall.mjs:22-87 |
| heal-better-sqlite3 分层策略 | scripts/heal-better-sqlite3.mjs:280-410 |
| esbuild 6 bundle + external 依赖清单 | package.json:82-85 |
| assert-bundle 禁 `__require` 垫片；assert-asymmetric-drift 钉 start.mjs | scripts/assert-bundle.mjs:16-43；scripts/assert-asymmetric-drift.mjs |
| CI 三平台矩阵 + doctor 步骤 | .github/workflows/ci.yml |
| version-sync 10 文件锁步 | scripts/version-sync.mjs:12-40；package.json:87 |
| SSRF 防护：DNS 拦截 + IP 分类 + 手动重定向 | server.ts:2838-3013,3168-3286 |
| fetch 缓存 = source 行，TTL 24h | server.ts:3118-3119,3288-3311；fetch-cache.ts:13 |
| ADR-0002 工具描述风格 / 0003 deny 文案 / 0004 统计口径 | docs/adr/ |

## 分析工具使用记录

- 4 个并行 explore 子代理：session 子系统 / adapter+hook / store+search / CLI+bootstrap（各自产出带 file:line 的 1500-1800 词报告，已并入 notes/）。
- 主线程精读：server.ts 关键段（140-470, 880-1105, 1560-2470, 2553-2702, 3678-3807, 4847-4991）、executor.ts 全文、security.ts 全文、routing-block.mjs 全文、start.mjs 头部、hooks.json、plugin.json、docs/platform-support.md。
