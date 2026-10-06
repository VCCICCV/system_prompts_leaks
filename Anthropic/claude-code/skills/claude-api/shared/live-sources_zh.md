# 实时文档源

此文件包含用于从 platform.claude.com 和 Agent SDK 仓库获取最新信息的 WebFetch URL。当用户需要自缓存内容上次更新以来可能已发生变化的最新数据时，请使用这些 URL。

## 何时使用 WebFetch

- 用户明确要求“最新”或“当前”信息
- 缓存数据似乎不正确
- 用户询问缓存内容未涵盖的功能
- 用户需要特定的 API 详情或示例

## Claude API 文档 URL

### 模型与定价

| 主题           | URL                                                                          | 提取提示                                                               |
| --------------- | ---------------------------------------------------------------------------- | ------------------------------------------------------------------------------- |
| 模型概览       | `https://platform.claude.com/docs/en/about-claude/models/overview.md`        | “提取所有 Claude 模型的当前模型 ID、上下文窗口和定价” |
| 迁移指南       | `https://platform.claude.com/docs/en/about-claude/models/migration-guide.md` | “提取升级到较新 Claude 模型时的重大变更、已弃用的参数以及各模型的迁移步骤” |
| 推出 Claude Fable 5 | `https://platform.claude.com/docs/en/about-claude/models/introducing-claude-fable-5.md` | “提取 Claude Fable 5 和 Claude Mythos 5 的能力、API 变更及上线阶段” |
| 定价           | `https://platform.claude.com/docs/en/about-claude/pricing.md`                | “提取输入和输出每百万 token 的当前定价”               |
| 成本优化       | `https://platform.claude.com/docs/en/about-claude/models/optimizing-for-cost-and-intelligence.md` | “提取已测量的成本优化项、缓存与批量处理的节省效果、不同努力等级与模型的每任务成本对比、预算控制以及多模型使用建议” |

### 核心功能

| 主题             | URL                                                                          | 提取提示                                                                      |
| ----------------- | ---------------------------------------------------------------------------- | -------------------------------------------------------------------------------------- |
| 扩展思维         | `https://platform.claude.com/docs/en/build-with-claude/extended-thinking.md` | “提取扩展思维的参数、budget_tokens 要求及使用示例” |
| 自适应思维         | `https://platform.claude.com/docs/en/build-with-claude/adaptive-thinking.md` | “提取自适应思维的设置、努力等级以及 Claude Opus 5.5 的使用示例”         |
| 努力参数         | `https://platform.claude.com/docs/en/build-with-claude/effort.md`            | “提取努力等级、成本与质量的权衡，以及其与思维模式的交互方式”        |
| 工具使用         | `https://platform.claude.com/docs/en/agents-and-tools/tool-use/overview.md`  | “提取工具定义的 Schema、tool_choice 选项及工具结果的处理方式”       |
| 流式传输         | `https://platform.claude.com/docs/en/build-with-claude/streaming.md`         | “提取流式事件类型、SDK 示例及最佳实践”                      |
| 提示词缓存       | `https://platform.claude.com/docs/en/build-with-claude/prompt-caching.md`    | “提取 cache_control 的用法、定价优势及实现示例”           |

### 媒体与文件

| 主题       | URL                                                                    | 提取提示                                                 |
| ----------- | ---------------------------------------------------------------------- | ----------------------------------------------------------------- |
| 视觉功能      | `https://platform.claude.com/docs/en/build-with-claude/vision.md`      | “提取支持的图片格式、尺寸限制及代码示例” |
| PDF 支持   | `https://platform.claude.com/docs/en/build-with-claude/pdf-support.md` | “提取PDF处理能力、限制及示例”         |

### API 操作

