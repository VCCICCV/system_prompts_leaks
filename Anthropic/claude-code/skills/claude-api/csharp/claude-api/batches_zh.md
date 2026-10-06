# 消息批处理 - C#

## 消息批处理 API

```csharp
var batch = await client.Messages.Batches.Create(new() {
    Requests = [
        new() { CustomID = "req-1", Params = new() { Model = "claude-opus-5-5", MaxTokens = 1024, Messages = [...] } },
    ],
});
// 轮询 client.Messages.Batches.Retrieve(batch.ID)，直到 ProcessingStatus 等于 "ended"，
// 然后遍历 client.Messages.Batches.Results(batch.ID)。
```

