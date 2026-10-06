# 工具使用 - TypeScript

有关概念性概述（工具定义、工具选择、提示），请参阅 [shared/tool-use-concepts.md](../../shared/tool-use-concepts.md)。

## 工具运行器（推荐）

**测试版：** 工具运行器在 TypeScript SDK 中处于测试阶段。

使用 `betaZodTool` 和 Zod 模式来定义带有 `run` 函数的工具，然后将其传递给 `client.beta.messages.toolRunner()`：

```typescript
import Anthropic from "@anthropic-ai/sdk";
import { betaZodTool } from "@anthropic-ai/sdk/helpers/beta/zod";
import { z } from "zod";

const client = new Anthropic();

const getWeather = betaZodTool({
  name: "get_weather",
  description: "获取某地当前天气",
  inputSchema: z.object({
    location: z.string().describe("城市和州，例如旧金山, CA"),
    unit: z.enum(["celsius", "fahrenheit"]).optional(),
  }),
  run: async (input) => {
    // 在此处实现您的逻辑
    return `72°F，晴朗，在${input.location}`;
  },
});

// 工具运行器负责处理代理循环并返回最终消息
const finalMessage = await client.beta.messages.toolRunner({
  model: "claude-opus-5-5",
  max_tokens: 16000,
  tools: [getWeather],
  messages: [{ role: "user", content: "巴黎的天气如何？" }],
});

console.log(finalMessage.content);
```

Zod 是可选的——如果您不想依赖 Zod，也可以使用 `@anthropic-ai/sdk/helpers/beta/json-schema` 中的 `betaTool()`，它接受原始 JSON Schema `inputSchema` 和 `run` 函数。

**工具运行器的主要优势：**

- 无需手动循环——SDK 会自动调用工具并将结果反馈回去
- 通过 Zod 模式（或通过 `betaTool()` 使用原始 JSON Schema）实现类型安全的工具输入
- 工具模式会根据 Zod 定义自动生成
- 当 Claude 不再有工具调用时，迭代会自动停止

### 使用工具运行器调用服务器端工具

运行器的 `tools` 数组除了可接收可执行工具外，还支持直接传入原始的服务器端工具定义（如 `web_search_20260209`、`web_fetch_20260209`、代码执行等）——只需传递该工具对象即可。服务器端工具在 Anthropic 的服务器上运行，因此不需要提供 `run` 函数。

**注意——截至 `@anthropic-ai/sdk` 0.110.0 版本，运行器不会自动恢复 `pause_turn` 状态。** 长时间运行的服务器端工具回合可能会以 `stop_reason: "pause_turn"` 结束。运行器只有在客户端工具产生结果后才会继续，因此暂停的回合会导致循环终止，并作为最终消息返回——既无错误，也无警告，只是答案被无声截断。如果在运行器中混用了服务器端工具，请在每次迭代时检查 `stop_reason`，并通过重新推送被暂停的助手回合来恢复：

```typescript
const params = {
  model: "claude-opus-5-5",
  max_tokens: 16000,
  tools: [getWeather, { type: "web_search_20260209", name: "web_search", max_uses: 5 }],
  messages: [{ role: "user", content: "比较两个来源对巴黎本周天气的预报" }],
};

const runner = client.beta.messages.toolRunner(params);

// 非流式：每次迭代都会返回一条完整消息
for await (const message of runner) {
  if (message.stop_reason === "pause_turn") {
    runner.pushMessages({ role: "assistant", content: message.content });
  }
}

// 流式替代方案——使用 `stream: true` 构建运行器（参数同上）。此时每次迭代返回的是一个流，而不是一条消息——仅检查 `message.stop_reason` 不会触发任何操作。需先解析流：
const streamingRunner = client.beta.messages.toolRunner({ ...params, stream: true });
for await (const stream of streamingRunner) {
  const message = await stream.finalMessage();
  if (message.stop_reason === "pause_turn") {
    streamingRunner.pushMessages({ role: "assistant", content: message.content });
  }
}
```

