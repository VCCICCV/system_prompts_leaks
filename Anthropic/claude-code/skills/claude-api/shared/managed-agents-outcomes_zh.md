# 托管代理 - 结果

**结果**将会话从“对话”提升为“工作”：您定义“完成”的标准，框架会执行迭代 -> 评分 -> 修订的循环，直到工件满足评分标准、达到 `max_iterations` 或被中断。一个独立的**评分器**（独立的上下文窗口）会根据您的评分标准对每次迭代进行打分，并将各指标的差距反馈给代理。

SDK 会在所有 `client.beta.sessions.*` 调用中自动设置 `managed-agents-2026-04-01` 测试版头；对于结果功能，无需额外设置其他头。

**代码生成默认行为：** 当为一项包含交付物（如工件、报告、拉取请求或数据集）的任务生成托管代理客户端代码时，请使用 `user.define_outcome` 而非 `user.message` 来启动会话——根据任务描述起草一份初始评分标准，并标记为供用户调整。将纯 `user.message` 启动方式保留给真正意义上的对话型会话。

---

## `user.define_outcome` 事件

结果并不是 `sessions.create()` 的一个字段。您先创建一个普通会话，然后发送 `user.define_outcome` 事件。代理在收到该事件后即开始工作——**请勿同时发送 `user.message`** 来启动它。

您可以通过在会话的 `initial_events` 数组中传入单个 `user.define_outcome`，将这两个调用合并为一次——同一个事件、相同的规则，只需一次往返（参见 `shared/managed-agents-core.md` 中的“通过 `initial_events` 种植会话”部分）。如果该数组中包含多个 `user.define_outcome`，或者其中缺少 `rubric` 字段，则整个创建请求将因 400 错误而被拒绝。

```python
session = client.beta.sessions.create(
    agent=AGENT_ID,
    environment_id=ENVIRONMENT_ID,
    title="Costco 的财务分析",
)

client.beta.sessions.events.send(
    session_id=session.id,
    events=[
        {
            "type": "user.define_outcome",
            "description": "构建 Costco 的 DCF 模型（.xlsx 格式）",
            "rubric": {"type": "text", "content": RUBRIC_MD},
            # 或者："rubric": {"type": "file", "file_id": rubric.id}
            "max_iterations": 5,  # 可选；默认 3，最大 20
        }
    ],
)
```

| 字段 | 类型 | 备注 |
|---|---|---|
| `type` | `"user.define_outcome"` | |
| `description` | 字符串 | 任务描述。这是代理努力达成的目标——无需再发送单独的 `user.message`。 |
| `rubric` | `{type: "text", content}` \| `{type: "file", file_id}` | **必填。** 使用 Markdown 格式，明确列出可独立评分的标准。可通过 `client.files.upload(...)` 上传一次，以便在不同会话间重复使用。 |
| `max_iterations` | 整数 | 可选。默认值为 **3**，最大值为 **20**。 |

该事件会在流中被回显，并附带由服务器分配的 `outcome_id` 和 `processed_at` 时间戳。

> **编写评分标准。** 应使用明确且可评分的标准（例如“CSV 文件中包含数值型的 `price` 列”），而非主观感受（如“数据看起来不错”）——因为评分器会独立评估每个标准，模糊的标准会导致循环过程不稳定。如果您没有现成的评分标准，可以让 Claude 分析一个已知良好的工件，并将其分析结果转化为评分标准。当为未提供评分标准的用户生成代码时，应根据其任务描述自行起草一份评分标准——包含 5–10 个具体指标，涵盖工件的格式、所需内容及质量下限——并将其作为初始模板供用户调整；切勿因未收到评分标准而省略结果定义步骤。

---

## 结果相关的事件

这些事件会与常规的 `agent.*` 和 `session.*` 事件一同出现在标准事件流中（`sessions.events.stream` / `.list`）。| 事件 | 负载亮点 | 含义 |
|---|---|---|
| `span.outcome_evaluation_start` | `outcome_id`、`iteration`（从0开始计数） | 评分器开始对第*N*次迭代进行评分。 |
| `span.outcome_evaluation_ongoing` | `outcome_id` | 评分器运行时的心跳信号。评分器的推理过程是不透明的——你只能看到它在工作，而看不到它的具体思路。 |
| `span.outcome_evaluation_end` | `outcome_evaluation_start_id`、`outcome_id`、`iteration`、`result`、`explanation`、`usage` | 评分器完成了一次迭代。`result`决定了后续流程（见下表）。 |

