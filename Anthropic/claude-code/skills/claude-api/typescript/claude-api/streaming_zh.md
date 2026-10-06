# 流式传输 - TypeScript

## 快速入门

```typescript
const stream = client.messages.stream({
  model: "claude-opus-5-5",
  max_tokens: 64000,
  messages: [{ role: "user", content: "写一个故事" }],
});

for await (const event of stream) {
  if (
    event.type === "content_block_delta" &&
    event.delta.type === "text_delta"
  ) {
    process.stdout.write(event.delta.text);
  }
}
```

---

## 处理不同内容类型

> **Fable 5 / Claude Opus 5.5 / Claude Opus 5 / Opus 4.8 / Opus 4.7 / Opus 4.6：** 使用 `thinking: {type: "adaptive"}`。在 Claude Opus 5.5 和 Claude Opus 5 上，省略 `thinking` 参数也会启用自适应模式（Claude Opus 5.5 不接受其他设置——`disabled` 和 `budget_tokens` 都为 400）。在较旧的模型上，请使用 `thinking: {type: "enabled", budget_tokens: N}`。

```typescript
const stream = client.messages.stream({
  model: "claude-opus-5-5",
  max_tokens: 64000,
  thinking: { type: "adaptive", display: "summarized" }, // 显示选项：默认情况下，在 Fable 5/5.1、Mythos 5/5.1、Claude Opus 5.5、Claude Opus 5、Opus 4.8/4.7、Claude Sonnet 5.5 和 Claude Sonnet 5 上会省略思考内容（显示为空）。
  messages: [{ role: "user", content: "分析这个问题" }],
});

for await (const event of stream) {
  switch (event.type) {
    case "content_block_start":
      switch (event.content_block.type) {
        case "thinking":
          console.log("\n[思考中...]");
          break;
        case "text":
          console.log("\n[回答:]");
          break;
      }
      break;
    case "content_block_delta":
      switch (event.delta.type) {
        case "thinking_delta":
          process.stdout.write(event.delta.thinking);
          break;
        case "text_delta":
          process.stdout.write(event.delta.text);
          break;
      }
      break;
  }
}
```

---

## 带工具使用的流式传输（工具运行器）

将工具运行器与 `stream: true` 一起使用。外层循环遍历工具运行器的迭代（消息），内层循环处理流式事件。`betaZodTool()` 没有 `eager_input_streaming` 选项，因此需要将其扩展到返回的工具对象上；否则，API 会缓冲每个工具输入参数，`input_json_delta` 会在最后一次性到达（默认规则：参见 `shared/tool-use-concepts.md` -> 紧急输入流式传输）：

```typescript
import Anthropic from "@anthropic-ai/sdk";
import { betaZodTool } from "@anthropic-ai/sdk/helpers/beta/zod";
import { z } from "zod";

const client = new Anthropic();

const getWeather = {
  ...betaZodTool({
    name: "get_weather",
    description: "获取某地当前天气",
    inputSchema: z.object({
      location: z.string().describe("城市和州，例如旧金山, CA"),
    }),
    run: async ({ location }) => `72°F 并且晴朗，在 ${location}`,
  }),
  eager_input_streaming: true, // 在工具输入生成时即开始流式传输
};

let runner = client.beta.messages.toolRunner({
  model: "claude-opus-5-5",
  max_tokens: 64000,
  tools: [getWeather],
  messages: [
    { role: "user", content: "巴黎和伦敦的天气如何？" },
  ],
  stream: true,
});

// 启用紧急输入流式传输后，SDK 会在每个工具输入块关闭时对其进行解析。工具运行器会根据 Zod 模式验证输入，如果输入无效则不会调用 `run()` 方法；无法解析的 JSON 将导致该次迭代被拒绝。仅在这种情况下才重新发起请求——API 错误会被重新抛出——并限制连续失败次数。已消耗的工具运行器不可再次迭代，因此重试时会根据 `runner.params` 构建一个新的运行器，其中包含到目前为止的对话内容（失败的那一轮并未添加），因此已完成的工具调用不会被重复执行。
//
// 工具运行器不会自动应用停止条件规则：请在每次迭代前检查 `stop_reason`，然后再让工具运行器执行该轮的工具调用。
class TruncatedToolInput extends Error {}
for (let attempt = 0; ; attempt++) {
  try {
    // 外层循环：每次工具运行器迭代
    for await (const messageStream of runner) {
      // 内层循环：处理本次迭代的流事件
      for await (const event of messageStream) {
        switch (event.type) {
          case "content_block_delta":
            switch (event.delta.type) {
              case "text_delta":
                process.stdout.write(event.delta.text);
                break;
              case "input_json_delta":
                // 工具输入片段——在预取式流传输中会立即到达
                process.stdout.write(event.delta.partial_json);
                break;
            }
            break;
        }
      }
      const message = await messageStream.finalMessage();
      attempt = 0; // 当前轮次已完成，连续失败计数重置
      // 即使工具输入被截断也可能通过模式校验，因此在工具运行器执行之前就停止；拒绝可能会导致工具调用中途中断，所以该轮次的工具都不再执行。pause_turn 不会由工具运行器自动恢复——参见 tool-use.md -> 服务器端工具。
      const hasToolUse = message.content.some((b) => b.type === "tool_use");
      if (message.stop_reason === "max_tokens" && hasToolUse) {
        throw new TruncatedToolInput("工具输入被截断；请提高 max_tokens 后重试");
      }
      if (message.stop_reason === "refusal") break;
      // 对纯文本回答使用 max_tokens 只会以截断的文本结束循环；工具运行器会将其作为最终消息返回。
    }
    break;
  } catch (err) {
    if (err instanceof Anthropic.APIError || err instanceof TruncatedToolInput || attempt >= 2) {
      throw err;
    }
    console.error("工具输入不是可解析的 JSON，正在重新发起本轮对话");
    runner = client.beta.messages.toolRunner({ ...runner.params });
  }
}
```

