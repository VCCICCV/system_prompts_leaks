# 托管代理 - TypeScript

> **此处未展示的绑定：** 本 README 涵盖了 TypeScript 中最常见的托管代理流程。如果您需要未在此处展示的类、方法、命名空间、字段或行为，请通过 WebFetch 获取 TypeScript SDK 仓库**或 `shared/live-sources.md` 中的相关文档页面**，而不要自行猜测。请勿根据 cURL 的请求格式或其他语言的 SDK 进行推断。

> **代理是持久化的——只需创建一次，并通过 ID 引用。** 请保存 `agents.create` 返回的代理 ID，并在后续每次调用 `sessions.create` 时传递该 ID；切勿在请求路径中重复调用 `agents.create`。**建议：** 将代理和环境定义为受版本控制的文件，并通过 `ant apply` 同步——参见 `shared/anthropic-cli.md`（其实时文档 URL 在 `shared/live-sources.md` 中）。CLI 负责控制平面（创建/更新）；您的代码负责数据平面（使用已存储 ID 的会话）。以下示例展示了在必须以编程方式进行资源配置时的代码内创建；但在生产环境中，创建调用应放在初始化阶段，而非请求路径中。

## 安装

```bash
npm install @anthropic-ai/sdk
```

## 客户端初始化

```typescript
import Anthropic from "@anthropic-ai/sdk";

// 默认方式——从环境变量中解析凭据：
// ANTHROPIC_API_KEY、ANTHROPIC_AUTH_TOKEN，或 `ant auth login` 创建的配置文件。
// 本地开发时优先使用此方式，不要硬编码 API 密钥。
const client = new Anthropic();

// 显式指定 API 密钥（仅在必须注入特定密钥时使用）
const client = new Anthropic({ apiKey: "your-api-key" });
```

---

## 创建环境

```typescript
const environment = await client.beta.environments.create(
  {
    name: "my-dev-env",
    config: {
      type: "cloud",
      networking: { type: "unrestricted" },
    },
  },
);
console.log(environment.id); // env_...
```

---

## 创建代理（必需的第一步）

> 警告：**不支持内联代理配置。** `model`/`system`/`tools` 属于代理对象，而非会话。始终先调用 `agents.create()`——会话仅接受 `agent: { type: "agent", id: agent.id }`。

### 最简配置

```typescript
// 1. 创建代理（可复用、带版本）
const agent = await client.beta.agents.create(
  {
    name: "Coding Assistant",
    model: "claude-opus-5-5",
    tools: [{ type: "agent_toolset_20260401", default_config: { enabled: true } }],
  },
);

// 2. 开始会话
const session = await client.beta.sessions.create(
  {
    agent: { type: "agent", id: agent.id, version: agent.version },
    environment_id: environment.id,
  },
);
console.log(session.id, session.status);
console.log(`追踪链接：https://platform.claude.com/workspaces/default/sessions/${session.id}`); // 如果 API 密钥不在 Default 工作空间，请将 'default' 替换为您的工作空间 ID
```

### 带系统提示和自定义工具

```typescript
const agent = await client.beta.agents.create(
  {
    name: "Code Reviewer",
    model: "claude-opus-5-5",
    system: "您是一位资深代码评审员。",
    tools: [
      { type: "agent_toolset_20260401", default_config: { enabled: true } },
      {
        type: "custom",
        name: "run_tests",
        description: "运行测试套件",
        input_schema: {
          type: "object",
          properties: {
            test_path: { type: "string", description: "测试文件路径" },
          },
          required: ["test_path"],
        },
      },
    ],
  },
);

const session = await client.beta.sessions.create(
  {
    agent: { type: "agent", id: agent.id, version: agent.version },
    environment_id: environment.id,
    title: "代码评审会话",
    resources: [
      {
        type: "github_repository",
        url: "https://github.com/owner/repo",
        mount_path: "/workspace/repo",
        authorization_token: process.env.GITHUB_TOKEN,
        branch: "main",
      },
    ],
  },
);
```

---

## 发送用户消息

```typescript
await client.beta.sessions.events.send(
  session.id,
  {
    events: [
      {
        type: "user.message",
        content: [{ type: "text", text: "审查认证模块" }],
      },
    ],
  },
);
```

> 提示：**流优先**：在发送消息之前（或同时）打开流。流仅传递在其打开之后发生的事件——先发送后打开流会导致早期事件以批处理形式缓冲到达。请参阅[引导模式](../../shared/managed-agents-events.md#steering-patterns)。

---

## 定义成果（交付物的默认启动）

当会话的任务是生成可检验的内容——如工件、报告或拉取请求时，请使用 `user.define_outcome` 而不是 `user.message` 来启动：框架会根据您的评分标准对每次迭代进行评估，代理会不断修改直至通过。二者选其一，切勿同时发送。有关事件参考和评分标准编写指南，请参阅[成果](../../shared/managed-agents-outcomes.md)。

```typescript
const STARTER_RUBRIC = `# 报告评分标准 - 初稿，请调整各项指标
- 输出为 /mnt/session/outputs/ 目录下的单个 report.md 文件
- 每个论断均标注来源网址
- 包含一张汇总表，每行对应一家竞争对手
- 价格为运行当日的最新价格，且每行注明数据来源
- 不得留有占位文本、待办事项或空章节
`;

