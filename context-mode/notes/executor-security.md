# 模块笔记：沙箱执行器与安全（executor.ts / runtime.ts / security.ts）

## 1. PolyglotExecutor（src/executor.ts, 785 行）

**事实**：
- `execute()`：mkdtemp → 写脚本 → `buildCommand` → spawn → 清理 tmpdir（Windows 上 rmSync 8 次重试防句柄占用，#788）。
- **tmp 目录解析有一个专门的怪招**：`OS_TMPDIR` 绕过被某些 shell 改写的 `TMPDIR`（macOS 用 `getconf DARWIN_USER_TEMP_DIR`、Linux 用 `mktemp -u -d` 并清空 TMPDIR env），防止临时文件污染工作区（executor.ts:99-111）。
- 语言包装：Go 无 `package` 前缀自动补 `package main`；PHP 无 `<?php` 自动补；Elixir 在 Mix 项目下自动 `Code.prepend_path(_build/dev/lib/*/ebin)`（executor.ts:332-347）。
- shell 脚本：POSIX 上写 `export PATH=<继承的 PATH>` 后再执行（防 rc 文件改坏 PATH）；Windows PowerShell 脚本加 UTF-8 BOM + `[Console]::InputEncoding`（防 5.1 用 ANSI 代码页吃中文）（executor.ts:63-92）。Windows Git Bash 上把裸 `mvn` 重写成 `mvn.cmd` 绕开 mingw 路径转换 bug（#782，executor.ts:136-171）。
- Rust 特殊：`__rust_compile_run__`，先 rustc 编译（最多 60s）再跑二进制（executor.ts:291-293, 377-407）。
- cwd 统一为**项目根**（issue #788 起所有语言，不只 shell），脚本文件用绝对路径传给 runtime。
- 进程控制：Unix `detached:true` 建进程组，超时 `process.kill(-pid, SIGKILL)` 杀整组；Windows `taskkill /F /T`；`background:true` 时超时→unref + 管道改 no-op 排空（不发 SIGPIPE）+ 记录 PID 供退出时清理（executor.ts:196-208, 475-508）。
- **字节硬上限 100MB**：stdout+stderr 合计超限立即杀进程并加注 `[output capped at 100MB — process killed]`（executor.ts:510-547）。
- 超时默认**不设**（issue #406）：由宿主的 RPC 超时治理，只有显式传值才计时（executor.ts:470-475）。

## 2. 环境净化（#buildSafeEnv，executor.ts:573-745）

**事实**——约 90 个变量的 denylist，注释里逐项给了 CVE/MITRE 依据：
- shell 注入类：`BASH_ENV`/`ENV`/`PROMPT_COMMAND`/`PS4`/`SHELLOPTS`/`BASHOPTS`/`CDPATH`/`INPUTRC`/`BASH_XTRACEFD`
- Node：`NODE_OPTIONS`/`NODE_PATH`；Python：`PYTHONSTARTUP/PYTHONHOME/PYTHONWARNINGS/PYTHONBREAKPOINT/PYTHONINSPECT`
- Ruby/Perl：`RUBYOPT/RUBYLIB`、`PERL5OPT/PERL5LIB/PERLLIB/PERL5DB`
- Go/Rust/C#/Erlang：`GOFLAGS/CGO_*`、`RUSTC*`、`DOTNET_*`（含 profiler 注入 `CORECLR_PROFILER*`）、`ERL_*`
- 动态链接器：`LD_PRELOAD`/`DYLD_INSERT_LIBRARIES`；OpenSSL：`OPENSSL_CONF/ENGINES`
- Git 命令注入：`GIT_TEMPLATE_DIR/GIT_CONFIG_GLOBAL/GIT_CONFIG_SYSTEM/GIT_EXEC_PATH/GIT_SSH*/GIT_ASKPASS`
- 前缀清扫：`BASH_FUNC_*`、`COMPlus_*`（.NET 旧别名）
- 强制覆盖：`TMPDIR=<沙箱目录>`、`HOME=<真实 HOME>`、`LANG=en_US.UTF-8`、`NO_COLOR=1`、Python 三个 UTF-8 变量；SSL_CERT_FILE 自动探测。

