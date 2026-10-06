# 受管代理——工具与技能

## 工具

### 服务器端工具 vs 客户端工具

| 类型 | 运行方 | 工作原理 |
|---|---|---|
| **预构建的 Claude 代理工具**（`agent_toolset_20260401`） | Anthropic，在会话容器中运行（适用于 `cloud` 环境；对于 `self_hosted` 环境，由**您的**工作进程提供并运行文件/Shell 脚本工具——参见 `shared/managed-agents-self-hosted-sandboxes.md`）。无论哪种环境，`web_search` 和 `web_fetch` 始终在 Anthropic 的服务器上运行。 | 文件操作、Shell 脚本、网络搜索等。可一次性启用所有功能，也可通过 `enabled: true/false` 单独配置；可通过 `allowed_domains` 和 `blocked_domains` 限制网络工具的使用范围。 |
| **MCP 工具**（`mcp_toolset`） | Anthropic 的编排层 | 由已连接的 MCP 服务器提供的能力。可通过工具集按服务器逐个授予访问权限。 |
| **自定义工具** | **您**——由您的应用处理调用并返回结果 | 代理发出 `agent.custom_tool_use` 事件，会话进入空闲状态，您再发送 `user.custom_tool_result` 事件。 |

**建议：** 通过 `agent_toolset_20260401` 启用所有预构建工具，然后根据需要逐一禁用。

**版本管理：** 工具集是版本化的静态资源。当底层工具发生变化时，会生成新的工具集版本（如 `_20260401`），以便您始终清楚所使用的具体内容。

### 代理工具集

`agent_toolset_20260401` 提供以下内置工具：

| 工具                   | 描述                              |
| ---------------------- | ---------------------------------------- |
| `bash` | 在 Shell 会话中执行 Bash 命令 |
| `read` | 从本地文件系统读取文件，包括文本、图片、PDF 和 Jupyter 笔记本 |
| `write` | 将文件写入本地文件系统 |
| `edit` | 对文件中的字符串进行替换 |
| `glob` | 使用 glob 模式进行快速文件匹配 |
| `grep` | 使用正则表达式进行文本搜索 |
| `web_fetch` | 从指定 URL 获取内容 |
| `web_search` | 在网络上搜索信息 |

启用完整工具集：

```json
{
  "tools": [
    { "type": "agent_toolset_20260401" }
  ]
}
```

### 单个工具的配置

可对单个工具覆盖默认设置。以下示例启用了除 `bash` 外的所有工具：

```json
{
  "tools": [
    {
      "type": "agent_toolset_20260401",
      "default_config": { "enabled": true },
      "configs": [
        { "name": "bash", "enabled": false }
      ]
    }
  ]
}
```

| 字段 | 必填 | 描述 |
|---|---|---|
| `type` | 是 | `"agent_toolset_20260401"` |
| `default_config` | 否 | 应用于所有工具。格式为 `{ "enabled": bool, "permission_policy": {...} }` |
| `configs` | 否 | 针对每个工具的覆盖配置：`[{ "name": "...", "type": "...", "enabled": bool, "permission_policy": {...} }]`。`name` 用于标识工具（取值见上表）；`type` 在请求中可省略（其值与 `name` 相同，由服务端推断），但在响应中始终存在。`web_search` 和 `web_fetch` 条目还可接受网络相关设置——详见下文“网络搜索与网页抓取设置”部分。 |

> **强类型 SDK**：`configs` 中的每个条目都是一个联合类型的一个成员，每个内置工具对应一个成员（共八个：`BetaManagedAgentsWebFetchToolConfigParams`、`...WebSearchToolConfigParams`、`...BashToolConfigParams` 等），并通过 `type` 进行区分。Python/TypeScript/Ruby 中仅包含 `name` + `enabled` + `permission_policy` 的字典或哈希保持不变。在 Go、Java、C# 和 PHP 中，`configs` 本身就是该联合类型——需根据各工具的具体类型来构建每个条目（Go：`BetaManagedAgentsAgentToolConfigUnionParamsUnion{OfWebFetch: &anthropic.BetaManagedAgentsWebFetchToolConfigParams{...}}`——分支包括 `OfBash` / `OfRead` / `OfWrite` / `OfEdit` / `OfGlob` / `OfGrep` / `OfWebFetch` / `OfWebSearch`；Java：`.addConfig(BetaManagedAgentsWebFetchToolConfigParams.builder()...build())`；C#：`new BetaManagedAgentsWebFetchToolConfigParams { Enabled = false }`；PHP：`BetaManagedAgentsWebFetchToolConfigParams::with(enabled: false)`）。如果代码是基于所有工具共享同一配置类型的 SDK 编写的，则必须更新其构造条目的方式。

### 权限策略

控制由服务器执行的工具（代理工具集 + MCP）是自动运行、等待您的批准，还是每次调用都由服务器进行评估。此设置不适用于自定义工具（此类工具由您的应用程序直接执行）。

