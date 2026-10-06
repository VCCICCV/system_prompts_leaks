# Claude API - Go

> **注意：** Go SDK 支持使用 `BetaToolRunner` 调用 Claude API 和 Beta 工具。目前，Go 尚未提供 Agent SDK。

## 安装

```bash
go get github.com/anthropics/anthropic-sdk-go
```

## 客户端初始化

```go
import (
    "github.com/anthropics/anthropic-sdk-go"
    "github.com/anthropics/anthropic-sdk-go/option"
)

// 默认方式（使用 ANTHROPIC_API_KEY 环境变量）
client := anthropic.NewClient()

// 显式指定 API 密钥
client := anthropic.NewClient(
    option.WithAPIKey("your-api-key"),
)
```

---

## 模型 ID

`anthropic.Model` 是 `string` 类型的别名，因此只需传入模型的普通 ID 即可，例如：`Model: "claude-opus-5-5"`。默认使用 Claude Opus 5.5，除非用户另有指定；如果用户要求使用 Fable 或最强大的模型，则使用 `"claude-fable-5-1"`；如果用户希望选择更经济的版本，则使用当前代次的模型——`"claude-sonnet-5-5"` 或 `"claude-haiku-4-5"`（完整对应表请参见 `shared/models.md`）。

SDK 还提供了类型化的 `anthropic.ModelClaude*` 常量，但这些常量的更新会滞后于新模型的发布——某个 SDK 版本可能只包含上一代模型的常量。请勿仅因为某个模型有对应的类型化常量就选择该模型；字符串形式的模型 ID 在所有 SDK 版本中均适用。在假设某个当前模型存在类型化常量之前，请先查看 SDK 的发布说明。

---

## 基本消息请求

```go
response, err := client.Messages.New(context.Background(), anthropic.MessageNewParams{
    Model:     "claude-opus-5-5",
    MaxTokens: 16000,
    Messages: []anthropic.MessageParam{
        anthropic.NewUserMessage(anthropic.NewTextBlock("法国的首都是哪里？")),
    },
})
if err != nil {
    log.Fatal(err)
}
for _, block := range response.Content {
    switch variant := block.AsAny().(type) {
    case anthropic.TextBlock:
        fmt.Println(variant.Text)
    }
}
```

---

## 思考模式

通过在 `MessageNewParams` 中设置 `Thinking`，可以启用 Claude 的内部推理功能。响应内容中会在最终的 `TextBlock` 之前包含 `ThinkingBlock`。

**对于 Claude 4.6 及更高版本的模型，推荐使用自适应思考模式。** Claude 会动态决定何时以及进行多少思考。结合 `effort` 参数，可以实现成本与质量的平衡。

相关定义位于 `anthropic-sdk-go/message.go` 中（`ThinkingConfigParamUnion`、`ThinkingConfigAdaptiveParam`）。

```go
// 没有 `ThinkingConfigParamOfAdaptive` 辅助函数，需直接构造联合体结构体字面量并取其变体的地址。
// 显示选项：在 Fable 5/5.1、Mythos 5/5.1、Claude Opus 5.5、Claude Opus 5、Opus 4.8/4.7、Claude Sonnet 5.5 和 Claude Sonnet 5 上，默认为省略（即不显示思考文本）。
adaptive := anthropic.ThinkingConfigAdaptiveParam{Display: anthropic.ThinkingConfigAdaptiveDisplaySummarized}
params := anthropic.MessageNewParams{
    Model:     "claude-opus-5-5",
    MaxTokens: 16000,
    Thinking:  anthropic.ThinkingConfigParamUnion{OfAdaptive: &adaptive},
    Messages: []anthropic.MessageParam{
        anthropic.NewUserMessage(anthropic.NewTextBlock("strawberry 中有多少个 r？")),
    },
}

resp, err := client.Messages.New(context.Background(), params)
if err != nil {
    log.Fatal(err)
}

// 内容中，ThinkingBlock 会出现在 TextBlock 之前
for _, block := range resp.Content {
    switch b := block.AsAny().(type) {
    case anthropic.ThinkingBlock:
        fmt.Println("[thinking]", b.Thinking)
    case anthropic.TextBlock:
        fmt.Println(b.Text)
    }
}
```

> **Fable 5、Claude Opus 5.5、Claude Opus 5、Opus 4.8、Opus 4.7、Opus 4.6 和 Sonnet 4.6：** 使用自适应思维（见上文）。在 Fable 5、Claude Opus 5.5、Claude Opus 5、Opus 4.8 和 4.7 中，`ThinkingConfigParamOfEnabled(budgetTokens)` 已被移除（若发送则返回 400 错误）；在 Opus 4.6 和 Sonnet 4.6 中已被弃用。  
> **Claude Opus 5.5：** 思维始终开启——请保持 `Thinking` 参数未设置（或发送等效的自适应联合参数）；`OfDisabled` 在任何努力级别都会返回 400 错误，`ThinkingConfigParamOfEnabled` 同样如此。改用 `OutputConfig` 下的 `Effort` 参数来控制深度——该模型的默认值为 `medium`，而 Claude Opus 5 的默认值为 `high`。  
> **Claude Opus 5：** 默认开启思维——保持 `Thinking` 参数未设置即启用自适应模式（自适应联合参数与此等效），这与 Opus 4.8/4.7 不同，在后者中未设置该参数意味着不启用思维。  
> **较旧模型：** 使用 `anthropic.ThinkingConfigParamOfEnabled(N)`（预算必须小于 `MaxTokens`，最低 1024）。