await client.beta.sessions.events.send(
  session.id,
  {
    events: [
      {
        type: "user.define_outcome",
        description: "撰写一份竞争对手定价报告，保存为 report.md",
        rubric: { type: "text", content: STARTER_RUBRIC },
        max_iterations: 5, // 可选；默认3次，最大20次
      },
    ],
  },
);
```

---

## 流式事件（SSE）

```typescript
// 流优先：同时打开流并发送
const [events] = await Promise.all([
  collectStream(session.id),
  client.beta.sessions.events.send(
    session.id,
    { events: [{ type: "user.message", content: [{ type: "text", text: "..." }] }] },
  ),
]);

// 独立的流式迭代：
const stream = await client.beta.sessions.events.stream(
  session.id,
);

for await (const event of stream) {
  switch (event.type) {
    case "agent.message":
      for (const block of event.content) {
        if (block.type === "text") {
          process.stdout.write(block.text);
        }
      }
      break;
    case "agent.custom_tool_use":
      // 自定义工具调用——此时会话处于空闲状态
      console.log(`\n自定义工具调用：${event.name}`);
      console.log(`输入：${JSON.stringify(event.input)}`);
      break;
    case "session.status_idle":
      console.log("\n--- 代理空闲 ---");
      break;
    case "session.status_terminated":
      console.log("\n--- 会话终止 ---");
      break;
  }
}
```

---

## 提供自定义工具结果

```typescript
await client.beta.sessions.events.send(
  session.id,
  {
    events: [
      {
        type: "user.custom_tool_result",
        custom_tool_use_id: "sevt_abc123",
        content: [{ type: "text", text: "所有42个测试均通过。" }],
      },
    ],
  },
);
```

---

## 轮询事件

```typescript
const events = await client.beta.sessions.events.list(
  session.id,
);
for (const event of events.data) {
  console.log(`${event.type}: ${event.id}`);
}
```

---

## 带有自定义工具的完整流式循环

```typescript
function runCustomTool(toolName: string, toolInput: unknown): string {
  if (toolName === "run_tests") {
    // 在此处实现您的工具逻辑
    return "所有测试均通过。";
  }
  return `未知工具: ${toolName}`;
}

async function runSession(client: Anthropic, sessionId: string) {
  while (true) {
    const stream = await client.beta.sessions.events.stream(
      sessionId,
    );

    const toolCalls: Anthropic.Beta.Sessions.BetaManagedAgentsAgentCustomToolUseEvent[] = [];

    for await (const event of stream) {
      if (event.type === "agent.message") {
        for (const block of event.content) {
          if (block.type === "text") {
            process.stdout.write(block.text);
          }
        }
      } else if (event.type === "agent.custom_tool_use") {
        toolCalls.push(event);
      } else if (event.type === "session.status_idle") {
        break;
      } else if (event.type === "session.status_terminated") {
        return;
      }
    }

    if (toolCalls.length === 0) break;

    // 处理自定义工具调用
    const results = toolCalls.map((call) => ({
      type: "user.custom_tool_result" as const,
      custom_tool_use_id: call.id,
      content: [{ type: "text" as const, text: runCustomTool(call.name, call.input) }],
    }));
  }
}
    await client.beta.sessions.events.send(
      sessionId,
      { events: results },
    );
  }
}
```

---

## 上传文件

```typescript
import fs from "fs";

const file = await client.beta.files.upload({
  file: fs.createReadStream("data.csv"),
  purpose: "agent",
});

// 在会话中使用
const session = await client.beta.sessions.create(
  {
    agent: { type: "agent", id: agent.id, version: agent.version },
    environment_id: environment.id,
    resources: [{ type: "file", file_id: file.id, mount_path: "/workspace/data.csv" }],
  },
);
```

---

## 列出并下载会话文件

列出代理在会话期间写入 `/mnt/session/outputs/` 的文件，然后将其下载。

```typescript
import fs from "fs";

// 列出与会话关联的文件
const files = await client.beta.files.list({
  scope_id: session.id,
  betas: ["managed-agents-2026-04-01"],
});
for (const f of files.data) {
  console.log(f.filename, f.size_bytes);

  // 下载并保存到本地
  const resp = await client.beta.files.download(f.id);
  const buffer = Buffer.from(await resp.arrayBuffer());
  fs.writeFileSync(f.filename, buffer);
}
```

> 提示：从 `session.status_idle` 发生到输出文件出现在 `files.list` 中，可能会有短暂的索引延迟（约1–3秒）。如果列表为空，可重试一两次。

---

## 会话管理

```typescript
// 获取会话详情
const session = await client.beta.sessions.retrieve("sesn_011CZxAbc123Def456");
console.log(session.status, session.usage);

// 列出会话
const sessions = await client.beta.sessions.list();

// 删除会话
await client.beta.sessions.delete("sesn_011CZxAbc123Def456");

// 归档会话
await client.beta.sessions.archive("sesn_011CZxAbc123Def456");
```

---

## MCP 服务器集成

```typescript
// 代理声明 MCP 服务器（此处未进行认证——认证信息存放在 Vault 中）
const agent = await client.beta.agents.create({
  name: "MCP Agent",
  model: "claude-opus-5-5",
  mcp_servers: [
    { type: "url", name: "my-tools", url: "https://my-mcp-server.example.com/sse" },
  ],
  tools: [
    { type: "agent_toolset_20260401", default_config: { enabled: true } },
    { type: "mcp_toolset", mcp_server_name: "my-tools" },
  ],
});

// 会话附加包含这些 MCP 服务器 URL 凭证的 Vault
const session = await client.beta.sessions.create({
  agent: agent.id,
  environment_id: environment.id,
  vault_ids: [vault.id],
});
```

有关创建 Vault 并添加凭证的详细信息，请参阅 `shared/managed-agents-tools.md` 中的“Vault”部分。