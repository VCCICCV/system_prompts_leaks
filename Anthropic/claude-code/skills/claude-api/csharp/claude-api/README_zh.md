# Claude API - C#

> **注意：** C# SDK 是 Anthropic 官方提供的 C# SDK。通过 Messages API 支持工具使用，并提供了一个用于自动执行工具调用循环的 Beta 版 `BetaToolRunner`。该 SDK 还支持 Microsoft.Extensions.AI 的 IChatClient 集成，包括函数调用和托管代理（Beta 版）。

## 命名空间参考

类型按命名空间组织。如果所需类型未在下方示例中列出，请先通过下表查找，不要因从网络获取 SDK 源代码而阻塞。

| `using` | 包含内容 |
|---|---|
| `Anthropic` | `AnthropicClient`、顶级选项 |
| `Anthropic.Models.Messages` | 非 Beta 版请求/响应类型——`MessageCreateParams`、`Model`、`Role`、`ContentBlock`、`TextBlock`、`ToolUseBlock`、`ToolResultBlockParam`、`Tool*`（工具定义类） |
| `Anthropic.Models.Beta.Messages` | Beta 端点对应类型——`MessageCreateParams`、`BetaMessage`、`BetaTool*`、`Speed`、`BetaRequestMcpServerUrlDefinition`、上下文编辑/压缩配置 |
| `Anthropic.Models.Beta` | 共享的 Beta 常量 |
| `Anthropic.Models.Beta.Files` | Files API 类型 |
| `Anthropic.Models.Messages.Batches` | Batch API 类型 |
| `Anthropic.Helpers.Beta` | `BetaToolRunner`、Beta 辅助工具 |
| `Anthropic.Exceptions` | `AnthropicApiException`、`AnthropicRateLimitException`、`Anthropic5xxException` 等——参见 `shared/error-codes.md` |
| `Anthropic.Bedrock` / `Anthropic.Vertex` / `Anthropic.Foundry` / `Anthropic.Aws` | 平台客户端（独立 NuGet 包）：`AnthropicBedrockMantleClient`、`AnthropicFoundryClient`、`AnthropicAwsClient` |

`client.Messages.*` 使用非 Beta 版类型；`client.Beta.Messages.*` 使用 `Anthropic.Models.Beta.Messages` 中的类型。两个命名空间都定义了 `MessageCreateParams`——请根据调用的客户端路径选择相应版本。

### 各功能的关键类型

请参考此表，而非直接反射 SDK 程序集。端点列指示应使用 `client.Messages.*` 还是 `client.Beta.Messages.*`。

| 功能 | 端点 | 关键 C# 类型（命名空间见上表） |
|---|---|---|
| 用户档案 | Beta | `client.Beta.UserProfiles.Create(...)` / `.Retrieve(id)` / `.List()`。将返回的用户档案 ID 传递给 Beta 版 Messages 调用。需要 Beta 头部——请查阅 SDK 的 Beta 头部参考以获取当前标志。 |
| 代理技能 | Beta | `BetaContainerParams`（包含 `Skills = [new BetaSkillParams { ... }]`）、`BetaCodeExecutionTool20250825`。`Betas = ["code-execution-2025-08-25"]`（技能已脱离 Beta——无 `skills-2025-10-02`）。通过 `client.Beta.Files.Download(fileId)` 下载输出。 |
| 顾问工具 | Beta | `BetaAdvisorTool20260301`——可能尚未包含在所有 SDK 版本中 |
| 缓存诊断 | Beta | `Diagnostics = new() { PreviousMessageID = ... }`、`BetaCacheControlEphemeral`、`BetaContentBlockParam` |
| 上下文编辑 | Beta | `ContextManagement = new BetaContextManagementConfig { Edits = [new BetaClearToolUses20250919Edit()] }`。`Betas = ["context-management-2025-06-27"]`（不是 `compact-2026-01-12`——那是用于 `BetaCompact20260112Edit`）。 |
| 记忆工具 | 非 Beta | `Tools = [new ToolUnion(new MemoryTool20250818())]` |
| 程序化工具调用 | 非 Beta | `CodeExecutionTool20260120`、`ToolResultBlockParam`、`ContentBlockParam` |
| 任务预算 | Beta | `BetaOutputConfig` 中的 `TaskBudget = new BetaTokenTaskBudget { ... }` |
| 工具搜索 | 非 Beta | `new ToolUnion(new ToolSearchToolRegex20251119 { Type = ToolSearchToolRegex20251119Type.ToolSearchToolRegex20251119 })`——必须显式设置 `Type`。 |
| 网络搜索 | 非 Beta | `new ToolUnion(new WebSearchTool20260209())`——最新变体，支持动态过滤（Claude Fable 5.1 + Claude Opus 5.5 + Claude Opus 5 + Opus 4.8/4.7/4.6 + Claude Sonnet 5.5 + Claude Sonnet 5 + Sonnet 4.6）。对于较旧模型或 Vertex，请使用 `WebSearchTool20250305()` |

