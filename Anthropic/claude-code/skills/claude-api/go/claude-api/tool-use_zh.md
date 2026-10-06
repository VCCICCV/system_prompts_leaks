# 工具使用 - Go

有关概念性概述（工具定义、工具选择、提示），请参阅 [shared/tool-use-concepts.md](../../shared/tool-use-concepts.md)。

## 工具使用

### 工具运行器（Beta - 推荐）

**Beta 版：** Go SDK 通过 `toolrunner` 包提供了 `BetaToolRunner`，用于自动执行工具使用循环。

```go
import (
    "context"
    "fmt"
    "log"

    "github.com/anthropics/anthropic-sdk-go"
    "github.com/anthropic/anthropic-sdk-go/toolrunner"
)

// 使用 jsonschema 标签定义工具输入，以自动生成 Schema
type GetWeatherInput struct {
    City string `json:"city" jsonschema:"required,description=城市名称"`
}

// 从结构体标签自动生成 Schema 并创建工具
weatherTool, err := toolrunner.NewBetaToolFromJSONSchema(
    "get_weather",
    "获取某城市的当前天气",
    func(ctx context.Context, input GetWeatherInput) (anthropic.BetaToolResultBlockParamContentUnion, error) {
        return anthropic.BetaToolResultBlockParamContentUnion{
            OfText: &anthropic.BetaTextBlockParam{
                Text: fmt.Sprintf("在 %s 的天气是晴朗，72°F", input.City),
            },
        }, nil
    },
)
if err != nil {
    log.Fatal(err)
}

// 创建一个可自动处理对话循环的工具运行器
runner := client.Beta.Messages.NewToolRunner(
    []anthropic.BetaTool{weatherTool},
    anthropic.BetaToolRunnerParams{
        BetaMessageNewParams: anthropic.BetaMessageNewParams{
            Model:     "claude-opus-5-5",
            MaxTokens: 16000,
            Messages: []anthropic.BetaMessageParam{
                anthropic.NewBetaUserMessage(anthropic.NewBetaTextBlock("巴黎的天气怎么样？")),
            },
        },
        MaxIterations: 5,
    },
)

// 运行直至 Claude 生成最终响应
message, err := runner.RunToCompletion(context.Background())
if err != nil {
    log.Fatal(err)
}

// RunToCompletion 返回 *BetaMessage；内容为 []BetaContentBlockUnion。
// 通过 AsAny() 进行类型断言——请注意 Beta 命名空间中的类型（BetaTextBlock，
// 而非 TextBlock）：
for _, block := range message.Content {
    switch block := block.AsAny().(type) {
    case anthropic.BetaTextBlock:
        fmt.Println(block.Text)
    }
}
```

**Go 工具运行器的主要特性：**

- 通过 `jsonschema` 标签从 Go 结构体自动生成 Schema
- 提供 `RunToCompletion()` 方法，便于一次性简单调用
- 提供 `All()` 迭代器，用于逐条处理对话中的每一条消息
- 提供 `NextMessage()` 方法，支持逐步迭代
- 提供流式版本 `NewToolRunnerStreaming()`，并配有 `AllStreaming()` 方法

### 手动循环

建议优先使用工具运行器。如需拦截、验证、记录日志或进行人工审核，请在工具的执行函数中设置检查点，或通过 `NextMessage()`/`All()` 步进运行器并逐条查看每条消息（运行器的公共 `Params` 字段允许您调整下一次请求的内容）——通常无需采用手动循环。仅当需要控制运行器无法暴露的功能时才退回到手动循环：例如，使用 `ToolParam` 定义工具、检查 `StopReason`、自行执行工具，并将 `tool_result` 块重新注入对话中。源自 `anthropic-sdk-go/examples/tools/main.go`。

