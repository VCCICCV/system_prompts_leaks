# 流式传输 - Python

## 快速入门

```python
with client.messages.stream(
    model="claude-opus-5-5",
    max_tokens=64000,
    messages=[{"role": "user", "content": "写一个故事"}]
) as stream:
    for text in stream.text_stream:
        print(text, end="", flush=True)
```

### 异步

```python
async with async_client.messages.stream(
    model="claude-opus-5-5",
    max_tokens=64000,
    messages=[{"role": "user", "content": "写一个故事"}]
) as stream:
    async for text in stream.text_stream:
        print(text, end="", flush=True)
```

### 低层级：`stream=True`

（上述的）`messages.stream()` 是推荐使用的辅助方法，它会累积状态并提供 `text_stream` 和 `get_final_message()` 接口。如果你只需要原始事件迭代器，并希望降低内存占用，可以改为在调用 `messages.create()` 时传入 `stream=True`：

```python
for event in client.messages.create(
    model="claude-opus-5-5",
    max_tokens=64000,
    messages=[{"role": "user", "content": "写一个故事"}],
    stream=True,
):
    print(event.type)
```

在这种方式下，系统不会为你自动累积最终消息。

---

## 处理不同内容类型

Claude 可能返回文本、思考块或工具调用。请分别进行适当处理：

> **Fable 5 / Claude Opus 5.5 / Claude Opus 5 / Opus 4.8 / Opus 4.7 / Opus 4.6：** 使用 `thinking: {type: "adaptive"}`。在 Claude Opus 5.5 和 Claude Opus 5 上，省略 `thinking` 参数也会得到自适应模式（Claude Opus 5.5 不接受其他设置——`disabled` 和 `budget_tokens` 均为 400）。在较旧的模型上，请使用 `thinking: {type: "enabled", budget_tokens: N}`。

```python
with client.messages.stream(
    model="claude-opus-5-5",
    max_tokens=64000,
    thinking={"type": "adaptive", "display": "summarized"},  # 显示选项：默认情况下，在 Fable 5/5.1、Mythos 5/5.1、Claude Opus 5.5、Claude Opus 5、Opus 4.8/4.7、Claude Sonnet 5.5 和 Claude Sonnet 5 上，思考内容会被省略（即显示为空）
    messages=[{"role": "user", "content": "分析这个问题"}]
) as stream:
    for event in stream:
        if event.type == "content_block_start":
            if event.content_block.type == "thinking":
                print("\n[思考中...]")
            elif event.content_block.type == "text":
                print("\n[回复:]")

        elif event.type == "content_block_delta":
            if event.delta.type == "thinking_delta":
                print(event.delta.thinking, end="", flush=True)
            elif event.delta.type == "text_delta":
                print(event.delta.text, end="", flush=True)
```

---

## 带有工具调用的流式传输Python 工具运行器支持流式处理：向 `client.beta.messages.tool_runner(...)` 传入 `stream=True`，每次迭代都会返回一个流，您可以逐事件地消费该流，并通过 `get_final_message()` 获取每轮累积的消息（参见 `shared/tool-use-concepts.md` 中的“工具运行器与手动循环”部分）。使用 `@beta_tool(eager_input_streaming=True)` 来声明工具运行器的工具，使其输入在生成时即开始流式传输（默认规则参见 `shared/tool-use-concepts.md` 中的“输入预取流式传输”部分）。运行器绝不会对无法解析的输入调用您的函数；`ValueError` 异常会在您迭代每轮流的过程中抛出，因此请将 `for ... in runner` 循环包裹起来，一旦发生错误，就从您在迭代过程中同步维护的历史记录中重新启动一个新的运行器。对于每个产出的流，请按以下顺序处理：首先获取 `message = stream.get_final_message()`，将其追加到对话历史中（作为助手的一轮），然后检查其 `stop_reason`——如果因 `max_tokens` 而终止且存在 `tool_use`，或者因 `refusal` 终止，则立即停止，且不再为该轮调用 `generate_tool_call_response()`（该方法用于执行工具）；只有在本轮继续的情况下，才追加 `runner.generate_tool_call_response()`（对应的用户工具结果轮次），这与 `tool-use.md` 中恢复 `pause_turn` 的方式完全一致。Python 运行器不暴露任何可读的 `params` 参数，已消费的运行器不可再次迭代，而缺少已延续轮次中工具结果部分的历史记录会被 API 拒绝。`pause_turn` 需由您自行恢复；最终的答案文本即为最后一轮的消息。

