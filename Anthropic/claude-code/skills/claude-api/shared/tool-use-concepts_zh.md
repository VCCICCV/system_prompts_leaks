# 工具使用概念

本文件介绍了使用 Claude API 进行工具调用的概念基础。有关特定语言的代码示例，请参阅 `python/`、`typescript/` 或其他语言目录。关于应暴露哪些工具、如何管理长时间运行的代理中的上下文以及缓存策略的决策启发式方法，请参阅 `agent-design.md`。

## 用户自定义工具

### 工具定义结构

> **注意：** 使用工具运行器（测试版）时，工具 Schema 会根据您的函数签名（Python）、Zod Schema（TypeScript）、带注解的类（Java）、`jsonschema` 结构体标签（Go）或 `BaseTool` 子类（Ruby）自动生成。下面的原始 JSON Schema 格式适用于手动方式——包括 PHP 的 `BetaRunnableTool`，它将一个手写的 Schema 包装在一个运行闭包中——或者适用于不支持工具运行器的 SDK。

每个工具都需要一个名称、描述以及其输入的 JSON Schema：

```json
{
  "name": "get_weather",
  "description": "获取某地当前天气",
  "input_schema": {
    "type": "object",
    "properties": {
      "location": {
        "type": "string",
        "description": "城市和州，例如：旧金山，加利福尼亚州"
      },
      "unit": {
        "type": "string",
        "enum": ["celsius", "fahrenheit"],
        "description": "温度单位"
      }
    },
    "required": ["location"]
  }
}
```

**工具定义的最佳实践：**

- 使用清晰、描述性的名称（例如：`get_weather`、`search_database`、`send_email`）
- 编写详细的描述——Claude 会根据这些描述决定何时调用该工具。请明确说明*何时*调用该工具，而不仅仅是它的功能（例如：“当用户询问当前价格或近期事件时调用此工具”）。在最近的 Opus 模型上，由于它们更保守地选择调用工具，因此在描述中加入触发条件可以显著提高调用概率。
- 为每个属性添加描述
- 对于具有固定取值集合的参数，使用 `enum`
- 在 `required` 中标记真正必需的参数；其他参数则设为可选，并提供默认值

---

### 立即输入流式传输（客户端工具流式请求的默认设置）

默认情况下，API 会在发出任何与工具输入相关的 `input_json_delta` 之前，**缓冲并验证每个工具输入参数**。对于一个很小的 `{"location": "Paris"}` 请求，这几乎不会被察觉；但对于需要接收文件内容、代码块或文档的工具来说，在整个参数生成完毕之前，流式传输中不会有任何数据到达（一个包含 2 万 token 的参数会导致流式传输中出现长达数分钟的静默空白）。如果将工具的 `eager_input_streaming: true` 设置为开启，则会关闭该工具的缓冲机制：片段会在生成时立即流式传输，第一个片段会立刻到达，且每个片段的长度也会更长。事件类型不变（`content_block_start` -> `input_json_delta` × N -> `content_block_stop`），只是时间和验证保证发生了变化。

**默认规则：** 当请求是流式传输的（如 `client.messages.stream(...)`、`stream=True`，或启用了流式传输的工具运行器）并且定义了用户自定义工具时，应将这些工具的 `eager_input_streaming: true` 设置为开启。不要将其用于非流式请求（会被忽略），也不要用于服务器端工具（如 `web_search`、`code_execution`、`mcp_toolset` 等——这些工具不接受该字段），或者当客户端无法处理无效 JSON 时。

```json
{
  "name": "write_file",
  "description": "将文本写入文件",
  "eager_input_streaming": true,
  "input_schema": {
    "type": "object",
    "properties": {
      "path": {"type": "string"},
      "contents": {"type": "string", "description": "文件的全部内容"}
    },
    "required": ["path", "contents"]
  }
}
```

工具运行器：Python 的 `@beta_tool(eager_input_streaming=True)` 会直接传递该参数。TypeScript 的 `betaZodTool()` 则没有专门的选项——需将该字段扩展到返回的工具对象上：`{ ...betaZodTool({ name, description, inputSchema, run }), eager_input_streaming: true }`。Go/Java/Ruby/C#/PHP：该字段位于工具参数类型中（如 `EagerInputStreaming`、`.eagerInputStreaming(true)`、`eager_input_streaming:`）。

**您需要放弃什么，以及如何处理。** 如果不启用缓冲，API 不会对参数进行验证或强制转换，因此累积的 `partial_json` 可能会（a）在达到 `max_tokens` 时被截断于参数的中途，或者（b）成为模型输出的无效 JSON。两种 SDK 都使用一种“宽容”的部分 JSON 解析器来累积数据，因此格式错误的输入通常会以静默截断的对象形式返回（未转义的内部引号提前结束字符串；尾部的垃圾数据会被丢弃），而不会抛出异常。请勿依赖异常处理。始终做到：

