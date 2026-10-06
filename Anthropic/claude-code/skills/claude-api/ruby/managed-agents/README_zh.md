# 托管代理 - Ruby

> **此处未列出的绑定：** 本 README 涵盖了 Ruby 中最常见的托管代理流程。如果您需要未在此处展示的类、方法、命名空间、字段或行为，请通过 WebFetch 获取 Ruby SDK 仓库**或相关文档页面**，路径位于 `shared/live-sources.md`，切勿根据 cURL 的形式或其他语言的 SDK 进行推测。

> **代理是持久化的——只需创建一次，并通过 ID 引用。** 请保存 `client.beta.agents.create` 返回的代理 ID，并在后续每次调用 `client.beta.sessions.create` 时传递该 ID；不要在请求路径中重复调用 `agents.create`。**建议：** 将代理和环境定义为受版本控制的文件，并通过 `ant apply` 同步——参见 `shared/anthropic-cli.md`（其实时文档 URL 位于 `shared/live-sources.md`）。CLI 负责控制平面（创建/更新）；您的代码负责数据平面（使用已保存 ID 的会话）。以下示例展示了在必须以编程方式进行资源配置时的代码内创建；但在生产环境中，创建调用应放在初始化阶段，而非请求路径中。

## 安装

```bash
gem install anthropic
```

## 客户端初始化

```ruby
require "anthropic"

# 默认方式（使用 ANTHROPIC_API_KEY 环境变量）
client = Anthropic::Client.new

# 显式指定 API 密钥
client = Anthropic::Client.new(api_key: "your-api-key")
```

> 注意：**尾部下划线：** Ruby SDK 使用 `system_:` 和 `send_(`（尾部下划线）来避免与 `Kernel#system` 和 `Kernel#send` 产生名称冲突。在所有托管代理代码中，请始终使用这些形式。

---

## 创建一个环境

```ruby
environment = client.beta.environments.create(
  name: "my-dev-env",
  config: {
    type: "cloud",
    networking: {type: "unrestricted"}
  }
)
puts "环境 ID: #{environment.id}" # env_...
```

---

## 创建一个代理（必需的第一步）

> 注意：**不存在内联代理配置。** `model`、`system_` 和 `tools` 属于代理对象，而非会话。请始终从 `client.beta.agents.create()` 开始；会话要么接收 `agent: agent.id`，要么接收类型化的哈希形式 `agent: {type: "agent", id: agent.id, version: agent.version}`。

### 最简示例

```ruby
# 1. 创建代理（可复用、带版本）
agent = client.beta.agents.create(
  name: "编程助手",
  model: :"claude-opus-5-5",
  system_: "你是一位有用的编程助手。",
  tools: [{type: "agent_toolset_20260401"}]
)

# 2. 启动会话
session = client.beta.sessions.create(
  agent: {type: "agent", id: agent.id, version: agent.version},
  environment_id: environment.id,
  title: "快速入门会话"
)
puts "会话 ID: #{session.id}"
puts "追踪链接: https://platform.claude.com/workspaces/default/sessions/#{session.id}"  # 如果 API 密钥不在默认工作空间，请将 'default' 替换为您的工作空间 ID
```

### 更新代理

更新操作会创建新版本；每个版本的代理对象都是不可变的。

```ruby
updated_agent = client.beta.agents.update(
  agent.id,
  version: agent.version,
  system_: "你是一位有用的编程助手。始终编写测试用例。"
)
puts "新版本: #{updated_agent.version}"

# 列出所有版本
client.beta.agents.versions.list(agent.id).auto_paging_each do |version|
  puts "版本 #{version.version}: #{version.updated_at.iso8601}"
end

# 归档代理
archived = client.beta.agents.archive(agent.id)
puts "归档时间: #{archived.archived_at.iso8601}"
```

---

## 发送用户消息

```ruby
client.beta.sessions.events.send_(
  session.id,
  events: [{
    type: "user.message",
    content: [{type: "text", text: "查看认证模块"}]
  }]
)
```