| 主题            | URL                                                                         | 提取提示                                                                                       |
| ---------------- | --------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------- |
| 批量处理        | `https://platform.claude.com/docs/en/build-with-claude/batch-processing.md` | “提取批量API端点、请求格式及结果轮询方法”                                  |
| 文件API         | `https://platform.claude.com/docs/en/build-with-claude/files.md`            | “提取文件上传、下载、在消息中引用、支持的文件类型，以及从files-api-2025-04-14迁移的步骤” |
| 令牌计数        | `https://platform.claude.com/docs/en/build-with-claude/token-counting.md`   | “提取令牌计数API的使用方法及示例”                                                         |
| 速率限制        | `https://platform.claude.com/docs/en/api/rate-limits.md`                    | “提取按层级和模型划分的当前速率限制”                                                     |
| 使用与费用管理API | `https://platform.claude.com/docs/en/manage-claude/usage-cost-api.md`       | “提取usage_report和cost_report端点、管理API密钥要求、filter和group_by维度、令牌相关字段及粒度限制” |
| 错误            | `https://platform.claude.com/docs/en/api/errors.md`                         | “提取HTTP错误码、含义及重试指南”                                                        |
| Amazon Bedrock    | `https://platform.claude.com/docs/en/build-with-claude/claude-on-amazon-bedrock.md` | “提取各语言的AnthropicBedrockMantle客户端、以`anthropic.`为前缀的模型ID、认证路径、功能可用性及适用区域” |
| AWS上的Claude平台 | `https://platform.claude.com/docs/en/build-with-claude/claude-platform-on-aws.md` | “提取各语言的AnthropicAWS客户端、SigV4认证、凭证优先级、短期API密钥、workspace_id及区域要求” |
| AWS上Claude平台的IAM操作 | `https://platform.claude.com/docs/en/api/claude-platform-on-aws-iam-actions.md` | “提取各项API功能所需的IAM操作名称、资源ARN及策略示例” |

### 管理API（组织管理）

| 主题                | URL                                                                     | 提取提示                                                                     |
| -------------------- | ----------------------------------------------------------------------- | ---------------------------------------------------------------------------- |
| 管理 API 指南      | `https://platform.claude.com/docs/en/manage-claude/admin-api.md`        | “提取管理 API 的认证、SDK/CLI 使用方法，以及成员/邀请/密钥管理”   |
| 管理 API 参考      | `https://platform.claude.com/docs/en/api/admin.md`                      | “提取管理 API 的端点参数、响应及分页信息”            |
| 工作空间           | `https://platform.claude.com/docs/en/manage-claude/workspaces.md`       | “通过 API 提取工作空间的创建、列出、归档及成员管理功能”                  |
| 速率限制 API      | `https://platform.claude.com/docs/en/manage-claude/rate-limits-api.md`  | “提取组织和工作空间的速率限制报告端点及其过滤条件”                    |
| WIF 管理           | `https://platform.claude.com/docs/en/manage-claude/wif-admin-api.md`    | “提取服务账户、联盟颁发者及联盟规则的管理功能”           |
| 使用与费用报告     | `https://platform.claude.com/docs/en/manage-claude/usage-cost-api.md`   | “提取使用情况和费用报告端点（仅支持 curl，不在 SDK 中）”                 |

### 工具

