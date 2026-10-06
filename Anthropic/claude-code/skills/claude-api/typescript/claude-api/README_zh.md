# Claude API - TypeScript

| 功能 | 命名空间 | 关键类型/调用 |
|---|---|---|
| 用户档案 | beta | `client.beta.userProfiles.create(...)` / `.retrieve(id)` / `.list()`。在调用 `client.beta.messages.create` 时，需传入返回的用户档案 ID。需要使用 beta 标头——请参阅 SDK 的 beta 标头参考以获取当前的启用标志。 |

## 安装

```bash
npm install @anthropic-ai/sdk
```

> **读取本地文件（ESM）：** 在 ES 模块中，`__dirname` 和 `__filename` 是**未定义**的——使用它们会在运行时抛出 `ReferenceError: __dirname is not defined` 错误。对于相对于工作目录的读取，请直接传入相对路径（`fs.readFileSync("./sample.png")`）。对于相对于脚本的路径，可从 `import.meta.url` 中推导出目录：`const here = path.dirname(fileURLToPath(import.meta.url))`。切勿在 ESM 的 `.ts` 文件中编写 `path.join(__dirname, ...)`。

## 客户端初始化

```typescript
import Anthropic from "@anthropic-ai/sdk";

// 默认方式——从环境变量中解析凭据：
// ANTHROPIC_API_KEY、ANTHROPIC_AUTH_TOKEN，或通过 `ant auth login` 登录的配置文件。
// 推荐在本地开发时使用此方式，不要硬编码 API 密钥。
const client = new Anthropic();

// 显式指定 API 密钥（仅在必须注入特定密钥时使用）
const client = new Anthropic({ apiKey: "your-api-key" });
```

---

## 基本消息请求

```typescript
const response = await client.messages.create({
  model: "claude-opus-5-5",
  max_tokens: 16000,
  messages: [{ role: "user", content: "法国的首都是哪里？" }],
});
// response.content 是 ContentBlock[]——一种联合类型。在访问 .text 之前，请先根据 .type 进行类型收窄（否则 TypeScript 会报错）。
for (const block of response.content) {
  if (block.type === "text") {
    console.log(block.text);
  }
}
```

---

## 系统提示

```typescript
const response = await client.messages.create({
  model: "claude-opus-5-5",
  max_tokens: 16000,
  system:
    "你是一位乐于助人的编程助手。请始终提供 Python 示例。",
  messages: [{ role: "user", content: "如何读取 JSON 文件？" }],
});
```

### 对话中途的系统消息（模型控制）

对于在对话中途出现的操作指令（模式切换、状态注入），应将 `{role: "system", ...}` 添加到 `messages` 数组中，而不是修改顶层的 `system` 字段——这样可以保留缓存的前缀，并确保操作者的权威性。此类消息必须紧跟在用户消息（或以服务工具调用结尾的助手消息）之后，且必须是 `messages` 数组中的最后一项，或者后面紧跟着一条助手消息；不能作为 `messages[0]`。不支持该功能的模型会返回 400 错误（“此模型不支持 role 'system'”）。有关何时使用这种方式以及何时使用顶层 `system`，请参阅 `shared/prompt-caching.md`。

```typescript
// 不需要 beta 标头——使用常规的 client.messages.create 即可。
const response = await client.messages.create({
  model: MODEL_ID, // 必须支持对话中途的系统消息
  max_tokens: 16000,
  system: [
    { type: "text", text: STABLE_SYSTEM, cache_control: { type: "ephemeral" } },
  ],
  messages: [
    ...history,
    { role: "user", content: userMessage },
    { role: "system", content: "已启用简洁模式——回复字数请控制在 40 字以内。" },
  ],
});
```

---

## 视觉（图像）

### URL 方式

```typescript
const response = await client.messages.create({
  model: "claude-opus-5-5",
  max_tokens: 16000,
  messages: [
    {
      role: "user",
      content: [
        {
          type: "image",
          source: { type: "url", url: "https://example.com/image.png" },
        },
        { type: "text", text: "请描述这张图片" },
      ],
    },
  ],
});
```

### Base64 方式