1. **在运行工具之前，先根据工具的 Schema 对解析后的输入进行验证。** 类型化的运行辅助函数会为您完成此操作，绝不会对验证失败的输入调用 `run` 方法：TypeScript 的 `betaZodTool`（基于 Zod）、Python 中带有类型化参数的 `@beta_tool` 装饰器、Java 的注解类、Go 的结构体标签，以及 Ruby 的 `BaseTool` 都会自动进行验证。而原生的 JSON Schema 辅助函数**不会**在运行时进行验证——TypeScript 的 `betaTool()` 和 PHP 的 `BetaRunnableTool` 会直接将解析得到的内容传递给 `run` 方法——因此，在使用这些辅助函数时，请务必在 `run` 方法内部进行验证（或者为该工具关闭 `eager_input_streaming` 模式）。在手动循环中，请自行验证（TypeScript 中使用 `schema.safeParse(block.input)`，Python 中使用 Pydantic 模型或显式类型检查），并将验证失败的情况视为与无效 JSON 相同。对于原始的 SSE 或 cURL 客户端，应严格地对累积的片段执行 `JSON.parse`，然后再进行验证。
2. 当出现 `tool_use` 块时，检查 `stop_reason == "max_tokens"`：被截断的输入通常仍能解析为有效的部分对象，因此这一条件可以捕获此类情况；此时应提高 `max_tokens` 后重试，而不是直接运行工具。此外，当 `stop_reason == "refusal"` 时也应停止处理——拒绝可能会在输入中途中断 `tool_use`，因此切勿执行该轮次的工具调用。
3. 仍然要捕获 SDK 在无法解析完全无效的 JSON 时抛出的异常：**Python** 会在流迭代器中抛出 `ValueError`（需将 `with client.messages.stream(...)` 块包裹起来）；**TypeScript** 会在 `content_block_stop` 时将输入具体化，因此此时抛出的错误会拒绝您正在等待的结果——如果您是逐事件迭代，则为 `for await (const event of stream)` 循环；否则为 `await stream.finalMessage()`——因此需将整个流的消费过程（包括迭代和最终读取）都包裹起来，就像 Python 指导中将整个 `with` 块包裹起来一样；**工具运行器**也会在其迭代过程中抛出同样的错误（需将循环包裹起来）。仅捕获该特定错误——将 SDK 的类型化 API 错误（如 `RateLimitError`、`AuthenticationError` 等）重新抛出，以免身份验证或限流失败被误认为是 JSON 格式错误——并限制重试次数。
4. 当验证失败且您仍持有 `tool_use` 块时（在 `finalMessage()` 或 `get_final_message()` 之后的手动循环中，或在原始 SSE 流中），不要运行工具；而是将原始文本作为错误结果返回给 Claude，以便其能够重试：

```json
{
  "type": "tool_result",
  "tool_use_id": "toolu_01...",
  "is_error": true,
  "content": "{\"INVALID_JSON\": \"<您收到的无法解析的输入>\"}"
}
```

如果 SDK 在块尚未完成时就已抛出异常（Python 流或任一工具运行器），则没有可回复的 `tool_use_id`，此时应重新发出请求。

构建该包装层时请使用 JSON 库（而非字符串拼接），以确保不良输入中的引号被正确转义。在默认的缓冲模式下，服务器本会将同一损坏的参数作为一个单独的字符串值返回；而“即时”模式只是将这一失败转移到客户端，并不会凭空制造它。

**可用性：** Claude API、AWS 上的 Claude Platform、Vertex AI 以及 Microsoft Foundry 支持所有当前模型（详见 `shared/platform-availability.md`）。在 Amazon Bedrock 上，只有较新的服务栈支持该字段（Opus 4.7 / 4.8 / 5、Fable 5、Sonnet 4.6 / 5）；较旧的 Bedrock 部署（Opus 4.5 / 4.6、Sonnet 4.0 / 4.5、Haiku 4.5）遇到未知字段时会返回 400 错误——在这种情况下请将其移除。`shared/platform-availability.md` 是该列表的权威来源。任何位于 API 前面的代理或网关也可能拒绝该字段；如果用户的代码指向自定义的 `base_url`，请将其关闭，除非他们确认上游确实是真正的 API。

---### 工具选择选项

控制 Claude 何时使用工具：

| 值                             | 行为                                      |
| -------------------------------- | ----------------------------------------- |
| `{"type": "auto"}`              | 由 Claude 自行决定是否使用工具（默认）    |
| `{"type": "any"}`               | Claude 必须至少使用一个工具              |
| `{"type": "tool", "name": "..."}` | Claude 必须使用指定的工具                |
| `{"type": "none"}`              | Claude 不得使用任何工具                  |

任何 `tool_choice` 值都可以附加 `"disable_parallel_tool_use": true`，以强制 Claude 在每次响应中至多调用一个工具。默认情况下，Claude 可能在一次响应中请求多次工具调用。

**Claude Fable 5.1、Claude Mythos 5.1、Claude Opus 5.5 和 Claude Sonnet 5.5 不支持强制工具调用：** 使用 `{"type": "any"}` 或 `{"type": "tool", "name": ...}` 时会返回 400 错误（错误信息为：“此模型不支持 tool_choice 类型为 'tool' 和 'any'。”——在 `count_tokens` 和批量请求中也会出现）。这是一项模型特定的限制（Claude Fable 5 和 Claude Opus 5 则支持这些选项）。由于 `auto` 无法保证一定会有工具调用，请检查是否确实发生了调用，若未发生则需重试。建议使用 `{"type": "auto"}`，并在提示中明确说明期望（例如：“请使用 get_weather 工具作答”）；对工具设置 `strict: true` 可以保留 `any` 提供的“参数符合 Schema”的保证；或者在仅需提取 JSON 的场景下使用结构化输出（通过 `output_config.format`）。`auto` 和 `none` 不受此影响；即使与 `auto` 搭配使用 `disable_parallel_tool_use`，也最多只会调用一次工具（而 `any`/`tool` 组合下的“恰好一次”行为已不再存在）。将 `tool_choice` 的 `any` 与 `strict: true` 配合使用时，仅适用于支持强制工具调用的模型。详情请参阅 `shared/model-migration.md` 中关于从 Claude Fable 5 迁移到 Claude Fable 5.1 的章节。

---

### 工具运行器与手动循环

**工具运行器（推荐）：** SDK 内置的工具运行器会自动处理代理式交互循环——它会调用 API、检测工具调用请求、执行您的工具函数、将结果反馈给 Claude，并重复这一过程，直到 Claude 停止发起工具调用。该功能现已在 Python、TypeScript、Java、Go、Ruby、PHP 和 C# SDK 中提供（处于 Beta 阶段）。Python SDK 还提供了 MCP 转换辅助工具（`anthropic.lib.tools.mcp`），可将 MCP 格式的工具、提示和资源转换为与工具运行器兼容的格式，具体请参阅 `python/claude-api/tool-use.md`。对于任何使用自定义工具的代理，**请优先选择工具运行器**。

**工具运行器并非黑盒——“我需要完全控制”通常并不是切换到手动循环的理由。** 每次迭代都会在工具执行前返回助手消息，允许您进行干预，因此大多数“细粒度控制”的需求无需手动编写循环即可满足：