| 主题          | URL                                                                                    | 提取提示                                                                        |
| -------------- | -------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------- |
| 代码执行       | `https://platform.claude.com/docs/en/agents-and-tools/tool-use/code-execution-tool.md` | “提取代码执行工具的设置、文件上传、容器复用及响应处理方式” |
| 计算机使用     | `https://platform.claude.com/docs/en/agents-and-tools/tool-use/computer-use-tool.md`   | “提取 computer_toolset_20260801 的设置（配置、成员工具、批量操作、结果中的 toolset_name）、兼容性矩阵，以及从 computer_20251124 的迁移步骤”             |
| Bash 工具      | `https://platform.claude.com/docs/en/agents-and-tools/tool-use/bash-tool.md`           | “提取 Bash 工具的 Schema、参考实现及安全注意事项”        |
| 文本编辑器     | `https://platform.claude.com/docs/en/agents-and-tools/tool-use/text-editor-tool.md`    | “提取文本编辑器工具的命令、Schema 及参考实现”                |
| 内存工具       | `https://platform.claude.com/docs/en/agents-and-tools/tool-use/memory-tool.md`         | “提取内存工具的命令、目录结构及实现模式”         |
| 工具搜索       | `https://platform.claude.com/docs/en/agents-and-tools/tool-use/tool-search-tool.md`    | “提取工具搜索的设置、适用场景及缓存交互方式”                          |
| 程序化工具调用 | `https://platform.claude.com/docs/en/agents-and-tools/tool-use/programmatic-tool-calling.md` | “提取 PTC 的设置、脚本执行模型及从代码中调用工具的方式”    |
| 技能           | `https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview.md`        | “提取技能文件夹结构、SKILL.md 格式及加载行为”                  |
| 技能指南       | `https://platform.claude.com/docs/en/build-with-claude/skills-guide.md`                | “提取 Skills API（`/v1/skills`）的使用方法，以及从 skills-2025-10-02 的迁移步骤” |

### 高级功能

| 主题              | URL                                                                           | 提取提示                                   |
| ------------------ | ----------------------------------------------------------------------------- | --------------------------------------------------- |
| 结构化输出         | `https://platform.claude.com/docs/en/build-with-claude/structured-outputs.md` | “提取 output_config.format 的用法及模式约束”                           |
| 压缩               | `https://platform.claude.com/docs/en/build-with-claude/compaction.md`         | “提取压缩的设置、触发配置以及使用压缩时的流式处理”             |
| 上下文编辑         | `https://platform.claude.com/docs/en/build-with-claude/context-editing.md`    | “提取上下文编辑的阈值、清除的内容及配置”            |
| 引用               | `https://platform.claude.com/docs/en/build-with-claude/citations.md`          | “提取引用的格式及实现方式”        |
| 上下文窗口         | `https://platform.claude.com/docs/en/build-with-claude/context-windows.md`    | “提取上下文窗口的大小及令牌管理” |

### 托管代理

