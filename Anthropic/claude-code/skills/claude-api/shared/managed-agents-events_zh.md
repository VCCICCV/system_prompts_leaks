# 受管代理 - 事件与控制

## 事件

### 发送事件

通过 `POST /v1/sessions/{id}/events` 向会话发送事件。

| 事件类型                | 发送时机                                        |
| ------------------------- | --------------------------------------------------- |
| `user.message`            | 发送用户消息 |
| `user.interrupt`          | 在代理运行时中断代理 |
| `user.tool_confirmation`  | 对因需审批而暂停的工具调用进行批准或拒绝（当设置为 `always_ask`，或在服务器无法做出判断时设置为 `auto`） |
| `user.custom_tool_result` | 提供自定义工具调用的结果 |
| `user.define_outcome`     | 启动基于评分标准的迭代循环 - 请参阅 `shared/managed-agents-outcomes.md` |
| `system.message`          | 为当前轮次及之后的所有轮次追加具有特权的系统级上下文；详见“会话中添加系统上下文”部分 |

#### 在会话中添加系统上下文（`system.message`）

代理定义中的 `system` 字段设置了会话的顶层系统提示，并在整个会话生命周期内保持不变。`system.message` 事件会以 `role: "system"` 的形式**追加**到会话的系统上下文中，而不会替换该提示。其内容适用于当前轮次及之后的所有轮次。可用于切换角色、调整约束条件，或引入运行时获取的、希望影响后续行为的上下文：

```python
client.beta.sessions.events.send(
    session.id,
    events=[
        {
            "type": "system.message",
            "content": [
                {"type": "text", "text": "用户的当前时区是 America/New_York。"},
            ],
        },
    ],
)
```

约束条件：

- **受模型限制：Claude Fable 5.1、Claude Mythos 5.1、Claude Fable 5、Claude Mythos 5、Claude Opus 5.5、Claude Opus 5、Claude Opus 4.8 和 Claude Sonnet 5.5（不包括 Claude Sonnet 5）。** 仅检查代理的**主模型**——`system.message` 仅作用于主线程，子代理模型不予考虑。如果主模型不受支持，该事件将被拒绝，并返回 `model_does_not_support_mid_conversation_system` 验证错误。
- **当会话处于 `stop_reason: requires_action` 空闲状态时**（即阻塞在 `user.custom_tool_result` 或 `user.tool_confirmation` 上），`system.message` 仅在与同一请求中的工具结果事件**紧随其后**时才会被接受。单独发送，或与 `user.message` 一同发送，均会被拒绝，直至待处理的工具事件得到解决。
- `content` 接受 1 至 1000 个文本项。

### 接收事件

提供三种方式：

1. **流式传输（SSE）**：`GET /v1/sessions/{id}/events/stream`——实时服务器发送事件。**长连接**——服务器会定期发送心跳以维持连接。
2. **轮询**：`GET /v1/sessions/{id}/events`——分页事件列表（查询参数：`limit` 默认 1000，`page`）。**立即返回**——这是普通的分页 GET 请求，而非长轮询。
3. **Webhook**：Anthropic 会将会话状态变更 POST 到您的 HTTPS 端点——负载较轻（仅包含 ID），并附带 HMAC 签名，需在控制台注册。详情请参阅 `shared/managed-agents-webhooks.md`。

**无代码检查——控制台会话查看器**（控制台侧边栏 -> **托管代理** -> **会话**；仅限开发者和管理员）。在用户自行解析流之前，可引导他们至此进行调试：会话列表（ID、名称、状态、代理、输入/输出令牌数、费用；按状态/创建时间筛选，按ID搜索）；多代理会话中每条线程对应一条轨道的**时间轴缩略图**；按模型请求分组的**对话记录**（思考、工具调用及其输入/结果、流式文本），配有**事件过滤器**框（可按ID、类型、工具名称或文本匹配；可在匹配之间设置步进）以及复制/下载为JSON（启用过滤器时导出已过滤内容）；还有**检查器**侧边面板（可通过`d`键切换），包含五个标签页——**会话**（详情、元数据、累计费用与预算对比图）、**事件**（按服务器顺序排列的原始事件，每条事件均为JSON，并提供**增量视图**，显示页面打开期间流式传输的消息）、**工具**（每个已配置工具的调用次数、失败次数、中位持续时间；可跳转至任意调用）、**资源**（挂载的文件、代码库、内存存储及各会话的内存变化、`/mnt/session/outputs`中的文件、`/workspace/skills`下的技能）、**线程**（状态、上下文大小、每条线程的费用；当前线程的上下文大小曲线图；可切换线程）。可在会话URL中使用`?event={event_id}`进行深度链接——这在错误报告中与`shared/managed-agents-core.md`中的控制台链接一同附上时非常方便。

