# 托管代理 - Python

> **此处未列出的绑定：** 本 README 涵盖了 Python 中最常见的托管代理流程。如果您需要未在此处展示的类、方法、命名空间、字段或行为，请通过 WebFetch 从 `shared/live-sources.md` 获取 Python SDK 仓库**或相关文档页面**，而不要自行猜测。请勿根据 cURL 的请求格式或其他语言的 SDK 进行推断。

> **代理是持久化的——只需创建一次，并通过 ID 引用。** 请保存 `agents.create` 返回的代理 ID，并在后续每次调用 `sessions.create` 时传递该 ID；切勿在请求路径中重复调用 `agents.create`。**建议：** 将代理和环境定义为受版本控制的文件，并通过 `ant apply` 同步——参见 `shared/anthropic-cli.md`（其实时文档 URL 在 `shared/live-sources.md` 中）。CLI 负责控制平面（创建/更新）；您的代码负责数据平面（使用已存储 ID 的会话）。以下示例展示了在必须以编程方式进行资源配置时的代码内创建；但在生产环境中，创建调用应放在初始化阶段，而非请求路径中。

## 安装

```bash
pip install anthropic
```

## 客户端初始化

```python
import anthropic

# 默认方式——从环境变量中解析凭据：
# ANTHROPIC_API_KEY、ANTHROPIC_AUTH_TOKEN，或通过 `ant auth login` 登录的配置文件。
# 推荐在本地开发时使用此方式，不要硬编码 API 密钥。
client = anthropic.Anthropic()

# 显式指定 API 密钥（仅在必须注入特定密钥时使用）
client = anthropic.Anthropic(api_key="your-api-key")
```

---

## 创建环境

```python
environment = client.beta.environments.create(
    name="my-dev-env",
    config={
        "type": "cloud",
        "networking": {"type": "unrestricted"},
    },
)
print(environment.id)  # env_...
```

---

## 创建代理（必需的第一步）

> 注意：**不支持内联代理配置。** `model`、`system` 和 `tools` 都属于代理对象，而非会话。始终先调用 `agents.create()`——会话仅接受 `agent={"type": "agent", "id": agent.id}`。

### 最简配置

```python
# 1. 创建代理（可复用、带版本）
agent = client.beta.agents.create(
    name="编程助手",
    model="claude-opus-5-5",
    tools=[{"type": "agent_toolset_20260401", "default_config": {"enabled": True}}],
)

# 2. 开始会话
session = client.beta.sessions.create(
    agent={"type": "agent", "id": agent.id, "version": agent.version},
    environment_id=environment.id,
)
print(session.id, session.status)
print(f"追踪链接：https://platform.claude.com/workspaces/default/sessions/{session.id}")  # 如果 API 密钥不在默认工作区，请将 'default' 替换为您的工作区 ID
```

### 带系统提示和自定义工具

```python
import os

agent = client.beta.agents.create(
    name="代码审查员",
    model="claude-opus-5-5",
    system="您是一位资深的代码审查员。",
    tools=[
        {"type": "agent_toolset_20260401"},
        {
            "type": "custom",
            "name": "run_tests",
            "description": "运行测试套件",
            "input_schema": {
                "type": "object",
                "properties": {
                    "test_path": {"type": "string", "description": "测试文件路径"}
                },
                "required": ["test_path"],
            },
        },
    ],
)

session = client.beta.sessions.create(
    agent={"type": "agent", "id": agent.id, "version": agent.version},
    environment_id=environment.id,
    title="代码审查会话",
    resources=[
        {
            "type": "github_repository",
            "url": "https://github.com/owner/repo",
            "mount_path": "/workspace/repo",
            "authorization_token": os.environ["GITHUB_TOKEN"],
            "branch": "main",
        }
    ],
)
```

---

## 发送用户消息

```python
client.beta.sessions.events.send(
    session_id=session.id,
    events=[
        {
            "type": "user.message",
            "content": [{"type": "text", "text": "请审查认证模块"}],
        }
    ],
)
```