```go
package main

import (
    "context"
    "encoding/json"
    "fmt"
    "log"

    "github.com/anthropics/anthropic-sdk-go"
)

func main() {
    client := anthropic.NewClient()

    // 1. 定义工具。ToolParam.InputSchema 使用 map，无需结构体标签。
    addTool := anthropic.ToolParam{
        Name:        "add",
        Description: anthropic.String("计算两个整数之和"),
        InputSchema: anthropic.ToolInputSchemaParam{
            Properties: map[string]any{
                "a": map[string]any{"type": "integer"},
                "b": map[string]any{"type": "integer"},
            },
        },
    }
    // ToolParam 必须包装在 ToolUnionParam 中，才能放入 Tools 切片
    tools := []anthropic.ToolUnionParam{{OfTool: &addTool}}

    messages := []anthropic.MessageParam{
        anthropic.NewUserMessage(anthropic.NewTextBlock("2 + 3 等于多少？")),
    }

    for {
        resp, err := client.Messages.New(context.Background(), anthropic.MessageNewParams{
            Model:     "claude-opus-5-5",
            MaxTokens: 16000,
            Messages:  messages,
            Tools:     tools,
        })
        if err != nil {
            log.Fatal(err)
        }

        // 2. 在处理工具调用之前，将助手的响应追加到对话历史中。
        //    resp.ToParam() 可以一次性将 Message 转换为 MessageParam。
        messages = append(messages, resp.ToParam())

        // 3. 遍历内容块。ContentBlockUnion 是一个扁平化的结构体；
        //    使用 block.AsAny().(type) 来根据实际变体进行类型断言。
        toolResults := []anthropic.ContentBlockParamUnion{}
        for _, block := range resp.Content {
            switch variant := block.AsAny().(type) {
            case anthropic.TextBlock:
                fmt.Println(variant.Text)
            case anthropic.ToolUseBlock:
                // 4. 解析工具输入。使用 variant.JSON.Input.Raw() 获取原始 JSON——
                //    block.Input 是 json.RawMessage，而不是解析后的值。
                var in struct {
                    A int `json:"a"`
                    B int `json:"b"`
                }
                if err := json.Unmarshal([]byte(variant.JSON.Input.Raw()), &in); err != nil {
                    log.Fatal(err)
                }
                result := fmt.Sprintf("%d", in.A+in.B)
                // 5. NewToolResultBlock(toolUseID, content, isError) 会为你构建
                //    ContentBlockParamUnion。block.ID 就是 tool_use_id。
                toolResults = append(toolResults,
                    anthropic.NewToolResultBlock(block.ID, result, false))
            }
        }

        // 6. 当 Claude 不再请求工具时退出循环
        if resp.StopReason != anthropic.StopReasonToolUse {
            break
        }

        // 7. 工具结果放入用户消息中（可变参数：所有结果在一轮中）
        messages = append(messages, anthropic.NewUserMessage(toolResults...))
    }
}
```

**关键 API 接口：**

| 符号 | 用途 |
|---|---|
| `resp.ToParam()` | 将 `Message` 响应转换为历史记录所需的 `MessageParam` |
| `block.AsAny().(type)` | 对 `ContentBlockUnion` 的不同变体进行类型断言 |
| `variant.JSON.Input.Raw()` | 工具输入的原始 JSON 字符串（用于 `json.Unmarshal`） |
| `anthropic.NewToolResultBlock(id, content, isError)` | 构建 `tool_result` 块 |
| `anthropic.NewUserMessage(blocks...)` | 将工具结果封装为用户的一轮消息 |
| `anthropic.StopReasonToolUse` | 用于检查循环终止的 `StopReason` 常量 |
| `anthropic.ToolUnionParam{OfTool: &t}` | 将 `ToolParam` 包装进联合类型，用于 `Tools:` 参数 |

---

## Anthropic 定义的工具

版本后缀的结构体名称带有 `Param` 后缀。`Name` 和 `Type` 是 `constant.*` 类型——零值也能正确序列化，因此使用 `{}` 即可。需将其包装在 `ToolUnionParam` 中，并设置对应的 `Of*` 字段。网络搜索和代码执行由服务器端执行；Bash 和文本编辑器则由客户端执行（您需要在本地处理 `tool_use` 事件——参见 `shared/tool-use-concepts.md`）。

```go
Tools: []anthropic.ToolUnionParam{
    {OfWebSearchTool20260209: &anthropic.WebSearchTool20260209Param{}},
    {OfBashTool20250124: &anthropic.ToolBash20250124Param{}},
    {OfTextEditor20250728: &anthropic.ToolTextEditor20250728Param{}},
    {OfCodeExecutionTool20260120: &anthropic.CodeExecutionTool20260120Param{}},
},
```

此外还提供：`WebFetchTool20260209Param`、`ToolSearchToolBm25_20251119Param`、`ToolSearchToolRegex20251119Param`。对于顾问工具和记忆工具，请在 `client.Beta.Messages.New` 中使用 beta 命名空间下的 `BetaAdvisorTool20260301Param` 和 `BetaMemoryTool20250818Param`。

### 顾问工具（beta 版）

服务器端运行，无需 `tool_result` 的往返交互。顾问模型的版本必须不低于执行模型（顶级模型）；不合法的组合将返回 400 错误。

```go
response, err := client.Beta.Messages.New(ctx, anthropic.BetaMessageNewParams{
    Model:     "claude-sonnet-5-5", // 执行模型
    MaxTokens: 4096,
    Tools: []anthropic.BetaToolUnionParam{
        {OfAdvisorTool20260301: &anthropic.BetaAdvisorTool20260301Param{
            Model: "claude-opus-5-5", // 顾问模型
        }},
    },
    Messages: []anthropic.BetaMessageParam{ /* ... */ },
    Betas:    []anthropic.AnthropicBeta{anthropic.AnthropicBetaAdvisorTool2026_03_01},
})
```

---