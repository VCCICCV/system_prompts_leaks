# 托管代理 - 常见客户端模式

在驱动托管代理会话时，您将在客户端编写的模式，基于实际可用的 SDK 示例。

代码示例采用 TypeScript；其他语言的结构相同，请参阅 `{lang}/managed-agents/README.md`（cURL 和 C#：`curl/managed-agents.md`）以获取等效实现。

---

## 1. 无损流式重连

**问题：** SSE 不支持回放。如果连接在会话中途断开，简单的重连会从“当前时刻”重新打开流，导致中间发出的所有事件都被悄然错过。

**解决方案：** 重连时，在消费实时流之前，先通过 `events.list()` 获取完整的事件历史，并在实时流追上时按事件 ID 进行去重。

```ts
const seenEventIds = new Set<string>()
const stream = await client.beta.sessions.events.stream(session.id)

// 流已打开并在服务器端缓冲。先读取历史记录。
for await (const event of client.beta.sessions.events.list(session.id)) {
  seenEventIds.add(event.id)
  handle(event)
}

// 尾随实时流。去重仅作用于 `handle()` 调用——终端检查必须对所有事件执行，即使是对已见过的事件，否则历史响应中包含的终端事件会被 `continue` 跳过，导致循环无法退出。
for await (const event of stream) {
  if (!seenEventIds.has(event.id)) {
    seenEventIds.add(event.id)
    handle(event)
  }
  if (event.type === 'session.status_terminated') break
  if (event.type === 'session.status_idle' && event.stop_reason.type !== 'requires_action') break
}
```

---

## 2. `processed_at`——排队与处理

流上的每个事件都带有 `processed_at`（ISO 8601 格式），该字段在事件处理完成后被设置。对于客户端发送的事件（`user.message`、`user.interrupt`、`user.tool_confirmation`），当事件在队列中等待处理时，此字段为 `null`；一旦代理处理了该事件，字段才会被填充——因此同一个事件会在流上出现两次，一次是 `null`，一次是有时间戳。（例外：当会话因预算耗尽而暂停时发送的 `user.interrupt` 会被接受但忽略——它根本不会出现在流中；详见 `shared/managed-agents-events.md` § 达到会话预算。）

**有三种事件类型会跳过排队阶段：** `user.define_outcome`、`user.custom_tool_result` 和 `user.tool_result` 在接收到后即被处理，并立即返回带有 `processed_at` 的响应。如果 UI 状态假设“首次出现时 `processed_at` 总是 `null`”，则这些事件将永远不会从待处理状态变为已确认状态——因此，首次看到时若 `processed_at` 已有值，则应视为已立即确认。

```ts
for await (const event of stream) {
  if (event.type === 'user.message') {
    if (event.processed_at == null) onQueued(event.id)
    else onProcessed(event.id, event.processed_at)
  }
}
```

利用这一点来驱动您所发送内容的待处理→已确认 UI 状态。如何将本地渲染的乐观消息映射到服务端分配的 `event.id` 是应用相关的（通常通过 `events.send()` 的返回值或 FIFO 顺序来实现）。

---

## 3. 中断正在运行的会话

将 `user.interrupt` 作为普通事件发送。会话会继续运行，直到到达一个安全边界，然后进入空闲状态。

```ts
await client.beta.sessions.events.send(session.id, {
  events: [{ type: 'user.interrupt' }],
})

// 消费流直至会话真正结束——完整逻辑参见模式 5。
for await (const event of stream) {
  if (event.type === 'session.status_terminated') break
  if (
    event.type === 'session.status_idle' &&
    event.stop_reason.type !== 'requires_action'
  ) break
}
```

参考：`interrupt.ts`——在检测到 `span.model_request_start` 时立即发送中断，消费流直至会话进入空闲状态，随后通过 `sessions.retrieve()` 进行验证。

---

## 4. `tool_confirmation` 往返流程当调用评估结果为 `ask` 时——即工具的 `permission_policy` 为 `{ type: 'always_ask' }`，或者其为 `{ type: 'auto' }` 且服务器未能做出明确判断时——`agent.tool_use` 或 `agent.mcp_tool_use` 事件会携带 `evaluated_permission === 'ask'`，此时会话将进入空闲状态，等待用户决策。请发送 `user.tool_confirmation` 响应。