| 策略 | 行为 |
|---|---|
| `always_allow` | 工具自动执行。代理工具集的默认值。 |
| `always_ask` | 会话会发出 `session.status_idle` 事件（`stop_reason.type: requires_action`），并暂停，直到您发送 `user.tool_confirmation` 事件。MCP 工具集的默认值。 |
| `auto` | 服务器会对每次调用（工具、输入及当前会话内容）进行评估，并**允许执行、拒绝执行，或暂停等待您的批准**。两种工具集均未将 `auto` 设为默认值。详见下文 § `auto` 部分。

```json
{
  "type": "agent_toolset_20260401",
  "default_config": {
    "enabled": true,
    "permission_policy": { "type": "always_allow" }
  },
  "configs": [
    { "name": "bash", "permission_policy": { "type": "always_ask" } }
  ]
}
```

**响应 `always_ask`**（以及被暂停的 `auto` 调用）：发送 `user.tool_confirmation` 事件，并将 `tool_use_id` 设置为触发该调用的 `agent.tool_use` 或 `agent.mcp_tool_use` 事件的**事件 ID**（`sevt_...`，而非 `toolu_` ID）。可在一次 `events` 请求中包含多个确认：

```js
{ "type": "user.tool_confirmation", "tool_use_id": "sevt_abc123", "result": "allow" }
{ "type": "user.tool_confirmation", "tool_use_id": "sevt_def456", "result": "deny", "deny_message": "请改读 .env.example 文件" }
```

在拒绝时可选填的 `deny_message` 会作为被拒绝的工具结果传递给代理，以便其调整应对策略。对于 `evaluated_permission` 不为 `"ask"` 的事件发送的 `user.tool_confirmation` 将被拒绝，并返回 400 错误——这同样适用于服务器在 `auto` 模式下已拒绝的调用；您的客户端无法覆盖这些决定。

#### `auto`——让服务器评估每次调用

在任何接受 `permission_policy` 的地方均可设置为 `{"type": "auto"}`：工具集的 `default_config` 或单个 `configs` 条目，无论是代理工具集还是 `mcp_toolset`。由于评估会考虑调用的输入及截至当时的会话内容，因此对同一工具的两次调用可能得到不同处理。每次调用仅有三种结果之一：

| 结果 | 发生情况 |
|---|---|
| **执行** | 服务器判定调用安全——按 `always_allow` 模式执行，无需通知您的客户端。 |
| **拒绝** | 服务器评估认为调用风险较高——工具不会执行。代理会收到一条错误的工具结果（`Permission to use {tool_name} has been denied.`，`is_error: true`），会话**将继续运行**，且您的客户端无法覆盖该拒绝。 |
| **暂停** | 服务器未能作出判断——会话将与 `always_ask` 模式下一样暂停；此时需通过 `user.tool_confirmation` 做出回应。 |

```json
{
  "name": "运维代理",
  "model": "claude-opus-5-5",
  "mcp_servers": [{ "type": "url", "name": "github", "url": "https://mcp.example.com/github" }],
  "tools": [
    {
      "type": "agent_toolset_20260401",
      "default_config": { "permission_policy": { "type": "auto" } },
      "configs": [{ "name": "bash", "permission_policy": { "type": "always_ask" } }]
    },
    {
      "type": "mcp_toolset",
      "mcp_server_name": "github",
      "default_config": { "permission_policy": { "type": "auto" } }
    }
  ]
}
```

在 Python、TypeScript 和 Ruby 中，可以使用与无类型字典/对象字面量/哈希表相同的结构来传递。对于强类型的 SDK（Go、Java、C#、PHP），需要为随每个 SDK 功能发布一起提供的 `auto` 策略生成相应的类型；在此之前，请使用无类型语言或通过 cURL / `ant` 构建请求。Python 和 TypeScript 也仅在添加该策略的版本中对 `{"type": "auto"}` 进行类型检查（而 Wire API 无论如何都接受它）。

**评估所信任的内容。** 服务器将会话内容视为评估的依据，而非必须执行的指令。您在 `user.message` 事件中发布的文本（包括您转发的最终用户输入）被视为您的意图，并可能导致服务器允许原本会被拒绝的调用——尽管某些调用无论何时都会被判定为高风险。同样的文字出现在工具结果、抓取的网页、MCP 服务器响应或会话线程之间的消息中，则不具有这种效力。如果您在 `user.message` 中转发不受信任的最终用户输入，服务器也会将其视为您的意图，并可能因此允许某次调用；请对那些您不希望该最终用户未经审核即可使用的工具设置 `always_ask`。

> **`auto` 并非人工审查环节。** 服务器判定为安全的调用会在任何人看到之前即已执行，其影响可能无法撤销。如果某个工具的调用必须由人工审核后才能执行，请对该工具使用 `always_ask`。