```typescript
import fs from "fs";

const imageData = fs.readFileSync("image.png").toString("base64");

const response = await client.messages.create({
  model: "claude-opus-5-5",
  max_tokens: 16000,
  messages: [
    {
      role: "user",
      content: [
        {
          type: "image",
          source: { type: "base64", media_type: "image/png", data: imageData },
        },
        { type: "text", text: "这张图片里有什么？" },
      ],
    },
  ],
});
```

---

## 提示词缓存

**缓存采用前缀匹配**——只要前缀中任意一处发生字节级变化，其后的所有内容都会失效。有关放置模式、架构指导（冻结的系统提示、确定性的工具调用顺序、易变内容的放置位置）以及静默失效审计清单，请参阅 `shared/prompt-caching.md`。

### 自动缓存（推荐）

使用顶级 `cache_control` 可自动缓存请求中最后一个可缓存的内容块：

```typescript
const response = await client.messages.create({
  model: "claude-opus-5-5",
  max_tokens: 16000,
  cache_control: { type: "ephemeral" }, // 自动缓存最后一个可缓存的内容块
  system: "您是这份大型文档的专家……",
  messages: [{ role: "user", content: "请总结要点" }],
});
```

### 手动缓存控制

如需更精细的控制，可在特定内容块上添加 `cache_control`：

```typescript
const response = await client.messages.create({
  model: "claude-opus-5-5",
  max_tokens: 16000,
  system: [
    {
      type: "text",
      text: "您是这份大型文档的专家……",
      cache_control: { type: "ephemeral" }, // 默认 TTL 为 5 分钟
    },
  ],
  messages: [{ role: "user", content: "请总结要点" }],
});
```// 使用显式 TTL（生存时间）
const response2 = await client.messages.create({
  model: "claude-opus-5-5",
  max_tokens: 16000,
  system: [
    {
      type: "text",
      text: "您是这份大型文档的专家……",
      cache_control: { type: "ephemeral", ttl: "1h" }, // 1小时 TTL
    },
  ],
  messages: [{ role: "user", content: "总结关键要点" }],
});
```

### 验证缓存命中情况

```typescript
console.log(response.usage.cache_creation_input_tokens); // 写入缓存的 token 数量（约 1.25 倍成本）
console.log(response.usage.cache_read_input_tokens);     // 从缓存中读取的 token 数量（约 0.1 倍成本）
console.log(response.usage.input_tokens);                // 未缓存的 token 数量（全额成本）
```

如果在重复的、前缀相同的请求中，`cache_read_input_tokens` 均为零，则说明存在无声失效机制——可能是系统提示中使用了 `Date.now()` 或 UUID、键的非确定性排序，或工具集发生变化。完整的审计表请参见 `shared/prompt-caching.md`。

---

## 扩展思考

> **Fable 5、Claude Opus 5.5、Claude Opus 5、Opus 4.8、Opus 4.7、Opus 4.6 和 Sonnet 4.6：** 使用自适应思维。在 Fable 5、Claude Opus 5.5、Claude Opus 5、Opus 4.8 和 4.7 中，`budget_tokens` 已被移除（若发送则会被忽略）；在 Opus 4.6 和 Sonnet 4.6 中已被弃用。  
> **Claude Opus 5.5：** 思维始终开启——可省略 `thinking` 参数（或发送 `{ type: "adaptive" }`，两者等效）；发送 `{ type: "disabled" }` 在任何努力级别都会返回 400 错误，设置思维预算也是如此。应改用 `output_config.effort` 来控制深度——该模型的默认值为 `medium`，而 Claude Opus 5 的默认值为 `high`。  
> **Claude Opus 5：** 默认启用思维——省略 `thinking` 参数时会自动启用自适应模式（即等同于 `{ type: "adaptive" }`），这与 Opus 4.8/4.7 不同，在后者中省略则表示不启用思维。只有在努力级别为 `high` 或更低时才接受 `{ type: "disabled" }`；若将其与 `xhigh` 或 `max` 搭配，则会返回 400 错误。  
> **较旧的模型：** 使用 `thinking: {type: "enabled", budget_tokens: N}`（必须小于 `max_tokens`，且最小为 1024）。```typescript
// Fable 5 / Claude Opus 5.5 / Claude Opus 5 / Opus 4.8 / 4.7 / 4.6：自适应思维（推荐）
const response = await client.messages.create({
  model: "claude-opus-5-5",
  max_tokens: 16000,
  thinking: { type: "adaptive", display: "summarized" }, // 显示选项：在 Fable 5/5.1、Mythos 5/5.1、Claude Opus 5.5、Claude Opus 5、Opus 4.8/4.7、Claude Sonnet 5.5 和 Claude Sonnet 5 上，默认会省略（思维内容为空）
  output_config: { effort: "high" }, // low | medium | high | xhigh | max
  messages: [
    { role: "user", content: "请逐步解答这道数学题..." },
  ],
});

