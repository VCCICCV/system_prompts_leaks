# 托管代理 - 概览

托管代理为每个会话 provision 一个容器，作为代理的工作空间。代理循环在 Anthropic 的编排层上运行；容器则是代理的 *工具* 执行的地方——包括 Bash 命令、文件操作和代码执行。您先创建一个持久化的 **代理** 配置（包含模型、系统提示、工具、MCP 服务器和技能），然后启动引用该配置的 **会话**。会话会向您流式返回事件；您可以向会话发送用户消息和工具执行结果。

## 警告：必经流程：代理（仅一次）→ 会话（每次运行）

**为什么代理是独立的对象：版本控制。** 代理是一个持久化且带版本的配置——每次更新都会生成一个新的不可变版本，而会话在创建时会固定到某个版本。这使您可以在不中断现有运行中会话的情况下迭代代理（调整提示、添加工具），在变更造成回退时进行回滚，并对不同版本进行 A/B 测试。如果每次运行都重新调用 `agents.create()`，这些功能都将无法实现。

每个会话都引用一个预先创建的 `/v1/agents` 对象。只需创建一次代理，保存其 ID，并在后续运行中重复使用。

| 步骤 | 请求 | 调用频率 |
|---|---|---|
| 1 | `POST /v1/agents` —— `model`、`system`、`tools`、`mcp_servers` 和 `skills` 在此处定义 | **仅一次。** 保存 `agent.id` **和** `agent.version`。 |
| 2 | `POST /v1/sessions` —— `agent: "agent_abc123"` 或 `{type: "agent", id, version}` | **每次运行。** 使用字符串简写时，默认使用最新版本。 |

如果您正准备在会话请求体中直接传入 `model`、`system` 或 `tools` 并调用 `sessions.create()`——请 **停止**。这些字段应在 `agents.create()` 中定义。会话只需提供一个 *指针* 即可。

**在生成代码时，应将初始化与运行时逻辑分离。** `agents.create()` 应放在初始化脚本中（或置于受保护的 `if agent_id is None:` 块中），而不是放在主路径的最前端。如果用户的代码在每次调用时都执行 `agents.create()`，就会不断积累孤立的代理，并白白承受创建延迟的开销。正确的做法是：将代理定义为受版本控制的文件，并通过 `ant apply` 同步到 `claude-lock.json` 中（参见 `shared/anthropic-cli.md`）——或者使用受保护的初始化脚本，将返回的 ID 持久化（写入配置文件、环境变量或密钥管理器），并在每次运行时加载该 ID 并调用 `sessions.create()`。

**若要更改代理的行为，请使用 `POST /v1/agents/{id}`，不要重新创建代理。** （对于使用 `ant apply` 管理的代理，直接编辑其配置文件并重新运行即可——如果在文件之外进行更新，下次 `ant apply` 将拒绝执行。）每次更新都会递增版本号；正在运行的会话会保持其固定的版本，而新会话则会获取最新版本（或通过 `{type: "agent", id, version}` 显式指定）。详情请参阅 `shared/managed-agents-core.md` 中的“代理”章节下的“版本控制”。若要在不影响代理对象的情况下，仅修改 **某个运行中会话** 的 `tools` 或 `mcp_servers`，请使用 `sessions.update()`（`vault_ids` 仅在会话创建时关联）——详情请参阅 `shared/managed-agents-core.md` 中的“在会话运行期间更新代理配置”部分。

## Beta 标头

托管代理目前处于 Beta 阶段。SDK 会自动设置必要的 Beta 标头：

| Beta 标头                    | 启用的功能                                      |
| ------------------------------ | ---------------------------------------------------- |
| `managed-agents-2026-04-01`    | 代理、环境、会话、事件、会话资源、会话线程、结果、多代理、保险库、凭据、部署 |
| `agent-memory-2026-07-22`      | 内存存储（在内存存储相关端点上替代 `managed-agents-2026-04-01`） |

**各 Beta 头应放置于何处：** SDK 会在 `client.beta.{agents,environments,sessions,vaults,deployments,deployment_runs}.*` 调用中自动设置 `managed-agents-2026-04-01`，并在 `client.beta.memory_stores.*` 调用中设置 `agent-memory-2026-07-22`。请勿在内存存储调用中添加 `managed-agents-2026-04-01`：如果在内存存储请求中同时发送这两个头，将返回 400 错误（将内存存储附加到会话属于会话调用，仍需使用 `managed-agents-2026-04-01`）。文件和技能 API 已退出 Beta 阶段，无需使用 Beta 头；即使请求中仍携带 `files-api-2025-04-14` 或 `skills-2025-10-02`，请求仍可正常执行，但返回的响应格式为旧版 Beta 格式。**例外——会话范围的文件列表：** 使用 `scope_id` 对 `files.list` 进行过滤时需要 `managed-agents-2026-04-01`，而 `client.beta.files` 不会自动添加该头，因此在调用 `client.beta.files.list({scope_id: session.id})` 时，请显式传递 `betas: ["managed-agents-2026-04-01"]`（在原生 HTTP 请求中，请发送 `anthropic-beta: managed-agents-2026-04-01`；在 `ant` CLI 中，请在 `ant beta:files list --scope-id` 命令后添加 `--beta managed-agents-2026-04-01`）。详情请参阅 `shared/managed-agents-environments.md` -> 会话输出。