## 3. runtime.ts（765 行）：runtime 探测

**事实**：
- 探测各语言可执行文件（`commandExists`/`runnableExists`）+ 版本；shell 白名单 `bash|sh|zsh|dash|pwsh|powershell|cmd`（runtime.ts:16-24）。
- JS/TS 优先 **Bun**（≥1.0，3-5x 提速），否则 node；TS 依次 fallback `bun → tsx → ts-node`（runtime.ts:293-367, 644-671）。
- Windows 上专门找 Git Bash（`KNOWN_GIT_BASH_PATHS`），并识别 WSL/system cmd 差异（176-237）。
- `resolveHookRuntime()`：hook 用什么运行时执行（Bun 有坑——0.x 有 ESM 回归——所以有版本门槛与缓存）（403-549）。
- `getAvailableLanguages()` 决定 `ctx_execute` 描述里动态渲染的语言列表与 `bunNote`（server.ts:1244-1247）。

## 4. security.ts（889 行）：策略求值引擎

**事实**：
- 读取用户已有的 **Claude 风格权限配置**（`.claude/settings.local.json` → `.claude/settings.json` → 各 adapter 全局 settings，security.ts:363-394, 426-490）。
- 模式解析：`Bash(cmd:argsGlob)` / `Tool(glob)`；`globToRegex` 支持 `command:*` 与 `sudo *` 两种格式（26-90）；文件路径用另一套 `fileGlobToRegex`（`**` 跨目录、`*` 不跨 `/`）（101-135）。
- **绕过防御**：`splitChainedCommands` 按 `&& || ; | & 换行` 拆链（正确处理引号/转义/`$()` 嵌套），`extractSubshellCommands` 递归提取 `$()` 与反引号内容，`collectCommandElements` 合并——任何一段命中 deny 即整体 deny（158-314, 512-563）。
- 决策：deny > ask > allow；跨策略文件按"最本地优先"；**默认 ask**（512-563）。
- 服务器侧只有 deny 模式（没有 UI 问用户）→ `evaluateCommandDenyOnly`（573-588）。
- **文件路径评估**：同时匹配原样路径、词法绝对路径、realpath 规范路径（防 `..` 穿越与符号链接逃逸）（616-656）。
- **项目边界包含判定**（#852）：`isPathInsideProject` 纯算法（`path.relative`），词法通过后还要 realpath 也通过；root 无法解析则 fail-open（686-741）。`evaluateProjectContainment` 给用户留了 **复用宿主 `permissions.allow: ["Read(/var/log/**)"]`** 的正规逃生门（766-783）。
- **非 shell 代码的内嵌命令扫描**：`SHELL_ESCAPE_PATTERNS` 逐语言正则（python 的 `os.system`/`subprocess.*` 含 list 形式、JS 的 `exec*`/`spawn*`、ruby 反引号、php 的 `shell_exec/exec/system/...`、go `exec.Command`、rust `Command::new`）（795-889）。

## 5. 安全层次总结

**事实+推断**：
1. **宿主权限 UI**（Claude Code 弹窗）——context-mode 用 `title`/annotations 帮助用户看清（#846/#852）。
2. **Hook 层（主执法层）**：PreToolUse 对 Bash 查 deny、对 curl/wget/WebFetch/大 Read 做重定向或拒绝（见 adapters-hooks 笔记）。
3. **MCP Server 层（兜底防线）**：每个工具 handler 入参再查一次同样策略 + 项目边界；抛异常一律 fail-open（"hooks 才是主执法层"，server.ts:1127-1131）。
4. **沙箱层**：env 净化 + 进程组隔离 + 100MB/超时护栏。
5. **网络层**：fetch 子进程做 SSRF（DNS 重绑定防不住的部分靠逐跳校验 + IP 分类）。
- **推断**：这是一套"防呆 + 防绕过"而**不是**强隔离（沙箱进程实际拥有用户全部文件系统权限，没有容器/沙箱 syscall 限制）——README 与代码注释都承认执行代码即用户权限（docs/adr/0003 等）。