所有**持久化**事件均带有`id`、`type`和`processed_at`（ISO 8601格式），在事件处理完成时设置。对于您发送的事件，在其仍排在更早事件之后等待处理时，`processed_at`为`null`——**但**`user.define_outcome`、`user.custom_tool_result`和`user.tool_result`除外，这些事件在接收时即被处理，并在返回时已填充`processed_at`。仅用于流式传输的预览事件`event_start`和`event_delta`（参见§实时预览）仅携带其所预览事件的`id`。

> 警告：**健壮的轮询（原生HTTP）。** 如果您绕过SDK自行编写轮询循环，请勿依赖`requests`或`httpx`的超时作为绝对时间上限——它们是**逐块读取**的超时，每次收到字节时都会重置。若响应缓慢流出（心跳消息、chunked编码体卡住、代理异常），即使设置`timeout=(5, 60)`或`httpx.Timeout(120)`，调用也可能无限期阻塞。这两种库均未内置“总壁钟”超时机制。如需硬性截止：在循环层级跟踪`time.monotonic()`，若单次请求超出预算则中断/取消（例如通过守时线程，或在异步httpx调用外包裹`asyncio.wait_for()`）。**优先使用SDK**——`client.beta.sessions.events.stream()`和`client.beta.sessions.events.list()`能合理地处理超时与重试。
>
> 如果`GET /v1/sessions/{id}/events`（分页接口）在响应头之后出现挂起，很可能是误调用了`GET /v1/sessions/{id}/events/stream`，或是服务端发生阻塞——请上报，不要将其视为客户端配置问题。

### 事件类型（接收时）

事件类型采用点号命名法，并按命名空间分组：

| 事件类型 | 描述 |
| --- | --- |
| `agent.message` | 助手的文本输出 |
| `agent.thinking` | 表示助手正在思考的进度信号——该事件**不**携带思考内容 |
| `agent.tool_use` | 助手使用了内置工具（`agent_toolset_20260401`）。携带 `evaluated_permission`（`allow`/`ask`/`deny`）以及通常的 `evaluation`——详见 `shared/managed-agents-tools.md` 中关于 `evaluated_permission` 和 `evaluation` 的说明 |
| `agent.tool_result` | 内置工具的执行结果 |
| `agent.mcp_tool_use` | 助手使用了 MCP 工具。携带 `evaluated_permission` 以及通常的 `evaluation`，与 `agent.tool_use` 相同 |
| `agent.mcp_tool_result` | MCP 工具的执行结果 |
| `agent.custom_tool_use` | 助手调用了自定义工具——会话进入空闲状态，您需以 `user.custom_tool_result` 进行响应 |
| `agent.thread_context_compacted` | 对话上下文已被压缩 |
| `session.status_idle` | 助手已完成当前任务，正在等待输入。此时可能通过 `user.message` 等待继续处理，也可能因等待 `user.custom_tool_result` 或 `user.tool_confirmation` 而阻塞，或因达到会话预算上限而暂停。附加的 `stop_reason` 提供了助手停止工作的具体原因。 |
| `session.status_running` | 会话已开始运行，助手正在积极工作。 |
| `session.status_rescheduled` | 会话在发生可重试错误后被（重新）调度，准备由编排系统接管。 |
| `session.status_terminated` | 会话已结束且不可恢复——**无论是在正常完成时还是出现错误时**，并非仅在出错时触发。 |
| `session.updated` | 会话更新至少修改了一个字段——仅携带变更的字段（如预算被移除时携带 `budget: null`）。 |
| `session.usage` | 会话累计用量及追踪的列表费用快照——详见下文“达到会话预算”部分。 |
| `session.error` | 处理过程中发生错误。 |
| `span.model_request_start` | 模型推理开始。 |
| `span.model_request_end` | 模型推理结束。 |
| `span.outcome_evaluation_start` / `_ongoing` / `_end` | 面向结果的会话中评分器的进展——详见 `shared/managed-agents-outcomes.md`。 |
| `session.thread_created` | 派生子线程生成（多智能体场景），或顾问咨询启动（线程名称为 `anthropic.advisor`）——详见 `shared/managed-agents-multiagent.md`。 |
| `session.thread_status_running` / `_idle` / `_rescheduled` / `_terminated` | 线程状态的转换——主要出现在多智能体会话中，但单智能体会话的主线程在因达到会话预算而暂停时也会发出 `_idle` 状态（详见“达到会话预算”部分）。`_idle` 状态会携带 `stop_reason`。 |
| `agent.thread_message_sent` / `_received` | 线程间消息，携带 `to_session_thread_id` / `from_session_thread_id`（多智能体场景）。 |

