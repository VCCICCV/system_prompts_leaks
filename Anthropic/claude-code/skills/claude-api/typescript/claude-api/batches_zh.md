# 消息批处理 API - TypeScript

批处理 API（`POST /v1/messages/batches`）以标准价格的50%异步处理消息 API 请求。

## 核心要点

- 每个批次最多支持10万条请求或256 MB
- 大多数批次在1小时内完成，最长不超过24小时
- 结果在创建后可保留29天
- 所有令牌使用费用减半
- 支持消息 API 的所有功能（视觉、工具、缓存等）

---

## 创建一个批处理

```typescript
import Anthropic from "@anthropic-ai/sdk";

const client = new Anthropic();

const messageBatch = await client.messages.batches.create({
  requests: [
    {
      custom_id: "request-1",
      params: {
        model: "claude-opus-5-5",
        max_tokens: 16000,
        messages: [
          { role: "user", content: "总结气候变化的影响" },
        ],
      },
    },
    {
      custom_id: "request-2",
      params: {
        model: "claude-opus-5-5",
        max_tokens: 16000,
        messages: [
          { role: "user", content: "解释量子计算的基本原理" },
        ],
      },
    },
  ],
});

console.log(`批处理 ID: ${messageBatch.id}`);
console.log(`状态: ${messageBatch.processing_status}`);
```

---

## 轮询以获取完成状态

```typescript
let batch;
while (true) {
  batch = await client.messages.batches.retrieve(messageBatch.id);
  if (batch.processing_status === "ended") break;
  console.log(
    `状态: ${batch.processing_status}, 正在处理: ${batch.request_counts.processing}`,
  );
  await new Promise((resolve) => setTimeout(resolve, 60_000));
}

console.log("批处理已完成！");
console.log(`成功: ${batch.request_counts.succeeded}`);
console.log(`出错: ${batch.request_counts.errored}`);
```

---

## 获取结果

```typescript
for await (const result of await client.messages.batches.results(
  messageBatch.id,
)) {
  switch (result.result.type) {
    case "succeeded":
      console.log(
        `[${result.custom_id}] ${result.result.message.content[0].text.slice(0, 100)}`,
      );
      break;
    case "errored":
      if (result.result.error.type === "invalid_request") {
        console.log(`[${result.custom_id}] 验证错误 - 请修复后重试`);
      } else {
        console.log(`[${result.custom_id}] 服务器错误 - 可安全重试`);
      }
      break;
    case "expired":
      console.log(`[${result.custom_id}] 已过期 - 请重新提交`);
      break;
  }
}
```

---

## 取消一个批处理

```typescript
const cancelled = await client.messages.batches.cancel(messageBatch.id);
console.log(`状态: ${cancelled.processing_status}`); // "canceling"
```