### `span.outcome_evaluation_end.result`

| `result` | 下一步 |
|---|---|
| `satisfied` | 会话进入`idle`状态。该结果为最终结果。 |
| `needs_revision` | 代理启动下一次迭代。 |
| `max_iterations_reached` | 不再进行评分器循环。代理可能会执行最后一次修改，然后会话进入`idle`状态。 |
| `failed` | 会话进入`idle`状态。说明评分标准与任务要求存在根本性不匹配（例如，任务描述与评分标准相互矛盾）。 |
| `interrupted` | 只要活动中的结果收到`user.interrupt`事件，就会发出此事件——**即使评价尚未开始**。此时，`outcome_evaluation_start_id`为空字符串而非事件ID，因此在使用它作为查找键之前请先检查。（但如果是在会话预算暂停状态下收到的中断，则会被接受并忽略——参见`shared/managed-agents-events.md`第“达到会话预算”一节。） |

```json
{
  "type": "span.outcome_evaluation_end",
  "id": "sevt_01jkl...",
  "outcome_evaluation_start_id": "sevt_01def...",
  "outcome_id": "outc_01a...",
  "result": "satisfied",
  "explanation": "所有12项标准均已满足：收入预测采用了5年的历史数据，……",
  "iteration": 0,
  "usage": { "input_tokens": 2400, "output_tokens": 350, "cache_creation_input_tokens": 0, "cache_read_input_tokens": 1800 },
  "processed_at": "2026-03-25T14:03:00Z"
}
```

---

## 查询状态与获取交付物

**状态**——您可以监听流中的`span.outcome_evaluation_end`事件，也可以轮询会话并读取`outcome_evaluations`：

```python
session = client.beta.sessions.retrieve(session.id)
for ev in session.outcome_evaluations:
    print(f"{ev.outcome_id}: {ev.result}")  # 输出：outc_01a...: satisfied
```

**交付物**——代理会将文件写入`/mnt/session/outputs/`目录。会话进入空闲状态后，可通过Files API以`scope_id=session.id`的方式获取这些文件。这与`shared/managed-agents-environments.md`中“会话输出”部分所描述的机制相同（包括`files.list`在指定`scope_id`时所需的`managed-agents-2026-04-01`头信息）。

---

## 交互规则与注意事项- **一次处理一个结果。** 只有在前一个结果的终端 `span.outcome_evaluation_end` 事件（`satisfied` / `max_iterations_reached` / `failed` / `interrupted`）完成后，才能通过发送下一个 `user.define_outcome` 来进行链式调用。会话会在多个链式结果之间保留历史记录。
- **引导是允许的，但不是必须的。** 您可以在结果处理过程中发送 `user.message` 事件来适当引导方向，但代理本身已经知道会持续工作直到达到终端状态，因此无需发送“继续”的提示。（例外：当会话因预算耗尽而暂停时，仅接受结算类事件——此时发送引导性质的 `user.message` 或链式 `user.define_outcome` 都会返回 400 错误；参见 `shared/managed-agents-events.md` 中关于“达到会话预算”的说明。）
- **`user.interrupt` 会暂停当前结果**——它会将结果标记为 `result: "interrupted"`，并将会话置为 `idle` 状态，等待新的结果或新一轮对话。（例外：如果在会话因预算耗尽而暂停时发送中断请求，则该请求会被接受但忽略，结果仍保持活动状态——参见 `shared/managed-agents-events.md` 中关于“达到会话预算”的说明。）
- **达到终端状态后，会话可再次使用**——您可以继续对话，也可以定义一个新的结果。
- **结果与会话创建字段无关。** 请勿在 `sessions.create()` 请求中包含 `outcome`、`rubric` 或 `description` 字段——结果始终通过 `user.define_outcome` 事件单独发送。
- **空闲检测逻辑保持不变。** 在您的轮询循环中，请继续使用 `event.type === 'session.status_idle' && event.stop_reason?.type !== 'requires_action'` 的条件——**不要**仅根据 `span.outcome_evaluation_end` 进行判断（当状态为 `needs_revision` 时，会话仍会继续运行）。详情请参阅 `shared/managed-agents-client-patterns.md` 中的模式 5。
  
有关原始 HTTP 数据结构以及除 Python 外的各语言 SDK 绑定，请参阅 WebFetch 文档：`https://platform.claude.com/docs/en/managed-agents/define-outcomes.md`（另请参考 `shared/live-sources.md`）。