当托管代理的绑定、行为或通信层细节未在缓存的 `shared/managed-agents-*.md` 概念文件或 `{lang}/managed-agents/README.md` 中涵盖时，请使用这些内容。| 主题                 | URL                                                                              | 提取提示                                                                               |
| --------------------- | -------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------- |
| 概述              | `https://platform.claude.com/docs/en/managed-agents/overview.md`                 | “提取高层次架构，以及代理/会话/环境/保险库之间的关联方式” |
| 快速入门            | `https://platform.claude.com/docs/en/managed-agents/quickstart.md`               | “提取从代理到环境、再到会话、最后到流的最小化端到端代码路径”              |
| 代理设置           | `https://platform.claude.com/docs/en/managed-agents/agent-setup.md`              | “提取代理的创建、更新、版本列表、归档等生命周期及参数”                   |
| 定义结果       | `https://platform.claude.com/docs/en/managed-agents/define-outcomes.md`          | “提取结果定义、评估钩子以及成功标准的配置”             |
| 会话              | `https://platform.claude.com/docs/en/managed-agents/sessions.md`                 | “提取会话的生命周期、状态转换、空闲/终止语义及恢复规则”    |
| 环境          | `https://platform.claude.com/docs/en/managed-agents/environments.md`             | “提取环境配置（云/网络）、管理端点及复用模型”          |
| 自托管沙盒 | `https://platform.claude.com/docs/en/managed-agents/self-hosted-sandboxes.md`    | “提取 config:{type:self_hosted}、ANTHROPIC_ENVIRONMENT_KEY、EnvironmentWorker.run/handle_item、environments.work.poller(drain)、beta_agent_toolset、ant beta:worker poll/run、基于 Webhook 的唤醒机制、内存存储（ANTHROPIC_WORK_SECRET、memory_sync_interval/memory_sync_deletes）” |
| 自托管沙盒——安全 | `https://platform.claude.com/docs/en/managed-agents/self-hosted-sandboxes-security.md` | “提取客户负责的部分（加固、出站流量控制、密钥保管、信任边界）与 Anthropic 无法承担的部分” |
| 事件与流式传输  | `https://platform.claude.com/docs/en/managed-agents/events-and-streaming.md`     | “提取事件流类型、以流为先的顺序、重连/去重机制以及引导模式”    |
| 工具                 | `https://platform.claude.com/docs/en/managed-agents/tools.md`                    | “提取内置工具集、自定义工具定义及工具结果的数据格式”                |
| 文件                 | `https://platform.claude.com/docs/en/managed-agents/files.md`                    | “提取文件上传、挂载路径、会话资源以及会话输出的列出与下载功能”  |
| 权限策略   | `https://platform.claude.com/docs/en/managed-agents/permission-policies.md`      | “提取权限策略类型（always_allow / always_ask / auto），三种 auto 结果，evaluated_permission 和 evaluation 事件字段，以及每项工具的配置” |
| 多代理           | `https://platform.claude.com/docs/en/managed-agents/multiagent-orchestration.md` | “提取多代理的组合模式、子代理的调用及结果交接方式”            |
| 可观测性         | `https://platform.claude.com/docs/en/managed-agents/observability.md`            | “提取托管代理暴露的日志记录、链路追踪及使用情况的遥测数据”                       |
| Webhook              | `https://platform.claude.com/docs/en/managed-agents/webhooks.md`                 | “提取 Webhook 端点注册、HMAC 签名验证、支持的事件类型及投递语义” |
| GitHub                | `https://platform.claude.com/docs/en/managed-agents/github.md`                   | “提取 github_repository 资源的结构、多仓库挂载及令牌轮换”             |
| MCP 连接器         | `https://platform.claude.com/docs/en/managed-agents/mcp-connector.md`            | “提取在代理上声明 MCP 服务器，以及在会话阶段通过保险库注入凭据的方式”     |
| 保险库                | `https://platform.claude.com/docs/en/managed-agents/vaults.md`                   | “提取保险库的创建、凭据添加/轮换、OAuth 刷新的结构及归档操作”                 |
| 技能                | `https://platform.claude.com/docs/en/managed-agents/skills.md`                   | “提取面向托管代理的技能打包与加载模型”                                  |
| 内存                | `https://platform.claude.com/docs/en/managed-agents/memory.md`                   | “提取内存资源的结构、作用域及生命周期”                                         |
| 上手指南            | `https://platform.claude.com/docs/en/managed-agents/onboarding.md`               | “提取首次运行的设置、先决条件以及账户和区域的要求”                      |
| 云容器      | `https://platform.claude.com/docs/en/managed-agents/cloud-containers.md`         | “提取云容器的运行时、镜像配置以及网络和存储的相关配置选项”                     |
| 迁移             | `https://platform.claude.com/docs/en/managed-agents/migration.md`                | “提取从早期 API/预览形态迁移到正式版托管代理的迁移路径”                 |

### Anthropic CLI

`ant` CLI 提供了对 Claude API 的终端访问。每个 API 资源都以子命令的形式暴露出来。这是将代理、环境、技能、记忆存储和部署作为版本控制文件进行管理的推荐方式（`ant apply` - 参见 `shared/anthropic-cli.md`），同时也为脚本编写和交互式检查提供了会话及其他所有 API 资源的访问入口。