### 发现类型和成员名称如果所需的类型或成员未在上述表格中列出，使用 `strings ~/.nuget/packages/anthropic/*/lib/*/Anthropic.dll | grep -i <term>` 就能快速且有效地定位类名和属性名。**请勿升级为通过 `dotnet run` 进行的反射探测**来精确地转储成员——首次编译已经足够耗时，在许多环境中甚至需要放到后台执行，否则会陷入轮询循环。相反，只需根据 `strings | grep` 找到的名称编写 `Program.cs`；如果成员名有误，编译器错误（`error CS1061: 'X' 不包含名为 'Y' 的定义`）会在几秒钟内指出问题所在，比任何反射探测都更快。

请注意，`strings` 命令不会显示协议格式中的蛇形命名字段名（如 `output_tokens`、`stop_reason`），因为这些字段在 DLL 中是以另一种方式存储的。**C# 属性是协议字段的帕斯卡命名等价物**（如 `response.Usage.OutputTokens`、`response.StopReason`）。如果你从文档中知道协议字段名，就直接写对应的帕斯卡命名属性并编译；不要去寻找蛇形命名的字符串。

### 最小可运行骨架

**编写一个简单的 `Program.cs` 主体**——先写 `using` 语句，再写顶级语句，如下所示。**不要**添加 `#!/usr/bin/env dotnet` shebang 或 `#:package Anthropic@*` 指令：这些是 .NET 基于文件的应用语法，当文件通过现有的 `.csproj` 编译时，会导致 `CS1024: 预处理器指令预期` 错误。标准项目设置（参见 [C# 快速入门](https://platform.claude.com/docs/en/get-started)：`dotnet new console` -> `dotnet add package Anthropic` -> 编辑 `Program.cs` -> `dotnet run`）会自动生成 `.csproj` 和包引用。

从以下代码开始——它可以直接编译。填充特定功能所需的字段；无需花费时间进行反射或 XML 文档检查来先发现类型名。

```csharp
using System;
using Anthropic;
using Anthropic.Models.Messages;       // 或 Anthropic.Models.Beta.Messages 用于测试版端点

AnthropicClient client = new();

var message = await client.Messages.Create(new MessageCreateParams
{
    Model = "claude-opus-5-5",
    MaxTokens = 1024,
    Messages = [ new() { Role = Role.User, Content = "Hello, Claude" } ],
});

Console.WriteLine(message);
```

对于测试版功能（任何带有 `anthropic-beta` 头部的功能），请使用测试版客户端路径和命名空间——整体结构相同：

```csharp
using System;
using Anthropic;
using Anthropic.Models.Beta.Messages;

AnthropicClient client = new();

var response = await client.Beta.Messages.Create(new MessageCreateParams
{
    Model = "claude-opus-5-5",
    MaxTokens = 4096,
    Betas = ["<beta-flag>"],
    Messages = [ new() { Role = Role.User, Content = "..." } ],
    // Tools = new BetaToolUnion[] { new BetaSomeTool { ... } },   // 用于工具功能
});

Console.WriteLine(response);
```

如果某个功能所需的类型名不在该文件中，请按照上述命名空间参考中的命名规则编写，并根据编译器输出进行修正——边写 `Program.cs` 边迭代，远比先做研究更高效。

### 常见 C# 编译错误

- **CS8803（顶级语句必须位于类型声明之前）：** 将所有 `record`/`class`/`struct` 定义**放在**最后一个顶级语句之后，即文件末尾。如果在 `var client = new AnthropicClient()` 之前定义了记录类型，将无法编译。
- **对 `Task<...Page>` 使用 `await foreach`：** `client.Models.List()` 返回的是 `Task<ModelListPage>`，不能直接异步枚举。应先等待其完成，再进行迭代：`var page = await client.Models.List(); foreach (var m in page.Items) {...}`。若需自动分页，请先查看页面类型是否提供了 `AutoPagingEachAsync()` 等方法，再考虑使用 `await foreach`。

## 安装

```bash
dotnet add package Anthropic
```

## 客户端初始化

```csharp
using Anthropic;

// 默认（使用 ANTHROPIC_API_KEY 环境变量）
AnthropicClient client = new();
// 显式 API 密钥（请使用环境变量，切勿硬编码密钥）
AnthropicClient client = new() {
    ApiKey = Environment.GetEnvironmentVariable("ANTHROPIC_API_KEY")
};
```

---

## 基本消息请求

```csharp
using Anthropic.Models.Messages;

var parameters = new MessageCreateParams
{
    Model = "claude-opus-5-5",
    MaxTokens = 16000,
    Messages = [new() { Role = Role.User, Content = "法国的首都是哪里？" }]
};
var response = await client.Messages.Create(parameters);

// ContentBlock 是一个联合类型包装器。.Value 可以解包为具体的变体对象，
// 然后使用 OfType<T> 过滤出所需类型。或者也可以使用下文“思考”部分所示的 TryPick* 模式。
foreach (var text in response.Content.Select(b => b.Value).OfType<TextBlock>())
{
    Console.WriteLine(text.Text);
}
```

---

## 思考模式

**对于 Claude 4.6 及以上版本的模型，推荐使用自适应思考模式。** Claude 会动态决定何时以及需要进行多少思考。

> **Fable 5、Claude Opus 5.5、Claude Opus 5、Opus 4.8、Opus 4.7、Opus 4.6 和 Sonnet 4.6：** 使用自适应思考模式（见下文）。`new ThinkingConfigEnabled { BudgetTokens = N }` 在 Fable 5、Claude Opus 5.5、Claude Opus 5、Opus 4.8 和 4.7 上已被移除（若发送则返回 400 错误）；在 Opus 4.6 和 Sonnet 4.6 上已弃用。  
> **Claude Opus 5.5：** 思考始终开启——可省略 `Thinking` 参数（或发送等价的 `ThinkingConfigAdaptive`）；`ThinkingConfigDisabled` 在任何努力级别上都会返回 400 错误，设置思考预算也同样如此。应改用 `OutputConfig.Effort` 来控制思考深度——该模型的默认值为 `medium`，而 Claude Opus 5 的默认值为 `high`。  
> **Claude Opus 5：** 默认启用思考——省略 `Thinking` 即为自适应模式（等同于 `ThinkingConfigAdaptive`），这与 Opus 4.8/4.7 不同，在后者中省略则表示不进行思考。`ThinkingConfigDisabled` 仅在 `high` 或更低的努力级别下有效；若与 `xhigh` 或 `max` 配合使用，则会返回 400 错误。  
> **较旧的模型：** 使用 `new ThinkingConfigEnabled { BudgetTokens = N }`（预算必须小于 `MaxTokens`，最低为 1024）。

```csharp
using Anthropic.Models.Messages;

var response = await client.Messages.Create(new MessageCreateParams
{
    Model = "claude-opus-5-5",
    MaxTokens = 16000,
    // ThinkingConfigParam? 可从具体的变体类隐式转换而来——无需额外的包装。
    // 显示选项：Fable 5/5.1、Mythos 5/5.1、Claude Opus 5.5、Claude Opus 5、Opus 4.8/4.7、Claude Sonnet 5.5 和 Claude Sonnet 5 默认省略显示（即不显示思考内容）。
    Thinking = new ThinkingConfigAdaptive { Display = Display.Summarized },
    Messages =
    [
        new() { Role = Role.User, Content = "计算：27 * 453" },
    ],
});

// Content 中，ThinkingBlock 位于 TextBlock 之前。使用 TryPick* 可以缩小联合类型的范围。
foreach (var block in response.Content)
{
    if (block.TryPickThinking(out ThinkingBlock? t))
    {
        Console.WriteLine($"[thinking] {t.Thinking}");
    }
    else if (block.TryPickText(out TextBlock? text))
    {
        Console.WriteLine(text.Text);
    }
}
```

另一种方法是使用 `.Select(b => b.Value).OfType<ThinkingBlock>()`（与基本消息示例中的 LINQ 写法相同）。

---

## 上下文编辑/压缩（Beta 版）

**Beta 命名空间前缀不一致**（经源码验证，参照 `src/Anthropic/Models/Beta/Messages/*.cs` @ 12.9.0）。无前缀的有：`MessageCreateParams`、`MessageCountTokensParams`、`Role`、`Speed`。**其余所有均带有 `Beta` 前缀**：`BetaMessageParam`、`BetaMessage`、`BetaContentBlock`、`BetaToolUseBlock`，以及所有块参数类型。未加前缀的 `Role` 若同时引入两个命名空间，将与 `Anthropic.Models.Messages.Role` 发生冲突（CS0104）。最安全的做法是仅导入 Beta 命名空间；若需混合使用，请为 Beta 的 `Role` 起别名：

```csharp
using Anthropic.Models.Beta.Messages;
using NonBeta = Anthropic.Models.Messages;  // 仅当您也需要非 Beta 类型时才这样做
// 此时：MessageCreateParams、BetaMessageParam、Role（Beta 版）、NonBeta.Role（如需）
```


`BetaMessage.Content` 是 `IReadOnlyList<BetaContentBlock>`——一种包含 15 种变体的区分联合类型。可通过 `TryPick*` 方法进行筛选。**响应中的 `BetaContentBlock` 不能直接赋值给参数中的 `BetaContentBlockParam`**——C# 中没有 `.ToParam()` 方法。可通过逐个转换每个块来实现往返操作：
```csharp
using Anthropic.Models.Beta.Messages;

var betaParams = new MessageCreateParams   // 无 Beta 前缀 - 参见上文的非前缀列表
{
    Model = "claude-opus-5-5",
    MaxTokens = 16000,
    Betas = ["compact-2026-01-12"],
    ContextManagement = new BetaContextManagementConfig
    {
        Edits = [new BetaCompact20260112Edit()],
    },
    Messages = messages,
};
BetaMessage resp = await client.Beta.Messages.Create(betaParams);

foreach (BetaContentBlock block in resp.Content)
{
    if (block.TryPickCompaction(out BetaCompactionBlock? compaction))
    {
        // 内容可能为 null - 因为压缩操作可能在服务端失败
        Console.WriteLine($"压缩摘要: {compaction.Content}");
    }
}

// 上下文编辑的元数据位于一个单独的可空字段中
if (resp.ContextManagement is { } ctx)
{
    foreach (var edit in ctx.AppliedEdits)
        Console.WriteLine($"清除了 {edit.ClearedInputTokens} 个 token");
}

// 往返转换：BetaMessageParam.Content 是 BetaMessageParamContent（字符串或列表的联合类型）。它会从 List<BetaContentBlockParam> 隐式转换，而不是从响应中的 IReadOnlyList<BetaContentBlock> 转换。逐个转换每个块：
List<BetaContentBlockParam> paramBlocks = [];
foreach (var b in resp.Content)
{
    if (b.TryPickText(out var t)) paramBlocks.Add(new BetaTextBlockParam { Text = t.Text });
    else if (b.TryPickCompaction(out var c)) paramBlocks.Add(new BetaCompactionBlockParam { Content = c.Content });
    // ... 根据需要处理其他变体
}
messages.Add(new BetaMessageParam { Role = Role.Assistant, Content = paramBlocks });
```

所有 15 种 `BetaContentBlock.TryPick*` 变体：`Text`、`Thinking`、`RedactedThinking`、`ToolUse`、`ServerToolUse`、`WebSearchToolResult`、`WebFetchToolResult`、`CodeExecutionToolResult`、`BashCodeExecutionToolResult`、`TextEditorCodeExecutionToolResult`、`ToolSearchToolResult`、`McpToolUse`、`McpToolResult`、`ContainerUpload`、`Compaction`。

**`BetaToolUseBlock.Input` 是 `IReadOnlyDictionary<string, JsonElement>`** - 通过键索引后调用 `JsonElement` 提取器：

```csharp
if (block.TryPickToolUse(out BetaToolUseBlock? tu))
{
    int a = tu.Input["a"].GetInt32();
    string s = tu.Input["name"].GetString()!;
}
```

---

## 努力参数

努力参数嵌套在 `OutputConfig` 下，而不是顶级属性。`ApiEnum<string, Effort>` 有从枚举到该类型的隐式转换，因此可以直接赋值 `Effort.High`。

```csharp
OutputConfig = new OutputConfig { Effort = Effort.High },
```

取值：`Effort.Low`、`Effort.Medium`、`Effort.High`、`Effort.Max`。与 `Thinking = new ThinkingConfigAdaptive()` 搭配使用，以实现成本与质量的平衡。

---

## 提示缓存

`System` 接受 `MessageCreateParamsSystem?` - 即 `string` 或 `List<TextBlockParam>` 的联合类型。没有 `SystemTextBlockParam`；请使用普通的 `TextBlockParam`。隐式转换需要具体的 `List<TextBlockParam>` 类型（数组字面量无法转换）。关于放置模式和静默失效审计清单，请参阅 `shared/prompt-caching.md`。

```csharp
System = new List<TextBlockParam> {
    new() {
        Text = longSystemPrompt,
        CacheControl = new CacheControlEphemeral(),  // 自动设置 Type = "ephemeral"
    },
},
```

`CacheControlEphemeral` 的可选 `Ttl`：`new() { Ttl = Ttl.Ttl1h }` 或 `Ttl.Ttl5m`。`CacheControl` 也存在于 `Tool.CacheControl` 和顶级的 `MessageCreateParams.CacheControl` 中。

可通过 `response.Usage.CacheCreationInputTokens` 和 `response.Usage.CacheReadInputTokens` 来验证缓存命中情况。

---

## Token 计数

```csharp
MessageTokensCount result = await client.Messages.CountTokens(new MessageCountTokensParams {
    Model = "claude-opus-5-5",
    Messages = [new() { Role = Role.User, Content = "Hello" }],
});
long tokens = result.InputTokens;
```

`MessageCountTokensParams.Tools` 使用的联合类型（`MessageCountTokensTool`）与 `MessageCreateParams.Tools`（`ToolUnion`）不同——如果要传递工具，编译器会在必要时提醒你。

---

## PDF/文档输入`DocumentBlockParam` 接受一个 `DocumentBlockParamSource` 联合类型：`Base64PdfSource` / `UrlPdfSource` / `PlainTextSource` / `ContentBlockSource`。`Base64PdfSource` 会自动将 `MediaType` 设置为 `"application/pdf"`，并将 `Type` 设置为 `"base64"`。

```csharp
new MessageParam {
    Role = Role.User,
    Content = new List<ContentBlockParam> {
        new DocumentBlockParam { Source = new Base64PdfSource { Data = base64String } },
        new TextBlockParam { Text = "Summarize this PDF" },
    },
}
```

---

## 快速模式（Beta 版）

```csharp
var response = await client.Beta.Messages.Create(new MessageCreateParams {
    Model = "claude-opus-5-5", MaxTokens = 4096,
    Speed = Speed.Fast,
    Betas = ["fast-mode-2026-02-01"],
    Messages = [...],
});
```

## 模型 API

```csharp
var page = await client.Models.List();
foreach (var m in page.Items) { Console.WriteLine(m.ID); }
var one = await client.Models.Retrieve("claude-opus-5-5");
```

## 长输出（128k）+ 预填充

在 `client.Messages` 中将 `MaxTokens` 设置为 128000，并使用流式传输路径（参见 `streaming.md`）。在 Claude 4 及更高版本的模型上，128k 输出是原生支持的——无需使用 `output-128k-*` Beta 标头或 Beta 命名空间。

**预填充助手消息**（在输入中添加一条末尾的 `Role.Assistant` 消息）在 Claude Fable 5.1、Claude Opus 5.5、Claude Opus 5、Opus 4.8、Opus 4.7、Opus 4.6 和 Sonnet 4.6 上**不被支持**——请求会返回 400 错误。请改用 `OutputConfig.Format`（结构化输出）。

## 停止详情

当 `StopReason` 为 `"refusal"` 时，响应中会包含结构化的 `StopDetails`：

```csharp
if (response.StopReason == "refusal" && response.StopDetails is { } details)
{
    Console.WriteLine($"Category: {details.Category}");
    Console.WriteLine($"Explanation: {details.Explanation}");
}
```

**拒绝回退（Claude Fable 5.1）——默认启用。** 回退功能是可选的：如果不启用回退，被拒绝的请求会直接停止。在 `claude-fable-5-1` 的代码中，默认情况下应在调用 Beta 消息时加入 `Fallbacks = [new(Model.ClaudeOpus4_8)]`，并启用 `AnthropicBeta.ServerSideFallback2026_06_01` Beta 功能——这样，如果出现政策拒绝，会在同一请求中由回退模型进行处理。该功能在 Amazon Bedrock、Vertex AI 和 Microsoft Foundry 上不可用——可在这些平台上使用客户端侧处理器：`new AnthropicClient { Handlers = [new BetaRefusalFallbackHandler { Fallbacks = [new(Model.ClaudeOpus4_8)] }] }`（命名空间为 `Anthropic.Helpers`），并通过 `BetaFallbackState.Create()` 创建每轮对话的状态，并使用 `using (fallbackState.Use()) { ... }` 进行作用域管理。完整的语义（计费、粘性路由、流式传输）以及可运行示例，请参见 `shared/model-migration.md` 中的“迁移到 Claude Fable 5.1”部分，以及 C# SDK 仓库中的 `examples/` 目录（通过 `shared/live-sources.md` 实现 WebFetch 示例）。

---

## 托管代理（Beta 版）

C# SDK 通过 `client.Beta.Agents`、`client.Beta.Sessions`、`client.Beta.Environments` 及相关命名空间支持托管代理。有关架构说明，请参阅 `shared/managed-agents-overview.md`；有关协议级参考，请参阅 `curl/managed-agents.md`。