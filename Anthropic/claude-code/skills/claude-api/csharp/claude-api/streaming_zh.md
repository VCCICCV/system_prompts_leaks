# 流式传输 - C#

## 流式传输

```csharp
using Anthropic.Models.Messages;

var parameters = new MessageCreateParams
{
    Model = "claude-opus-5-5",
    MaxTokens = 64000,
    Messages = [new() { Role = Role.User, Content = "写一首俳句" }]
};

await foreach (RawMessageStreamEvent streamEvent in client.Messages.CreateStreaming(parameters))
{
    if (streamEvent.TryPickContentBlockDelta(out var delta) &&
        delta.Delta.TryPickText(out var text))
    {
        Console.Write(text.Text);
    }
}
```

**`RawMessageStreamEvent` 的 TryPick 方法**（命名时去掉了 `Message` 和 `Raw` 前缀）：`TryPickStart`、`TryPickDelta`、`TryPickStop`、`TryPickContentBlockStart`、`TryPickContentBlockDelta`、`TryPickContentBlockStop`。没有 `TryPickMessageStop` 方法，请使用 `TryPickStop`。

---