| 主题         | URL                                                     | 提取提示                                                                                  |
| ------------- | ------------------------------------------------------- | -------------------------------------------------------------------------------------------------- |
| Anthropic CLI | `https://platform.claude.com/docs/en/cli-sdks-libraries/cli/quickstart.md` | “提取 CLI 的安装、认证、命令结构，以及如何发送第一条请求” |
| `ant apply` | `https://platform.claude.com/docs/en/cli-sdks-libraries/cli/apply.md` | “提取按资源类型划分的文件布局、文件类型的推断方式、文件间的路径引用、`claude-lock.json`、各标志位（`--dry-run`、`--yes`、`--force`、`--prune`、`--upgrade`、`--lock-file`），以及 CI 配置” |
| `ant beta:sessions connect` | `https://platform.claude.com/docs/en/cli-sdks-libraries/cli/sessions-connect.md` | “提取交互式会话查看器：快捷键、工具调用的允许/拒绝提示、`--web` 本地查看器及其 URL 和有效期规则” |
| 认证概览 | `https://platform.claude.com/docs/en/manage-claude/authentication.md` | “提取凭证选项（API 密钥、交互式 OAuth 登录、工作负载身份联合）及各自的适用场景” |
| WIF 参考 | `https://platform.claude.com/docs/en/manage-claude/wif-reference.md`  | “提取凭证优先级顺序、配置文件的 Schema，以及配置目录的布局” |

---

## Claude API SDK 仓库

当缓存的 `{lang}/` 技能文件或上述 managed-agents 文档中未涵盖某个绑定（类、方法、命名空间、字段）时，请通过 WebFetch 获取这些仓库。SDK 包含对 `/v1/agents`、`/v1/sessions`、`/v1/environments` 及相关资源的 Beta 版 managed-agents 支持——可在仓库中搜索 `BetaManagedAgents`、`beta.agents`、`beta.sessions`，或该语言对应的等效命名空间。| SDK        | URL                                                      | 提取提示                                                                                                       |
| ---------- | -------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------- |
| Python     | `https://github.com/anthropics/anthropic-sdk-python`     | “提取 beta managed-agents 的命名空间、类及方法签名（`client.beta.agents`、`client.beta.sessions`）” |
| TypeScript | `https://github.com/anthropics/anthropic-sdk-typescript` | “提取 beta managed-agents 的命名空间、类及方法签名（`client.beta.agents`、`client.beta.sessions`）” |
| Java       | `https://github.com/anthropics/anthropic-sdk-java`       | “提取 beta managed-agents 的类、构建器及方法签名（`client.beta().agents()`、`BetaManagedAgents*`）” |
| Go         | `https://github.com/anthropics/anthropic-sdk-go`         | “提取 beta managed-agents 的类型及方法签名（`client.Beta.Agents`、`BetaManagedAgents*` 事件类型）”      |
| Ruby       | `https://github.com/anthropics/anthropic-sdk-ruby`       | “提取 beta managed-agents 的方法及参数结构（`client.beta.agents`、`client.beta.sessions`）”               |
| C#         | `https://github.com/anthropics/anthropic-sdk-csharp`     | “提取 beta managed-agents 的类及方法签名（NuGet 包，`BetaManagedAgents*` 类型）”                 |
| PHP        | `https://github.com/anthropics/anthropic-sdk-php`        | “提取 beta managed-agents 的类及方法签名（`$client->beta->agents`、`BetaManagedAgents*` 参数）”      |

每个 SDK 仓库的 `examples/` 目录下还包含可运行的示例程序，其中包括拒绝回退/`fallbacks` 示例（客户端中间件注册、回退状态以及服务端的 `fallbacks` 参数）。请直接获取这些语言的完整示例代码，而非翻译其他语言的示例。

### SDK 主版本升级指南

关于在不同主版本之间升级 SDK 的权威变更列表。随附的 `{lang}/claude-api/sdk-upgrade.md` 是可执行版本；当两者不一致时，以仓库中的指南为准。

| SDK                | URL                                                                         | 提取提示                                                                                                   |
| ------------------ | --------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------- |
| Python（0.x -> 1.x） | `https://github.com/anthropics/anthropic-sdk-python/blob/main/MIGRATION.md` | “提取所有破坏性变更及其前后代码、新的最低 Python 版本，以及升级命令” |

---

## 回退策略

如果 WebFetch 失败（网络问题、URL 变更）：

1. 使用各语言文件中缓存的内容（请注意缓存日期）
2. 告知用户数据可能已过时
3. 建议用户直接访问 platform.claude.com 或 GitHub 仓库