每次暂停与恢复都会消耗一次 `max_iterations` 计数，因此即使设置了最大迭代次数，运行仍可能以暂停状态结束——在信任结果前，请务必检查最终消息的 `stop_reason`（循环结束后，调用您所迭代的运行器上的 `.done()` 方法以获取最终消息）。或者，您也可以使用下文中的手动循环，它会显式处理 `pause_turn` 状态。---

## 手动代理循环

优先使用工具运行器。仅在需要控制运行器无法暴露的功能时才退回到手动循环（例如，自定义传输、SDK 无法构建的请求格式，或避免依赖某个处于 Beta 阶段的特性——运行器本身处于 Beta 状态，并且通过 `stream: true` 支持按 token 流式输出）。人机协作审批并不需要手动循环——可以在工具的 `run()` 函数内部设置拦截逻辑（返回“用户拒绝”的结果），或者在每次迭代之间检查待处理的 `tool_use` 块并调用 `setMessagesParams()`。

如果确实需要手动循环：

```typescript
import Anthropic from "@anthropic-ai/sdk";

const client = new Anthropic();
const tools: Anthropic.Tool[] = [...]; // 您的工具定义
let messages: Anthropic.MessageParam[] = [{ role: "user", content: userInput }];

while (true) {
  const response = await client.messages.create({
    model: "claude-opus-5-5",
    max_tokens: 16000,
    tools: tools,
    messages: messages,
  });

  if (response.stop_reason === "end_turn") break;

  // 服务端工具调用已达到迭代上限；追加助手回复后重新发送以继续
  if (response.stop_reason === "pause_turn") {
    messages.push({ role: "assistant", content: response.content });
    continue;
  }

  const toolUseBlocks = response.content.filter(
    (b): b is Anthropic.ToolUseBlock => b.type === "tool_use",
  );

  messages.push({ role: "assistant", content: response.content });

  const toolResults: Anthropic.ToolResultBlockParam[] = [];
  for (const tool of toolUseBlocks) {
    const result = await executeTool(tool.name, tool.input);
    toolResults.push({
      type: "tool_result",
      tool_use_id: tool.id,
      content: result,
    });
  }

  messages.push({ role: "user", content: toolResults });
}
```

### 流式手动循环