## 阅读指南

| 用户想要...                       | 阅读这些文档                                        |
| -------------------------------------- | ------------------------------------------------------- |
| **从零开始 / “帮我设置一个代理”** | `shared/managed-agents-onboarding.md` - 引导式访谈（WHERE->WHO->WHAT->WATCH），然后输出代码 |
| 了解 API 的工作原理           | `shared/managed-agents-core.md`                         |
| 查看完整的端点参考            | `shared/managed-agents-api-reference.md`                |
| **创建一个代理**（必经的第一步） | `shared/managed-agents-core.md`（代理章节）+ 语言文件 |
| 更新/版本化一个代理                | `shared/managed-agents-core.md`（代理 -> 版本管理）- 更新，不要重新创建 |
| 创建一个会话                       | `shared/managed-agents-core.md` + `{lang}/managed-agents/README.md`（cURL/C#: `curl/managed-agents.md`） |
| 配置工具和权限        | `shared/managed-agents-tools.md`                        |
| 限制 `web_search` / `web_fetch` 可访问的站点；本地化搜索；对抓取内容进行上限控制 | `shared/managed-agents-tools.md`（§ 网络搜索与网页抓取设置）- 在工具集的 `configs` 条目中设置 `allowed_domains` / `blocked_domains` / `user_location` / `max_content_tokens`；**不是**环境的 `networking` |
| 设置 MCP 服务器                     | `shared/managed-agents-tools.md`（MCP 服务器章节）  |
| 流式接收事件 / 处理工具调用        | `shared/managed-agents-events.md` + 语言文件       |
| 通过 Webhook 获得会话状态变化的通知（无需轮询） | `shared/managed-agents-webhooks.md` - 控制台注册的端点，HMAC 校验，精简负载 + 拉取 |
| 定义结果 / 基于评分标准的迭代循环 | `shared/managed-agents-outcomes.md` - `user.define_outcome` 事件、评分器、`span.outcome_evaluation_*` 事件 |
| 协调多个代理 / 子代理 / 线程 | `shared/managed-agents-multiagent.md` - 代理配置中的 `multiagent: {type: "coordinator", agents: [...]}`、会话线程、跨线程的工具确认 |
| 设置环境                    | `shared/managed-agents-environments.md` + 语言文件 |
| 在自有基础设施 / VPC 中执行工具（自托管沙盒） | `shared/managed-agents-self-hosted-sandboxes.md` - `config:{type:"self_hosted"}`、`ANTHROPIC_ENVIRONMENT_KEY`、`EnvironmentWorker.run()` / `ant beta:worker poll` |
| 上传文件 / 附加代码库            | `shared/managed-agents-environments.md`（资源）     |
| 让代理在不同会话间保持持久记忆 | `shared/managed-agents-memory.md` - 内存存储、`memory_store` 会话资源、前置条件、版本控制/脱敏。在自托管沙盒中：参见 `shared/managed-agents-self-hosted-sandboxes.md` § 内存存储（SDK 工作进程同步本地副本） |
| 无需代码即可查看会话信息（对话记录、各工具统计、费用、线程） | `shared/managed-agents-events.md` - 控制台会话查看器说明；深度链接 `?event={event_id}` |
| 将代理/环境/技能以版本控制的形式保存（`ant apply`）；通过 Shell 调用 API | `shared/anthropic-cli.md` - `ant apply`、`claude-lock.json`、`--transform`、`@file` 内联 |
| 存储凭据（MCP 认证、CLI/SDK 的 API 密钥） | `shared/managed-agents-tools.md`（保险库章节）- `mcp_oauth` / `static_bearer` / `environment_variable` |
| 调用需要密钥的非 MCP API / CLI | `shared/managed-agents-tools.md`（保险库章节）- `environment_variable` 凭据，在出口时注入。若不适用（如自托管沙盒），参见 `shared/managed-agents-client-patterns.md` 模式 9，通过自定义工具将密钥保留在服务端 |
| 按定时任务周期性运行代理 | `shared/managed-agents-scheduled-deployments.md` - 部署、部署运行、暂停/自动暂停 |
| 为会话设置硬性美元预算上限 | `shared/managed-agents-core.md`（§ 会话预算）- 创建会话时设置 `budget`，达到预算时暂停，可修改或移除以恢复。对于部署：参见 `shared/managed-agents-scheduled-deployments.md` § 部署预算 |
| 固定模型推理的运行位置（数据驻留） | `shared/managed-agents-core.md`（§ 推理地域固定）- 代理上的 `model.inference_geo`、会话级覆盖、角色配置一致性 |
| 从代码库加载技能而非上传 | `shared/managed-agents-tools.md`（§ 从 GitHub 仓库加载技能）- 会话启动时自动发现根目录下的 `.claude/skills` |
| 为会话配备顾问，可在回合中随时咨询 | `shared/managed-agents-multiagent.md`（§ 顾问）- 角色配置中的 `{type: "advisor", model}`、咨询线程、明文传递 vs 脱敏传递 |

