# 工具使用 - C#

有关概念性概述（工具定义、工具选择、提示），请参阅 [shared/tool-use-concepts.md](../../shared/tool-use-concepts.md)。

## 工具使用

### 定义工具

`Tool`（非 `ToolParam`）带有 `InputSchema` 记录。`InputSchema.Type` 由构造函数自动设置为 `"object"`，请勿手动设置。`ToolUnion` 具有从 `Tool` 的隐式转换，该转换由集合表达式 `[...]` 触发。

```csharp
using System.Text.Json;
using Anthropic.Models.Messages;

var parameters = new MessageCreateParams
{
    Model = "claude-opus-5-5",
    MaxTokens = 16000,
    Tools = [
        new Tool {
            Name = "get_weather",
            Description = "获取指定地点的当前天气",
            InputSchema = new() {
                Properties = new Dictionary<string, JsonElement> {
                    ["location"] = JsonSerializer.SerializeToElement(
                        new { type = "string", description = "城市名称" }),
                },
                Required = ["location"],
            },
        },
    ],
    Messages = [new() { Role = Role.User, Content = "巴黎的天气如何？" }],
};
```

源自 `anthropic-sdk-csharp/src/Anthropic/Models/Messages/Tool.cs` 和 `ToolUnion.cs:799`（隐式转换）。

有关循环模式，请参阅 [共享工具使用概念](../../shared/tool-use-concepts.md)。  
### 将响应内容转换为后续助手消息

在助手回合中回显 Claude 的响应时，**没有 `.ToParam()` 辅助方法**——需手动将每个 `ContentBlock` 变体重建为其对应的 `*Param` 类型。切勿使用 `new ContentBlockParam(block.Json)`：虽然可以编译并序列化，但 `.Value` 仍为 `null`，因此 `TryPick*`/`Validate()` 会失败（仅是 JSON 的降级透传，而非类型化的路径）。

```csharp
using Anthropic.Models.Messages;

Message response = await client.Messages.Create(parameters);

// 没有 .ToParam() - 需按变体逐一重建。每种 *Param 类型到 ContentBlockParam 的隐式转换意味着无需显式包装。
List<ContentBlockParam> assistantContent = [];
List<ContentBlockParam> toolResults = [];
foreach (ContentBlock block in response.Content)
{
    if (block.TryPickText(out TextBlock? text))
    {
        assistantContent.Add(new TextBlockParam { Text = text.Text });
    }
    else if (block.TryPickThinking(out ThinkingBlock? thinking))
    {
        // 必须保留签名 - API 会拒绝篡改
        assistantContent.Add(new ThinkingBlockParam
        {
            Thinking = thinking.Thinking,
            Signature = thinking.Signature,
        });
    }
    else if (block.TryPickRedactedThinking(out RedactedThinkingBlock? redacted))
    {
        assistantContent.Add(new RedactedThinkingBlockParam { Data = redacted.Data });
    }
    else if (block.TryPickToolUse(out ToolUseBlock? toolUse))
    {
        // ToolUseBlock 要求提供 Caller；而 ToolUseBlockParam.Caller 是可选的，不要复制它
        assistantContent.Add(new ToolUseBlockParam
        {
            ID = toolUse.ID,
            Name = toolUse.Name,
            Input = toolUse.Input,
        });
        // 执行工具；为每个 tool_use 块收集一个结果 - 如果任何 tool_use ID 缺少对应的 tool_result，API 会拒绝后续请求。
        string result = ExecuteYourTool(toolUse.Name, toolUse.Input);
        toolResults.Add(new ToolResultBlockParam
        {
            ToolUseID = toolUse.ID,
            Content = result,
        });
    }
}

// 后续消息：前序消息 + 助手回显 + 用户提供的工具结果
List<MessageParam> followUpMessages =
[
    .. parameters.Messages,
    new() { Role = Role.Assistant, Content = assistantContent },
    new() { Role = Role.User, Content = toolResults },
];
```

`ToolResultBlockParam` 没有元组构造函数——请使用对象初始化器。`Content` 是字符串或列表的联合类型；普通 `string` 会自动进行隐式转换。

---

## 结构化输出

```csharp
OutputConfig = new OutputConfig {
    Format = new JsonOutputFormat {
        Schema = new Dictionary<string, JsonElement> {
            ["type"] = JsonSerializer.SerializeToElement("object"),
            ["properties"] = JsonSerializer.SerializeToElement(
                new { name = new { type = "string" } }),
            ["required"] = JsonSerializer.SerializeToElement(new[] { "name" }),
        },
    },
},
```

`JsonOutputFormat.Type` 由构造函数自动设置为 `"json_schema"`。`Schema` 是必填项。

---

## Anthropic 定义的工具

网络搜索、Bash、文本编辑器和代码执行是 Anthropic 定义的内置模式的工具。其中，网络搜索和代码执行在服务器端执行；Bash 和文本编辑器则在客户端执行（您需要在本地处理 `tool_use`——参见 `shared/tool-use-concepts.md`）。工具类型名称带有版本后缀；构造函数会自动设置 `name` 和 `type`。**请务必使用 `new ToolUnion(...)` 显式包装每个工具。**

```csharp
Tools = [
    new ToolUnion(new WebSearchTool20260209()),
    new ToolUnion(new ToolBash20250124()),
    new ToolUnion(new ToolTextEditor20250728()),
    new ToolUnion(new CodeExecutionTool20260120()),
],
```

此外还提供：`new ToolUnion(new WebFetchTool20260209())`、`new ToolUnion(new MemoryTool20250818())`。`WebSearchTool20260209` 的可选参数包括：`AllowedDomains`、`BlockedDomains`、`MaxUses`、`UserLocation`。

---

## 工具运行器（Beta 版）

C# SDK 提供了一个用于自动工具执行循环的 `BetaToolRunner`。您可以使用原始 JSON 模式定义工具，运行器将负责 API 调用、工具执行以及结果反馈的整个流程。

```csharp
using Anthropic.Models.Beta.Messages;

// 按照上述“工具使用”部分所示的方式定义工具并创建参数，
// 但需使用 Beta 命名空间中的类型（如 BetaToolUnion 等）。
var runner = client.Beta.Messages.ToolRunner(betaParams);

await foreach (BetaMessage message in runner)
{
    foreach (var block in message.Content)
    {
        if (block.TryPickText(out var text))
        {
            Console.WriteLine(text.Text);
        }
    }
}
```

---