当您需要在手动循环中进行流式处理时，请使用 `client.messages.stream()` 和 `finalMessage()` 替代 `.create()`。每次迭代都会流式返回文本增量；`finalMessage()` 会收集完整的 `Message` 对象，以便您可以检查 `stop_reason` 并提取工具调用块。为每个工具设置 `eager_input_streaming: true`，使大型输入在生成时即可流式传输；此时服务器将不再对其进行校验，因此在执行前请根据工具的 Schema 对每个解析后的输入进行验证，遇到 `max_tokens` 或 `refusal` 时应停止，并仅捕获 SDK 的 JSON 解析错误（参见 `shared/tool-use-concepts.md` 中的“输入预流”部分）。Schema 校验并非路径校验：模型提供的 `path` 是不可信的输出，因此在写入之前应将其限制在项目根目录内（参见同一文件中的文本编辑器安全说明）：
```typescript
import Anthropic from "@anthropic-ai/sdk";
import nodePath from "path";
import { z } from "zod";

const client = new Anthropic();
const ROOT = nodePath.resolve(process.cwd());
const WriteFileInput = z.object({ path: z.string(), contents: z.string() });
const tools: Anthropic.Tool[] = [
  {
    name: "write_file",
    description: "将文本写入指定路径的文件",
    eager_input_streaming: true, // 在生成过程中流式传输大型输入
    input_schema: {
      type: "object",
      properties: { path: { type: "string" }, contents: { type: "string" } },
      required: ["path", "contents"],
    },
  },
];
let messages: Anthropic.MessageParam[] = [{ role: "user", content: userInput }];
let jsonRetries = 0;

while (true) {
  const stream = client.messages.stream({
    model: "claude-opus-5-5",
    max_tokens: 64000,
    tools,
    messages,
  });

  // 每次迭代时流式输出文本增量
  stream.on("text", (delta) => {
    process.stdout.write(delta);
  });

  // finalMessage() 返回完整的消息对象，无需手动处理 .on("message")、.on("error") 或 .on("abort")。
  // 启用输入流式传输后，如果工具输入完全无法解析，则会抛出错误。仅在这种情况下才会重试；API 错误则直接抛出。
  let message: Anthropic.Message;
  try {
    message = await stream.finalMessage();
    jsonRetries = 0; // 重试次数限制为同一轮中连续失败的次数
  } catch (err) {
    if (err instanceof Anthropic.APIError || jsonRetries++ >= 2) throw err;
    console.error("工具输入不是可解析的 JSON，重新发出本轮请求");
    continue;
  }

  if (message.stop_reason === "end_turn") break;
  // 如果发生拒绝，可能会在输入过程中中断工具调用；这种情况下不应执行该轮的任何工具。
  if (message.stop_reason === "refusal") break;

  // 服务器端工具调用次数达到上限；追加助手回复并重新发送以继续对话。
  if (message.stop_reason === "pause_turn") {
    messages.push({ role: "assistant", content: message.content });
    continue;
  }

  const toolUseBlocks = message.content.filter(
    (b): b is Anthropic.ToolUseBlock => b.type === "tool_use",
  );
  if (toolUseBlocks.length === 0) break; // 其他终止原因

  // 当工具输入因达到最大 token 数而被截断时，通常仍能解析为一个有效的部分对象；此时应检查停止原因，并通过提高 max_tokens 来重试，而不是对截断的输入执行工具。
  if (message.stop_reason === "max_tokens") {
    throw new Error("工具输入因 max_tokens 被截断；请提高 max_tokens 后重试");
  }

  messages.push({ role: "assistant", content: message.content });

  const toolResults: Anthropic.ToolResultBlockParam[] = [];
  for (const tool of toolUseBlocks) {
    // SDK 的容错解析器可能会静默地截断输入（例如，在未转义的内部引号处），因此在执行前需进行验证。
    const parsed = WriteFileInput.safeParse(tool.input);
    if (!parsed.success) {
      toolResults.push({
        type: "tool_result",
        tool_use_id: tool.id,
        is_error: true,
        content: JSON.stringify({ INVALID_JSON: JSON.stringify(tool.input) }),
      });
      continue;
    }
    // `path` 是不可信的模型输出：在写入之前，需解析其绝对路径并拒绝任何试图逃出项目根目录的路径（如包含 ".." 或绝对路径）——仅靠模式校验无法确保这一点。此检查是基于字符串的；如果根目录中存在符号链接目录，还应使用 fs.realpath 进行规范化处理（参见 shared/tool-use-concepts.md 中关于文本编辑器安全性的说明）。
    const target = nodePath.resolve(ROOT, parsed.data.path);
    const relative = nodePath.relative(ROOT, target);
    if (relative === ".." || relative.startsWith(".." + nodePath.sep) || nodePath.isAbsolute(relative)) {
      toolResults.push({
        type: "tool_result",
        tool_use_id: tool.id,
        is_error: true,
        content: "路径逃出了项目根目录",
      });
      continue;
    }
    toolResults.push({
      type: "tool_result",
      tool_use_id: tool.id,
      content: await executeTool(tool.name, { ...parsed.data, path: target }),
    });
  }

messages.push({ role: "user", content: toolResults });
}
```

> **重要提示：** 不要将 `.on()` 事件包裹在 `new Promise()` 中来收集最终消息，而是使用 `stream.finalMessage()`。SDK 内部会处理所有错误、中断和完成状态。