流还会回显用户发送的事件（`user.message`、`user.interrupt`、`user.tool_confirmation`、`user.tool_result`、`user.custom_tool_result`、`user.define_outcome`），但当会话因预算而暂停时发送的 `user.interrupt` 会被接收并忽略，不会出现在流中（详见“达到会话预算”部分）。

仅限流传输的增量预览事件（`event_start`、`event_delta`）是唯一不遵循 `{domain}.{action}` 命名规范的例外——详见下文“实时预览”部分；它们绝不会出现在 `GET /v1/sessions/{id}/events` 中。

---

## 实时预览

默认情况下，助手的文本会以缓冲的 `agent.message` 事件形式到达流中——仅在产生这些文本的模型请求完成后才会发出。借助“实时预览”，您可以在模型仍在生成时逐步渲染这些文本。缓冲的 `agent.message` 始终是权威记录；即使客户端忽略预览，仍会收到完整且正确的流数据。其传输格式**并非** Messages API 流式传输：增量类型为 `content_delta`，而非 `content_block_delta`，因此 Messages API 的累积代码无法直接沿用。**按每次流连接选择加入**，通过添加 `event_deltas[]` 查询参数来实现，每个事件类型重复一次即可进行预览。允许的值为：`agent.message`、`agent.thinking`；任何其他值或请求中包含超过 100 个值都会返回 400 错误。**两个流端点均支持此参数**：会话级流（`GET /v1/sessions/{id}/events/stream`）以及每个会话线程的独立流（`GET /v1/sessions/{sid}/threads/{tid}/stream`）。在 Shell 中，请对 URL 加上引号，或将方括号百分号编码为 `%5B%5D`——直接使用 `[]` 会被解释为通配符模式。

**预览是线程范围的。** 每个连接仅预览其所读取的线程。子线程的预览会在该子线程的流上发送，**绝不会**被跨线程发布到会话级流；会话级流的预览始终限定在主线程范围内。如需实时查看子代理生成的文本，可打开该子代理的线程流——参见 `shared/managed-agents-multiagent.md`。每个连接运行一个累加器实例。

```python
stream = client.beta.sessions.events.stream(
    session_id=session.id,
    event_deltas=["agent.message"],
)
```

当预览的事件开始时，流会发出一个 `event_start` 事件，携带即将发生的事件的 `type` 和 `id`；对于 `agent.message`，其后会跟随多个 `event_delta` 事件，逐步传递增量文本：

```js
{"type": "event_start", "event": {"type": "agent.message", "id": "sevt_01abc..."}}
{"type": "event_delta", "event_id": "sevt_01abc...", "delta": {"type": "content_delta", "index": 0, "content": {"type": "text", "text": "Here is the summary"}}}
```

`event_start` 和 `event_delta` 本身没有独立的 `id` 或 `processed_at` 字段——它们唯一携带的标识就是所预览事件的 `id`。对于 `agent.thinking`，**仅**会发出 `event_start` 事件（表示“思考已开始”），之后不会再有增量事件，且作为预览结束的缓冲 `agent.thinking` 事件也不包含任何思考内容。它只是一个进度信号，而非内容载体；其中并无可供读取的内容。

