# Claude API - cURL / 原始 HTTP

当用户需要原始 HTTP 请求，或使用没有官方 SDK 的编程语言时，可参考这些示例。

## 设置

```bash
export ANTHROPIC_API_KEY="您的API密钥"
```

---

## 基本消息请求

```bash
curl https://api.anthropic.com/v1/messages \
  -H "Content-Type: application/json" \
  -H "x-api-key: $ANTHROPIC_API_KEY" \
  -H "anthropic-version: 2023-06-01" \
  -d '{
    "model": "claude-opus-5-5",
    "max_tokens": 16000,
    "messages": [
      {"role": "user", "content": "法国的首都是哪里？"}
    ]
  }'
```

### 解析响应

使用 `jq` 从 JSON 响应中提取字段。请勿使用 `grep` 或 `sed`——因为 JSON 字符串可能包含任意字符，而正则表达式解析在遇到引号、转义字符或多行内容时会失效。

```bash
# 捕获响应并提取字段
response=$(curl -s https://api.anthropic.com/v1/messages \
  -H "Content-Type: application/json" \
  -H "x-api-key: $ANTHROPIC_API_KEY" \
  -H "anthropic-version: 2023-06-01" \
  -d '{"model":"claude-opus-5-5","max_tokens":16000,"messages":[{"role":"user","content":"你好"}]}')

# 打印第一个文本块（-r 参数会去除 JSON 引号）
echo "$response" | jq -r '.content[0].text'

# 读取用量字段
input_tokens=$(echo "$response" | jq -r '.usage.input_tokens')
output_tokens=$(echo "$response" | jq -r '.usage.output_tokens')

# 读取停止原因（用于工具调用循环）
stop_reason=$(echo "$response" | jq -r '.stop_reason')

# 提取所有文本块（content 是一个数组；筛选 type=="text"）
echo "$response" | jq -r '.content[] | select(.type == "text") | .text'
```


---

## 流式传输（SSE）

```bash
curl https://api.anthropic.com/v1/messages \
  -H "Content-Type: application/json" \
  -H "x-api-key: $ANTHROPIC_API_KEY" \
  -H "anthropic-version: 2023-06-01" \
  -d '{
    "model": "claude-opus-5-5",
    "max_tokens": 64000,
    "stream": true,
    "messages": [{"role": "user", "content": "写一首俳句"}]
  }'
```

响应是一个服务器发送事件（SSE）流：

```
event: message_start
data: {"type":"message_start","message":{"id":"msg_...","type":"message",...}}

event: content_block_start
data: {"type":"content_block_start","index":0,"content_block":{"type":"text","text":""}}

event: content_block_delta
data: {"type":"content_block_delta","index":0,"delta":{"type":"text_delta","text":"你好"}}

event: content_block_stop
data: {"type":"content_block_stop","index":0}

event: message_delta
data: {"type":"message_delta","delta":{"stop_reason":"end_turn"},"usage":{"output_tokens":12}}

event: message_stop
data: {"type":"message_stop"}
```

---

## 工具调用

```bash
curl https://api.anthropic.com/v1/messages \
  -H "Content-Type: application/json" \
  -H "x-api-key: $ANTHROPIC_API_KEY" \
  -H "anthropic-version: 2023-06-01" \
  -d '{
    "model": "claude-opus-5-5",
    "max_tokens": 16000,
    "tools": [{
      "name": "get_weather",
      "description": "获取某地当前天气",
      "input_schema": {
        "type": "object",
        "properties": {
          "location": {"type": "string", "description": "城市名称"}
        },
        "required": ["location"]
      }
    }],
    "messages": [{"role": "user", "content": "巴黎的天气如何？"}]
  }'
```

当 Claude 返回 `tool_use` 块时，请将结果重新发送：
```bash
curl https://api.anthropic.com/v1/messages \
  -H "Content-Type: application/json" \
  -H "x-api-key: $ANTHROPIC_API_KEY" \
  -H "anthropic-version: 2023-06-01" \
  -d '{
    "model": "claude-opus-5-5",
    "max_tokens": 16000,
    "tools": [{
      "name": "get_weather",
      "description": "获取某地当前天气",
      "input_schema": {
        "type": "object",
        "properties": {
          "location": {"type": "string", "description": "城市名称"}
        },
        "required": ["location"]
      }
    }],
    "messages": [
      {"role": "user", "content": "巴黎的天气如何？"},
      {"role": "assistant", "content": [
        {"type": "text", "text": "让我查一下天气。"},
        {"type": "tool_use", "id": "toolu_abc123", "name": "get_weather", "input": {"location": "巴黎"}}
      ]},
      {"role": "user", "content": [
        {"type": "tool_result", "tool_use_id": "toolu_abc123", "content": "72°F，晴天"}
      ]}
    ]
  }'
```

---

## 提示缓存

将 `cache_control` 放在稳定前缀的最后一块上。有关放置模式和静默失效器审计清单，请参阅 `shared/prompt-caching.md`。

```bash
curl https://api.anthropic.com/v1/messages \
  -H "Content-Type: application/json" \
  -H "x-api-key: $ANTHROPIC_API_KEY" \
  -H "anthropic-version: 2023-06-01" \
  -d '{
    "model": "claude-opus-5-5",
    "max_tokens": 16000,
    "system": [
      {"type": "text", "text": "<大型共享提示...>", "cache_control": {"type": "ephemeral"}}
    ],
    "messages": [{"role": "user", "content": "请总结要点"}]
  }'
```