```ts
for await (const event of stream) {
  if ((event.type === 'agent.tool_use' || event.type === 'agent.mcp_tool_use') && event.evaluated_permission === 'ask') {
    await client.beta.sessions.events.send(session.id, {
      events: [{
        type: 'user.tool_confirmation',
        tool_use_id: event.id,         // 不是 toolu_ ID，使用 event.id
        result: 'allow',               // 或者 'deny'
        // deny_message: '...',        // 可选，仅在 result 为 'deny' 时使用
      }],
    })
  }
}
```

要点：
- `tool_use_id` 是 `event.id`（通常是 `sevt_...`），**不是** `toolu_...` ID。
- `result` 的值为 `'allow' | 'deny'`。可使用 `deny_message` 向模型说明拒绝的原因——该信息会回传给代理。
- 存在多个待处理的工具调用时：对每个 `evaluated_permission === 'ask'` 的 `agent.tool_use` 或 `agent.mcp_tool_use` 事件分别响应一次。
- 应基于 `evaluated_permission === 'ask'` 进行判断，而非依据您配置的策略——这同时适用于 `always_ask` 和 `auto` 且无法确定的情况。在 `auto` 策略下，如果服务器判定为拒绝（`evaluated_permission === 'deny'`，且 `evaluation.evaluated_permission.reason_code === 'high_risk'`），则不会进入此流程：代理会收到错误的工具调用结果，会话仍将继续运行；向此类调用发送确认会导致 400 错误。
- 记录 `event.evaluation` 以供审计（包括 `type` 和 `reason_code`），并容忍未识别的 `type` 或 `reason_code`——对已知值进行分支处理，未知值则直接传递。

参考：`tool-permissions.ts`。

---

## 5. 正确处理空闲中断逻辑

不要仅根据 `session.status_idle` 来中断会话。会话可能会短暂进入空闲状态——例如，在并行执行工具调用之间、等待 `user.tool_confirmation` 或等待 `user.custom_tool_result` 时。应在会话处于空闲状态且 `stop_reason` 类型不为 `requires_action` 时中断（如终端状态或预算耗尽——后者只能通过更新预算来恢复，因此除非您打算更改或移除预算，否则应中断），或者在 `session.status_terminated` 时中断。

```ts
for await (const event of stream) {
  handle(event)
  if (event.type === 'session.status_terminated') break
  if (event.type === 'session.status_idle') {
    if (event.stop_reason.type === 'requires_action') continue // 等待您的操作——继续处理
    break // end_turn、retries_exhausted 或 budget_reached——参见下方列表
  }
}
```

`session.status_idle` 下的 `stop_reason.type` 取值：
- `requires_action`——代理正在等待客户端事件（工具确认、自定义工具结果）。请继续处理，不要中断。**自托管例外情况：** 如果会话因无待处理的 `agent.tool_use` / `agent.mcp_tool_use`（`ask`）或 `agent.custom_tool_use` 而进入 `requires_action` 空闲状态，则表示工作进程未能完成其任务（通常为内存存储挂载错误，仅记录在工作进程主机上）。对此不应无限期地继续处理——应上报问题、修复主机，并发送 `user.interrupt` 以重新排队该任务（参见 `shared/managed-agents-self-hosted-sandboxes.md` § 内存存储 -> 故障排除）。
- `retries_exhausted`——不可恢复的失败。中断后，请通过 `sessions.retrieve()` 检查错误状态。
- `end_turn`——正常结束。
- `budget_reached`——会话达到支出上限并暂停。这不是终端状态，也无法通过任何事件恢复：需更改（通常上调）或移除会话的 `budget` 才能恢复，否则视为已完成。紧接此空闲状态之前会有一个包含最终费用的 `session.usage` 事件。详情参见 `shared/managed-agents-core.md` § 会话预算。

---

## 6. 处理空闲后的状态写入竞争问题SSE 流会在会话的可查询状态反映出来之前，略微提前发出 `session.status_idle` 事件。那些在会话处于空闲状态时立即调用 `sessions.delete()` 或 `sessions.archive()` 的客户端，会间歇性地收到 400 错误，提示“正在运行中，无法删除或归档”。

在清理前轮询：