- **人工介入审批/门控**：可在工具的执行函数中设置门控（返回“用户拒绝”结果而不实际执行），或在返回的消息中检查工具调用，并通过 `set_messages_params()` / `setMessagesParams()` / `append_messages()` / `pushMessages()` 来覆盖待处理的请求，在工具执行前决定是否批准。只要您未进行干预，运行器就会自动执行您的函数。
- **错误拦截**：在工具结果返回给 Claude 之前对其进行检查（通过 `generate_tool_call_response()` / `generateToolResponse()`）；您可以提前终止流程或自行处理错误。
- **结果修改**：在工具结果返回前对其进行修改（例如添加用于提示缓存的 `cache_control` 头，或对输出进行转换）。
- **每轮重试/参数调整**：例如，提高 `max_tokens` 并重新执行被截断的对话回合；还可通过 `max_iterations` 对整个循环设置最大迭代次数。
- **流式传输和自动压缩**均受支持。这些钩子是 SDK 的辅助功能，而不是独立的 API 参数——有关确切的方法名称和示例，请在 `shared/live-sources.md` 中列出的各语言 SDK 仓库中通过 WebFetch 查阅“Claude API SDK 仓库”部分（工具运行器相关的辅助函数位于每个仓库的 `tools.md` 或 `helpers.md` 中）。随附的 `python/claude-api/tool-use.md` 和 `typescript/claude-api/tool-use.md` 展示了基本的工具运行器配置。

**不要因为以下误解而改用手动循环：**

- 工具运行器并不依赖 Zod 或 Pydantic——`betaTool()`（TS）和 `@beta_tool`（Python）接受原生 JSON Schema；其他 SDK 则使用普通的结构体/映射/类。
- 运行器让检测最终一轮变得更容易，而不是更难——当 Claude 停止调用工具时，迭代即告结束，最后生成的消息就是最终响应。大多数 SDK 还提供一次性变体（如 `runner.until_done()`、`runner.runUntilDone()` 或 `RunToCompletion()`）。
- 确认/审批机制可以与运行器配合使用（详见下文的安全性部分）。

**手动代理循环：** 只有在您希望完全掌控整个循环时才应采用这种方式——例如，当您需要访问运行器无法提供的控制能力（如自定义传输层、SDK 无法构造的请求格式，或在不支持逐 token 流式输出的 SDK 上实现逐 token 流）、不愿引入 Beta 版依赖，或者您的控制流程不适合运行器的每轮钩子（如在循环中途穿插无关任务）。审批门控、日志记录、拦截、结果修改和条件执行等功能并不需要手动循环——工具运行器已经覆盖了这些场景（见上文）。循环直至 `stop_reason == "end_turn"`，始终将完整的 `response.content` 追加到内容中以保留工具调用块，并确保每个 `tool_result` 都包含对应的 `tool_use_id`。

**服务端工具的停止原因：** 使用服务端工具（代码执行、网络搜索等）时，API 会在服务器端执行采样循环。如果该循环达到默认的 10 次迭代上限，响应的 `stop_reason` 将为 `"pause_turn"`。要继续处理，只需重新发送用户消息和助手响应并发起另一条 API 请求——服务器会从中断处自动恢复。切勿额外添加类似“继续”的用户消息，API 能够识别尾部的 `server_tool_use` 块并自动恢复。

```python
# 在代理循环中处理 pause_turn
if response.stop_reason == "pause_turn":
    messages = [
        {"role": "user", "content": user_query},
        {"role": "assistant", "content": response.content},
    ]
    # 发起另一条 API 请求——服务器会自动恢复
    response = client.messages.create(
        model="claude-opus-5-5", messages=messages, tools=tools
    )
```

**注意：** 截至 `@anthropic-ai/sdk` 0.110.0 和 `anthropic` 0.116.0 版本，SDK 的工具运行器不会自动恢复 `pause_turn`——被暂停的本轮会终止运行器，并作为最终消息返回，且不会抛出错误。在 TypeScript 中，您可以在迭代体内恢复（将已暂停的助手回合重新推入运行器）；而在 Python 中，运行器无法在循环中恢复——您需要在暂停回合的基础上重新启动一个新的运行器，或者在手动循环中处理 `pause_turn`。具体模式请参阅各语言的 `tool-use.md` 文档。

设置一个 `max_continuations` 上限（例如 5），以防止出现无限循环。完整指南请参见：`https://platform.claude.com/docs/en/build-with-claude/handling-stop-reasons`> **安全性：**每当 Claude 请求时，工具运行器都会自动执行您的工具函数。对于具有副作用的工具（如发送电子邮件、修改数据库、金融交易），请对输入进行验证，并将破坏性操作置于人工审批之后。**工具运行器和手动循环都支持此功能**——在工具运行器中，您可以在工具的运行函数内设置闸门（提示用户并返回“用户拒绝”结果而不执行），或者在每个 yielded 消息中检查工具调用，并通过 `set_messages_params()` / `setMessagesParams()` 控制消息历史，在工具运行之前允许或拒绝（只有在您不干预的情况下，它才会自动执行您的函数）；而在手动循环中，您可以在调用函数之前直接进行控制。

---

### 处理工具结果

当 Claude 使用工具时，响应中会包含一个 `tool_use` 块。您必须：

1. 使用提供的输入执行工具
2. 在 `tool_result` 消息中返回结果
3. 继续对话

**工具结果中的错误处理：**当工具执行失败时，请设置 `"is_error": true"` 并提供一条清晰的错误信息。Claude 通常会确认该错误，并尝试其他方法或请求进一步说明。

**多次工具调用：**Claude 可以在一次响应中请求多个工具。在继续之前，请先处理所有调用——将所有结果汇总到一条 `user` 消息中一并返回。

---

## 服务器端工具：代码执行

代码执行工具允许 Claude 在一个安全的沙箱容器中运行代码。与用户自定义工具不同，服务器端工具运行在 Anthropic 的基础设施上——您无需在客户端执行任何操作。只需提供工具定义，其余部分由 Claude 自动处理。

### 关键信息

