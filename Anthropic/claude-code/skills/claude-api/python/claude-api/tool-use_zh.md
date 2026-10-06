# 工具使用 - Python

有关概念性概述（工具定义、工具选择、技巧），请参阅 [shared/tool-use-concepts.md](../../shared/tool-use-concepts.md)。

## 工具运行器（推荐）

**测试版：** 工具运行器在 Python SDK 中处于测试阶段。

使用 `@beta_tool` 装饰器将工具定义为带类型的函数，然后将其传递给 `client.beta.messages.tool_runner()`：

```python
import anthropic
from anthropic import beta_tool

client = anthropic.Anthropic()

@beta_tool
def get_weather(location: str, unit: str = "celsius") -> str:
    """获取某地的当前天气。

    Args:
        location: 城市和州，例如旧金山，加利福尼亚州。
        unit: 温度单位，可以是“摄氏”或“华氏”。
    """
    # 在此处实现您的逻辑
    return f"{location} 当前 72°F，晴朗"

# 工具运行器会自动处理代理循环
runner = client.beta.messages.tool_runner(
    model="claude-opus-5-5",
    max_tokens=16000,
    tools=[get_weather],
    messages=[{"role": "user", "content": "巴黎的天气怎么样？"}],
)

# 每次迭代都会生成一条 BetaMessage；当 Claude 完成时，迭代即停止
for message in runner:
    print(message)
```

对于异步用法，请使用 `@beta_async_tool` 和 `async def` 函数。

**工具运行器的主要优势：**

- 无需手动循环——SDK 会自动调用工具并将结果反馈回去
- 通过装饰器实现类型安全的工具输入
- 工具 Schema 会根据函数签名自动生成
- 当 Claude 不再有工具调用时，迭代会自动停止

### 使用工具运行器调用服务器端工具

运行器的 `tools` 列表除了接受装饰过的工具外，还支持原生的服务器端工具定义（如 `web_search_20260209`、`web_fetch_20260209` 以及代码执行）——只需传入对应的字典即可。由于服务器端工具在 Anthropic 的服务器上运行，因此无需实现任何函数。

**注意——截至 `anthropic` 0.116.0 版本，运行器不会自动恢复 `pause_turn` 状态。** 长时间运行的服务器端工具回合可能会以 `stop_reason: "pause_turn"` 结束。运行器只有在客户端工具产生结果后才会继续，因此暂停的回合会导致循环终止，并作为最终消息返回——既无错误提示，也无警告，只是答案被无声截断。与 TypeScript 运行器不同，Python 运行器无法在循环中途恢复：只要没有客户端工具被调用，它就会无条件退出，且 `runner.append_messages(...)` 也无法阻止这一退出行为。要处理 `pause_turn`，可以在迭代过程中同步保存对话历史，然后在暂停的回合之后重新启动运行器：

```python
messages = [{"role": "user", "content": user_input}]

max_restarts = 5  # 限制 `pause_turn` 的重启次数，与 `max_continuations` 的建议一致
restarts = 0
while True:
    runner = client.beta.messages.tool_runner(
        model="claude-opus-5-5",
        max_tokens=16000,
        tools=tools,  # 可以混合使用 `@beta_tool` 函数和服务器端工具定义
        messages=messages,
    )
    last = None
    for message in runner:
        last = message
        # 同步保存对话历史——运行器会自行维护一份副本，但不对外暴露
        messages.append({"role": "assistant", "content": message.content})
        tool_response = runner.generate_tool_call_response()  # 已缓存；工具仍只会运行一次
        if tool_response is not None:
            messages.append(tool_response)
    if last is None or last.stop_reason != "pause_turn":
        break
    restarts += 1
    if restarts > max_restarts:
        raise RuntimeError("放弃：达到最大重启次数后回合仍处于暂停状态")
    # 如果在回合中间暂停：`messages` 已经包含了暂停的助手消息，
    # 因此下一次运行器会从中断处继续
```