```ts
let s
for (let i = 0; i < 10; i++) {
  s = await client.beta.sessions.retrieve(session.id)
  if (s.status !== 'running') break
  await new Promise(r => setTimeout(r, 200))
}
if (s?.status !== 'running') {
  await client.beta.sessions.archive(session.id)
} // 否则：等待 2 秒后仍处于运行状态——不归档，让其自行结束或升级处理
```

---

## 7. 先开启流，再发送事件

请务必**先**打开流，**再**发送启动事件。否则，代理可能会在您的消费者尚未连接时就处理该事件并发出首个事件，导致您错过这些事件。

```ts
const stream = await client.beta.sessions.events.stream(session.id)
await client.beta.sessions.events.send(session.id, {
  events: [{ type: 'user.message', content: [{ type: 'text', text: 'Hello' }] }],
})
for await (const event of stream) { /* ... */ }
```

使用 `Promise.all([stream, send])` 的写法也可以，但先开启流的方式更简单，且效果相同——流一旦打开就会开始缓冲数据。

---

## 8. 文件挂载的注意事项

**挂载的资源与您上传的文件具有不同的 `file_id`。** 创建会话时会生成一个会话范围内的副本。

```ts
const uploaded = await client.beta.files.upload({ file, purpose: 'agent_resource' })
// uploaded.id         -> 原始文件
const session = await client.beta.sessions.create({
  /* ... */
  resources: [{ type: 'file', file_id: uploaded.id, mount_path: '/workspace/data.csv' }],
})
// session.resources[0].file_id !== uploaded.id  <- ID 不同
```

请通过 `files.delete(uploaded.id)` 删除原始文件；会话范围内的副本将随会话一起被垃圾回收。`mount_path` 必须是绝对路径——详情请参阅 `shared/managed-agents-environments.md`。

---

## 9. 非 MCP API 和 CLI 的密钥管理——通过自定义工具将其保留在宿主机端

**问题：** 您希望代理调用需要密钥（API 密钥、令牌、服务账号凭据）的第三方 API 或运行 CLI，但又无法或不愿将密钥交给密钥管理系统。

**首先检查：** 对于云环境，目前首选方案是使用密钥管理系统的 `environment_variable` 凭证——代理的 Shell 只能看到一个占位符，而真正的密钥会在出站时被替换。请参阅 `shared/managed-agents-tools.md` 中的“密钥管理”部分。如果该方案不适用，则可以采用以下方式：**自托管沙箱**（目前尚不支持环境变量凭据）、通过本地格式校验拒绝占位符的客户端、必须始终不出您基础设施的密钥，或需要在宿主机端运行二进制文件的调用。

**解决方案：** 将认证后的调用移到您的端执行。在代理中声明一个自定义工具；当代理发出 `agent.custom_tool_use` 事件时，由您的编排器（即读取 SSE 流的进程）使用自己的凭据执行该调用，并以 `user.custom_tool_result` 事件作出响应。容器永远不会接触到密钥。

```ts
// 代理模板：声明工具，无需提供凭据
tools: [{ type: 'custom', name: 'linear_graphql', input_schema: { /* query, vars */ } }]

// 编排器：使用宿主机凭据处理调用
for await (const event of stream) {
  if (event.type === 'agent.custom_tool_use' && event.name === 'linear_graphql') {
    const result = await linear.request(event.input.query, event.input.vars) // 宿主机的密钥
    await client.beta.sessions.events.send(session.id, {
      events: [{
        type: 'user.custom_tool_result',
        custom_tool_use_id: event.id,
        content: [{ type: 'text', text: JSON.stringify(result) }],
      }],
    })
  }
}
```

同样的模式也适用于 `gh` CLI、本地脚本或其他需要宿主机侧认证或二进制文件的场景。**安全提示：** 这不会暴露任何公共端点。`agent.custom_tool_use` 会通过您已使用 Anthropic API 密钥保持打开状态的 SSE 流到达，而 `user.custom_tool_result` 则会通过同一密钥下的 `events.send()` 返回。您的编排器是客户端，而非服务器——没有任何未经身份验证的服务在监听。

**请勿为规避问题而在系统提示或用户消息中嵌入 API 密钥。** 提示和消息会存储在会话的事件历史中，可通过 `events.list()` 获取，并包含在压缩摘要中——一旦将密钥放置于此处，它将在会话生命周期内被持久保存，并可通过 API 读取。