**累积与对账模式。** 将预览视为一个以 `(event_id, index)` 为键的临时缓冲区。收到 `event_start` 时，为声明的 `id` 创建一个空条目。每收到一个 `event_delta`，就将 `delta.content.text` 追加到对应 `(event_id, delta.index)` 的位置，并渲染当前的完整文本。当缓冲的 `agent.message` 到达时，根据 `id` 匹配并**丢弃已累积的预览**，转而渲染消息的内容。这些标识始终一致：`event_start.event.id`、每个 `event_delta.event_id` 以及缓冲事件的 `id` 都是同一个值。正常情况下，顺序固定为：`session.status_running` → `span.model_request_start` → `event_start` → `event_delta`* → 缓冲的 `agent.message` → `span.model_request_end`。如果本轮发生错误或被中断，缓冲事件可能永远不会到达，但 `span.model_request_end` 仍会如期发出——一旦发现未对账的预览，请立即关闭。Python、TypeScript 和 Go SDK 均提供了实现该模式的累加器辅助工具；在其他 SDK 中，则需按照手动模式处理生成的事件类型。

**该模式依赖于两项保证：** 按到达顺序将预览的增量事件拼接起来，并以 `(event_id, index)` 为键，最终得到缓冲事件中 `content[index].text` 的一个前缀（只是前缀，不一定是全部文本——在负载较高时可能会丢失部分增量）；并且每个连接针对每个 `event_id` 至多发出一个 `event_start`，而缓冲事件则是该连接为该 `id` 发送的最后一项内容。**局限性：**
- **尽力而为** - 在高负载情况下，服务器可能会丢弃某个事件的增量数据；您会收到一个连续的前缀，但不会再收到该事件的其他增量。缓冲的 `agent.message` 仍然会完整到达。切勿将累积的预览视为最终结果。
- **重连时不回放** - 增量仅在已订阅的连接打开期间发送给该连接；这适用于会话级流和每个线程流。在模型请求开始后建立的连接不会收到该未完成事件的任何增量。断开连接后，请遵循§“断线重连时的合并模式”中的处理方式——历史事件获取会返回间隙期间发出的所有缓冲事件；错过的增量无法重新请求。
- **单线程、仅文本** - 预览仅涵盖连接正在读取的线程上的助手文本。工具调用、工具结果、MCP 结果以及任何*其他*线程上的活动均不会在该连接上进行预览。
- **永不持久化** - `event_start` / `event_delta` 仅存在于实时 SSE 流中，绝不会出现在 `GET /v1/sessions/{id}/events` 或任何线程的事件历史中。

**故障排除：**

| 您看到 | 含义 |
| --- | --- |
| 缓冲了事件但没有 `event_start` / `event_delta` | 此连接未订阅（`event_deltas[]` 是按连接而非按会话设置的），或者本轮对话是在另一个线程上运行的。可通过列出 `GET /v1/sessions/{sid}/threads` 来确认具体是哪个线程。 |
| 流 URL 返回 404 | 路径或 ID 错误，或者请求中缺少 managed-agents 的 Beta 标头——线程端点受 Beta 限制，若无该标头则不存在。线程路径为 `/threads/{tid}/stream`，**不是** `/threads/{tid}/events/stream`（不存在）或 `/events/stream`（仅限会话级别）。 |
| `event_deltas` 命名错误返回 400 | 只接受 `agent.message` 和 `agent.thinking`，最多 100 个值。 |

---

## 引导模式

通过事件接口驱动会话的实用模式。

### 先开启流再发事件

**先打开流，再发送事件。** 流只推送在其打开之后发生的事件，不会回放当前状态或历史事件。如果先发送消息再打开流，早期事件（包括快速的状态切换）会以批量形式被缓冲，您将失去实时响应的能力。

```ts
// 正确——同时开启流并发送
const [response] = await Promise.all([
  streamEvents(sessionId),   // 打开 SSE 连接
  sendMessage(sessionId, text),
]);

// 错误——流打开前发送的事件会以单批缓冲到达
await sendMessage(sessionId, text);
const response = await streamEvents(sessionId);
```

**若需完整历史记录，**请使用 `GET /v1/sessions/{id}/events`（分页列表）——流仅提供从连接建立起的实时事件。

### 断线重连后的处理

**SSE 流不支持回放。** 如果连接中断（如 httpx 读超时或网络短暂波动）并重新连接，您只能收到重新连接后发出的事件，间隙期间产生的事件将从流中丢失。

