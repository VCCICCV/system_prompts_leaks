# 平台可用性

哪些功能在哪个云服务商平台上可用。**本表格是本技能的唯一权威来源**——其他地方的各功能章节均指向此处，不再重复说明其可用性。当为第三方平台（Bedrock、Vertex、Foundry）或 AWS 上的 Claude Platform 编写代码时，请首先查阅本表格；如果某项功能在该平台上不受支持，则应使用第一方 Claude API 或采用其他实现方式。

列说明：**1P** = 第一方 Claude API，**P-AWS** = AWS 上的 Claude Platform（由 Anthropic 运营，功能当日同步），**Bedrock** = Amazon Bedrock，**Vertex** = Google Cloud Vertex AI，**Foundry** = Microsoft Foundry。其中，“Yes”表示正式商用，“beta”表示测试版，“No”表示不支持，“unconfirmed”表示撰写本文时尚未确认是否支持。| 功能 | 1P | P‑AWS | Bedrock | Vertex | Foundry | 备注 |
|---|---|---|---|---|---|---|
| 消息、流式传输、工具使用 | 是 | 是 | 是 | 是 | 是 | 核心 API |
| PDF 输入 | 是 | 是 | 是 | 是 | 是 | |
| 结构化输出/严格工具使用 | 是 | 是 | 是 | 是 | 是 | |
| 自适应思考/精力调节 | 是 | 是 | 是 | 是 | 是 | |
| 扩展思考 | 是 | 是 | 是 | 是 | 是 | |
| 提示缓存（5 分钟、1 小时） | 是 | 是 | 是 | 是 | 是 | |
| 自动提示缓存 | 是 | 是 | 是 | 是 | 是 | 旧版 Bedrock 集成（Opus 4.6 及更早版本）会以 400 错误拒绝顶层 `cache_control`，仅支持显式断点 |
| 令牌计数 | 是 | 是 | 是 | 是 | 是 | |
| 引用 | 是 | 是 | 是 | 是 | 是 | |
| 搜索结果内容块 | 是 | 是 | 是 | 是 | 是 | |
| 细粒度工具流式传输 | 是 | 是 | 是 | 是 | 是 | Bedrock：仅在较新的服务堆栈上启用 `eager_input_streaming`（Opus 4.7/4.8/5、Fable 5、Sonnet 4.6/5）；较旧的部署（Opus 4.5/4.6、Sonnet 4.0/4.5、Haiku 4.5）会在该字段上返回 400 错误 |
| 压缩 | 测试版 | 测试版 | 测试版 | 测试版 | 测试版 | |
| 上下文编辑 | 测试版 | 测试版 | 测试版 | 测试版 | 测试版 | |
| 上下文窗口（100 万） | 是 | 是 | 是 | 是 | 是 | |
| `inference_geo`（数据驻留） | 是 | 是 | 否 | 否 | 否 | |
| **服务器端工具** | | | | | | |
| &nbsp;&nbsp;网络搜索 | 是 | 是 | 否 | 是 | 是 | Vertex：仅支持基础版 `web_search_20250305`（不支持 `_20260209` 的动态过滤）。Foundry Azure 版：仅支持基础版 `web_search_20250305` |
| &nbsp;&nbsp;网页抓取 | 是 | 是 | 否 | 否 | 是 | Foundry Azure 版：仅支持基础版 `web_fetch_20250910` |
| &nbsp;&nbsp;代码执行 | 是 | 是 | 否 | 否 | 是 | Foundry：仅 Anthropic 部署支持；Azure 部署返回 400 错误 |
| &nbsp;&nbsp;工具搜索 | 是 | 是 | 是 | 是 | 是 | Bedrock：仅 InvokeModel API 支持，Converse 不支持 |
| &nbsp;&nbsp;顾问工具 | 测试版 | 测试版 | 否 | 否 | 否 | |
| **客户端实现的工具** | | | | | | |
| &nbsp;&nbsp;Bash、文本编辑器、记忆 | 是 | 是 | 是 | 是 | 是 | |
| &nbsp;&nbsp;计算机使用 | 测试版 | 测试版 | 测试版 | 测试版 | 测试版 | `computer_20251124` 及更早版本：五个平台均处于测试阶段。Claude Opus 5.5 在 Claude API 和 Google Cloud 上仅接受 `computer_toolset_20260801`（正式版，不含测试版标头），但在 Amazon Bedrock 上仍接受 `computer_20251124`（参见 `shared/model-migration.md` -> 迁移到 Claude Opus 5.5，破坏性变更第 4 条）。Claude Sonnet 5.5 在 Claude API 和 Google Cloud 上仅接受该工具集，但在 Amazon Bedrock 上仍接受 `computer_20251124`，并在所有平台上拒绝 `computer_20250124`（参见 `shared/model-migration.md` -> 迁移到 Claude Sonnet 5.5，破坏性变更第 4 条） |
| **代理/编排** | | | | | | |
| &nbsp;&nbsp;代理技能（Messages API） | 是 | 是 | 否 | 否 | 测试版 | Foundry：仅 Anthropic 部署支持；Azure 部署返回 400 错误 |
| &nbsp;&nbsp;程序化工具调用 | 是 | 是 | 否 | 否 | 是 | Foundry：仅 Anthropic 部署支持；Azure 部署返回 400 错误 |
| &nbsp;&nbsp;MCP 连接器 | 测试版 | 测试版 | 否 | 否 | 测试版 | |
| &nbsp;&nbsp;托管代理 | 测试版 | 测试版 | 否 | 否 | 否 | Foundry：否（推断所得；Foundry 文档中亦未提及） |
| &nbsp;&nbsp;自托管沙盒 | 测试版 | 测试版 | 否 | 否 | 否 | P‑AWS：工作节点通过 IAM/SigV4 或 AWS 控制台 API 密钥 + `AnthropicSelfHostedEnvironmentAccess` 进行身份验证（控制台环境密钥在此处无效）；自托管环境中的会话无法附加记忆存储；`GET /v1/environments/{id}/work` 列表端点不受支持，其他工作端点正常 |
| **API 端点** | | | | | | |
| &nbsp;&nbsp;消息批处理 | 是 | 是 | 否 | 否 | 否 | |
| &nbsp;&nbsp;文件 API | 是 | 是 | 否 | 否 | 测试版 | Foundry：仅 Anthropic 部署支持；Azure 部署返回 400 错误 |
| &nbsp;&nbsp;模型 API | 是 | 是 | 否 | 否 | 否 | |
| **其他** | | | | | | |
| &nbsp;&nbsp;对话中途系统消息 | 是 | 是 | 是 | 是 | 否 | Claude Opus 5、Claude Opus 5.5、Claude Opus 4.8、Claude Fable 5、Claude Fable 5.1、Claude Mythos 5、Claude Mythos 5.1、Claude Sonnet 5.5；Claude Sonnet 5 不支持。Bedrock：InvokeModel 直通模式，不支持 ARN 版本的模型 |
| &nbsp;&nbsp;对话中途工具变更 | 测试版 | 测试版 | 测试版 | 测试版 | 否 | 适用的模型同对话中途系统消息；测试版 `mid-conversation-tool-changes-2026-07-01` |
| &nbsp;&nbsp;回合范围内的（`clear_at`）系统消息 | 测试版 | 测试版 | 测试版 | 测试版 | 否 | 适用的模型同对话中途系统消息；测试版 `mid-conversation-system-clear-at-2026-08-21`（在 Bedrock/Vertex 上需以测试版值传递） |
| &nbsp;&nbsp;每条消息的“精力”（系统消息 `output_config`） | 测试版 | 未确认 | 未确认 | 测试版 | 未确认 | Claude Fable 5.1、Claude Mythos 5.1、Claude Opus 5、Claude Opus 5.5、Claude Sonnet 5.5（仅适用于思考模式——若设置为 `between_tools` 则返回 400 错误）；测试版 `mid-conversation-output-config-2026-07-01`；在 Claude API 和 Google Cloud 上，任何发送该标头的组织均可使用（Claude Platform on AWS/Bedrock/Foundry 未确认；Bedrock 上 Claude Opus 5 被排除） |
| &nbsp;&nbsp;`thinking.display: "updates"` | 测试版 | 测试版 | 测试版 | 测试版 | 测试版 | Claude Fable 5.1、Claude Mythos 5.1、Claude Fable 5、Claude Opus 5.5、Claude Sonnet 5.5（具备自适应思考功能）；测试版 `thinking-display-updates-2026-08-18`（各平台按测试版值传递）；若未设置，则 `"updates"` 会被视为未知的 `display` 值而被拒绝 |
| &nbsp;&nbsp;思考块绑定控制 | 测试版 | 测试版 | 测试版 | 测试版 | 未确认 | `thinking.block_binding` + `input_transformations`；测试版 `thinking-binding-controls-2026-08-01`（Claude API、Claude Platform on AWS、Bedrock 和 Vertex 使用同一测试版名称——Bedrock：`anthropic_beta` 请求体字段，Vertex：`anthropic-beta` HTTP 标头）；Foundry 未确认；凡标头被拒绝之处，应去除后重试；历史编辑的强制执行本身遵循 `shared/model-migration.md` 中关于账户年龄的规定——从 Claude Fable 5 迁移到 Claude Fable 5.1 |
| &nbsp;&nbsp;服务器端“回退” | 测试版 | 测试版 | 否 | 否 | 否 | `"default"` -> 测试版 `server-side-fallback-2026-07-01`；数组形式 -> 测试版 `server-side-fallback-2026-06-01` |
| &nbsp;&nbsp;快速模式 | 测试版 | 否 | 否 | 否 | 否 | 研究预览，测试版 `fast-mode-2026-02-01`，仅限第一方 API（Claude Opus 5 / Opus 4.8，价格分别为 $10 / $50；Claude Opus 5.5，价格分别为 $8 / $40） |
| &nbsp;&nbsp;缓存诊断 | 测试版 | 否 | 否 | 否 | 否 | 仅限第一方 API |
| &nbsp;&nbsp;任务预算 | 测试版 | 测试版 | 否 | 否 | 否 | 测试版标头 `task-budgets-2026-03-13`；第三方可用性未予说明——暂按不支持处理 |