> **循环中的错误处理：** 使用 SDK 的类型化异常（例如 `Anthropic.RateLimitError`、`Anthropic.APIError`），具体示例请参见[错误处理](./README.md#error-handling)。不要通过字符串匹配来检查错误信息。

> **SDK 类型：** 对于所有与 API 相关的数据结构，请使用 `Anthropic.MessageParam`、`Anthropic.Tool`、`Anthropic.ToolUseBlock`、`Anthropic.ToolResultBlockParam`、`Anthropic.Message` 等类型，不要重新定义等效的接口。

---

## 处理工具结果

```typescript
const response = await client.messages.create({
  model: "claude-opus-5-5",
  max_tokens: 16000,
  tools: tools,
  messages: [{ role: "user", content: "巴黎的天气怎么样？" }],
});

for (const block of response.content) {
  if (block.type === "tool_use") {
    const result = await executeTool(block.name, block.input);

    const followup = await client.messages.create({
      model: "claude-opus-5-5",
      max_tokens: 16000,
      tools: tools,
      messages: [
        { role: "user", content: "巴黎的天气怎么样？" },
        { role: "assistant", content: response.content },
        {
          role: "user",
          content: [
            { type: "tool_result", tool_use_id: block.id, content: result },
          ],
        },
      ],
    });
  }
}
```

---

## 工具选择

`tool_choice` 默认为 `{ type: "auto" }`。强制调用（即 `{ type: "any" }` 或 `{ type: "tool", name: ... }`）在 Claude Opus 5.5、Claude Sonnet 5.5、Claude Fable 5.1 和 Claude Mythos 5.1 上会返回 400 错误；而 Claude Opus 5、Claude Sonnet 5 及更早版本则接受这种设置。建议通过提示词来引导模型，并通过 `strict: true` 来确保模式约束：

```typescript
const response = await client.messages.create({
  model: "claude-opus-5-5",
  max_tokens: 16000,
  tools: tools.map((tool) => ({ ...tool, strict: true })), // 模式必须设置 additionalProperties: false
  messages: [{ role: "user", content: "巴黎的天气怎么样？请使用 get_weather 工具。" }],
});
// 自动模式不保证一定会调用工具——需检查是否有 `tool_use` 块，如果没有则重新提示。
```

---

## Anthropic 定义的工具

`type` 字面量带有版本后缀；`name` 在每个接口中是固定的。网络搜索和代码执行由服务器端执行；Bash 和文本编辑器由客户端执行（您需要在本地处理 `tool_use` 事件——详见 `shared/tool-use-concepts.md`）。只需传递普通的对象字面量即可，因为 `ToolUnion` 类型可以通过结构兼容性推导出来。**`name` 和 `type` 的组合必须与接口一致**：将 `str_replace_based_edit_tool`（20250728 版本名称）与 `text_editor_20250124`（期望的是 `str_replace_editor`）混用会导致 TS2322 错误。

**不要将其标注为 `Tool[]`**——`Tool` 只是自定义工具的变体。让结构类型推导从 `tools` 参数中进行，或者如果您确实需要，可以标注为 `Anthropic.Messages.ToolUnion[]`：

```typescript
// 良好的做法：让类型推导自动完成，无需显式标注
const response = await client.messages.create({
  model: "claude-opus-5-5",
  max_tokens: 16000,
  tools: [
    { type: "text_editor_20250728", name: "str_replace_based_edit_tool" },
    { type: "bash_20250124", name: "bash" },
    { type: "web_search_20260209", name: "web_search" },
    { type: "code_execution_20260120", name: "code_execution" },
  ],
  messages: [{ role: "user", content: "..." }],
});

// 不良的做法：这会导致 TS2352 错误——`Tool` 只代表自定义工具的变体
// const tools: Anthropic.Tool[] = [{ type: "text_editor_20250728", ... }]
```

| 接口 | `name` | `type` |
|---|---|---|
| `ToolTextEditor20250124` | `str_replace_editor` | `text_editor_20250124` |
| `ToolTextEditor20250429` | `str_replace_based_edit_tool` | `text_editor_20250429` |
| `ToolTextEditor20250728` | `str_replace_based_edit_tool` | `text_editor_20250728` |
| `ToolBash20250124` | `bash` | `bash_20250124` |
| `WebSearchTool20260209` | `web_search` | `web_search_20260209` |
| `WebFetchTool20260209` | `web_fetch` | `web_fetch_20260209` |
| `CodeExecutionTool20260120` | `code_execution` | `code_execution_20260120` |

**不要混用测试版和正式版类型**：如果您调用 `client.beta.messages.create()`，返回的 `content` 将是 `BetaContentBlock[]` 类型——您不能直接将其传递给非测试版的 `ContentBlockParam[]`，除非对每个元素进行缩小范围的类型断言。

---

## 代码执行

### 基本用法

```typescript
import Anthropic from "@anthropic-ai/sdk";

const client = new Anthropic();

const response = await client.messages.create({
  model: "claude-opus-5-5",
  max_tokens: 16000,
  messages: [
    {
      role: "user",
      content:
        "计算 [1, 2, 3, 4, 5, 6, 7, 8, 9, 10] 的均值和标准差",
    },
  ],
  tools: [{ type: "code_execution_20260120", name: "code_execution" }],
});
```

### 读取本地文件（ESM 注意事项）

在 ES 模块中，`__dirname` 不存在。对于相对于脚本的路径，请使用 `import.meta.url`：

```typescript
import { readFileSync } from "fs";
import { fileURLToPath } from "url";
import { dirname, join } from "path";

const __dirname = dirname(fileURLToPath(import.meta.url));
const pdfBytes = readFileSync(join(__dirname, "sample.pdf"));
```

或者，如果脚本运行于已知目录下，也可以使用相对于当前工作目录的路径：`readFileSync("./sample.pdf")`。

### 上传文件以进行分析

```typescript
import Anthropic, { toFile } from "@anthropic-ai/sdk";
import { createReadStream } from "fs";

const client = new Anthropic();

// 1. 上传文件
const uploaded = await client.beta.files.upload({
  file: await toFile(createReadStream("sales_data.csv"), undefined, {
    type: "text/csv",
  }),
});

// 2. 传递给代码执行工具
const response = await client.messages.create(
  {
    model: "claude-opus-5-5",
    max_tokens: 16000,
    messages: [
      {
        role: "user",
        content: [
          {
            type: "text",
            text: "分析这些销售数据，展示趋势并生成可视化图表。",
          },
          { type: "container_upload", file_id: uploaded.id },
        ],
      },
    ],
    tools: [{ type: "code_execution_20260120", name: "code_execution" }],
  },
);
```

### 获取生成的文件

```typescript
import path from "path";
import fs from "fs";

const OUTPUT_DIR = "./claude_outputs";
await fs.promises.mkdir(OUTPUT_DIR, { recursive: true });

for (const block of response.content) {
  if (block.type === "bash_code_execution_tool_result") {
    const result = block.content;
    if (result.type === "bash_code_execution_result" && result.content) {
      for (const fileRef of result.content) {
        if (fileRef.type === "bash_code_execution_output") {
          const metadata = await client.beta.files.retrieveMetadata(
            fileRef.file_id,
          );
          const downloadResponse = await client.beta.files.download(fileRef.file_id);
          const fileBytes = Buffer.from(await downloadResponse.arrayBuffer());
          const safeName = path.basename(metadata.filename);
          if (!safeName || safeName === "." || safeName === "..") {
            console.warn(`跳过无效文件名：${metadata.filename}`);
            continue;
          }
          const outputPath = path.join(OUTPUT_DIR, safeName);
          await fs.promises.writeFile(outputPath, fileBytes);
          console.log(`已保存：${outputPath}`);
        }
      }
    }
  }
}
```

### 容器复用

```typescript
// 第一次请求：设置环境
const response1 = await client.messages.create({
  model: "claude-opus-5-5",
  max_tokens: 16000,
  messages: [
    {
      role: "user",
      content: "安装 tabulate，并创建包含示例用户数据的 data.json 文件",
    },
  ],
  tools: [{ type: "code_execution_20260120", name: "code_execution" }],
});

// 复用容器
// container 字段可为 null——仅在使用服务端代码执行时才需设置
const containerId = response1.container!.id;

const response2 = await client.messages.create({
  container: containerId,
  model: "claude-opus-5-5",
  max_tokens: 16000,
  messages: [
    {
      role: "user",
      content: "读取 data.json 并以格式化表格形式显示",
    },
  ],
  tools: [{ type: "code_execution_20260120", name: "code_execution" }],
});
```

---

## 记忆工具

### 基本用法

```typescript
const response = await client.messages.create({
  model: "claude-opus-5-5",
  max_tokens: 16000,
  messages: [
    {
      role: "user",
      content: "请记住，我首选的语言是 TypeScript。",
    },
  ],
  tools: [{ type: "memory_20250818", name: "memory" }],
});
```

### SDK 内存辅助工具

使用 `betaMemoryTool` 并配合 `MemoryToolHandlers` 实现：

```typescript
import {
  betaMemoryTool,
  type MemoryToolHandlers,
} from "@anthropic-ai/sdk/helpers/beta/memory";

const handlers: MemoryToolHandlers = {
  async view(command) { ... },
  async create(command) { ... },
  async str_replace(command) { ... },
  async insert(command) { ... },
  async delete(command) { ... },
  async rename(command) { ... },
};

const memory = betaMemoryTool(handlers);

const runner = client.beta.messages.toolRunner({
  model: "claude-opus-5-5",
  max_tokens: 16000,
  tools: [memory],
  messages: [{ role: "user", content: "记住我的偏好" }],
});

for await (const message of runner) {
  console.log(message);
}
```

如需完整的实现示例，请参考 WebFetch：
- `https://github.com/anthropics/anthropic-sdk-typescript/blob/main/examples/tools-helpers-memory.ts`

---

## 结构化输出

### JSON 输出（推荐使用 Zod）

```typescript
import Anthropic from "@anthropic-ai/sdk";
import { z } from "zod";
import { zodOutputFormat } from "@anthropic-ai/sdk/helpers/zod";

const ContactInfoSchema = z.object({
  name: z.string(),
  email: z.string(),
  plan: z.string(),
  interests: z.array(z.string()),
  demo_requested: z.boolean(),
});

const client = new Anthropic();

const response = await client.messages.parse({
  model: "claude-opus-5-5",
  max_tokens: 16000,
  messages: [
    {
      role: "user",
      content:
        "提取：Jane Doe（jane@co.com）想要企业版，对 API 和 SDK 感兴趣，并希望预约演示。",
    },
  ],
  output_config: {
    format: zodOutputFormat(ContactInfoSchema),
  },
});

// 如果解析失败，parsed_output 为 null - 需要进行断言或检查
console.log(response.parsed_output!.name); // "Jane Doe"
```

### 严格工具调用

```typescript
const response = await client.messages.create({
  model: "claude-opus-5-5",
  max_tokens: 16000,
  messages: [
    {
      role: "user",
      content: "为 2 名乘客预订 3 月 15 日飞往东京的航班。",
    },
  ],
  tools: [
    {
      name: "book_flight",
      description: "预订前往某目的地的航班",
      strict: true,
      input_schema: {
        type: "object",
        properties: {
          destination: { type: "string" },
          date: { type: "string", format: "date" },
          passengers: {
            type: "integer",
            enum: [1, 2, 3, 4, 5, 6, 7, 8],
          },
        },
        required: ["destination", "date", "passengers"],
        additionalProperties: false,
      },
    },
  ],
});
```

---

## 代理技能

通过 `container.skills` 和 beta 路径上的 `code_execution` 工具启用由 Anthropic 管理的技能（例如 `pptx`）。两者都需要使用 beta 标头。输出会以文件形式出现在响应内容中，可通过 Files API 根据文件 ID 下载。

```typescript
const response = await client.beta.messages.create({
  model: "claude-opus-5-5",
  max_tokens: 16000,
  container: {
    skills: [{ type: "anthropic", skill_id: "pptx", version: "latest" }],
  },
  tools: [{ type: "code_execution_20260521", name: "code_execution" }],
  betas: ["code-execution-2025-08-25"],
  messages: [{ role: "user", content: "创建一个关于 X 的 3 张幻灯片的演示文稿。" }],
});
// 在 response.content 中找到 file_id，然后：
// await client.beta.files.download(fileId)
```