仅当您不使用工具运行器，且需要在使用工具时进行逐 token 流式处理时，才应采用下面的手动循环模式。为每个用户定义的工具设置 `eager_input_streaming: True`。启用预取流式传输后，服务器将不再验证输入：Python SDK 的宽容解析器会对截断的输入返回部分对象（检查 `stop_reason == "max_tokens"`），并对格式错误的 JSON 默默截断并返回（在运行工具前务必验证解析后的输入）；只有完全无法解析的 JSON 才会**从流式迭代器中**抛出 `ValueError`，因此该保护措施应包裹在流式处理逻辑中，而非包裹在最终消息的读取操作上。模式验证并非路径验证：模型提供的 `path` 是不可信的输出，因此在写入文件前务必将该路径限制在项目根目录内（参见 `shared/tool-use-concepts.md` 中的文本编辑器安全提示）：

```python
import json
from pathlib import Path

ROOT = Path.cwd().resolve()

tools = [
    {
        "name": "write_file",
        "description": "将文本写入指定路径的文件",
        "eager_input_streaming": True,  # 大量输入随生成即时流式传输
        "input_schema": {
            "type": "object",
            "properties": {
                "path": {"type": "string"},
                "contents": {"type": "string"},
            },
            "required": ["path", "contents"],
        },
    }
]

messages = [{"role": "user", "content": task}]
json_retries = 0

while True:
    try:
        with client.messages.stream(
            model="claude-opus-5-5",
            max_tokens=64000,
            tools=tools,
            messages=messages,
        ) as stream:
            for event in stream:
                if event.type == "text":
                    print(event.text, end="", flush=True)
                elif event.type == "input_json":
                    # 工具输入片段——在预取流式传输下会立即到达
                    print(event.partial_json, end="", flush=True)
            response = stream.get_final_message()
        json_retries = 0  # 上限针对单轮内的连续失败次数
    except ValueError:
        # SDK 完全无法解析的 JSON。异常在工具调用块完成之前抛出，
        # 因此没有可供回复的 tool_use_id；需重新发出该轮请求（有限次数）。
        # API 错误不属于 ValueError，会直接向外抛出。
        json_retries += 1
        if json_retries > 2:
            raise
        continue

    # 服务端工具达到迭代上限：追加本轮对话并重新发送
    if response.stop_reason == "pause_turn":
        messages.append({"role": "assistant", "content": response.content})
        continue

    tool_uses = [b for b in response.content if b.type == "tool_use"]
    if response.stop_reason == "refusal" 或者没有 tool_uses：
        # 结束本轮，得到纯文本答案，或因拒绝而终止（可能在工具输入中途被切断）：
        # 此时无需执行任何工具
        break
    if response.stop_reason == "max_tokens":
        # 截断的工具输入会被解析为有效的部分对象，不应执行。
        raise RuntimeError("工具输入被截断；请提高 max_tokens 后重试")

    # SDK 的容错解析器可能会返回被静默截断或拼写错误的输入（例如在未转义的内部引号处），因此需要先进行验证。
    tool_results = []
    for block in tool_uses:
        args = block.input
        if not (isinstance(args, dict) and isinstance(args.get("path"), str)
                and isinstance(args.get("contents"), str)):
            tool_results.append({"type": "tool_result", "tool_use_id": block.id, "is_error": True,
                                 "content": json.dumps({"INVALID_JSON": json.dumps(args)})})
            continue
        # `path` 是不可信的模型输出：在写入之前，需解析该路径并拒绝任何试图逃出项目根目录的行为（如“..”、绝对路径、符号链接）——仅靠模式验证无法检查这一点。
        target = (ROOT / args["path"]).resolve()
        if not target.is_relative_to(ROOT):
            tool_results.append({"type": "tool_result", "tool_use_id": block.id, "is_error": True,
                                 "content": "路径逃出了项目根目录"})
            continue
        tool_results.append({"type": "tool_result", "tool_use_id": block.id,
                             "content": run_tool(block.name, {**args, "path": str(target)})})
    messages.append({"role": "assistant", "content": response.content})
    messages.append({"role": "user", "content": tool_results})
```