> 提示：**先开启流**：在发送消息之前（或同时）打开流。流仅传递在其打开之后发生的事件——如果在发送消息后再打开流，早期的事件会以一个批次的形式缓冲后到达。请参阅[引导模式](../../shared/managed-agents-events.md#steering-patterns)。

---

## 定义成果（交付物的默认启动方式）

当会话的任务是产出可检验的成果——如文档、报告或拉取请求时，请使用 `user.define_outcome` 而不是 `user.message` 来启动：框架会根据您的评分标准对每次迭代进行打分，代理会持续修改直至通过。二者选其一，切勿同时使用。有关事件参考及评分标准编写指南，请参阅[成果](../../shared/managed-agents-outcomes.md)。

```python
STARTER_RUBRIC = """# 报告评分标准 - 初版，可根据需要调整各项指标
- 输出为 /mnt/session/outputs/ 目录下的单个 report.md 文件
- 每条论断均标注来源网址
- 包含一张汇总表，每行对应一家竞争对手
- 价格为运行当日的最新价格，且每行注明数据来源
- 不得留有任何占位文本、待办事项或空缺部分
"""

client.beta.sessions.events.send(
    session_id=session.id,
    events=[
        {
            "type": "user.define_outcome",
            "description": "撰写一份竞争对手定价报告，保存为 report.md",
            "rubric": {"type": "text", "content": STARTER_RUBRIC},
            "max_iterations": 5,  # 可选；默认3次，最大20次
        }
    ],
)
```

---

## 流式事件（SSE）

```python
import json

# 先开启流：先打开流，然后在流保持活动期间发送事件
with client.beta.sessions.events.stream(
    session_id=session.id,
) as stream:
    client.beta.sessions.events.send(
        session_id=session.id,
        events=[{"type": "user.message", "content": [{"type": "text", "text": "..."}]}],
    )
    for event in stream:
        ...  # 处理事件

# 独立的流式迭代：
with client.beta.sessions.events.stream(
    session_id=session.id,
) as stream:
    for event in stream:
        if event.type == "agent.message":
            for block in event.content:
                if block.type == "text":
                    print(block.text, end="", flush=True)
        elif event.type == "agent.custom_tool_use":
            # 自定义工具调用——此时会话处于空闲状态
            print(f"\n自定义工具调用：{event.name}")
            print(f"输入：{json.dumps(event.input)}")
            # 发送结果回传（见下文）
        elif event.type == "session.status_idle":
            print("\n--- 代理处于空闲状态 ---")
        elif event.type == "session.status_terminated":
            print("\n--- 会话已终止 ---")
            break
```

---

## 提供自定义工具结果

```python
client.beta.sessions.events.send(
    session_id=session.id,
    events=[
        {
            "type": "user.custom_tool_result",
            "custom_tool_use_id": "sevt_abc123",
            "content": [{"type": "text", "text": "所有42个测试均已通过。"}],
        }
    ],
)
```

---

## 轮询事件

```python
events = client.beta.sessions.events.list(
    session_id=session.id,
)
for event in events.data:
    print(f"{event.type}: {event.id}")
```

> 警告：**优先使用 SDK，而非直接使用 requests 或 httpx。** 如果您自行编写轮询循环，请不要假设 `timeout=(5, 60)` 或 `httpx.Timeout(120)` 会限制整个调用的总时长——它们都是按**每个数据块**设置的读取超时（每接收一个字节就会重置），因此即使响应缓慢地逐字节返回，也可能导致程序永远阻塞。若需要明确的绝对时间截止，应在循环层级使用 `time.monotonic()` 进行计时并主动退出，或使用 `asyncio.wait_for()` 进行包装。详情请参阅[接收事件](../../shared/managed-agents-events.md#receiving-events)。

---

## 包含自定义工具的完整流式处理循环

```python
import json


def run_custom_tool(tool_name: str, tool_input: dict) -> str:
    """执行自定义工具并返回结果"""
    if tool_name == "run_tests":
        # 在此处实现您的工具逻辑
        return "所有测试均已通过。"
    return f"未知工具：{tool_name}"


def run_session(client, session_id: str):
    """流式接收事件并处理自定义工具调用"""
    while True:
        with client.beta.sessions.events.stream(
            session_id=session_id,
        ) as stream:
            tool_calls = []
            for event in stream:
                if event.type == "agent.message":
                    for block in event.content:
                        if block.type == "text":
                            print(block.text, end="", flush=True)
                elif event.type == "agent.custom_tool_use":
                    tool_calls.append(event)
                elif event.type == "session.status_idle":
                    break
                elif event.type == "session.status_terminated":
                    return

        if not tool_calls:
            break

        # 处理自定义工具调用
        results = []
        for call in tool_calls:
            result = run_custom_tool(call.name, call.input)
            results.append({
                "type": "user.custom_tool_result",
                "custom_tool_use_id": call.id,
                "content": [{"type": "text", "text": result}],
            })

        client.beta.sessions.events.send(
            session_id=session_id,
            events=results,
        )
```

---

## 上传文件

```python
with open("data.csv", "rb") as f:
    file = client.beta.files.upload(
        file=f,
    )

# 在会话中使用该文件
session = client.beta.sessions.create(
    agent={"type": "agent", "id": agent.id, "version": agent.version},
    environment_id=environment.id,
    resources=[{"type": "file", "file_id": file.id, "mount_path": "/workspace/data.csv"}],
)
```

---

## 列出并下载会话文件

列出会话期间代理写入 `/mnt/session/outputs/` 目录的文件，并将其下载。

```python
# 列出与会话关联的文件
files = client.beta.files.list(
    scope_id=session.id,
    betas=["managed-agents-2026-04-01"],
)
for f in files.data:
    print(f.filename, f.size_bytes)
    # 下载每个文件并保存到本地
    file_content = client.beta.files.download(f.id)
    file_content.write_to_file(f.filename)
```

> 提示：在 `session.status_idle` 发生后，输出文件出现在 `files.list` 中可能会有短暂的索引延迟（约1–3秒）。如果列表为空，可重试一两次。

---

## 会话管理

```python
# 获取会话详情
session = client.beta.sessions.retrieve(session_id="sesn_011CZxAbc123Def456")
print(session.status, session.usage)

# 列出会话
sessions = client.beta.sessions.list()

# 删除会话
client.beta.sessions.delete(session_id="sesn_011CZxAbc123Def456")

# 归档会话
client.beta.sessions.archive(session_id="sesn_011CZxAbc123Def456")
```

---

## MCP 服务器集成

```python
# 代理声明 MCP 服务器（此处未进行认证——认证信息存放在 Vault 中）
agent = client.beta.agents.create(
    name="MCP Agent",
    model="claude-opus-5-5",
    mcp_servers=[
        {"type": "url", "name": "my-tools", "url": "https://my-mcp-server.example.com/sse"},
    ],
    tools=[
        {"type": "agent_toolset_20260401", "default_config": {"enabled": True}},
        {"type": "mcp_toolset", "mcp_server_name": "my-tools"},
    ],
)

# 会话附加包含这些 MCP 服务器 URL 凭证的 Vault
session = client.beta.sessions.create(
    agent=agent.id,
    environment_id=environment.id,
    vault_ids=[vault.id],
)
```

有关创建 Vault 并添加凭证的详细信息，请参阅 `shared/managed-agents-tools.md` 的“Vaults”章节。