- 运行于隔离容器中（1 核 CPU、5 GiB 内存、5 GiB 磁盘空间）
- 无网络访问权限（完全沙箱化）
- 预装 Python 3.11 及数据科学相关库
- 容器可保留 30 天，并可在不同请求间重复使用
- 与网页搜索/网页抓取工具一起使用时免费；否则每组织每月前 1,550 小时免费，超出部分按每小时 0.05 美元收费

### 工具定义

该工具无需 schema，只需在 `tools` 数组中声明即可：

```json
{
  "type": "code_execution_20260120",
  "name": "code_execution"
}
```

Claude 会自动获得 `bash_code_execution`（运行 Shell 命令）和 `text_editor_code_execution`（创建/查看/编辑文件）的使用权限。

### 预装 Python 库

- **数据科学**：pandas、numpy、scipy、scikit-learn、statsmodels
- **可视化**：matplotlib、seaborn
- **文件处理**：openpyxl、xlsxwriter、pillow、pypdf、pdfplumber、python-docx、python-pptx
- **数学**：sympy、mpmath
- **实用工具**：tqdm、python-dateutil、pytz、sqlite3

您还可以在运行时通过 `pip install` 安装额外的包。

### 支持上传的文件类型

| 类型   | 扩展名                         |
| ------ | ------------------------------ |
| 数据   | CSV、Excel（.xlsx/.xls）、JSON、XML |
| 图像   | JPEG、PNG、GIF、WebP           |
| 文本   | .txt、.md、.py、.js 等         |

### 容器复用

您可以在不同请求之间复用容器以保持状态（文件、已安装的包、变量等）。从首次响应中提取 `container_id`，并在后续请求中传递。

### 响应结构

响应中包含文本块与工具结果块交替出现：

- `text` —— Claude 的解释
- `server_tool_use` —— Claude 正在执行的操作
- `bash_code_execution_tool_result` —— 代码执行输出（可通过 `return_code` 检查执行是否成功）
- `text_editor_code_execution_tool_result` —— 文件操作结果

> **安全性：**在将下载的文件写入磁盘之前，请务必使用 `os.path.basename()` 或 `path.basename()` 对文件名进行清理，以防止路径遍历攻击。请将文件写入专门的输出目录。

---

## 服务器端工具：网页搜索与网页抓取网络搜索和网页抓取使 Claude 能够在互联网上进行搜索并获取页面内容。这些工具在服务器端运行——您只需提供工具定义，Claude 就会自动处理查询、抓取和结果的后续处理。

### 工具定义

```json
[
  { "type": "web_search_20260209", "name": "web_search" },
  { "type": "web_fetch_20260209", "name": "web_fetch" }
]
```

### 动态过滤（适用于 Claude Opus 5.5 / Claude Opus 5 / Fable 5 / Opus 4.8 / Opus 4.7 / Opus 4.6 / Claude Sonnet 5.5 / Sonnet 5 / Sonnet 4.6）

`web_search_20260209` 和 `web_fetch_20260209` 版本支持**动态过滤**——Claude 会编写并执行代码，在搜索结果进入上下文窗口之前对其进行筛选，从而提升准确性和 token 利用效率。动态过滤功能内置于这些工具版本中，并会自动启用；您无需单独声明 `code_execution` 工具，也无需传递任何 beta 标头。

```json
{
  "tools": [
    { "type": "web_search_20260209", "name": "web_search" },
    { "type": "web_fetch_20260209", "name": "web_fetch" }
  ]
}
```

如果不使用动态过滤，旧版的 `web_search_20250305` 仍然可用。

> **注意：** 只有当您的应用出于自身目的（如数据分析、文件处理、可视化）需要代码执行时，才应单独引入 `code_execution` 工具；若将其与 `_20260209` 系列网络工具同时使用，会创建第二个执行环境，可能导致模型混淆。

---

## 服务器端工具：程序化工具调用

在常规工具使用模式下，每次工具调用都会形成一次往返交互：Claude 发起调用，结果进入其上下文，Claude 进行推理后再调用下一个工具。这种链式调用会累积延迟并消耗大量 token——而其中大部分中间数据实际上不会再被使用。

通过程序化工具调用，Claude 可以将这些调用组合成一段脚本。该脚本在代码执行容器中运行；当脚本调用某个工具时，容器会暂停，工具执行完毕后，结果直接返回到正在运行的代码中（而非进入 Claude 的上下文）。随后，脚本按正常控制流程处理这些结果，最终只有最终输出才会返回给 Claude。当需要串联多个工具调用，或中间结果较大且应在进入上下文窗口前进行筛选时，可采用此方式。如需完整文档，请使用 WebFetch：

- URL：`https://platform.claude.com/docs/en/agents-and-tools/tool-use/programmatic-tool-calling`

---

## 服务器端工具：工具搜索

工具搜索工具使 Claude 能够从大型工具库中动态发现工具，而无需将所有工具定义加载到上下文窗口中。当您拥有大量工具但每次请求仅涉及其中少数几个时，可使用此工具。发现的工具 Schema 会追加到请求中，而不是直接替换——这样可以保留提示缓存（参见 `agent-design.md` 第“代理的缓存”章节）。

如需完整文档，请使用 WebFetch：

- URL：`https://platform.claude.com/docs/en/agents-and-tools/tool-use/tool-search-tool`

---

## 对话中途更改工具（Beta 版）

**Beta 标头：`mid-conversation-tool-changes-2026-07-01`；适用模型：Claude Opus 5、Claude Opus 5.5、Claude Opus 4.8、Claude Fable 5、Claude Fable 5.1、Claude Mythos 5、Claude Mythos 5.1 以及 Claude Sonnet 5.5——不包括 Claude Sonnet 5；Microsoft Foundry 不支持此功能（可用性参见 `shared/platform-availability.md`）。** 通常情况下，`tools` 在一次对话的整个生命周期内是固定的——对其进行编辑会改变提示前缀的最前端部分，并导致整个缓存失效（参见 `prompt-caching.md` 第“失效层级”章节）。此功能允许您在多轮对话之间添加或移除工具，同时保持缓存前缀不变。