**合并模式：** 每次（重新）连接时，将流与历史事件获取并行执行，并按事件 ID 去重：

```python
def connect_with_consolidation(client, session_id):
    # 1. 先打开 SSE 流
    stream = client.beta.sessions.events.stream(session_id=session_id)

    # 2. 获取历史事件以覆盖可能的间隙
    history = client.beta.sessions.events.list(
        session_id=session_id,
    )

    # 3. 先输出历史事件，再输出流中的事件——按 event.id 去重
    seen = set()
    for ev in history.data:
        seen.add(ev.id)
        yield ev
    for ev in stream:
        if ev.id not in seen:
            seen.add(ev.id)
            yield ev
```

### 消息队列

**您无需等待响应即可发送下一条消息。** 用户事件会在服务器端排队并按顺序处理。这对于用户频繁发送后续消息的聊天桥接场景非常有用：
```ts
// 三个消息都进入同一个会话；代理按顺序处理它们
await sendMessage(sessionId, "总结一下 README");
await sendMessage(sessionId, "顺便也看看 CONTRIBUTING 指南");
await sendMessage(sessionId, "然后把两者对比一下");
// 流式传输一次——代理将这三个请求作为一个连贯的回合进行响应
```

事件可以随时发送到会话中。无需等待特定的会话状态，即可通过 `client.beta.sessions.events.send()` 将新事件加入队列。一个例外：当会话因预算耗尽而暂停（`stop_reason: budget_reached`）时，仅接受结算事件——此时发送 `user.message` 会返回 400 错误。详见§ 会话预算耗尽。

### 中断

`user.interrupt` 事件会**跳过队列**（优先于所有待处理的用户消息），并强制会话进入空闲状态。例外情况：当会话因预算耗尽而暂停时，中断请求会被接收但忽略——它不会被持久化，也不会改变任何状态（见§ 会话预算耗尽）。可用于“停止”、“算了”或“取消”类指令：

```ts
await client.beta.sessions.events.send(sessionId, {
  events: [{ type: 'user.interrupt' }],
});
```

代理会在任务中途停止。它不会将中断视为一条消息，只是简单地终止当前操作。可随后发送一条 `user` 事件，说明接下来应如何处理。如果已有结果正在生成，中断还会标记 `span.outcome_evaluation_end.result: "interrupted"`（参见 `shared/managed-agents-outcomes.md`），但在预算暂停状态下，中断会被接收但忽略（见§ 会话预算耗尽）。

**被中断的回合将以 `stop_reason: end_turn` 结束**——这与正常结束的回合具有相同的值。没有专门用于中断的停止原因，因此在仅根据 `stop_reason` 判断时，无法区分这两种情况；请记录您已发送了中断请求。

**对于已经处于空闲状态的会话，中断通常不会产生任何效果。** 唯一的例外是自托管环境中的会话：如果其工作进程未能成功完成所声明的任务（例如内存存储挂载错误），该会话会以 `stop_reason: requires_action` 的状态保持空闲，并且没有错误事件；此时发送 `user.interrupt` 可将该任务重新排队，供下一次工作进程获取（参见 `shared/managed-agents-self-hosted-sandboxes.md` § 内存存储 -> 故障排除）。

**在多代理会话中，省略 `session_thread_id` 会中断所有未归档的线程，包括主线程**——并非仅针对主线程。若要中断某一线程，请指定 `session_thread_id`。详情请参阅 `shared/managed-agents-multiagent.md`。

> **注意**：在当前实现中，中断事件的 ID 可能为空。排查问题时，请结合 `processed_at` 时间戳以及前后相关事件的 ID 进行分析。（此规则不适用于在预算上限处发送的中断——此类事件不会被持久化，因此无从定位。）

### 会话预算耗尽

使用预算创建的会话（参见 `shared/managed-agents-core.md` § 会话预算）会在超出预算时暂停，而非继续消耗。每次向模型发起请求前，平台都会检查已消耗的列表成本是否达到上限；若已达到，则暂停相应线程，会话将进入空闲状态，并设置 `stop_reason: budget_reached`，而不会直接终止。在流式传输中，暂停会以三个事件依次出现：