#### `evaluated_permission` 和 `evaluation`——查看每次调用是如何被评估的

在**任何**策略下，每个 `agent.tool_use` 和 `agent.mcp_tool_use` 事件都会携带 `evaluated_permission`（`"allow" | "ask" | "deny"`）——即权限检查的结果。大多数事件还会包含一个 `evaluation` 对象，其 `type` 属性指明了产生该结果的策略；在 `auto` 策略下，还会额外显示服务器的判定结果，并在 `ask` 或 `deny` 的情况下提供一个 `reason_code`：

```json
{
  "type": "agent.tool_use",
  "id": "sevt_01pqr...",
  "name": "bash",
  "input": { "command": "rm -rf /workspace/reports" },
  "evaluated_permission": "deny",
  "evaluation": {
    "type": "auto",
    "evaluated_permission": { "type": "deny", "reason_code": "high_risk" }
  },
  "processed_at": "2026-03-25T14:05:12Z"
}
```

| `evaluation` | 顶层 `evaluated_permission` | 含义 |
|---|---|---|
| `{"type": "always_allow"}` | `"allow"` | 最终确定的策略是 `always_allow`；调用已执行。 |
| `{"type": "always_ask"}` | `"ask"` | 最终确定的策略是 `always_ask`；调用已暂停等待您的批准。 |
| `{"type": "auto", "evaluated_permission": {"type": "allow"}}` | `"allow"` | 服务器判定调用安全；调用已执行。 |
| `{"type": "auto", "evaluated_permission": {"type": "ask", "reason_code": "indeterminate"}}` | `"ask"` | 服务器未能做出明确判断；调用已暂停等待您的批准。 |
| `{"type": "auto", "evaluated_permission": {"type": "deny", "reason_code": "high_risk"}}` | `"deny"` | 服务器认为该调用风险较高并予以拒绝。 |

- 在 `auto` 表单中，嵌套的 `evaluated_permission.type` 始终等于事件最上层的 `evaluated_permission`。
- `reason_code` 供您的客户端用于分支判断，并记录在审计日志中——不应作为文本向最终用户展示。
- 当代理调用会话中未启用的工具时（服务器在不评估任何策略的情况下直接拒绝：`evaluated_permission: "deny"`，且无 `evaluation`），以及在该字段尚未存在时记录的事件中，`evaluation` 字段将**不存在**；对于这些事件，可将其视为 `"allow"` 对应的 `always_allow`，或 `"ask"` 对应的 `always_ask`。
- 请在客户端代码中设计为能够容忍其无法识别的 `evaluation.type` 或 `reason_code`。
- `agent.custom_tool_use` 事件不携带上述两个字段（自定义工具不受权限策略约束）。

如需仅启用特定工具，请先关闭默认设置，再逐个启用所需工具：

```json
{
  "tools": [
    {
      "type": "agent_toolset_20260401",
      "default_config": { "enabled": false },
      "configs": [
        { "name": "bash", "enabled": true },
        { "name": "read", "enabled": true }
      ]
    }
  ]
}
```

### 网络搜索与网页抓取设置（域名过滤）

`web_search` 和 `web_fetch` 即使在任何环境类型下，也始终在 Anthropic 的服务器上运行，因此环境的 `networking` 策略对其**不适用**（参见 `shared/managed-agents-environments.md` 中的“网络”部分）。要控制它们可访问的范围，请在工具的 `configs` 配置项中设置 `allowed_domains`（仅允许这些主机）**或** `blocked_domains`（禁止访问这些主机）——同一配置项中不得同时设置两者。每个工具均维护独立的列表。控制台中的组织级网络搜索/抓取设置仅适用于 Messages API，而不适用于 Managed Agents 会话。

```json
{
  "type": "agent_toolset_20260401",
  "configs": [
    {
      "type": "web_search",
      "name": "web_search",
      "allowed_domains": ["docs.example.com", "arxiv.org"],
      "user_location": { "type": "approximate", "country": "US", "timezone": "America/Los_Angeles" }
    },
    {
      "type": "web_fetch",
      "name": "web_fetch",
      "blocked_domains": ["ads.example.com"],
      "max_content_tokens": 50000
    }
  ]
}
```

| 设置 | 适用对象 | 描述 |
|---|---|---|
| `allowed_domains` | `web_search`、`web_fetch` | 工具仅可访问的主机列表。与同一配置项中的 `blocked_domains` 互斥。 |
| `blocked_domains` | `web_search`、`web_fetch` | 工具不可访问的主机列表。 |
| `max_content_tokens` | `web_fetch` | 抓取内容中进入上下文的文本 token 数量上限（正整数），二进制内容（如 PDF）不受此限制。 |
| `user_location` | `web_search` | `{ "type": "approximate", city?、region?、country?（ISO 3166-1 两位大写代码）、timezone?（IANA 标准时区） }`——至少需填写一个可选字段。 |