> 提示：**优先流式处理**：在发送消息之前（或同时）打开流。流仅传递在其打开之后发生的事件——先发送后打开流会导致早期事件以批处理方式缓冲到达。请参阅[引导模式](../../shared/managed-agents-events.md#steering-patterns)。

---

## 流式接收事件（SSE）

```ruby
# 先打开流，再发送用户消息
stream = client.beta.sessions.events.stream_events(session.id)

client.beta.sessions.events.send_(
  session.id,
  events: [{
    type: "user.message",
    content: [{type: "text", text: "总结仓库的README文件"}]
  }]
)
stream.each do |event|
  case event.type
  in :"agent.message"
    event.content.each { |block| print block.text }
  in :"agent.tool_use"
    puts "\n[正在使用工具：#{event.name}]"
  in :"session.status_idle"
    break
  in :"session.error"
    puts "\n[错误：#{event.error&.message || "未知"}]"
    break
  else
    # 忽略其他事件类型
  end
end
```

> 注意：事件的 `.type` 是一个 Symbol（注意与 `:"agent.message"` 区分，而不是 `"agent.message"`）。

### 重新连接与尾随

在会话中途重新连接时，应先列出历史事件以去重，然后再尾随实时事件：

```ruby
require "set"

stream = client.beta.sessions.events.stream_events(session.id)

# 流已打开并处于缓冲状态。在尾随实时事件之前先列出历史记录。
seen_event_ids = Set.new
client.beta.sessions.events.list(session.id).auto_paging_each { |past| seen_event_ids << past.id }

# 尾随实时事件，跳过已见过的事件
stream.each do |event|
  next if seen_event_ids.include?(event.id)
  seen_event_ids << event.id
  case event.type
  in :"agent.message"
    event.content.each { |block| print block.text }
  in :"session.status_idle"
    break
  else
    # 忽略其他事件类型
  end
end
```

---

## 提供自定义工具结果

> 注意：用于 `user.custom_tool_result` 的 Ruby managed-agents 绑定尚未在本技能或应用示例源码中记录。请参考 `shared/managed-agents-events.md` 了解数据格式，并参阅 `anthropic` Ruby gem 仓库获取对应的参数说明。

---

## 轮询事件

```ruby
client.beta.sessions.events.list(session.id).auto_paging_each do |event|
  puts "#{event.type}: #{event.id}"
end
```

---

## 上传文件

```ruby
require "pathname"

file = client.beta.files.upload(file: Pathname("data.csv"))
puts "文件 ID：#{file.id}"

# 在会话中挂载文件
session = client.beta.sessions.create(
  agent: agent.id,
  environment_id: environment.id,
  resources: [
    {
      type: "file",
      file_id: file.id,
      mount_path: "/workspace/data.csv"
    }
  ]
)
```

### 在现有会话中添加和管理资源

```ruby
# 向已打开的会话附加额外的文件
resource = client.beta.sessions.resources.add(
  session.id,
  type: "file",
  file_id: file.id
)
puts resource.id # "sesrsc_01ABC..."

# 列出会话中的资源
listed = client.beta.sessions.resources.list(session.id)
listed.data.each { |entry| puts "#{entry.id} #{entry.type}" }

# 卸载资源
client.beta.sessions.resources.delete(resource.id, session_id: session.id)
```

---

## 列出并下载会话文件

```ruby
files = client.beta.files.list(scope_id: "sesn_abc123", betas: ["managed-agents-2026-04-01"])
content = client.beta.files.download(files.data[0].id)
File.binwrite("output.txt", content.read)
```

---

## 会话管理

```ruby
# 列出环境
environments = client.beta.environments.list

# 获取特定环境
env = client.beta.environments.retrieve(environment.id)

# 归档环境（只读，已有会话继续运行）
client.beta.environments.archive(environment.id)

# 删除环境（仅当没有会话引用该环境时）
client.beta.environments.delete(environment.id)

# 删除会话
client.beta.sessions.delete(session.id)
```

---

## MCP 服务器集成

```ruby
# 代理声明 MCP 服务器（此处未进行认证——认证信息存放在保险库中）
agent = client.beta.agents.create(
  name: "GitHub Assistant",
  model: :"claude-opus-5-5",
  mcp_servers: [
    {
      type: "url",
      name: "github",
      url: "https://api.githubcopilot.com/mcp/"
    }
  ],
  tools: [
    {type: "agent_toolset_20260401"},
    {type: "mcp_toolset", mcp_server_name: "github"}
  ]
)

# 会话挂载包含这些 MCP 服务器 URL 凭证的保险库
session = client.beta.sessions.create(
  agent: {type: "agent", id: agent.id, version: agent.version},
  environment_id: environment.id,
  vault_ids: [vault.id]
)
```

有关创建保险库及添加凭证，请参阅 `shared/managed-agents-tools.md` 中的“保险库”部分。

---

## 保险库
```ruby
# 创建一个保险库
vault = client.beta.vaults.create(
  display_name: "Alice",
  metadata: {external_user_id: "usr_abc123"}
)
puts vault.id # "vlt_01ABC..."

# 添加一个 OAuth 凭证
credential = client.beta.vaults.credentials.create(
  vault.id,
  display_name: "Alice 的 Slack",
  auth: {
    type: "mcp_oauth",
    mcp_server_url: "https://mcp.slack.com/mcp",
    access_token: "xoxp-...",
    expires_at: "2026-04-15T00:00:00Z",
    refresh: {
      token_endpoint: "https://slack.com/api/oauth.v2.access",
      client_id: "1234567890.0987654321",
      scope: "channels:read chat:write",
      refresh_token: "xoxe-1-...",
      token_endpoint_auth: {
        type: "client_secret_post",
        client_secret: "abc123..."
      }
    }
  }
)

# 轮换凭证（例如在刷新令牌后）
client.beta.vaults.credentials.update(
  credential.id,
  vault_id: vault.id,
  auth: {
    type: "mcp_oauth",
    access_token: "xoxp-new-...",
    expires_at: "2026-05-15T00:00:00Z",
    refresh: {refresh_token: "xoxe-1-new-..."}
  }
)

# 归档一个保险库
client.beta.vaults.archive(vault.id)
```

---

## GitHub 仓库集成

将 GitHub 仓库挂载为会话资源（保险库中保存 GitHub MCP 凭证）：

```ruby
session = client.beta.sessions.create(
  agent: agent.id,
  environment_id: environment.id,
  vault_ids: [vault.id],
  resources: [
    {
      type: "github_repository",
      url: "https://github.com/org/repo",
      mount_path: "/workspace/repo",
      authorization_token: "ghp_your_github_token"
    }
  ]
)
```

在同一会话中使用多个仓库：

```ruby
resources = [
  {
    type: "github_repository",
    url: "https://github.com/org/frontend",
    mount_path: "/workspace/frontend",
    authorization_token: "ghp_your_github_token"
  },
  {
    type: "github_repository",
    url: "https://github.com/org/backend",
    mount_path: "/workspace/backend",
    authorization_token: "ghp_your_github_token"
  }
]
```

轮换某个仓库的授权令牌：

```ruby
listed = client.beta.sessions.resources.list(session.id)
repo_resource_id = listed.data.first.id

client.beta.sessions.resources.update(
  repo_resource_id,
  session_id: session.id,
  authorization_token: "ghp_your_new_github_token"
)
```