for (const block of response.content) {
  if (block.type === "thinking") {
    console.log("思考过程:", block.thinking);
  } else if (block.type === "text") {
    console.log("响应:", block.text);
  }
}
```

---

## 错误处理

请使用 SDK 的类型化异常类，切勿通过字符串匹配来检查错误信息：

```typescript
import Anthropic from "@anthropic-ai/sdk";

try {
  const response = await client.messages.create({...});
} catch (error) {
  if (error instanceof Anthropic.BadRequestError) {
    console.error("请求错误:", error.message);
  } else if (error instanceof Anthropic.AuthenticationError) {
    console.error("无效的 API 密钥");
  } else if (error instanceof Anthropic.RateLimitError) {
    console.error("已达到速率限制 - 请稍后重试");
  } else if (error instanceof Anthropic.APIError) {
    console.error(`API 错误 ${error.status}:`, error.message);
  }
}
```

所有类都继承自 `Anthropic.APIError`，并带有类型化的 `status` 字段。请按照从具体到一般的顺序进行检查。完整的错误码参考请参见 [shared/error-codes.md](../../shared/error-codes.md)。

---

## 多轮对话

API 是无状态的——每次都需要发送完整的对话历史。可以使用 `Anthropic.MessageParam[]` 来为消息数组添加类型：

```typescript
const messages: Anthropic.MessageParam[] = [
  { role: "user", content: "我叫 Alice。" },
  { role: "assistant", content: "你好，Alice！很高兴认识你。" },
  { role: "user", content: "我叫什么名字？" },
];

const response = await client.messages.create({
  model: "claude-opus-5-5",
  max_tokens: 16000,
  messages: messages,
});
```

**规则：**

- 允许连续的同角色消息——API 会将它们合并为一轮对话
- 第一条消息必须是 `user`
- 对于所有 API 数据结构，请使用 SDK 类型（如 `Anthropic.MessageParam`、`Anthropic.Message`、`Anthropic.Tool` 等），不要重新定义等效的接口

---

### 压缩功能（长对话）

> **测试版，适用于 Fable 5、Claude Opus 5.5、Claude Opus 5、Opus 4.8、Opus 4.7、Opus 4.6 和 Sonnet 4.6。** 当对话接近 20 万上下文窗口时，压缩功能会在服务器端自动对早期上下文进行摘要。API 会返回一个 `compaction` 块；在后续请求中，您必须将其原样传回——追加的是整个 `response.content`，而不仅仅是文本内容。

```typescript
import Anthropic from "@anthropic-ai/sdk";

const client = new Anthropic();
const messages: Anthropic.Beta.BetaMessageParam[] = [];

async function chat(userMessage: string): Promise<string> {
  messages.push({ role: "user", content: userMessage });

  const response = await client.beta.messages.create({
    betas: ["compact-2026-01-12"],
    model: "claude-opus-5-5",
    max_tokens: 16000,
    messages,
    context_management: {
      edits: [{ type: "compact_20260112" }],
    },
  });

  // 追加完整内容——压缩块必须保留
  messages.push({ role: "assistant", content: response.content });

  const textBlock = response.content.find(
    (b): b is Anthropic.Beta.BetaTextBlock => b.type === "text",
  );
  return textBlock?.text ?? "";
}