或者，您也可以使用下方的手动循环，该方法会显式处理 `pause_turn`。

---

## MCP 工具转换辅助工具

**测试版。** 将 [MCP（模型上下文协议）](https://modelcontextprotocol.io/) 的工具、提示和资源转换为 Anthropic API 兼容的类型，以便与工具运行器配合使用。需要先运行 `pip install anthropic[mcp]`（Python 3.10+）。> **注意：** Claude API 还支持 `mcp_servers` 参数，允许 Claude 直接连接到远程 MCP 服务器。当您需要本地 MCP 服务器、提示词、资源，或对 MCP 连接有更多控制时，请改用这些辅助工具。

### 使用工具运行器的 MCP 工具

```python
from anthropic import AsyncAnthropic
from anthropic.lib.tools.mcp import async_mcp_tool
from mcp import ClientSession
from mcp.client.stdio import stdio_client, StdioServerParameters

client = AsyncAnthropic()

async with stdio_client(StdioServerParameters(command="mcp-server")) as (read, write):
    async with ClientSession(read, write) as mcp_client:
        await mcp_client.initialize()

        tools_result = await mcp_client.list_tools()
        # tool_runner 是同步的——返回的是运行器对象，而非协程
        runner = client.beta.messages.tool_runner(
            model="claude-opus-5-5",
            max_tokens=16000,
            messages=[{"role": "user", "content": "使用可用工具"}],
            tools=[async_mcp_tool(t, mcp_client) for t in tools_result.tools],
        )
        async for message in runner:
            print(message)
```

对于同步调用，请使用 `mcp_tool` 而不是 `async_mcp_tool`。

### MCP 提示词

```python
from anthropic.lib.tools.mcp import mcp_message

prompt = await mcp_client.get_prompt(name="my-prompt")
response = await client.beta.messages.create(
    model="claude-opus-5-5",
    max_tokens=16000,
    messages=[mcp_message(m) for m in prompt.messages],
)
```

### 将 MCP 资源作为内容

```python
from anthropic.lib.tools.mcp import mcp_resource_to_content

resource = await mcp_client.read_resource(uri="file:///path/to/doc.txt")
response = await client.beta.messages.create(
    model="claude-opus-5-5",
    max_tokens=16000,
    messages=[{
        "role": "user",
        "content": [
            mcp_resource_to_content(resource),
            {"type": "text", "text": "请总结这份文档"},
        ],
    }],
)
```

### 将 MCP 资源上传为文件

```python
from anthropic.lib.tools.mcp import mcp_resource_to_file

resource = await mcp_client.read_resource(uri="file:///path/to/data.json")
uploaded = await client.beta.files.upload(file=mcp_resource_to_file(resource))
```

如果无法转换 MCP 值（例如，不支持的内容类型如音频，或不支持的 MIME 类型），转换函数会抛出 `UnsupportedMCPValueError` 异常。

---

## 手动代理循环

建议优先使用上述工具运行器。仅当需要访问运行器未提供的控制功能时（例如自定义传输、SDK 无法构建的请求格式，或避免依赖处于测试阶段的功能——而运行器本身仍处于测试阶段）才退回到手动循环。“人机协作”审批并不一定需要使用手动循环——您可以在工具函数内部设置审批机制（返回“用户拒绝”的结果），或者在 `for message in runner:` 循环体中检查待处理的 `tool_use` 块，并调用 `runner.set_messages_params()`。

如果您确实需要使用手动循环：

```python
import anthropic

client = anthropic.Anthropic()
tools = [...]  # 您的工具定义
messages = [{"role": "user", "content": user_input}]

# 自主代理循环：持续运行，直到 Claude 停止调用工具
while True:
    response = client.messages.create(
        model="claude-opus-5-5",
        max_tokens=16000,
        tools=tools,
        messages=messages
    )

    # 如果 Claude 已完成（不再调用工具），则退出循环
    if response.stop_reason == "end_turn":
        break

    # 服务端工具调用次数已达上限；重新发送消息以继续
    if response.stop_reason == "pause_turn":
        messages = [
            {"role": "user", "content": user_input},
            {"role": "assistant", "content": response.content},
        ]
        continue

    # 从响应中提取工具调用块
    tool_use_blocks = [b for b in response.content if b.type == "tool_use"]

    # 将助手的响应（包括工具调用块）追加到消息列表中
    messages.append({"role": "assistant", "content": response.content})

    # 执行每个工具并收集结果
    tool_results = []
    for tool in tool_use_blocks:
        result = execute_tool(tool.name, tool.input)  # 您的实现
        tool_results.append({
            "type": "tool_result",
            "tool_use_id": tool.id,  # 必须与工具调用块的 ID 匹配
            "content": result
        })

    # 将工具结果作为用户消息追加到消息列表中
    messages.append({"role": "user", "content": tool_results})

# 最终响应文本
final_text = next(b.text for b in response.content if b.type == "text")
```

---

## 处理工具结果

```python
response = client.messages.create(
    model="claude-opus-5-5",
    max_tokens=16000,
    tools=tools,
    messages=[{"role": "user", "content": "巴黎的天气如何？"}]
)

for block in response.content:
    if block.type == "tool_use":
        tool_name = block.name
        tool_input = block.input
        tool_use_id = block.id

        result = execute_tool(tool_name, tool_input)

        followup = client.messages.create(
            model="claude-opus-5-5",
            max_tokens=16000,
            tools=tools,
            messages=[
                {"role": "user", "content": "巴黎的天气如何？"},
                {"role": "assistant", "content": response.content},
                {
                    "role": "user",
                    "content": [{
                        "type": "tool_result",
                        "tool_use_id": tool_use_id,
                        "content": result
                    }]
                }
            ]
        )
```

---

## 多次工具调用

```python
tool_results = []

for block in response.content:
    if block.type == "tool_use":
        result = execute_tool(block.name, block.input)
        tool_results.append({
            "type": "tool_result",
            "tool_use_id": block.id,
            "content": result
        })

# 一次性发送所有结果
if tool_results:
    followup = client.messages.create(
        model="claude-opus-5-5",
        max_tokens=16000,
        tools=tools,
        messages=[
            *previous_messages,
            {"role": "assistant", "content": response.content},
            {"role": "user", "content": tool_results}
        ]
    )
```

---

## 工具结果中的错误处理

```python
tool_result = {
    "type": "tool_result",
    "tool_use_id": tool_use_id,
    "content": "错误：未找到位置 'xyz'。请提供有效城市名称。",
    "is_error": True
}
```

---

## 工具选择

`tool_choice` 默认为 `{"type": "auto"}`。在 Claude Opus 5.5、Claude Sonnet 5.5、Claude Fable 5.1 和 Claude Mythos 5.1 上，强制调用（`{"type": "any"}` 或 `{"type": "tool", "name": ...}`）会返回 400 错误；而 Claude Opus 5、Claude Sonnet 5 及更早版本则接受这种设置。建议通过提示词来引导模型，并通过 `strict: true` 来确保 schema 的有效性：

```python
response = client.messages.create(
    model="claude-opus-5-5",
    max_tokens=16000,
    tools=[{**tool, "strict": True} for tool in tools],  # schema 必须设置 additionalProperties: false
    messages=[{"role": "user", "content": "巴黎的天气如何？请使用 get_weather 工具。"}]
)
# auto 不保证一定会调用工具——需检查是否有 tool_use 块，若无则重新提示
```

---

## 代码执行

### 基本用法

```python
import anthropic

client = anthropic.Anthropic()

response = client.messages.create(
    model="claude-opus-5-5",
    max_tokens=16000,
    messages=[{
        "role": "user",
        "content": "计算 [1, 2, 3, 4, 5, 6, 7, 8, 9, 10] 的均值和标准差"
    }],
    tools=[{
        "type": "code_execution_20260120",
        "name": "code_execution"
    }]
)

for block in response.content:
    if block.type == "text":
        print(block.text)
    elif block.type == "bash_code_execution_tool_result":
        print(f"stdout: {block.content.stdout}")
```

### 上传文件进行分析

```python
# 1. 上传文件
uploaded = client.beta.files.upload(file=open("sales_data.csv", "rb"))

# 2. 通过 container_upload 块将文件传递给代码执行
response = client.messages.create(
    model="claude-opus-5-5",
    max_tokens=16000,
    messages=[{
        "role": "user",
        "content": [
            {"type": "text", "text": "分析这份销售数据，展示趋势并生成可视化图表。"},
            {"type": "container_upload", "file_id": uploaded.id}
        ]
    }],
    tools=[{"type": "code_execution_20260120", "name": "code_execution"}]
)
```

### 获取生成的文件

```python
import os

OUTPUT_DIR = "./claude_outputs"
os.makedirs(OUTPUT_DIR, exist_ok=True)

for block in response.content:
    if block.type == "bash_code_execution_tool_result":
        result = block.content
        if result.type == "bash_code_execution_result" and result.content:
            for file_ref in result.content:
                if file_ref.type == "bash_code_execution_output":
                    metadata = client.beta.files.retrieve_metadata(file_ref.file_id)
                    file_content = client.beta.files.download(file_ref.file_id)
                    # 使用 basename 防止路径遍历；验证结果
                    safe_name = os.path.basename(metadata.filename)
                    if not safe_name or safe_name in (".", ".."):
                        print(f"跳过无效文件名: {metadata.filename}")
                        continue
                    output_path = os.path.join(OUTPUT_DIR, safe_name)
                    file_content.write_to_file(output_path)
                    print(f"已保存: {output_path}")
```

### 容器复用

```python
# 第一次请求：设置环境
response1 = client.messages.create(
    model="claude-opus-5-5",
    max_tokens=16000,
    messages=[{"role": "user", "content": "安装 tabulate 并创建包含示例数据的 data.json"}],
    tools=[{"type": "code_execution_20260120", "name": "code_execution"}]
)

# 从响应中获取容器 ID
container_id = response1.container.id

# 第二次请求：复用同一容器
response2 = client.messages.create(
    container=container_id,
    model="claude-opus-5-5",
    max_tokens=16000,
    messages=[{"role": "user", "content": "读取 data.json 并以格式化表格显示"}],
    tools=[{"type": "code_execution_20260120", "name": "code_execution"}]
)
```

### 响应结构

```python
for block in response.content:
    if block.type == "text":
        print(block.text)  # Claude 的解释
    elif block.type == "server_tool_use":
        print(f"正在运行: {block.name} - {block.input}")  # Claude 正在执行的操作
    elif block.type == "bash_code_execution_tool_result":
        result = block.content
        if result.type == "bash_code_execution_result":
            if result.return_code == 0:
                print(f"输出: {result.stdout}")
            else:
                print(f"错误: {result.stderr}")
        else:
            print(f"工具错误: {result.error_code}")
    elif block.type == "text_editor_code_execution_tool_result":
        print(f"文件操作: {block.content}")
```

---

## 记忆工具

### 基本用法

```python
import anthropic

client = anthropic.Anthropic()

response = client.messages.create(
    model="claude-opus-5-5",
    max_tokens=16000,
    messages=[{"role": "user", "content": "请记住，我偏好的编程语言是 Python。"}],
    tools=[{"type": "memory_20250818", "name": "memory"}],
)
```

### SDK 记忆助手

继承 `BetaAbstractMemoryTool` 类：

```python
from anthropic.lib.tools import BetaAbstractMemoryTool

class MyMemoryTool(BetaAbstractMemoryTool):
    def view(self, command): ...
    def create(self, command): ...
    def str_replace(self, command): ...
    def insert(self, command): ...
    def delete(self, command): ...
    def rename(self, command): ...

memory = MyMemoryTool()

# 与工具运行器一起使用
runner = client.beta.messages.tool_runner(
    model="claude-opus-5-5",
    max_tokens=16000,
    tools=[memory],
    messages=[{"role": "user", "content": "请记住我的偏好"}],
)

for message in runner:
    print(message)
```

如需完整的实现示例，请参阅 WebFetch：

- `https://github.com/anthropics/anthropic-sdk-python/blob/main/examples/memory/basic.py`

---

## 结构化输出

### JSON 输出（推荐使用 Pydantic）

```python
from pydantic import BaseModel
from typing import List
import anthropic

class ContactInfo(BaseModel):
    name: str
    email: str
    plan: str
    interests: List[str]
    demo_requested: bool

client = anthropic.Anthropic()

response = client.messages.parse(
    model="claude-opus-5-5",
    max_tokens=16000,
    messages=[{
        "role": "user",
        "content": "提取：Jane Doe（jane@co.com）希望购买企业版，对 API 和 SDK 感兴趣，并希望预约演示。"
    }],
    output_format=ContactInfo,
)

# response.parsed_output 是一个经过验证的 ContactInfo 实例
contact = response.parsed_output
print(contact.name)           # "Jane Doe"
print(contact.interests)      # ["API", "SDKs"]
```

### 原始 Schema

```python
response = client.messages.create(
    model="claude-opus-5-5",
    max_tokens=16000,
    messages=[{
        "role": "user",
        "content": "提取信息：John Smith（john@example.com）想要企业版套餐。"
    }],
    output_config={
        "format": {
            "type": "json_schema",
            "schema": {
                "type": "object",
                "properties": {
                    "name": {"type": "string"},
                    "email": {"type": "string"},
                    "plan": {"type": "string"},
                    "demo_requested": {"type": "boolean"}
                },
                "required": ["name", "email", "plan", "demo_requested"],
                "additionalProperties": False
            }
        }
    }
)

import json
# output_config.format 确保第一个块是包含有效 JSON 的文本
text = next(b.text for b in response.content if b.type == "text")
data = json.loads(text)
```

### 严格工具使用

```python
response = client.messages.create(
    model="claude-opus-5-5",
    max_tokens=16000,
    messages=[{"role": "user", "content": "为2名乘客预订3月15日飞往东京的航班"}],
    tools=[{
        "name": "book_flight",
        "description": "预订前往目的地的航班",
        "strict": True,
        "input_schema": {
            "type": "object",
            "properties": {
                "destination": {"type": "string"},
                "date": {"type": "string", "format": "date"},
                "passengers": {"type": "integer", "enum": [1, 2, 3, 4, 5, 6, 7, 8]}
            },
            "required": ["destination", "date", "passengers"],
            "additionalProperties": False
        }
    }]
)
```

### 同时使用两者

```python
response = client.messages.create(
    model="claude-opus-5-5",
    max_tokens=16000,
    messages=[{"role": "user", "content": "计划下个月去巴黎旅行"}],
    output_config={
        "format": {
            "type": "json_schema",
            "schema": {
                "type": "object",
                "properties": {
                    "summary": {"type": "string"},
                    "next_steps": {"type": "array", "items": {"type": "string"}}
                },
                "required": ["summary", "next_steps"],
                "additionalProperties": False
            }
        }
    },
    tools=[{
        "name": "search_flights",
        "description": "搜索可用航班",
        "strict": True,
        "input_schema": {
            "type": "object",
            "properties": {
                "destination": {"type": "string"},
                "date": {"type": "string", "format": "date"}
            },
            "required": ["destination", "date"],
            "additionalProperties": False
        }
    }]
)
```