这两种操作都是附加在 `messages[]` 中的 `{"role": "system", ...}` 消息上的内容块，并且都通过 `tool_reference` 按名称引用某个工具：

```python
# 移除 - 必须紧挨着一条助手消息之前，或位于 messages 的最后。
{"role": "system", "content": [
    {"type": "tool_removal", "tool": {"type": "tool_reference", "name": "get_weather"}},
]}
```# 工具添加 - 前置声明并启用延迟加载的工具。
{"role": "system", "content": [
    {"type": "tool_addition", "tool": {"type": "tool_reference", "name": "get_forecast"}},
]}
```

**您计划添加的工具必须已在 `tools[]` 中以 `"defer_loading": True` 声明。** 延迟加载的工具在请求中已知，但直到通过 `tool_addition` 显式引入之前，不会被加载到模型的上下文中：

```python
tools = [
    {"name": "get_weather", "description": "获取天气",
     "input_schema": {"type": "object", "properties": {"city": {"type": "string"}}}},
    {"name": "get_forecast", "description": "获取5日预报",
     "input_schema": {"type": "object", "properties": {"city": {"type": "string"}}},
     "defer_loading": True},
]
```

**若要更改工具的定义**，需分两次请求：第一次发送针对旧定义的 `tool_removal`，第二次在后续请求中使用更新后的 `tools[]` 继续对话。

> 警告：早期预览版本使用了不同的 beta 标头和不同的块格式；两者均已弃用。请使用 `mid-conversation-tool-changes-2026-07-01` 并配合 `tool_addition` / `tool_removal` / `tool_reference` 使用。

SDK 类型定义滞后于这些块——在 Python 中可将其作为普通字典传递，或在 TypeScript 中添加 `@ts-expect-error` 注解。

**与工具搜索的对比：** 工具搜索用于*发现*——Claude 会自行从庞大的工具库中找到所需工具。而会话中工具变更则用于*控制*——您的应用明确决定工具集发生了变化（例如模式切换、资源可用性变化或需要撤销某项能力），并予以说明。

---

## 代理技能（Messages API）

代理技能封装了特定任务的指令和文件，Claude 在相关场景下会加载这些内容（例如 Anthropic 预构建的 `pptx`、`xlsx`、`pdf`、`docx` 技能）。在 **Messages API** 上，技能通过 `container` 参数与代码执行工具一同启用——这**不是**托管代理界面，也**不**使用 `client.beta.agents` / `sessions` / `environments`。可用性参见 `shared/platform-availability.md`。

每次请求需满足以下条件：

1. 调用 `client.beta.messages.create(...)` 并启用 `code-execution-2025-08-25` beta 标志（技能功能已脱离 beta，无需 `skills-2025-10-02` 标头）。
2. `container={"skills": [{"type": "anthropic", "skill_id": "<id>", "version": "latest"}]}`——该技能列表决定了执行容器内可用的技能。
3. `tools=[{"type": "code_execution_20260521", "name": "code_execution"}]`——技能通过容器内的代码执行来运行。

```python
response = client.beta.messages.create(
    model="claude-opus-5-5", max_tokens=16000,
    betas=["code-execution-2025-08-25"],
    container={"skills": [{"type": "anthropic", "skill_id": "pptx", "version": "latest"}]},
    tools=[{"type": "code_execution_20260521", "name": "code_execution"}],
    messages=[{"role": "user", "content": "创建一份关于 X 的三页演示文稿"}],
)
```

生成的文件（`.pptx`、`.xlsx` 等）均写入容器内，响应中会为每个文件携带一个文件 ID。可通过将该 ID 传递给 Files API（`client.files.download(file_id)` 或 `GET /v1/files/{id}/content`）进行下载。

可通过 `GET /v1/skills` 列出可用技能（无需 beta 标头）。

---

## MCP 连接器（Beta 版）

MCP 连接器使 Claude 能够直接从 Messages API 调用托管在远程 MCP 服务器上的工具——Anthropic 在服务端负责建立 MCP 连接。调用 `client.beta.messages.create(...)` 时需启用 `mcp-client-2025-11-20` beta 标志。可用性参见 `shared/platform-availability.md`。

**需同时提供两个参数：**

- `mcp_servers`——服务器连接定义数组：`[{"type": "url", "url": "<server URL>", "name": "<server-name>", "authorization_token": "<optional>"}]`
- `tools`——必须包含一个引用服务器名称的 `mcp_toolset` 条目：`[{"type": "mcp_toolset", "mcp_server_name": "<server-name>"}]`

工具集中指定的 `mcp_server_name` 必须与 `mcp_servers` 中的某个 `name` 匹配。缺少 `mcp_toolset` 条目将导致验证错误——`mcp_servers` 中的每个服务器都必须且只能由一个工具集引用。

```python
client.beta.messages.create(
    model="claude-opus-5-5", max_tokens=1024,
    betas=["mcp-client-2025-11-20"],
    mcp_servers=[{"type": "url", "url": "https://example/sse", "name": "example-mcp"}],
    tools=[{"type": "mcp_toolset", "mcp_server_name": "example-mcp"}],
    messages=[...],
)
```

Go 语言使用类型化常量 `anthropic.AnthropicBetaMCPClient2025_11_20`；旧版常量 `...2025_04_04` 已弃用。

工具集的可选字段包括：`default_config`（所有工具的默认配置，例如用于白名单模式的 `{"enabled": false}`）以及 `configs`（按工具名称键控的单个工具覆盖配置）。

---

## 工具使用示例

您可以在工具定义中直接提供示例调用，以展示使用模式并减少参数错误。这有助于 Claude 理解如何正确格式化工具输入，尤其适用于具有复杂架构的工具。

如需完整文档，请访问 WebFetch：
- URL: `https://platform.claude.com/docs/en/agents-and-tools/tool-use/implement-tool-use`

---

## 客户端工具：计算机使用

计算机使用使 Claude 能够与桌面环境交互（截屏、鼠标、键盘操作）。这是一种客户端工具——由您的应用程序提供运行环境并执行 Claude 请求的操作；Anthropic 会实时处理截屏和操作请求，但不托管运行环境，也不保留数据。