**运行时行为：** 如果 `web_fetch` 调用的 URL 不在其允许列表内，则会向代理返回错误结果（`agent.tool_result` 中的 `is_error: true`，且内容标识为 `url_not_allowed`）；`web_search` 则会静默地忽略不在其允许列表内的搜索结果。在控制台中，代理表单提供了针对网络工具的允许/阻止列表控件；`user_location` 和 `max_content_tokens` 可在代理的“原始视图”中进行设置。

**域名列表规则**（违规将在创建或更新代理时，以及在创建或更新会话并提供 `tools` 时引发 400 `invalid_request_error` 错误；错误信息中会指明具体是哪个列表及零起索引，例如：“allowed_domains.0: 不支持 IP 地址...”）：- 每个列表最多包含1–64个域名，每个域名长度为1–255个字符。空列表将被拒绝——如无限制，请省略该字段或发送`null`。列表中重复的域名也将被拒绝。
- 仅支持纯主机名，例如`example.com`，不支持`https://example.com`、`example.com:443`或`*.example.com`。域名不区分大小写；末尾的单个斜杠会被忽略。
- 列表中的一个域名会覆盖其自身及其所有子域名（例如`example.com`会覆盖`docs.example.com`，但`docs.example.com`不会覆盖`example.com`或`api.example.com`）。`www.`被视为普通子域名——若要同时覆盖带`www.`和不带`www.`的域名，请列出不含`www.`的主域名。
- 以下内容将被拒绝：任何形式的IP地址；裸顶级域名或注册后缀（如`com`、`co.uk`）；单标签域名（如`intranet`）；`localhost`以及以`.localhost`、`.local`、`.internal`、`.localdomain`、`.invalid`结尾的主机名；非ASCII字符（需使用`xn--` Punycode编码）。
- `web_fetch`工具的域名不允许携带路径。`web_search`工具的域名可以带有路径后缀（如`example.com/blog`，且路径中不得包含空格、`?`、`#`、`$`、`,`、`|`、`^`、`!`），但服务提供商会将其视为URL模式进行匹配——建议优先使用纯主机名。
- 同时可能存在服务提供商特定的拒绝情况：Anthropic爬虫可能无法访问某些域名，或者提供的`user_location.country`值无效（消息末尾显示“不是该搜索服务提供商支持的国家”），又或者IANA时区代码无效。

会话在首次初始化工具时会重新检查配置；如果先前接受的设置不再有效，会话将发出`session.error`事件并进入`idle`状态，且不会重试。可通过更新会话工具（参见`shared/managed-agents-core.md`中的“会话期间更新代理配置”部分）来修复此问题，并同步更新代理，以便新会话能够获得修复；随后再发送一条新的`user.message`。

**多代理层叠机制**（参见`shared/managed-agents-multiagent.md`）：通往某个线程的所有列表会同时生效——一名名单代理不仅受自身列表约束，还受调用它的所有代理的列表以及协调器当前列表的约束。白名单取交集，黑名单取并集，因此名单代理只能缩小范围，而不能放宽限制。若白名单互不相交，则工具虽仍可用，但每次调用都会因`url_not_allowed`而失败（工具描述会告知模型原因）——请确保名单代理的白名单始终包含在协调器的白名单范围内。`max_content_tokens`和`user_location`**不**叠加使用：优先采用自身的值，其次为调用方的值，最后是协调器的值。`{"type": "self"}`条目遵循协调器的设定。结果评估器（参见`shared/managed-agents-outcomes.md`）在运行时不会使用网络工具。更新处于空闲状态的会话所使用的工具，将从下一轮开始改变协调器对所有线程的列表；而名单代理自身的列表则保持在会话创建时的定义不变。

**与Messages API的`web_search_20260209`/`web_fetch_20260209`工具相比：** 允许域名和禁止域名的语义相同，但最大条目数为64，`web_fetch`工具的域名不允许带路径，且没有`max_uses`、`citations`或`cache_control`等参数。若从Messages API迁移，这些参数将由每请求设置改为在代理上一次性配置。

### 自定义工具（客户端侧）

自定义工具由**您的应用**而非Anthropic执行。流程如下：

1. 代理决定使用该工具时，会话会发出带有输入参数的`agent.custom_tool_use`事件；
2. 会话进入`idle`状态，等待您的响应；
3. 您的应用执行该工具；
4. 您通过`user.custom_tool_result`事件返回执行结果；
5. 会话恢复为`running`状态。

无需权限策略——因为执行者是您自己。

```json
{
  "tools": [
    {
      "type": "custom",
      "name": "get_weather",
      "description": "获取某城市的当前天气。",
      "input_schema": {
        "type": "object",
        "properties": {
          "city": { "type": "string", "description": "城市名称" }
        },
        "required": ["city"]
      }
    }
  ]
}
```