对于 1 小时 TTL：`"cache_control": {"type": "ephemeral", "ttl": "1h"}`。请求体中的顶级 `"cache_control"` 会自动放置在最后一个可缓存块上。可通过响应中的 `usage.cache_creation_input_tokens` 和 `usage.cache_read_input_tokens` 字段验证缓存命中情况。

---

## 扩展思考

> **Fable 5、Claude Opus 5.5、Claude Opus 5、Opus 4.8、Opus 4.7、Opus 4.6 和 Sonnet 4.6：** 使用自适应思考。`budget_tokens` 在 Fable 5、Claude Opus 5.5、Claude Opus 5、Opus 4.8 和 4.7 中已被移除（如果发送则为 400）；在 Opus 4.6 和 Sonnet 4.6 中已弃用。  
> **Claude Opus 5.5：** 思考始终开启——省略 `thinking`（或发送 `{"type": "adaptive"}`，效果相同）；`{"type": "disabled"}` 在任何努力级别都会返回 400 错误，思考预算也是如此。改用 `output_config.effort` 控制深度——该模型的默认值为 `medium`，而 Claude Opus 5 的默认值为 `high`。  
> **Claude Opus 5：** 默认开启思考——省略 `"thinking"` 即为自适应（等同于 `{"type": "adaptive"}`），这与 Opus 4.8/4.7 不同，在后者中省略意味着不进行思考。`{"type": "disabled"}` 只能在 `high` 或更低的努力级别下接受；若与 `xhigh`/`max` 搭配，则会返回 400 错误。  
> **较旧的模型：** 使用 `"type": "enabled"` 并设置 `"budget_tokens": N`（必须小于 `max_tokens`，最低 1024）。

```bash
# Fable 5 / Claude Opus 5.5 / Claude Opus 5 / Opus 4.8 / 4.7 / 4.6：自适应思考（推荐）
curl https://api.anthropic.com/v1/messages \
  -H "Content-Type: application/json" \
  -H "x-api-key: $ANTHROPIC_API_KEY" \
  -H "anthropic-version: 2023-06-01" \
  -d '{
    "model": "claude-opus-5-5",
    "max_tokens": 16000,
    "thinking": {
      "type": "adaptive",
      "display": "summarized"
    },
    "output_config": {
      "effort": "high"
    },
    "messages": [{"role": "user", "content": "请一步一步解决这个问题..."}]
  }'
```

---

## 拒绝回退（Claude Fable 5.1）——默认启用在 `claude-fable-5-1` 上，安全分类器可能会拒绝请求（HTTP 200 状态码，且 `stop_reason: "refusal"`）。回退机制是**需主动启用**的：若未启用，请求将直接停止。请默认包含 `fallbacks` 参数及其对应的测试版标头；当策略判定拒绝时，API 会在同一调用中使用回退模型重新执行该请求。中途被拒绝的部分按正常费率计费，而后续由回退模型处理的部分则按其自身的费率计费；若在任何输出产生之前即被拒绝，请参阅[拒绝如何计费](https://platform.claude.com/docs/en/build-with-claude/refusals-and-fallback#how-refusals-are-billed)。

```bash
response=$(curl -s https://api.anthropic.com/v1/messages \
  -H "Content-Type: application/json" \
  -H "x-api-key: $ANTHROPIC_API_KEY" \
  -H "anthropic-version: 2023-06-01" \
  -H "anthropic-beta: server-side-fallback-2026-06-01" \
  -d '{
    "model": "claude-fable-5-1",
    "max_tokens": 16000,
    "fallbacks": [{"model": "claude-opus-4-8"}],
    "messages": [{"role": "user", "content": "Hello"}]
  }')

# 输出生成消息的模型
echo "$response" | jq -r '.model'

# 最终响应中的拒绝标志表示整个流程都被拒绝
echo "$response" | jq -r '.stop_reason'

# 切换点：每运行一次并被拒绝的模型都会对应一个回退块
echo "$response" | jq -r '.content[] | select(.type == "fallback") | "\(.from.model) 被拒绝；\(.to.model) 继续"'

# 服务来源信号——适用于连续多轮的情况，这类情况不会产生回退块。
# 可与 stop_reason 配合使用：回退模型本身也可能拒绝。
if [ "$(echo "$response" | jq -r '.stop_reason')" != "refusal" ] && \
   echo "$response" | jq -e '[.usage.iterations[]? | select(.type == "fallback_message")] | length > 0' > /dev/null; then
  echo "本轮由回退模型提供服务"
fi
```

对于这种数组形式的参数，标头必须精确为 `server-side-fallback-2026-06-01`；较新的标量形式 `fallbacks: "default"` 则应使用 `server-side-fallback-2026-07-01`（详见 `shared/model-migration.md` -> 迁移到 Claude Opus 5 -> 新 API 功能），并将任一标头与另一种形式搭配使用都会返回 400 错误。该参数在 Batches API 中被拒绝，在 Amazon Bedrock、Vertex AI 和 Microsoft Foundry 中不可用。完整语义（包括粘性路由、计费、流式传输以及回退轮次的回显）请参见 `shared/model-migration.md` -> 迁移到 Claude Fable 5.1 -> `refusal` 停止原因。

---

## 必填标头

| 标头              | 值              | 说明                |
| ------------------- | ------------------ | -------------------------- |
| `Content-Type`      | `application/json` | 必填                   |
| `x-api-key`         | 您的 API 密钥       | 用于身份验证             |
| `anthropic-version` | `2023-06-01`       | API 版本                |
| `anthropic-beta`    | 测试功能 ID       | 使用测试功能时必填     |