**两种请求形式。** 当前形式为 **计算机工具集**——在 Claude API 和 Google Cloud 上已正式发布，无需 beta 标头：`tools` 中仅有一个条目 `{"type": "computer_toolset_20260801"}`，**无 `name`** 且无显示尺寸，并可选地通过 `configs` 地图关闭部分成员工具（例如 `{"zoom": {"enabled": false}}`；默认情况下所有 17 个成员工具均开启，包括 `zoom`）。Claude 的调用为 `tool_use` 块，其 `name` 即为对应成员工具名（`screenshot`、`left_click`、`type`、`zoom` 等），并带有 `"toolset_name": "computer"` 标识，通常每轮会有多个调用；下一轮用户消息中需返回与之对应的 `tool_result`，**每个结果均需回显 `"toolset_name": "computer"`**（仅 `screenshot` 和 `zoom` 需附带图像，其余只需返回 `OK`）。坐标基于您返回的完整截屏像素空间，即使经过缩放后亦然，且截屏必须符合模型的图像限制。较早的 `computer_20251124` 工具（beta 版本 `computer-use-2025-11-24`，包含 `name: "computer"` 条目及 `display_width_px` / `display_height_px`，动作在 `input.action` 中描述）仍在支持它的模型和平台上继续运行——Bedrock、AWS 上的 Claude Platform 和 Foundry 目前仅提供早期 beta 版本——且这两种形式不能共用同一请求。**Claude Opus 5.5 仅接受工具集形式**：`computer_20251124` 在该模型上会返回 400 错误（参见 `shared/model-migration.md` -> 迁移到 Claude Opus 5.5 -> 重大变更 4，其中列出了请求和代理循环方面的变化；可在 Claude Opus 5 上测试，该模型同时支持两种形式）。**Claude Sonnet 5.5 在 Claude API 和 Google Cloud 上仅接受工具集形式**（`computer_20251124` 在该模型上会返回 400 错误；Amazon Bedrock 仍支持该工具，其他平台均不支持 `computer_20250124`）——详情参见 `shared/model-migration.md` -> 迁移到 Claude Sonnet 5.5 -> 重大变更 4。

如需完整文档（成员参考、批量操作、扩展性以及 `computer_20251124` 的迁移步骤），请访问 WebFetch：
- URL: `https://platform.claude.com/docs/en/agents-and-tools/computer-use/overview`

---

## 上下文编辑

上下文编辑用于清除长时间运行的代理在多轮对话中积累的过时工具结果和思考片段。与压缩（摘要）不同，上下文编辑是剪枝操作——被清除的内容会被移除，而非替换。当旧的工具输出不再相关，而又希望保持对话结构的同时精简对话记录时，可使用此功能。**Beta 版。** 使用 `client.beta.messages.*` 配合 beta 版的 `context-management-2025-06-27`。通过 `context_management.edits` 进行配置，策略类型为 `clear_tool_uses_20250919`（清除旧的工具调用结果；可选的 `clear_tool_inputs: true` 也会清除工具调用参数）或 `clear_thinking_20251015`（清除思考块）。这些**不是**压缩类型——与 beta 版 `compact-2026-01-12` 搭配的 `compact_20260112` 是独立的压缩功能。

如需完整文档，请使用 WebFetch：

- URL：`https://platform.claude.com/docs/en/build-with-claude/context-editing`

---

## 服务器端工具：Advisor（Beta 版）

Advisor 工具将一种速度更快、成本更低的**执行模型**（请求中的顶级 `model`）与一种智能更高的**顾问模型**（工具定义中的 `model` 字段）搭配使用，在生成过程中提供战略指导。执行模型负责大部分的 token 生成；顾问模型则用于规划。可用性：请参阅 `shared/platform-availability.md`。

### 工具定义

```json
{
  "type": "advisor_20260301",
  "name": "advisor",
  "model": "claude-opus-4-8"
}
```

工具定义中的可选字段：

- `max_uses`：限制每次请求中顾问的调用次数。超过该限制时，`advisor_tool_result` 块的 `content` 将为错误对象 `{"type": "advisor_tool_result_error", "error_code": "max_uses_exceeded"}`——即下方负载结构表中 content 联合类型的第三种成员。
- `max_tokens`：限制顾问每次调用的总输出量（包括思考内容和文本）。达到上限时，结果块会携带 `stop_reason: "max_tokens"`，并且在执行模型看到的建议后会附加截断提示；同时，服务器还会在顾问的提示中插入一个剩余 token 预算块，使其自动调整以符合上限。
- `caching`：对顾问自身提示的缓存控制，其结构与缓存断点相同：“caching”: {"type": "ephemeral", "ttl": "5m"}（`ttl` 可为 `"5m"` 或 `"1h"`，默认为 `"5m"`）。每次调用都会以该 TTL 写入一条缓存记录，以便对话中后续的调用能够读取稳定的前缀。若省略，则顾问的提示不会被缓存。