### MCP服务器

MCP（模型上下文协议）服务器提供了标准化的第三方能力接口（例如Asana、GitHub、Linear）。**配置分为代理端和保险库两部分：**1. **代理创建** 用于声明要连接的服务器（`type`、`name`、`url`——无需认证）。代理的 `mcp_servers` 数组中没有 `auth` 字段。
2. **Vault** 用于存储 OAuth 凭证。在创建会话时，通过 `vault_ids` 进行关联。

这样可以将敏感信息从可复用的代理定义中隔离出来。每个 Vault 凭证都与一个 MCP 服务器 URL 绑定；Anthropic 会根据 URL 将凭证与服务器进行匹配。

**代理端——声明服务器（无需认证）：**

| 字段 | 必填 | 描述 |
|---|---|---|
| `type` | 是 | `"url"` |
| `name` | 是 | 唯一名称——由 `mcp_toolset.mcp_server_name` 引用 |
| `url` | 是 | MCP 服务器的端点 URL（支持流式 HTTP 传输） |

```json
{
  "mcp_servers": [
    { "type": "url", "name": "linear", "url": "https://mcp.linear.app/mcp" }
  ],
  "tools": [
    { "type": "mcp_toolset", "mcp_server_name": "linear" }
  ]
}
```

**会话端——关联 Vault：**

```json
{
  "agent": "agent_abc123",
  "environment_id": "env_abc123",
  "vault_ids": ["vlt_abc123"]
}
```

> 提示：**按工具启用**：`mcp_toolset` 接受 `default_config: {enabled: false}` 和 `configs: [{name, enabled: true}]` 的配置，以实现白名单模式。MCP 的 `configs` 条目仅包含 `name`（即服务器上报的工具名称）、`enabled` 和 `permission_policy`，不包含 `type` 字段，也不包含 `web_search` 或 `web_fetch` 在代理工具集中接受的 Web 相关设置。

> 提示：**在运行中的会话上更换工具或 MCP 服务器**：当会话处于 `idle` 状态时，可以通过 `sessions.update()` 替换 `agent.tools` 和 `agent.mcp_servers`——这是一种会话级别的覆盖，不会影响代理对象本身。`vault_ids` 只能在创建时设置。详情请参阅 `shared/managed-agents-core.md` 中的“在会话中途更新代理配置”部分。

**大型工具输出**：如果某个工具返回的内容超过 **10 万字符（约 2.5 万个 token）**，输出将自动转储到沙盒中的文件；代理会收到截断的预览内容以及文件路径，并可通过 `read` 操作获取完整内容。无需额外配置。该阈值以 *字符数* 计算，不仅适用于内置代理工具，也适用于 MCP 工具。

**无效的 Vault 凭证不会阻止会话创建**：如果为已声明的 MCP 服务器提供的 Vault 凭证无效，会话仍会成功创建；`session.error` 事件会记录 MCP 认证失败的信息，并在下一次会话状态从 `idle` 切换到 `running` 时重试认证。

> 警告：**MCP 认证令牌 ≠ REST API 令牌**。托管的 MCP 服务器（如 `mcp.notion.com`、`mcp.linear.app` 等）通常需要 **OAuth Bearer 令牌**，而非服务自身的原生 API 密钥。Notion 的 `ntn_` 集成令牌可用于 Notion 的 REST API 认证，但**不能**作为 Notion MCP 服务器的 Vault 凭证使用。这是两种不同的认证体系。

### Vaults——凭据存储

**Vaults** 是 Anthropic 代表您管理凭据的存储空间。凭据分为两类：

- **MCP 凭证**（`mcp_oauth`、`static_bearer`）——以 `mcp_server_url` 为键。当代理连接到该 URL 的服务器时，令牌会自动注入。**匹配过程经过规范化处理，而非严格逐字节比对**：协议和主机名会被转换为小写，且会去除默认端口和尾部斜杠，因此主机名大小写、显式指定的默认端口或尾部斜杠都不会影响匹配。但如果路径、子域名或非默认端口不同，则会导致匹配失败。如果没有匹配项，连接将以未认证状态尝试。`mcp_oauth` 令牌会通过标准的 OAuth 2.0 `refresh_token` 授权机制自动刷新。这是认证 MCP 服务器的唯一方式。
- **环境变量**（`environment_variable`）——以 `secret_name`（环境变量名）为键。沙盒中只会看到一个**透明占位符**；真正的密钥会在出站请求时**在出口处**被替换进去。此功能适用于任何通过环境变量进行认证的服务：命令行工具（如 `aws`、`gcloud`、`stripe`）、SDK，或直接通过 `bash` 工具发出的 `curl` 请求。您提供的机密字段（`token`、`access_token`、`refresh_token`、`client_secret`、`secret_value`）均为只写，绝不会在 API 响应中返回。

#### 凭证与沙箱