1. `session.thread_status_idle`，附带 `stop_reason: budget_reached`，每个暂停的线程都会收到这一事件。当某个线程的最后一次请求既超过预算上限又完成了本轮对话时，该线程会报告 `stop_reason: end_turn`，而会话整体仍显示 `budget_reached`——请关注**会话级别的** `stop_reason`，而非线程级别的，以判断是否发生暂停。
2. `session.usage`——会话累计用量及已追踪列表成本的快照。
3. `session.status_idle`，附带 `stop_reason: budget_reached`。`session.usage` 事件始终紧接在此空闲事件之前。

在上限状态下，会话仅接受**结算类事件**（`user.tool_confirmation`、`user.tool_result`、`user.custom_tool_result`、`user.interrupt`）；任何启动新工作的事件，包括`user.message`，都会导致400错误。当会话因预算耗尽而暂停（所有线程均处于上限状态）时，发送的`user.interrupt`事件会被接收但忽略：它不会出现在事件列表中，也不会改变任何状态。通过提高或移除预算即可继续执行。当一个线程在等待工具调用而另一个线程处于上限暂停状态时，会话级别的`stop_reason`为`requires_action`，而非`budget_reached`——结算该调用并不会触发模型请求，因此应按常规方式响应。

**没有任何事件能够恢复处于上限状态的会话。** 应改为更新会话的预算：将其设为高于已消耗列表成本的值（可高于或低于旧上限），或通过设置`"budget": null`来移除预算。经确认的预算更新会自动恢复被暂停的工作。

**`session.usage`** 记录了会话的累计令牌总量、`list_cost`（以美分为单位并四舍五入）、`active_seconds`（并发线程的重叠时间仅计算一次——这是计费所依据的运行时长）、`server_tool_use`计数（包括`web_search_requests`和`web_fetch_requests`——后者目前仅为信息性字段，始终为0，因为网页抓取暂不计费），以及已设置预算时的预算值。该字段会出现在事件列表和会话流中——流读取器无需额外获取即可看到达到上限时的最终成本；子线程的各自流中则不包含此信息。相同的累计数据也存在于会话对象的`usage`字段中，且每个线程的`usage`字段记录各自的`list_cost`和`active_seconds`——但各线程的成本**不**会累加为会话总值：会话数值还包含了会话的总运行时间，并且各项数据均独立四舍五入，因此会话数值才是权威数据。若需强制执行支出限额，应直接设置预算，而非轮询使用情况并自行中断会话——平台会在每次模型请求前进行检查。

### 事件负载
部分事件除了状态变更本身外，还会携带有用的元数据：
- `session.status_idle`事件包含`stop_reason`字段，详细说明会话停止的原因及用户需要采取何种进一步操作。
```json
{
  "id": "sevt_456",
  "processed_at": "2026-04-07T04:27:43.197Z",
  "stop_reason": {
    "event_ids": [
      "sevt_123"
    ],
    "type": "requires_action"
  },
  "type": "status_idle"
}
```
- `span.model_request_end`事件包含`model_usage`字段，用于成本追踪与效率分析：
```json
{
  "type": "span.model_request_end",
  "id": "sevt_456",
  "is_error": false,
  "model_request_start_id": "sevt_123",
  "model_usage": {
    "cache_creation_input_tokens": 0,
    "cache_read_input_tokens": 6656,
    "input_tokens": 3571,
    "output_tokens": 727
  },
  "processed_at": "2026-04-07T04:11:32.189Z"
}
```
- `agent.thread_context_compacted`事件——在对话历史被压缩以适应上下文时发出。包含`pre_compaction_tokens`字段，便于了解被压缩的数据量：
```json
{
  "id": "sevt_abc123",
  "processed_at": "2026-03-24T14:05:15.787Z",
  "type": "agent.thread_context_compacted"
}
```

### 归档
会话结束后，请将其归档以释放资源：
```ts
await client.beta.sessions.archive(sessionId);
```
> 归档**会话**属于常规清理工作——会话是针对单次运行的临时资源。**请勿将此操作推广至代理或环境**：它们是持久且可复用的资源，一旦归档即不可恢复（无法解档；新建会话也无法引用）。详情请参阅`shared/managed-agents-overview.md`中的“常见误区”部分。