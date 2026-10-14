# codemaps

代码库深度分析笔记：用 AI agent 对开源项目做「从零到能改代码」的架构全解，全程只读分析，产出中文笔记与自包含 HTML 文档。

## 收录

### [context-mode](./context-mode/) — mksglu/context-mode `v1.0.169` @ `dbec980`

MCP 服务器 + 多平台客户端插件，目标是节省 AI 编码代理的 context window：把原始大数据留在沙箱，只回传答案；对话压缩（compact）后按需恢复会话状态。

### [rea](./rea/) — morluto/rea @ `b11fe63bf`

「Reverse Engineer Anything」：本地运行的逆向工程工具，以 MCP server + CLI 双形态暴露同一套分析工作流，产出带证据（Evidence）与不确定性（Unknown）的结论。

## 每个项目的结构

| 文件 | 内容 |
|---|---|
| `overview.md` | 定位、受众、技术栈、如何运行 |
| `architecture.md` | 模块划分、关键抽象、数据流 |
| `notes/<模块>.md` | 分模块细节笔记 |
| `questions.md` | 未解问题清单 |
| `sources.md` | 关键结论的证据出处 |
| `architecture.html` | 单文件「核心架构全解」，零外部请求、离线可双击打开 |

## 约定

- 结论按 **事实**（附文件:行 / 命令出处）/ **推断** / **未知** 三分法标注
- 对被分析仓库全程只读，不改动任何文件
- 分析流程规范见 [AGENTS.md](./AGENTS.md)

> 笔记由 AI agent 按上述流程生成：观察到的记为事实并附出处，推导的标推断，没弄清的记未知。