凭据存储于保险库中；这些凭据**永远不会进入沙箱**。这是一项刻意设置的安全边界——在沙箱中运行的代码（包括代理写入的任何内容）都无法读取或窃取已存入保险库的凭据，即使在提示注入攻击下亦然。相反，凭据会在请求离开沙箱后，由 Anthropic 侧的代理进行注入：

- **MCP 工具调用**会通过 Anthropic 侧的代理路由，该代理从保险库获取凭据并将其添加到出站请求中。
- **对关联 GitHub 仓库的 Git 操作**（如 `git pull`、`git push`，以及 GitHub REST API 调用）会通过 Git 代理路由，该代理以相同方式注入 `github_repository` 资源的 `authorization_token`。
- **环境变量凭据**在沙箱中以不透明占位符形式出现；真实值仅在出站时、且仅针对该凭据允许访问的主机才会替换占位符。替换仅适用于请求的**头和正文**——嵌入在**URL 路径**中的秘密不会被替换，因此路径式秘密端点（如 Slack 入站 Webhook URL）无法存入保险库；请改用基于头部的身份验证（例如，对于 Slack，在 `Authorization` 头中使用 bot token，并通过 `chat.postMessage` 进行调用）。

**当保险库凭据不适用时**（例如自托管沙箱——目前尚不支持 `environment_variable`），**请注册自定义工具**：代理会发出 `agent.custom_tool_use` 事件，您的编排器（已持有该凭据）将执行调用，并通过同一经过身份验证的事件流返回 `user.custom_tool_result`。不会暴露任何公共端点；沙箱始终无法看到该秘密。详情请参阅 `shared/managed-agents-client-patterns.md` 中的模式 9。

**切勿将 API 密钥作为变通方案放入系统提示或用户消息中**——它们会保留在会话的事件历史中。

> 此前在内部被称为 TAT（工具/租户访问令牌）。

**流程如下**：

1. 创建一个保险库（`client.beta.vaults.create(...)`）——根据您的模型，可以为每个租户/用户创建一个，也可以共用一个。
2. 向保险库添加凭据（`client.beta.vaults.credentials.create(...)`）——MCP 凭据按 MCP 服务器 URL 进行索引；环境变量凭据则按 `secret_name` 索引。
3. 在创建会话时，通过 `vault_ids: ["vlt_..."]` 引用该保险库。
4. Anthropic 会在 OAuth 令牌到期前自动刷新，并在运行时注入机密。

**MCP OAuth 凭据的结构**：

```json
{
  "display_name": "Notion（workspace-foo）",
  "auth": {
    "type": "mcp_oauth",
    "mcp_server_url": "https://mcp.notion.com/mcp",
    "access_token": "<当前访问令牌>",
    "expires_at": "2026-04-02T14:00:00Z",
    "refresh": {
      "refresh_token": "<刷新令牌>",
      "client_id": "<您的 OAuth 客户端 ID>",
      "token_endpoint": "https://api.notion.com/v1/oauth/token",
      "token_endpoint_auth": { "type": "none" }
    }
  }
}
```

`refresh` 部分实现了自动刷新功能——`token_endpoint` 是 Anthropic 发送 `refresh_token` 授权请求的地址。`token_endpoint_auth` 是一种可区分联合类型：

| `type` | 结构 | 使用场景 |
|---|---|---|
| `"none"` | `{type: "none"}` | 公开 OAuth 客户端（无密钥） |
| `"client_secret_basic"` | `{type: "client_secret_basic", client_secret: "..."}` | 保密客户端，通过 HTTP Basic 认证传递密钥 |
| `"client_secret_post"` | `{type: "client_secret_post", client_secret: "..."}` | 保密客户端，密钥放在请求体中 |

如果您只有访问令牌而没有刷新能力，则可完全省略 `refresh` 部分——它将在过期前正常工作，过期后代理将失去访问权限。

> 小贴士：**获取 OAuth 令牌**。如何获取初始访问令牌和刷新令牌取决于具体的 MCP 服务器，请查阅其文档。获得后，按照上述结构将其存储为保险库凭据；Anthropic 将从此处通过 `refresh.token_endpoint` 自动刷新。**环境变量凭据形状**：

```json
{
  "display_name": "沙盒环境的 Twilio API 密钥",
  "auth": {
    "type": "environment_variable",
    "secret_name": "TWILIO_API_KEY",
    "secret_value": "sk-your-secret-here",
    "networking": {
      "type": "limited",
      "allowed_hosts": ["api.twilio.com", "*.twilio.com"]
    }
  }
}
```

`networking.allowed_hosts` 用于控制该密钥可被替换到哪些出站目标主机——可以设置为 `{"type": "limited", "allowed_hosts": [...]}`，或者在无法预先枚举所有域名时设置为 `{"type": "unrestricted"}`。强烈建议进行限制：这能防止密钥被发送至未经授权的主机。

