# Claude API - Python

## 安装

```bash
pip install anthropic
```

## 客户端初始化

```python
import anthropic

# 默认方式：从环境变量中获取凭证：
# ANTHROPIC_API_KEY、ANTHROPIC_AUTH_TOKEN，或通过 `ant auth login` 配置的认证信息。
# 本地开发时推荐使用此方式，不要硬编码 API 密钥。
client = anthropic.Anthropic()

# 显式指定 API 密钥（仅在必须注入特定密钥时使用）
client = anthropic.Anthropic(api_key="your-api-key")

# 异步客户端
async_client = anthropic.AsyncAnthropic()
```

---

## 客户端配置

### 每次请求的覆盖设置

使用 `with_options()` 可以在不改变客户端默认配置的情况下，为单次调用覆盖部分设置：

```python
client.with_options(timeout=5.0, max_retries=5).messages.create(
    model="claude-opus-5-5",
    max_tokens=1024,
    messages=[{"role": "user", "content": "Hello"}],
)
```

### 超时设置

默认请求超时时间为 10 分钟。可以传入一个浮点数（单位：秒）或使用 `anthropic.Timeout` 对超时进行更精细的控制。当请求超时时，SDK 会抛出 `anthropic.APITimeoutError` 异常，并根据 `max_retries` 设置进行重试。

```python
client = anthropic.Anthropic(timeout=20.0)
client = anthropic.Anthropic(
    timeout=anthropic.Timeout(60.0, read=5.0, write=10.0, connect=2.0),
)
```