// 当上下文规模较大时，压缩功能会自动触发
console.log(await chat("帮我构建一个 Python 爬虫"));
console.log(await chat("增加对 JavaScript 渲染页面的支持"));
console.log(await chat("现在再加上限速和错误处理"));
```

---

## 停止原因响应中的 `stop_reason` 字段指示模型停止生成的原因：

| 值           | 含义                                                         |
| --------------- | --------------------------------------------------------------- |
| `end_turn`      | Claude 自然结束其回复                                          |
| `max_tokens`    | 达到 `max_tokens` 限制 - 可以调大该值或使用流式输出             |
| `stop_sequence` | 匹配了自定义停止序列                                           |
| `tool_use`      | Claude 想调用工具 - 执行该工具后继续生成                       |
| `pause_turn`    | 模型暂停，可恢复（代理流程）                                   |
| `refusal`       | Claude 因安全原因拒绝 - 请查看 `stop_details`                  |

### 结构化停止详情

当 `stop_reason` 为 `"refusal"` 时，响应中会包含一个 `stop_details` 对象，提供关于拒绝的结构化信息：

```typescript
if (response.stop_reason === "refusal" && response.stop_details) {
  console.log(`类别: ${response.stop_details.category}`); // 例如 "cyber"、"bio"、"reasoning_extraction"、"frontier_llm"，或 null——详见文档获取完整列表
  console.log(`解释: ${response.stop_details.explanation}`);
}
```

### 拒绝回退机制（Claude Fable 5.1）——默认启用

回退机制是**可选的**：若不启用，被拒绝的请求将直接停止。在 `claude-fable-5-1` 中，默认应在服务端代码中包含 `fallbacks` 参数；当策略判定拒绝时，API 会在同一调用中使用回退模型重新执行相同请求。中途拒绝仍按正常费率计费，而救援部分则按回退模型的费率计费，并自动应用缓存重定价；若在任何输出产生之前即被拒绝，请参阅[拒绝的计费方式](https://platform.claude.com/docs/en/build-with-claude/refusals-and-fallback#how-refusals-are-billed)。

```typescript
const response = await client.beta.messages.create({
  model: "claude-fable-5-1",
  max_tokens: 16000,
  betas: ["server-side-fallback-2026-06-01"],
  fallbacks: [{ model: "claude-opus-4-8" }],
  messages: [{ role: "user", content: "..." }],
});

// 切换点：每运行一次并被拒绝的模型都会对应一个回退块
for (const block of response.content) {
  if (block.type === "fallback") {
    console.log(`${block.from.model} 被拒绝；${block.to.model} 继续`);
  }
}

// 服务来源标识——适用于持续回合，此类情况下不会出现回退块。
// 配合 stop_reason 使用：回退模型本身也可能拒绝。
const fallbackRan = (response.usage.iterations ?? []).some(
  (entry) => entry.type === "fallback_message",
);
if (fallbackRan && response.stop_reason !== "refusal") {
  console.log(`由 ${response.model} 提供服务`);
}
```

最终响应中的 `stop_reason: "refusal"` 表示整个链条都被拒绝。对于这种数组形式，标头必须精确为 `server-side-fallback-2026-06-01`；较新的标量形式 `fallbacks: "default"` 则使用 `server-side-fallback-2026-07-01`（详见 `shared/model-migration.md` -> 迁移到 Claude Opus 5 -> 新 API 功能），并将两种标头混用会导致 400 错误。该参数在 Batches API 上被拒绝，在 Amazon Bedrock、Vertex AI 和 Microsoft Foundry 上不可用——可在这些平台上改用客户端的 `betaRefusalFallbackMiddleware`。完整的语义（包括粘性路由、计费、流式传输以及回退回合的回显）详见 `shared/model-migration.md` -> 迁移到 Claude Fable 5.1 -> `refusal` 停止原因。

---

## 成本优化策略

### 1. 对重复上下文使用提示缓存

```typescript
// 自动缓存（最简单——仅缓存最后一个可缓存块）
const response = await client.messages.create({
  model: "claude-opus-5-5",
  max_tokens: 16000,
  cache_control: { type: "ephemeral" },
  system: largeDocumentText, // 例如 50KB 的上下文
  messages: [{ role: "user", content: "总结要点" }],
});
// 第一次请求：全额费用
// 后续请求：对于已缓存的部分，费用约便宜90%
```

### 2. 在发送请求前进行令牌计数

```typescript
const countResponse = await client.messages.countTokens({
  model: "claude-opus-5-5",
  messages: messages,
  system: system,
});

const estimatedInputCost = countResponse.input_tokens * 0.000004; // 每100万令牌4美元
console.log(`预计输入成本：$${estimatedInputCost.toFixed(4)}`);
```