**`injection_location`**（可选，与 `networking` 同级）用于控制在出站请求的**何处**插入该密钥——格式为 `{header: bool, body: bool}`。这两者是独立的：`allowed_hosts` 决定被替换后的请求可以访问**哪些主机**；而 `injection_location` 则决定在这些主机的所有请求中，密钥会被插入到**请求的哪些部分**。大多数服务会从请求头中读取 API 密钥，因此 `{"header": true}` 是更为严格的配置——因为请求体通常由代理正在处理的内容组装而成，所以请求体往往是更大的暴露面。如果某个位置被禁用，则占位符**既不会被替换，也不会被移除**——原样保留的不透明占位符字符串会按字面意思发送给第三方。

| 操作 | `injection_location` 的语义 |
|---|---|
| 创建凭据 | 完全省略该字段 -> 默认启用两个位置。提供该对象 -> 未指定的字段默认为 `false`（即 `{"header": true}` 仅启用请求头）。 |
| 更新凭据 | 字段**单独合并**——`{"body": false}` 禁用请求体插入，而请求头保持不变。对于正在运行的会话，更新会在下一次操作时生效。 |

凭据必须至少启用一个位置；若创建或更新导致两个位置均被禁用，则返回 400 错误；对整个对象或任一字段显式传入 `null` 也会返回 400（应改为省略）。响应中始终会返回两个字段及其解析后的值。

> 警告：**在控制台中创建的凭据默认仅启用请求头**——与 API 不同，API 中省略该字段会同时启用两个位置。如果您的客户端在请求体中发送密钥（例如表单编码的令牌请求），占位符会原样传递，服务端会以自身的认证错误拒绝该请求。请在控制台表单中勾选请求体注入，或使用 `{"injection_location": {"body": true}}` 来 POST 凭据。

> 警告：**两层网络配置，缺一不可。** 凭据上的 `networking.allowed_hosts` 控制的是**哪些请求会使用该密钥**，而非**哪些请求被允许**。代理还必须能够在**环境层面**访问该域名（设置为 `unrestricted`，或将其列入环境的 `allowed_hosts` 中——参见 `shared/managed-agents-environments.md`）。如果任一层缺少该域名，则使用密钥替换后的请求都会失败。

> 警告：**客户端侧验证的注意事项。** 替换发生在出站时，而非沙盒内部——那些在发起网络请求前本地验证凭据**格式**的客户端（例如检查密钥是否以 `sk-` 开头的 CLI）会看到不透明的占位符，并可能在启动时失败。如果客户端在任何网络调用之前就拒绝了凭据，原因就在于此。

> 提示：**尽量缩小密钥的作用范围。** 代理所能执行的操作完全取决于密钥的权限；如果密钥的权限超出任务所需，一旦代理行为异常，就会扩大影响范围。

**自托管沙盒不支持此功能**——`environment_variable` 类型的凭据需要 Anthropic 管理的出站通道。详情请参阅 `shared/managed-agents-self-hosted-sandboxes.md`。

**约束条件（适用于所有凭据类型）：**

- **每个保险库的键必须唯一。** `mcp_server_url`（MCP 凭据）和 `secret_name`（环境变量凭据）在保险库中的所有有效凭据中必须是唯一的；重复的键会导致 409 错误。
- **键不可更改。** 密钥值、`display_name`，以及（对于环境变量凭据）`injection_location` 可以更新；若要更改 `mcp_server_url`、`secret_name`、`token_endpoint` 或 `client_id`，需先归档该凭据并创建一条新记录。归档操作会清除密钥内容，并释放该键以便替换。
- **每个保险库最多可存储 20 条凭据。**
- 凭据按原样存储，**仅在会话运行时才会进行验证**——无效凭据会在会话过程中引发身份验证或下游错误，这些错误会被报告，但不会阻止会话继续。

**作用域：** 保险库与工作空间绑定。拥有 API 工作空间“开发者”及以上角色的用户均可创建、读取（仅限元数据——密钥为只写）及关联保险库。`vault_ids` 可在会话 **创建** 时设置，但无法通过会话更新来修改（SDK 文档注释说明：“暂不支持；设置此字段的请求将被拒绝”）。

---

## 技能

技能是可复用的、基于文件系统的资源，为您的代理提供特定领域的专业知识：包括工作流、上下文和最佳实践，从而将通用型代理转变为专业型代理。与提示词（针对一次性任务的对话级指令）不同，技能按需加载，无需在多次对话中反复提供相同的指导。

技能有两种方式传递给代理：一是通过代理的 `skills` 数组 **附加**；二是从挂载到会话的 **GitHub 仓库加载**（详见下文“来自 GitHub 仓库的技能”）。当任务相关时，代理会自动使用这些技能：