使用 `betaZodTool` 时，工具运行器会在调用 `run` 之前根据 Zod 模式对每个工具输入进行校验（而 `betaTool()` 的 JSON Schema 工具不会在运行时校验——需在 `run` 中自行校验），这样可以捕获那些宽松解析器允许通过的格式错误输入（如缺少字段或字段类型错误）；对于完全无法解析的 JSON，`for await` 循环会直接抛出异常，因此需要在外层捕获并重新抛出 API 错误，同时基于 `runner.params` 构建新的工具运行器，并设置连续失败次数上限——由于 `tool_use` 块未完成，也就没有 `tool_use_id` 可用于返回带有 `is_error` 标志的结果。停止原因的判断规则由您自行决定，而非由工具运行器控制：在调用 `finalMessage()` 后检查每一轮的 `stop_reason`——当本轮包含 `tool_use` 时，若因 `max_tokens` 停止（截断的输入可能仍能通过模式校验；截断的文本回答则直接返回），应停止；遇到 `refusal` 也应停止；至于 `pause_turn`，则需由您手动恢复（工具运行器不会自动恢复——参见 `tool-use.md` -> 使用工具运行器的服务器端工具，以及 `shared/tool-use-concepts.md` -> 预取式输入流）。

---

## 获取最终消息

```typescript
const stream = client.messages.stream({
  model: "claude-opus-5-5",
  max_tokens: 64000,
  messages: [{ role: "user", content: "你好" }],
});

for await (const event of stream) {
  // 处理事件...
}

const finalMessage = await stream.finalMessage();
console.log(`已使用 token 数：${finalMessage.usage.output_tokens}`);
```

---

## 流事件类型

| 事件类型            | 描述                 | 触发时机                     |
| --------------------- | --------------------------- | --------------------------------- |
| `message_start`       | 包含消息元数据   | 开始时触发一次             |
| `content_block_start` | 新内容块开始     | 文本或工具调用块开始时触发 |
| `content_block_delta` | 内容增量更新     | 每个 token 或数据块时触发   |
| `content_block_stop`  | 内容块结束       | 块结束时触发               |
| `message_delta`       | 消息级更新       | 包含 `stop_reason` 和用量信息 |
| `message_stop`        | 消息结束         | 结束时触发一次             |

## 最佳实践

1. **始终刷新输出**——使用 `process.stdout.write()` 实现即时显示
2. **处理部分响应**——如果流中断，可能会得到不完整的内容
3. **跟踪 token 使用量**——`message_delta` 事件包含用量信息
4. **使用 `finalMessage()`**——即使在流式传输过程中，也能获取完整的 `Anthropic.Message` 对象。不要将 `.on()` 事件包装成 `new Promise()`——`finalMessage()` 在内部已处理所有完成、错误和中止状态
5. **为 Web 界面缓冲**——考虑在渲染前先缓冲几个 token，以避免频繁操作 DOM
6. **使用 `stream.on("text", ...)` 处理增量**——`text` 事件直接提供增量字符串，比手动过滤 `content_block_delta` 事件更简单
7. **用于带流式传输的代理循环**——参见 `tool-use.md` 中的 [流式手动循环](./tool-use.md#streaming-manual-loop) 部分，了解如何将 `stream()` 和 `finalMessage()` 与工具调用循环结合使用

## 原始 SSE 格式

如果使用原生 HTTP（而非 SDK），流将以服务器发送事件（SSE）的形式返回：
```
事件：message_start
数据：{"type":"message_start","message":{"id":"msg_...","type":"message",...}}

事件：content_block_start
数据：{"type":"content_block_start","index":0,"content_block":{"type":"text","text":""}}

事件：content_block_delta
数据：{"type":"content_block_delta","index":0,"delta":{"type":"text_delta","text":"Hello"}}

事件：content_block_stop
数据：{"type":"content_block_stop","index":0}

事件：message_delta
数据：{"type":"message_delta","delta":{"stop_reason":"end_turn"},"usage":{"output_tokens":12}}

事件：message_stop
数据：{"type":"message_stop"}
```