`anthropic` 1.x 基于 [`httpx2`](https://pypi.org/project/httpx2/) 构建，而非 `httpx`。`anthropic.Timeout` 即为 `httpx2.Timeout`；如果您自行导入 HTTP 库，请使用 `import httpx2 as httpx`——如果使用来自 `httpx` 包的对象（如 `httpx.Timeout`、`httpx.Client`、传输层及限流相关类），则会在请求时被拒绝或导致失败。针对原有基于 `httpx` 的代码，我们提供了 [v1 迁移指南](https://github.com/anthropics/anthropic-sdk-python/blob/main/MIGRATION.md)以及 `/claude-api upgrade python` 工具。

### 重试机制

SDK 会对连接错误、408、409、429 以及所有 5xx 状态码的响应自动进行指数退避式的重试（默认重试 2 次）。您可以在客户端级别或通过 `with_options()` 设置 `max_retries`；将 `max_retries` 设为 0 可禁用重试功能。

### 异步性能（aiohttp 后端）

对于高并发的异步工作负载，请安装 `anthropic[aiohttp]`，并传入 `DefaultAioHttpClient`，而不是默认的 httpx2 后端：

```python
from anthropic import AsyncAnthropic, DefaultAioHttpClient

async with AsyncAnthropic(http_client=DefaultAioHttpClient()) as client:
    ...
```

### 自定义 HTTP 客户端（代理、基础 URL）

请使用 `DefaultHttpxClient` 或 `DefaultAsyncHttpxClient`，而不是直接使用原生的 `httpx2.Client`（也绝不要使用来自 `httpx` 包的客户端），以确保 SDK 的默认超时和连接限制得以保留：

```python
from anthropic import Anthropic, DefaultHttpxClient

client = Anthropic(
    base_url="http://my.test.server.example.com:8083",  # 或者通过 ANTHROPIC_BASE_URL 环境变量设置
    http_client=DefaultHttpxClient(proxy="http://my.test.proxy.example.com"),
)
```

### 日志记录

将 `ANTHROPIC_LOG` 设置为 `debug`（或 `info`），即可通过标准的 `logging` 模块启用 SDK 的日志记录功能。---

## 基本消息请求

```python
response = client.messages.create(
    model="claude-opus-5-5",
    max_tokens=16000,
    messages=[
        {"role": "user", "content": "法国的首都是哪里？"}
    ]
)
# response.content 是一个内容块对象列表（TextBlock、ThinkingBlock、ToolUseBlock 等）。在访问 .text 之前，请先检查 .type。
for block in response.content:
    if block.type == "text":
        print(block.text)
```

---

## 系统提示

```python
response = client.messages.create(
    model="claude-opus-5-5",
    max_tokens=16000,
    system="你是一位乐于助人的编程助手。请始终使用 Python 示例。",
    messages=[{"role": "user", "content": "如何读取一个 JSON 文件？"}]
)
```

### 会话中期的系统消息（由模型控制）

对于在对话中途到达的操作员指令（模式切换、注入状态），应将 `{"role": "system", ...}` 追加到 `messages` 中，而不是编辑顶层的 `system`——这样可以保留缓存的前缀，并传递操作员的权限。此类指令必须紧跟在用户消息（或以服务器工具调用结尾的助手消息）之后，且必须是 `messages` 中的最后一项，或者后面紧跟着一个助手回合；不能作为 `messages[0]`。不支持该功能的模型会返回 400 错误（“此模型不支持 role 'system'”）。关于何时使用这种方式而非顶层 `system`，请参阅 `shared/prompt-caching.md`。

```python
response = client.messages.create(
    model=MODEL_ID,  # 必须支持对话中途的系统消息
    max_tokens=16000,
    system=[{"type": "text", "text": STABLE_SYSTEM, "cache_control": {"type": "ephemeral"}}],
    messages=history + [
        {"role": "user", "content": user_message},
        {"role": "system", "content": "已启用简洁模式——请将回复控制在40字以内。"},
    ],
)  # 不需要添加 beta 头部——使用常规的 client.messages.create 即可
```

---

## 视觉（图像）

### Base64 编码

```python
import base64

with open("image.png", "rb") as f:
    image_data = base64.standard_b64encode(f.read()).decode("utf-8")

response = client.messages.create(
    model="claude-opus-5-5",
    max_tokens=16000,
    messages=[{
        "role": "user",
        "content": [
            {
                "type": "image",
                "source": {
                    "type": "base64",
                    "media_type": "image/png",
                    "data": image_data
                }
            },
            {"type": "text", "text": "这张图里有什么？"}
        ]
    }]
)
```

### URL 引用

```python
response = client.messages.create(
    model="claude-opus-5-5",
    max_tokens=16000,
    messages=[{
        "role": "user",
        "content": [
            {
                "type": "image",
                "source": {
                    "type": "url",
                    "url": "https://example.com/image.png"
                }
            },
            {"type": "text", "text": "请描述这张图"}
        ]
    }]
)
```

---

## 提示词缓存

通过缓存大型上下文来降低成本（最高可节省90%）。**缓存采用前缀匹配机制**——只要前缀中任何位置发生字节级变化，其后的所有内容都会失效。有关放置模式、架构指导（冻结系统提示、确定性工具顺序、易变内容的放置位置）以及静默无效化审计清单，请参阅 `shared/prompt-caching.md`。

### 自动缓存（推荐）

使用顶层的 `cache_control` 可自动缓存请求中最后一个可缓存的块，无需为每个内容块单独标注：

```python
response = client.messages.create(
    model="claude-opus-5-5",
    max_tokens=16000,
    cache_control={"type": "ephemeral"},  # 自动缓存最后一个可缓存的块
    system="您是这份大型文档的专家……",
    messages=[{"role": "user", "content": "请总结要点"}]
)
```

### 手动缓存控制

如需更精细的控制，可在特定内容块上添加 `cache_control`：

```python
response = client.messages.create(
    model="claude-opus-5-5",
    max_tokens=16000,
    system=[{
        "type": "text",
        "text": "您是这份大型文档的专家……",
        "cache_control": {"type": "ephemeral"}  # 默认 TTL 为 5 分钟
    }],
    messages=[{"role": "user", "content": "请总结要点"}]
)

# 显式指定 TTL（生存时间）
response = client.messages.create(
    model="claude-opus-5-5",
    max_tokens=16000,
    system=[{
        "type": "text",
        "text": "您是这份大型文档的专家……",
        "cache_control": {"type": "ephemeral", "ttl": "1h"}  # TTL 为 1 小时
    }],
    messages=[{"role": "user", "content": "请总结要点"}]
)
```

### 验证缓存命中情况
```python
print(response.usage.cache_creation_input_tokens)  # 写入缓存的 token 数量（约 1.25 倍成本）
print(response.usage.cache_read_input_tokens)      # 从缓存中读取的 token 数量（约 0.1 倍成本）
print(response.usage.input_tokens)                 # 未缓存的 token 数量（全额成本）
```

如果在重复的、具有相同前缀的请求中，`cache_read_input_tokens` 始终为零，则说明存在无声失效机制——可能是系统提示中使用了 `datetime.now()` 或 UUID，或者 `json.dumps()` 的输出未排序，又或是工具集发生了变化。完整的审计表请参见 `shared/prompt-caching.md`。

---

## 扩展思考

> **Fable 5、Claude Opus 5.5、Claude Opus 5、Opus 4.8、Opus 4.7 和 Sonnet 4.6：** 使用自适应思考模式。`budget_tokens` 在 Fable 5、Claude Opus 5.5、Claude Opus 5、Opus 4.8 和 4.7 中已被移除（若发送则为 400）；在 Opus 4.6 和 Sonnet 4.6 中已弃用。  
> **Claude Opus 5.5：** 思考模式始终开启——可省略 `thinking` 参数（或发送 `{"type": "adaptive"}`，效果等同）；若发送 `{"type": "disabled"}`，无论何种努力程度都会返回 400 错误，设置思考预算也是如此。请改用 `output_config.effort` 来控制深度——该模型的默认值为 `medium`，而 Claude Opus 5 的默认值为 `high`。  
> **Claude Opus 5：** 默认启用思考模式——省略 `thinking` 参数时会自动启用自适应模式（等同于 `{"type": "adaptive"}`），这与 Opus 4.8/4.7 不同，在后者中省略则表示不启用思考。只有在努力程度为 `high` 或更低时才接受 `{"type": "disabled"}`；若与 `xhigh` 或 `max` 搭配，则会返回 400 错误。  
> **较旧的模型：** 请使用 `thinking: {type: "enabled", budget_tokens: N}`（必须小于 `max_tokens`，最小值为 1024）。

```python
# Fable 5 / Claude Opus 5.5 / Claude Opus 5 / Opus 4.8 / 4.7 / 4.6：自适应思考（推荐）
response = client.messages.create(
    model="claude-opus-5-5",
    max_tokens=16000,
    thinking={"type": "adaptive", "display": "summarized"},  # 显示选项：默认省略（即不显示思考内容），适用于 Fable 5/5.1、Mythos 5/5.1、Claude Opus 5.5、Claude Opus 5、Opus 4.8/4.7、Claude Sonnet 5.5 和 Claude Sonnet 5
    output_config={"effort": "high"},  # low | medium | high | xhigh | max
    messages=[{"role": "user", "content": "请逐步解答..."}]
)

# 访问思考和响应内容
for block in response.content:
    if block.type == "thinking":
        print(f"思考过程：{block.thinking}")
    elif block.type == "text":
        print(f"响应内容：{block.text}")
```

---

## 错误处理

```python
import anthropic

try:
    response = client.messages.create(...)
except anthropic.BadRequestError as e:
    print(f"请求错误：{e.message}")
except anthropic.AuthenticationError:
    print("API 密钥无效")
except anthropic.PermissionDeniedError:
    print("API 密钥缺少必要权限")
except anthropic.NotFoundError:
    print("模型或端点不存在")
except anthropic.RateLimitError as e:
    retry_after = int(e.response.headers.get("retry-after", "60"))
    print(f"请求受限，请 {retry_after} 秒后重试。")
except anthropic.APIStatusError as e:
    if e.status_code >= 500:
        print(f"服务器错误（{e.status_code}）。请稍后重试。")
    else:
        print(f"API 错误：{e.message}")
except anthropic.APIConnectionError:
    print("网络错误。请检查您的网络连接。")
```

---

## 响应辅助工具

每个响应对象都公开 `_request_id` 属性（由 `request-id` 头部填充）——向 Anthropic 报告故障时请记录此 ID。尽管名称以下划线开头，但该属性是公开的。

```python
message = client.messages.create(...)
print(message._request_id)       # req_018EeWyXxfu5pfWkrYcMdjWG
print(message.to_json())          # 序列化 Pydantic 模型
print(message.to_dict())          # 转换为普通字典
```

如需访问原始响应头或其他元数据，请使用 `.with_raw_response` 方法：

```python
raw = client.messages.with_raw_response.create(
    model="claude-opus-5-5",
    max_tokens=1024,
    messages=[{"role": "user", "content": "你好"}],
)
print(raw.headers.get("request-id"))
message = raw.parse()  # 这是 messages.create() 本应返回的 Message 对象
```

---

## 多轮对话

API 是无状态的——每次都需要发送完整的对话历史。

```python
class ConversationManager:
    """使用 Claude API 管理多轮对话。"""

    def __init__(self, client: anthropic.Anthropic, model: str, system: str = None):
        self.client = client
        self.model = model
        self.system = system
        self.messages = []

    def send(self, user_message: str, **kwargs) -> str:
        """发送消息并获取回复。"""
        self.messages.append({"role": "user", "content": user_message})

        response = self.client.messages.create(
            model=self.model,
            max_tokens=kwargs.get("max_tokens", 16000),
            system=self.system,
            messages=self.messages,
            **kwargs
        )

        assistant_message = next(
            (b.text for b in response.content if b.type == "text"), ""
        )
        self.messages.append({"role": "assistant", "content": assistant_message})

        return assistant_message

# 使用示例
conversation = ConversationManager(
    client=anthropic.Anthropic(),
    model="claude-opus-5-5",
    system="你是一个乐于助人的助手。"
)

response1 = conversation.send("我叫爱丽丝。")
response2 = conversation.send("我叫什么名字？")  # Claude 记住了“爱丽丝”
```

**规则：**

- 允许连续的同角色消息——API 会将它们合并为一轮对话
- 第一条消息必须是 `user`
- 在支持模型上，允许在对话中途插入 `role: "system"` 消息（无需添加 beta 标头）——参见上文“对话中的系统消息”部分

---

### 压缩（长对话）

> **Beta 版功能，适用于 Fable 5、Claude Opus 5.5、Claude Opus 5、Opus 4.8、Opus 4.7、Opus 4.6 和 Sonnet 4.6。** 当对话接近 20 万上下文窗口时，服务器端会自动对早期上下文进行压缩总结。API 会返回一个 `compaction` 块；在后续请求中必须将其原样传回——请追加整个 `response.content`，而不仅仅是文本内容。

```python
import anthropic

client = anthropic.Anthropic()
messages = []

def chat(user_message: str) -> str:
    messages.append({"role": "user", "content": user_message})

    response = client.beta.messages.create(
        betas=["compact-2026-01-12"],
        model="claude-opus-5-5",
        max_tokens=16000,
        messages=messages,
        context_management={
            "edits": [{"type": "compact_20260112"}]
        }
    )

    # 追加完整内容——必须保留压缩块
    messages.append({"role": "assistant", "content": response.content})

    return next(block.text for block in response.content if block.type == "text")

# 当上下文量较大时，压缩会自动触发
print(chat("帮我构建一个 Python 网页爬虫"))
print(chat("增加对 JavaScript 渲染页面的支持"))
print(chat("现在再加上限速和错误处理"))
```

---

## 停止原因

响应中的 `stop_reason` 字段指明了模型停止生成的原因：

| 值         | 含义                     |
|------------|--------------------------|
| `end_turn` | Claude 自然结束其回复     |
| `max_tokens` | 达到 `max_tokens` 上限——可调高上限或使用流式输出 |
| `stop_sequence` | 遇到自定义停止序列       |
| `tool_use` | Claude 想调用工具——执行该工具后继续 |
| `pause_turn` | 模型暂停，可恢复（代理流程） |
| `refusal`  | Claude 因安全原因拒绝——请查看 `stop_details` |

### 结构化停止详情

当 `stop_reason` 为 `"refusal"` 时，响应中会包含一个 `stop_details` 对象，其中提供了关于拒绝的结构化信息：
```python
if response.stop_reason == "refusal" and response.stop_details:
    print(f"类别: {response.stop_details.category}")   # 例如 "cyber"、"bio"、"reasoning_extraction"、"frontier_llm"，或 None - 完整列表请参阅文档
    print(f"解释: {response.stop_details.explanation}")
```

### 拒绝回退（Claude Fable 5.1）- 默认启用

回退功能是**可选的**：如果不启用回退，被拒绝的请求将直接停止。在 `claude-fable-5-1` 中，默认包含服务端的 `fallbacks` 参数；当模型拒绝时，API 会在同一调用中使用回退模型重新处理该请求。中途拒绝仍按正常费率计费，而救援调用则按回退模型的费率计费，并自动应用缓存重定价；如果在任何输出产生之前就发生拒绝，请参阅[拒绝的计费方式](https://platform.claude.com/docs/en/build-with-claude/refusals-and-fallback#how-refusals-are-billed)。

```python
response = client.beta.messages.create(
    model="claude-fable-5-1",
    max_tokens=16000,
    betas=["server-side-fallback-2026-06-01"],
    fallbacks=[{"model": "claude-opus-4-8"}],
    messages=[{"role": "user", "content": "..."}],
)

# 切换点：每运行并拒绝一次的模型都会有一个回退块
for block in response.content:
    if block.type == "fallback":
        print(f"{block.from_.model} 拒绝；{block.to.model} 继续")

# 服务来源信号——适用于持续回退的情况，此时不会出现回退块。
# 可与 stop_reason 配合使用：回退模型本身也可能拒绝。
fallback_ran = any(
    entry.type == "fallback_message" for entry in response.usage.iterations or []
)
if fallback_ran and response.stop_reason != "refusal":
    print(f"由 {response.model} 提供服务")
```

最终响应中的 `stop_reason: "refusal"` 表示整个链条都被拒绝了。对于这种数组形式，标头必须精确为 `server-side-fallback-2026-06-01`；较新的标量形式 `fallbacks: "default"` 则使用 `server-side-fallback-2026-07-01`（详见 `shared/model-migration.md` -> 迁移到 Claude Opus 5 -> 新 API 功能），并将两种形式的标头混用会导致 400 错误。该参数在 Batches API 上被拒绝，在 Amazon Bedrock、Vertex AI 和 Microsoft Foundry 上不可用——可在这些平台上通过注册客户端侧的 `BetaRefusalFallbackMiddleware` 来实现。完整语义（持续路由、计费、流式传输、回退回合的回显）请参见 `shared/model-migration.md` -> 迁移到 Claude Fable 5.1 -> `refusal` 停止原因。

---

## 成本优化策略

### 1. 对重复上下文使用提示缓存

```python
# 自动缓存（最简单——仅缓存最后一个可缓存块）
response = client.messages.create(
    model="claude-opus-5-5",
    max_tokens=16000,
    cache_control={"type": "ephemeral"},
    system=large_document_text,  # 例如 50KB 的上下文
    messages=[{"role": "user", "content": "总结要点"}]
)

# 第一次请求：全额收费
# 后续请求：缓存部分约便宜 90%
```

### 2. 选择合适的模型

```python
# 大多数任务默认使用 Opus
response = client.messages.create(
    model="claude-opus-5-5",  # $4.00/$20.00 每 100 万 tokens
    max_tokens=16000,
    messages=[{"role": "user", "content": "解释量子计算"}]
)

# 高吞吐量生产工作负载使用 Sonnet
standard_response = client.messages.create(
    model="claude-sonnet-5-5",  # $2.00/$10.00 每 100 万 tokens
    max_tokens=16000,
    messages=[{"role": "user", "content": "总结本文档"}]
)

# 简单且对速度要求高的任务才使用 Haiku
simple_response = client.messages.create(
    model="claude-haiku-4-5",  # $1.00/$5.00 每 100 万 tokens
    max_tokens=256,
    messages=[{"role": "user", "content": "判断这条评论是正面还是负面"}]
)
```

### 3. 在请求前进行 token 计数

```python
count_response = client.messages.count_tokens(
    model="claude-opus-5-5",
    messages=messages,
    system=system
)

estimated_input_cost = count_response.input_tokens * 0.000004  # $4/100万 tokens
print(f"预计输入成本：${estimated_input_cost:.4f}")
```

---

## 使用指数退避重试

> **注意：** Anthropic SDK 会自动对速率限制（429）和服务器错误（5xx）进行指数退避重试。您可以通过 `max_retries` 参数（默认值为 2）来配置此行为。只有在需要超出 SDK 提供的默认行为时，才应实现自定义重试逻辑。

```python
import time
import random
import anthropic

def call_with_retry(
    client: anthropic.Anthropic,
    max_retries: int = 5,
    base_delay: float = 1.0,
    max_delay: float = 60.0,
    **kwargs
):
    """使用指数退避重试调用 API。"""
    last_exception = None

    for attempt in range(max_retries):
        try:
            return client.messages.create(**kwargs)
        except anthropic.RateLimitError as e:
            last_exception = e
        except anthropic.APIStatusError as e:
            if e.status_code >= 500:
                last_exception = e
            else:
                raise  # 客户端错误（4xx，但不包括 429）不应重试

        delay = min(base_delay * (2 ** attempt) + random.uniform(0, 1), max_delay)
        print(f"第 {attempt + 1}/{max_retries} 次重试，等待 {delay:.1f} 秒")
        time.sleep(delay)

    raise last_exception
```