| 类型 | 内容 |
|---|---|
| **Anthropic 预置技能** | 常见文档处理任务（PowerPoint、Excel、Word、PDF）。按名称引用（如 `xlsx`）。 |
| **自定义技能** | 您通过技能 API 在组织内创建的技能。按 `skill_id` 加可选的 `version` 引用。 |

**每个代理最多可关联 20 个技能。** 代理创建时使用 `managed-agents-2026-04-01` 版本；用于管理自定义技能定义的独立技能 API 已退出 Beta 阶段，无需指定 Beta 标头。

### 在会话中启用技能

技能通过 `agents.create()` 方法附加到 **代理** 定义中：

```ts
const agent = await client.beta.agents.create(
  {
    name: "财务代理",
    model: "claude-opus-5-5",
    system: "您是一位财务分析代理。",
    skills: [
      { type: "anthropic", skill_id: "xlsx" },
      { type: "custom", skill_id: "skill_abc123", version: "latest" },
    ],
  }
);
```

Python：

```python
agent = client.beta.agents.create(
    name="财务代理",
    model="claude-opus-5-5",
    system="您是一位财务分析代理。",
    skills=[
        {"type": "anthropic", "skill_id": "xlsx"},
        {"type": "custom", "skill_id": "skill_abc123", "version": "latest"},
    ]
)
```

**技能引用字段：**

| 字段 | Anthropic 技能 | 自定义技能 |
|---|---|---|
| `type` | `"anthropic"` | `"custom"` |
| `skill_id` | 技能名称（如 `"xlsx"`、`"docx"`、`"pptx"`、`"pdf"`） | 技能 API 中的技能 ID（如 `"skill_abc123"`） |
| `version` | `"latest"` 或具体版本号 | `"latest"` 或具体版本号 |

`version` 对于两种类型的技能都是可选的，默认值为 `"latest"`——并非仅适用于自定义技能。

### 来自 GitHub 仓库的技能

技能也可以存放在您的代码库中。当会话通过 `github_repository` 资源挂载一个仓库时（参见 `shared/managed-agents-environments.md` 中的“GitHub 仓库”部分），会话启动时会扫描该仓库的根目录下的 `.claude/skills` 目录，找到的每项技能都将对代理可用：代理可以看到每项技能的名称、描述及其沙盒路径，并在任务匹配时读取该技能的 `SKILL.md` 文件（以及其附带的所有脚本和资源）。**代理可以在 `.claude/skills/<技能名>/` 目录下发现任何技能**——该目录位于仓库根目录下，深度为一层。以下位置的技能无法被发现：一个空的 `.claude/skills/SKILL.md` 文件（没有对应的技能目录）、嵌套更深的文件（如 `.claude/skills/tools/code-review/SKILL.md`）、位于 `.claude` 外部的 `skills/` 目录，或位于某个包子目录内的 `.claude/skills` 目录（尽管在代理读取该子树下的文件时仍可能被识别）。`SKILL.md` 文件的格式与上传的自定义技能相同。

> 警告：**仓库中的技能即为代理的指令——请将其视为信任边界的一部分。** 任何能够向已挂载的仓库提交代码的人（例如合并了外部 Pull Request、依赖项被入侵或贡献者）都可以添加或修改 `.claude/skills/` 中的内容，而平台会在会话启动时直接加载这些内容，且不经过任何审核步骤——这意味着像 `bash` 和 `web_fetch` 这样的会话工具一旦接收到注入的指令，便具备了实际的执行能力。请仅挂载您信任的仓库，并在有外部贡献者参与的仓库上线前对其 `.claude/skills/` 内容进行审计。

规则：
- **仅限云端沙箱**——自建的沙箱不支持 `github_repository` 资源，因此无法加载仓库中的技能。
- **仅在会话启动时扫描一次**，扫描的是当时检出的仓库状态（即资源指定的 `checkout` 分支或提交，若未指定则使用默认分支）。会话期间推送的新提交不会被纳入扫描范围——如需使用更新后的技能，请开启一个新的会话。已加入到*运行中*会话的仓库也不会被扫描。
- **与附加技能共存**。如果某个仓库技能与附加技能（或来自其他已挂载仓库的技能）同名，则两者均可使用，且各自会通过其路径被声明。

### 技能 API

| 操作             | 方法   | 路径                                            |
| --------------------- | -------- | ----------------------------------------------- |
| 创建技能          | `POST`   | `/v1/skills`                                    |
| 列举技能           | `GET`    | `/v1/skills`                                    |
| 获取技能             | `GET`    | `/v1/skills/{id}`                               |
| 删除技能          | `DELETE` | `/v1/skills/{id}`                               |
| 创建版本        | `POST`   | `/v1/skills/{id}/versions`                      |
| 列举版本         | `GET`    | `/v1/skills/{id}/versions`                      |
| 获取版本           | `GET`    | `/v1/skills/{id}/versions/{version}`            |
| 删除版本        | `DELETE` | `/v1/skills/{id}/versions/{version}`            |