## 常见误区

- **先创建代理，再创建会话——无一例外**：会话的 `agent` 字段**仅**接受字符串形式的 ID 或 `{type: "agent", id, version}`。`model`、`system`、`tools`、`mcp_servers`、`skills` 是 `POST /v1/agents` 接口的**顶级字段**，绝不能在 `sessions.create()` 中使用。如果用户尚未创建代理，那么创建代理就是所有示例的第一步。
  
- **代理只需创建一次，而非每次运行时都创建**：`agents.create()` 是一项初始化步骤。请保存返回的 `agent_id` 并重复使用，不要在核心业务流程的入口处每次都调用 `agents.create()`。如果代理的配置需要变更，请使用 `POST /v1/agents/{id}`；每次更新都会生成一个新版本，会话可以固定到特定版本以确保结果的可复现性。

- **MCP 认证通过 Vault 管理**：代理的 `mcp_servers` 数组仅声明 `{type, name, url}`（不含认证信息）。凭据存储在 Vault 中（通过 `client.beta.vaults.credentials.create`），并通过 `vault_ids` 关联到会话。Anthropic 会利用存储的刷新令牌自动刷新 OAuth 令牌。Vault 还可用于存储非 MCP 服务的 `environment_variable` 凭据（如 CLI、SDK 或直接 API 调用），这些凭据会在出站时被注入，且在沙盒中始终不可见。

- **首次运行前务必校验资源完整性**：如果会话提出了明确的需求，但缺少工具、凭据、数据挂载或上下文，系统将在运行过程中发现缺失并最终失败。在创建会话之前，请确保任务中的每项操作都能映射到已配置的工具或 MCP 服务器，每个 MCP 服务器都有对应的 Vault 凭据，并且所有引用的文件或主机均已挂载且可访问。协助用户设置时，可参考 `shared/managed-agents-onboarding.md` 第 3 节“飞行前可行性检查”进行校验。

- **使用流式传输获取事件**：`GET /v1/sessions/{id}/events/stream` 是实时接收代理输出的主要方式。

- **SSE 流不支持回放——断线后需合并重连**：如果在 `agent.tool_use`、`agent.mcp_tool_use` 或 `agent.custom_tool_use` 尚未完成（前两者需等待 `user.tool_confirmation`，后者需等待 `user.custom_tool_result`）时流中断，会话将陷入死锁（客户端断开连接→会话空闲→重新连接→但客户端未提供确认或结果）。每次（重新）连接时，请先通过 `GET /v1/sessions/{id}/events/stream` 打开流，再通过 `GET /v1/sessions/{id}/events` 获取事件，按事件 ID 去重后再继续处理。详情参见 `shared/managed-agents-events.md` 的“断线后重连”部分。

- **不要依赖 HTTP 客户端库的超时作为绝对时间上限**：`requests` 的 `timeout=(c, r)` 和 `httpx.Timeout(n)` 都是**每块数据**的读取超时，它们会随着每个字节的到来而重置，因此即使网络缓慢也可能导致请求无限阻塞。若需对原始 HTTP 轮询设定硬性截止时间，请在循环层级使用 `time.monotonic()` 进行计时，并在超时时主动退出。建议优先使用 SDK 提供的 `sessions.events.stream()` 或 `sessions.events.list()`，而非自行实现 HTTP 请求。详情参见 `shared/managed-agents-events.md` 的“接收事件”部分。

- **消息队列机制**：在会话处于 `running` 或 `idle` 状态时均可发送事件，系统会按顺序处理。无需等待上一条消息的响应即可发送下一条。例外情况：当会话因预算耗尽而暂停（`stop_reason: budget_reached`）时，仅允许发送结算类事件；可通过修改或移除预算来恢复执行（参见 `shared/managed-agents-core.md` 第节“会话预算”）。

- **环境的 `config.type` 只能是 `"cloud"` 或 `"self_hosted"`**：“cloud”表示容器运行在 Anthropic 的基础设施上；“self_hosted”则将工具执行迁移到您的自有环境中（详见 `shared/managed-agents-self-hosted-sandboxes.md`）。

- **归档操作为永久性**：对代理、环境、会话、Vault、凭据或内存存储执行归档后，该资源将变为只读且不可撤销。对于代理、环境和内存存储而言，归档后的资源无法被新会话引用（已有会话仍可继续使用）。切勿将 `.archive()` 作为生产环境中的清理手段——**在归档前务必与用户确认**。