**顾问模型的能力必须不低于执行模型。** 如果配对无效，将返回 `400 invalid_request_error`。有效的配对如下：| 执行器（请求 `model`） | 有效顾问（工具 `model`） |
|---|---|
| `claude-haiku-4-5` / `claude-sonnet-4-6` | `claude-mythos-5-1`、`claude-fable-5-1`、`claude-mythos-5`、`claude-fable-5`、`claude-opus-5-5`、`claude-opus-5`、`claude-opus-4-8`、`claude-opus-4-7`、`claude-opus-4-6`、`claude-sonnet-5-5`、`claude-sonnet-5` 或 `claude-sonnet-4-6` |
| `claude-sonnet-5` | `claude-mythos-5-1`、`claude-fable-5-1`、`claude-mythos-5`、`claude-fable-5`、`claude-opus-5-5`、`claude-opus-5`、`claude-opus-4-8`、`claude-opus-4-7`、`claude-sonnet-5-5` 或 `claude-sonnet-5` |
| `claude-opus-4-6` | `claude-mythos-5-1`、`claude-fable-5-1`、`claude-mythos-5`、`claude-fable-5`、`claude-opus-5-5`、`claude-opus-5`、`claude-opus-4-8`、`claude-opus-4-7`、`claude-opus-4-6`、`claude-sonnet-5-5` 或 `claude-sonnet-5` |
| `claude-opus-4-7` / `claude-opus-4-8` | `claude-mythos-5-1`、`claude-fable-5-1`、`claude-mythos-5`、`claude-fable-5`、`claude-opus-5-5`、`claude-opus-5`、`claude-opus-4-8`、`claude-opus-4-7` 或 `claude-sonnet-5-5` |
| `claude-opus-5-5` / `claude-opus-5` / `claude-fable-5` / `claude-mythos-5` | `claude-mythos-5-1`、`claude-fable-5-1`、`claude-mythos-5`、`claude-fable-5`、`claude-opus-5-5` 或 `claude-opus-5` |
| `claude-fable-5-1` / `claude-mythos-5-1` | `claude-mythos-5-1` 或 `claude-fable-5-1`——这些执行器（如 `claude-opus-5-5`）会拒绝强制的 `tool_choice`，因此需要在提示中引导顾问调用 |
| `claude-sonnet-5-5` | `claude-mythos-5-1`、`claude-fable-5-1`、`claude-mythos-5`、`claude-fable-5`、`claude-opus-5-5`、`claude-opus-5` 或 `claude-sonnet-5-5`——Claude Opus 4.8/4.7/4.6 和 Claude Sonnet 5 及 Sonnet 4.6 的顾问会返回 400 错误；所有被接受的顾问都会返回加密的 `advisor_redacted_result`，且该执行器会拒绝强制的 `tool_choice`，因此需要在提示中引导顾问调用 |

> 警告：**不同顾问模型的负载格式有所不同。** 响应块始终为 `advisor_tool_result`；变化的是其 **`content`** 字段，它是一个可区分联合类型：  
>  
> | `content` 类型 | 字段 | 发生时机 |  
> |---|---|---|  
> | `advisor_result` | `text`、`stop_reason` | 顾问返回明文（如 Opus 4.8） |  
> | `advisor_redacted_result` | `encrypted_content`、`stop_reason` | 顾问返回加密输出——Claude Opus 5.5、Claude Opus 5、Claude Fable 5.1、Claude Mythos 5.1、Claude Fable 5、Claude Mythos 5、Claude Sonnet 5.5 |  
> | `advisor_tool_result_error` | `error_code` | 会话失败——`max_uses_exceeded`、`prompt_too_long`、`too_many_requests`、`overloaded`、`unavailable`、`execution_time_exceeded` 或 `model_not_found` |  
>  
> 因此，请根据 `advisor_tool_result.content` 的类型进行分支判断，而不是根据整个块的类型。如果代码无条件读取 `.text`，那么从 Claude Opus 5.5 或 Claude Opus 5 的顾问那里将什么都得不到，因为有效载荷是在 `encrypted_content` 字段下——而且你无法直接读取它，只能将其原样回传。

通过 `client.beta.messages.create(...)` 并携带 `betas=["advisor-tool-2026-03-01"]`（或 `anthropic-beta: advisor-tool-2026-03-01` 头部）来调用。在多轮对话中，将完整的 `response.content`——包括所有 `advisor_tool_result` 块——在下一轮时追加回 `messages` 列表中。如果在后续轮次中从 `tools` 中移除顾问工具，而历史记录中仍包含 `advisor_tool_result` 块，则 API 会返回 400 错误。

> **托管代理中的顾问：** CMA 会话也支持顾问功能，其配置方式是作为代理的多智能体名单中的一个 `{"type": "advisor", "model"}` 条目，而非工具定义——没有 `max_uses`/`max_tokens`/`caching` 等选项，建议会以线程事件的形式发送到会话的事件流中，而不是作为 `advisor_tool_result` 块。详情请参阅 `shared/managed-agents-multiagent.md` -> 顾问。

---

## 客户端工具：记忆

记忆工具使 Claude 能够通过一个记忆文件目录，在不同对话之间存储和检索信息。Claude 可以创建、读取、更新和删除在会话间持久保存的文件。

### 关键事实

- 客户端工具——您通过自己的实现来控制存储
- 支持的命令：`view`、`create`、`str_replace`、`insert`、`delete`、`rename`
- 操作位于 `/memories` 目录中的文件
- Python、TypeScript 和 Java SDK 提供了用于实现内存后端的辅助类/函数

> **安全须知：** 切勿在内存文件中存储 API 密钥、密码、令牌或其他敏感信息。对于个人身份信息（PII），请务必谨慎——在持久化用户数据之前，请先查阅相关数据隐私法规（如 GDPR、CCPA）。参考实现未内置访问控制；在多用户系统中，应在您的工具处理程序中为每个用户实现独立的内存目录和身份验证机制。

有关完整的实现示例，请参阅 WebFetch：

- 文档：`https://platform.claude.com/docs/en/agents-and-tools/tool-use/memory-tool.md`

---

## 客户端工具：Bash 与文本编辑器

Bash 工具和文本编辑器工具是 **Anthropic 定义的无模式工具**。只需按 `type` 和 `name` 声明即可——输入模式已内置于模型中，不可修改。**请勿传递 `input_schema`**，也请勿定义一个名为 `"bash"` 的自定义工具，因为那样会创建一个不具备内置行为的用户定义工具。

这两种工具均为 **客户端执行**：Claude 会返回一个 `tool_use` 块，您的代码在本地执行相应操作，然后将结果以 `tool_result` 形式返回。API 是无状态的；在各轮对话之间，由您的应用程序维护 shell 会话或文件系统状态。

### Bash 工具声明

```json
{"type": "bash_20250124", "name": "bash"}
```