要禁用：`anthropic.ThinkingConfigParamUnion{OfDisabled: &anthropic.ThinkingConfigDisabledParam{}}`。在 Claude Opus 5 上，此设置仅在努力级别为 `high` 或更低时有效——若搭配 `xhigh` 或 `max` 则会返回 400 错误；而在 Claude Opus 5.5 上，无论何种努力级别均会返回 400 错误，请改用较低的 `Effort` 设置。

---

## 提示缓存

`System` 是 `[]TextBlockParam`；在最后一个块上设置 `CacheControl` 可将工具和系统一同缓存。关于放置模式及静默失效器审计清单，请参阅 `shared/prompt-caching.md`。

```go
System: []anthropic.TextBlockParam{{
    Text:         longSystemPrompt,
    CacheControl: anthropic.NewCacheControlEphemeralParam(), // 默认 TTL 为 5 分钟
}},
```

若需 1 小时 TTL：`anthropic.CacheControlEphemeralParam{TTL: anthropic.CacheControlEphemeralTTLTTL1h}`。此外，在 `MessageNewParams` 上还有一个顶层的 `CacheControl`，可自动应用于最后一个可缓存的块。

可通过 `resp.Usage.CacheCreationInputTokens` 和 `resp.Usage.CacheReadInputTokens` 验证缓存命中情况。

---

## 停止详情

当 `StopReason` 为 `anthropic.StopReasonRefusal` 时，响应中会包含结构化的 `StopDetails`：

```go
if resp.StopReason == anthropic.StopReasonRefusal {
    fmt.Println("类别:", resp.StopDetails.Category)     // 例如 "cyber"、"bio"、"reasoning_extraction"、"frontier_llm"，或 ""——完整列表请参阅文档
    fmt.Println("解释:", resp.StopDetails.Explanation)
}
```

**拒绝回退（Claude Fable 5.1）——默认启用。** 回退功能是可选的：若不启用，被拒绝的请求将直接停止。在 `claude-fable-5-1` 代码中，默认应包含 `Fallbacks: []anthropic.BetaFallbackParam{{Model: "claude-opus-4-8"}}`，并在 `client.Beta.Messages.New` 中启用 `anthropic.AnthropicBetaServerSideFallback2026_06_01` 测试版——对于政策性拒绝，将在同一调用内由回退模型重新处理。该功能在 Amazon Bedrock、Vertex AI 和 Microsoft Foundry 上不可用——可在这些平台上注册客户端中间件：`option.WithMiddleware(betafallback.BetaRefusalFallbackMiddleware(...))`，来自 `lib/betafallback`，并通过 `betafallback.WithBetaFallbackState(&betafallback.BetaFallbackState{})` 维护每轮对话的状态。完整的语义（计费、粘性路由、流式传输）及可运行示例，请参阅 `shared/model-migration.md` 中的“迁移到 Claude Fable 5.1”部分，以及 Go SDK 仓库中的 `examples/` 目录（通过 `shared/live-sources.md` 实现 WebFetch）。

---

## PDF / 文档输入

通用辅助函数 `NewDocumentBlock` 接受任意来源类型，`MediaType` 和 `Type` 会自动设置。

```go
b64 := base64.StdEncoding.EncodeToString(pdfBytes)

msg := anthropic.NewUserMessage(
    anthropic.NewDocumentBlock(anthropic.Base64PDFSourceParam{Data: b64}),
    anthropic.NewTextBlock("总结这份文档"),
)
```

其他来源：`URLPDFSourceParam{URL: "https://..."}`、`PlainTextSourceParam{Data: "..."}`。

---

## 上下文编辑 / 精简（测试版）

使用带有 `ContextManagement` 的 `Beta.Messages.New`，相关参数位于 `BetaMessageNewParams` 中。没有专门的 `NewBetaAssistantMessage`——请使用 `.ToParam()` 进行往返转换。

```go
params := anthropic.BetaMessageNewParams{
    Model:     "claude-opus-5-5",
    MaxTokens: 16000,
    Betas:     []anthropic.AnthropicBeta{"compact-2026-01-12"},
    ContextManagement: anthropic.BetaContextManagementConfigParam{
        Edits: []anthropic.BetaContextManagementConfigEditUnionParam{
            {OfCompact20260112: &anthropic.BetaCompact20260112EditParam{}},
        },
    },
    Messages: []anthropic.BetaMessageParam{ /* ... */ },
}

resp, err := client.Beta.Messages.New(ctx, params)
if err != nil {
    log.Fatal(err)
}

// 往返转换：通过 .ToParam() 将响应追加到历史记录中
params.Messages = append(params.Messages, resp.ToParam())
// 从响应中读取压缩块
for _, block := range resp.Content {
    if c, ok := block.AsAny().(anthropic.BetaCompactionBlock); ok {
        fmt.Println("压缩摘要:", c.Content)
    }
}
```

其他编辑类型：`BetaClearToolUses20250919EditParam`、`BetaClearThinking20251015EditParam`——这些需要设置 `Betas: []anthropic.AnthropicBeta{"context-management-2025-06-27"}`，而不是 `compact-2026-01-12`。