---

## 获取最终消息

```python
with client.messages.stream(
    model="claude-opus-5-5",
    max_tokens=64000,
    messages=[{"role": "user", "content": "你好"}]
) as stream:
    for text in stream.text_stream:
        print(text, end="", flush=True)

    # 流式传输结束后获取完整消息
    final_message = stream.get_final_message()
    print(f"\n\n使用的 token 数量：{final_message.usage.output_tokens}")
```

---

## 带进度更新的流式传输

```python
def stream_with_progress(client, **kwargs):
    """带进度更新的流式响应函数。"""
    total_tokens = 0
    content_parts = []

    with client.messages.stream(**kwargs) as stream:
        for event in stream:
            if event.type == "content_block_delta":
                if event.delta.type == "text_delta":
                    text = event.delta.text
                    content_parts.append(text)
                    print(text, end="", flush=True)

            elif event.type == "message_delta":
                if event.usage and event.usage.output_tokens is not None:
                    total_tokens = event.usage.output_tokens

        final_message = stream.get_final_message()

    print(f"\n\n[使用的 token 数量：{total_tokens}]")
    return "".join(content_parts)
```

---

## 流式传输中的错误处理

```python
try:
    with client.messages.stream(
        model="claude-opus-5-5",
        max_tokens=64000,
        messages=[{"role": "user", "content": "写一个故事"}]
    ) as stream:
        for text in stream.text_stream:
            print(text, end="", flush=True)
except anthropic.APIConnectionError:
    print("\n连接中断，请重试。")
except anthropic.RateLimitError:
    print("\n已达到速率限制，请稍后再试。")
except anthropic.APIStatusError as e:
    print(f"\nAPI 错误：{e.status_code}")
```

---

## 流事件类型

| 事件类型            | 描述                 | 触发时机                     |
| --------------------- | --------------------------- | --------------------------------- |
| `message_start`       | 包含消息元数据   | 开始时触发一次             |
| `content_block_start` | 新内容块开始     | 每个文本/工具调用块开始时触发 |
| `content_block_delta` | 内容增量更新     | 每个 token/片段时触发      |
| `content_block_stop`  | 内容块结束       | 每个块结束时触发           |
| `message_delta`       | 消息级更新       | 包含停止原因及用量信息     |
| `message_stop`        | 消息完成         | 结束时触发一次             |

## 最佳实践

1. **始终刷新输出** - 使用 `flush=True` 可立即显示 token。
2. **处理部分响应** - 如果流中断，可能会得到不完整的响应内容。
3. **跟踪 token 使用情况** - `message_delta` 事件包含用量信息。
4. **设置超时** - 根据应用需求设置合适的超时时间。
5. **默认使用流式传输** - 即使在流式传输中，也可通过 `.get_final_message()` 获取完整响应，从而获得超时保护，而无需单独处理每个事件。
6. **非流式传输且 `max_tokens` 过大时会引发 `ValueError`** - SDK 会拒绝预计超过约 10 分钟的非流式请求（空闲连接会被断开）。可通过传递 `stream=True` 或使用 `messages.stream()` 来规避此限制，或者显式覆盖 `timeout` 参数。