| 语言 | 声明 |
|---|---|
| Python / TypeScript / Ruby / cURL | 纯对象 `{"type": "bash_20250124", "name": "bash"}` |
| Go | `anthropic.ToolUnionParam{OfBashTool20250124: &anthropic.ToolBash20250124Param{}}` |
| Java | 从 `com.anthropic.models.messages` 中调用 `.addTool(ToolBash20250124.builder().build())` |
| C# | 从 `Anthropic.Models.Messages` 中使用 `Tools = [new ToolBash20250124()]` |
| PHP | `tools: [new \Anthropic\Messages\ToolBash20250124()]` |

Claude 的 `tool_use.input` 包含 `{"command": "<string>"}` 或 `{"restart": true}`。请优先检查是否为 `restart`（重置会话并返回确认字符串）；否则运行 `command` 并返回合并后的标准输出与标准错误输出。

> **安全须知——命令来自不可信的模型输出。** 请在隔离环境中（容器、虚拟机或受限用户）执行，并应用允许列表以限制可执行文件的范围，同时拒绝使用 shell 操作符（如 `&&`、`|`、`;`、`` ` ``、`$()`）；设置超时和资源限制；记录每一条命令。仅使用黑名单是不够的。

### 文本编辑器工具声明

```json
{"type": "text_editor_20250728", "name": "str_replace_based_edit_tool"}
```

可选字段：`max_characters` 用于限制 `view` 输出的字符数。Java 提供了一个强类型的 `ToolTextEditor20250728` 构建器（`com.anthropic.models.messages`）；其他静态类型 SDK 也遵循相同的命名规范——具体类名请参见 `{lang}/claude-api/tool-use.md` 中的“Anthropic 定义的工具”部分。

> **安全须知——`path` 来自不可信的模型输出。请将所有文件操作限定在固定的项目根目录内。** 在执行任何命令之前，应将模型提供的 `path` 解析为其规范形式，并确保其仍位于您的项目根目录内；若发现路径逃逸（如 `..`、符号链接、根目录外的绝对路径、URL 编码的路径遍历如 `%2e%2e%2f`），则应拒绝该请求。请使用您所在语言的内置路径处理工具（例如 Python 中的 `pathlib.Path.resolve()`，然后检查 `.is_relative_to(root)`）。切勿直接对原始的 `path` 值调用 `open()`、`writeFile` 或 `unlink`。

`tool_use.input.command` 可能是以下之一：

| `command` | 其他输入 | 操作 |
|---|---|---|
| `view` | `path`，可选的 `view_range` | 返回文件内容或目录列表 |
| `create` | `path`，`file_text` | 使用 `file_text` 创建或覆盖文件。如果文件已存在，则创建备份。 |
| `str_replace` | `path`，`old_str`，`new_str` | 替换恰好一次；若匹配次数为 0 或 >1，则报错 |
| `insert` | `path`，`insert_line`，`insert_text` | 在第 `insert_line` 行之后插入 `insert_text`（0 表示文件开头） |

对于这两种工具，发生错误时请返回 `{"type": "tool_result", "tool_use_id": "...", "content": "<error text>", "is_error": true}`，以便 Claude 能够进行恢复。

---

## 结构化输出

结构化输出将 Claude 的响应限制为遵循特定的 JSON 模式，从而确保输出有效且可解析。这并不是一种单独的工具——它只是增强了 Messages API 的响应格式和/或工具参数验证。

目前提供两项功能：

- **JSON 输出**（`output_config.format`）：控制 Claude 的响应格式
- **严格工具使用**（`strict: true`）：保证工具参数模式的有效性

**支持的模型：** Claude Fable 5、Claude Mythos 5、Claude Fable 5.1、Claude Mythos 5.1、Claude Opus 5.5、Claude Opus 5、Claude Opus 4.8、Claude Sonnet 5.5、Claude Sonnet 5 和 Claude Haiku 4.5。旧版模型（Claude Opus 4.5、Claude Opus 4.1）也支持结构化输出。

> **建议：** 使用 `client.messages.parse()`，它会自动根据您的模式对响应进行验证。直接使用 `messages.create()` 时，请使用 `output_config: {format: {...}}`。部分 SDK 方法（如 `.parse()`）也接受便捷的 `output_format` 参数，但 API 层面的标准参数是 `output_config.format`。

### JSON 模式的限制

**支持：**

- 基本类型：对象、数组、字符串、整数、数字、布尔值、空值
- `enum`、`const`、`anyOf`、`allOf`、`$ref`/`$def`
- 字符串格式：`date-time`、`time`、`date`、`duration`、`email`、`hostname`、`uri`、`ipv4`、`ipv6`、`uuid`
- `additionalProperties: false`（所有对象均需设置）

**不支持：**

- 递归模式
- 数值约束（`minimum`、`maximum`、`multipleOf`）
- 字符串约束（`minLength`、`maxLength`）
- 复杂的数组约束
- `additionalProperties` 设置为除 `false` 之外的任何值

Python 和 TypeScript SDK 会自动处理不支持的约束，将其从发送给 API 的模式中移除，并在客户端进行验证。

### 重要说明

- **首次请求延迟：** 新模式会产生一次性编译开销。后续使用相同模式的请求将使用 24 小时缓存。
- **拒绝原因：** 如果 Claude 因安全原因拒绝生成（`stop_reason: "refusal"`），则输出可能不符合您的模式。
- **令牌限制：** 如果 `stop_reason: "max_tokens"`，输出可能会不完整。请增加 `max_tokens`。
- **不兼容：** 引用（返回 400 错误）、消息预填充。
- **兼容：** 批量 API、流式传输、令牌计数、扩展思考。

---

## 有效使用工具的技巧

1. **提供详细描述：** Claude 非常依赖描述来理解何时以及如何使用工具。
2. **使用具体工具名称：** `get_current_weather` 比 `weather` 更好。
3. **验证输入：** 在执行前务必验证工具输入。
4. **优雅地处理错误：** 返回清晰的错误信息，以便 Claude 能够做出调整。
5. **限制工具数量：** 工具过多会使模型困惑——保持工具集聚焦。
6. **测试工具交互：** 确认 Claude 在各种场景下都能正确使用工具。

有关详细的工具使用文档，请参阅 WebFetch：

- 网址：`https://platform.claude.com/docs/en/agents-and-tools/tool-use/overview`