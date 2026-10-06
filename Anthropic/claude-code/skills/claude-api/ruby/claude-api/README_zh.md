# Claude API - Ruby

> **注意：** Ruby SDK 支持 Claude API。工具运行器目前以 Beta 版本通过 `client.beta.messages.tool_runner()` 提供。Agent SDK 尚未提供 Ruby 版本。

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

---

## 基本消息请求

```ruby
message = client.messages.create(
  model: :"claude-opus-5-5",
  max_tokens: 16000,
  messages: [
    { role: "user", content: "法国的首都是哪里？" }
  ]
)
# content 是一个由多态块对象组成的数组（TextBlock、ThinkingBlock、ToolUseBlock 等）。.type 是一个 Symbol——与 :text 比较，而不是 "text"。
# .text 在非 TextBlock 类型的条目上调用时会抛出 NoMethodError 异常。
message.content.each do |block|
  puts block.text if block.type == :text
end
```

---

## 扩展思考功能

> **Fable 5、Claude Opus 5.5、Claude Opus 5、Opus 4.8、Opus 4.7、Opus 4.6 和 Sonnet 4.6：** 使用自适应思考模式。在 Fable 5、Claude Opus 5.5、Claude Opus 5、Opus 4.8 和 4.7 上，`budget_tokens` 参数已被移除（若传入则为 400）；在 Opus 4.6 和 Sonnet 4.6 上已弃用。  
> **Claude Opus 5.5：** 思考功能始终开启——可省略 `thinking` 参数（或传入 `{ type: "adaptive" }`，效果相同）；传入 `{ type: "disabled" }` 无论何种努力程度都会返回 400 错误，设置思考预算也是如此。请改用 `output_config.effort` 来控制思考深度——该模型的默认值为 `medium`，而 Claude Opus 5 的默认值为 `high`。  
> **Claude Opus 5：** 思考功能默认开启——省略 `thinking:` 即表示启用自适应模式（等同于传入 `{ type: "adaptive" }`），这与 Opus 4.8/4.7 不同，在后者中省略则意味着不进行思考。只有在努力程度为 `high` 或更低时才允许传入 `{ type: "disabled" }`；若与 `xhigh` 或 `max` 搭配，则会返回 400 错误。  
> **较旧的模型：** 使用 `thinking: { type: "enabled", budget_tokens: N }`（N 必须小于 `max_tokens`，且最小为 1024）。

```ruby
message = client.messages.create(
  model: :"claude-opus-5-5",
  max_tokens: 16000,
  thinking: { type: "adaptive" },
  messages: [{ role: "user", content: "计算：27 * 453" }]
)

message.content.each do |block|
  case block.type
  when :thinking then puts "思考：#{block.thinking}"
  when :text then puts "响应：#{block.text}"
  end
end
```

---

## 提示缓存

`system_:`（末尾带下划线——避免与 `Kernel#system` 冲突）接受一个文本块数组；在最后一个块上设置 `cache_control`。普通哈希可通过 `OrHash` 类型别名使用。有关放置模式和静默失效器审计清单，请参阅 `shared/prompt-caching.md`。

```ruby
message = client.messages.create(
  model: :"claude-opus-5-5",
  max_tokens: 16000,
  system_: [
    { type: "text", text: long_system_prompt, cache_control: { type: "ephemeral" } }
  ],
  messages: [{ role: "user", content: "总结要点" }]
)
```

若需 1 小时的 TTL：`cache_control: { type: "ephemeral", ttl: "1h" }`。此外，`messages.create` 还提供一个顶级的 `cache_control:` 参数，可自动应用于最后一个可缓存的块。

可通过 `message.usage.cache_creation_input_tokens` 和 `message.usage.cache_read_input_tokens` 来验证缓存命中情况。

---

## 停止详情

当 `stop_reason` 为 `:refusal` 时，响应中会包含结构化的 `stop_details`：

```ruby
if message.stop_reason == :refusal && message.stop_details
  puts "类别：#{message.stop_details.category}"     # 例如：:cyber、:bio、:reasoning_extraction、:frontier_llm，或 nil——完整列表请参阅文档
  puts "解释：#{message.stop_details.explanation}"
end
```

**拒绝回退（Claude Fable 5.1）——默认启用。** 回退功能为可选：若不启用，被拒绝的请求将直接停止。新的 `claude-fable-5-1` 代码应默认包含服务端的 `fallbacks` 参数（使用 beta 头部 `server-side-fallback-2026-06-01`，并在 beta 版的消息调用中设置 `fallbacks: [{model: "claude-opus-4-8"}]`）。具体的 Ruby 绑定（以及针对无服务端支持的提供商的客户端中间件）未在此处说明——请通过 WebFetch 从 Ruby SDK 仓库的 `shared/live-sources.md` 中获取 `examples/` 目录；完整语义参见 `shared/model-migration.md` 中的“迁移到 Claude Fable 5.1”章节下的“拒绝”停止原因部分。

---

## Beta 功能

`betas:` 仅在 `client.beta.messages.create` 接口中有效，非 Beta 路径则不适用。

### 任务预算

```ruby
response = client.beta.messages.create(
  model: :"claude-opus-5-5",
  max_tokens: 16000,
  output_config: { task_budget: { type: :tokens, total: 64_000 } },
  tools: [...],
  messages: [...],
  betas: ["task-budgets-2026-03-13"]
)
```

---

## 错误类型

`APIStatusError` 提供了一个 `.type` 字段，用于程序化地对错误进行分类：

```ruby
begin
  client.messages.create(...)
rescue Anthropic::Errors::APIStatusError => e
  puts e.type  # :rate_limit_error、:overloaded_error 等
end
```