# 受管代理——多代理会话

协调代理可以在一个会话内将任务委派给其他代理。所有代理**共享容器和文件系统**；每个代理都在自己的**线程**中运行——这是一个上下文隔离的事件流，拥有独立的对话历史、模型、系统提示、工具、MCP 服务器以及该代理自身配置中的技能。线程是持久化的：协调代理可以向先前调用的子代理发送后续请求，而该子代理会保留之前的交互记录。

SDK 会在所有 `client.beta.{agents,sessions}.*` 调用中自动设置 `managed-agents-2026-04-01` 测试版标头；对于多代理场景，无需额外设置任何标头。

---

## 何时使用——先从 `self` 开始，再添加更廉价的工作者

**如果代理的工作可以拆分为多个独立的部分**——需要调研多个来源、处理大量文件或记录，或者任务模式是“分别研究 N 个事项，然后进行汇总”——或者单个任务会导致其上下文被大量内容填满时，**请使用多代理会话，而不是单一的长时间单线程循环。** 每个被委派的任务都在独立的线程中以全新的上下文窗口运行，这些线程在同一容器内并行执行，最终只有各子代理的报告会返回，从而保持协调代理的上下文规模较小。您无需编写任何编排代码：协调代理会自动获得委派工具，并自行决定何时使用；您的客户端仍然只需创建一个会话并读取一个流。

**第一步——最小且有效的团队配置就是代理本身。** 添加一个 `multiagent` 块，其中仅包含 `{"type": "self"}`。此时，协调代理可以将自包含的子任务交给自身的副本——这些副本使用相同的模型、系统提示和工具，但不再具备进一步委派的能力——并将它们的报告进行整合。除此之外，其他部分均无需更改。

```python
agent = client.beta.agents.create(
    name="研究助理",
    description="端到端地完成一项研究任务。可生成副本专门负责某个范围明确的子问题。",
    model="claude-opus-5-5",
    system="你是一名研究助理。当请求可以拆分为多个独立的子问题时，请将每个子问题委派给你的一个副本，每个副本负责一项自包含的任务，随后验证并整合它们的报告。",
    tools=[{"type": "agent_toolset_20260401"}],
    multiagent={"type": "coordinator", "agents": [{"type": "self"}]},  # 与单代理相比唯一的改动
)

session = client.beta.sessions.create(agent=agent.id, environment_id=env.id)  # 未发生变化
```

**步骤 2——将以阅读为主的任务迁移到更经济的模型上。** 外包的研究工作主要涉及检索、阅读和信息提取：输入的 token 数量多，但所需的深度推理较少。在一款规模较小的现代表型模型（如 Claude Haiku 4.5，或当任务需要更多判断力时使用 Claude Sonnet 5.5）上创建第二个代理，为其设置一个范围明确的 `system` 提示，并仅赋予其所需的工具，然后将其与 `self` 并列列出。名册中的条目仅作为引用：该代理独立运行于其自身的 `model`、`system` 和 `tools`，其产生的 token 按该模型的计费标准单独结算。大模型将 token 资源用于规划、核查和综合；而小模型则承担大部分的阅读工作。```python
worker = client.beta.agents.create(
    name="网络研究员",
    description="快速、低成本的只读型研究员。只需提供一个范围明确的问题，它会进行搜索、阅读，并附上来源报告发现。",
    model="claude-haiku-4-5",
    system="请准确回答所给的问题。根据需要尽可能多地搜索和阅读，然后简洁地报告发现，并为每项陈述提供来源网址或文件路径。",
    tools=[{
        "type": "agent_toolset_20260401",
        "default_config": {"enabled": False},
        "configs": [{"name": n, "enabled": True} for n in ("读取", "全局匹配", "正则匹配", "网页抓取", "网络搜索")],
    }],
)
```lead = client.beta.agents.create(
    name="研究负责人",
    description="负责规划和整合研究工作。可复制一个副本，专门负责一项大型子分析。",
    model="claude-opus-5-5",
    system="规划任务。将每个独立且需大量阅读的问题委派给网络研究员，每次仅生成一个独立的任务，并可并行处理多个任务。验证与最终整合工作由自己完成；仅在需要充分发挥自身能力的子分析时，才生成自己的副本。",
    tools=[{"type": "agent_toolset_20260401"}],
    multiagent={"type": "协调者", "agents": [worker.id, {"type": "self"}]},
)
```

**步骤3 - 增加专职专家。** 当子任务需要不同技能时，为每项任务配备一个专属代理——配备独立的模型、针对性更强的`system`提示，以及该任务所需的工具——并将这些代理按ID列在`self`旁边。在此例中，负责人自行做出调整，将同一份评审简报分别发送给多个只读评审线程进行独立审查（一个已登记的代理可以多次生成），同时将一份独立的简报交给测试编写者；随后，它对各评审结果进行去重，逐一核对代码，并将修复方案及总结留作己用。

```python
reviewer = client.beta.agents.create(
    name="并发性评审员",
    description="专用于审查竞态条件、死锁、更新丢失以及重试/幂等性相关缺陷的只读评审员。提供变更文件路径及必须满足的不变式，以文件:行号的形式报告发现。针对同一处变更，可同时生成多个实例进行独立评审。",
    model="claude-sonnet-5-5",
    system="仅审查指定的文件。重点关注并发性问题：未同步的共享状态、锁顺序、非原子性的读-修改-写操作、缺少幂等性的重试等。每项发现均以文件:行号、触发该问题的执行序列及建议的修复方案形式报告；若未发现问题，也应明确说明。",
    tools=[{"type": "agent_toolset_20260401", "default_config": {"enabled": False},
            "configs": [{"name": n, "enabled": True} for n in ("read", "glob", "grep")]}],
)
test_writer = client.beta.agents.create(
    name="测试编写者",
    description="负责编写并运行测试用例。提供模块路径、待验证的行为及测试命令，添加测试文件、执行测试，并以输出形式报告结果。",
    model="claude-sonnet-5-5",
    system="根据所给行为编写有针对性的测试用例，使用指定命令运行，并报告通过/失败结果、相关输出以及新增文件的路径。不得修改非测试代码；若被测代码存在问题，应予以反馈。",
    tools=[{"type": "agent_toolset_20260401", "default_config": {"enabled": True},
            "configs": [{"name": n, "enabled": False} for n in ("web_fetch", "web_search")]}],
)
lead = client.beta.agents.create(
    name="工程负责人",
    description="负责规划并实施代码变更，整合各专家的报告。可复制一个副本，专门负责一项独立的变更。",
    model="claude-opus-5-5",
    system="首先自行完成变更。随后并行地将变更路径及不变式发送给三位并发性评审员，将模块路径及测试命令发送给测试编写者。合并并去重评审结果，在采取行动前逐一核对代码，修复问题后交由测试编写者重新运行。设计决策及最终总结由自己保留。",
    工具=[{"类型": "agent_toolset_20260401"}],
    多智能体={"类型": "协调器", "智能体": [reviewer.id, test_writer.id, {"类型": "self"}]},
)
```

同样的架构也适用于由不同专长的组件构成的流水线：一个快速的文档提取器（例如基于 Claude Haiku 4.5），它为每个输入文档生成一个 JSON 文件；一个校验器，用于将每个文件与其源文档进行比对；以及一个总控模块，负责应用修正并将最终表格写入 `/mnt/session/outputs/`。在每个任务中都指定输入和输出路径：线程共享容器的文件系统，但彼此之间不共享对话内容。

- **适用场景：** 并行开展跨来源的研究；在不填满协调器上下文的情况下处理大量材料；使用职责明确、工具集精简的专职智能体，而非让单个智能体承担所有工具。 **不适用场景：** 小型的单步任务——每次委派都会产生一次往返通信和重新 briefing 的开销。
- **为协调器编写 `name` 和 `description`。** 协调器会根据每个 roster 条目的名称和描述来决定要启动哪个智能体（`self` 条目会以协调器自身的名称列出），因此请说明每个智能体擅长什么以及应交给它什么任务。roster 中的名称必须唯一；不要将任何智能体命名为 `self`。
- **在协调器的 `system` 提示中说明如何委派工作**——哪些任务该交给谁、每次可以同时委派多少个、自己保留哪些、以及哪些任务小到不值得委派（`shared/model-migration.md` 中的“向子智能体委派”示例提示可作为起点）。子智能体无法看到协调器的任何对话，因此每个任务都必须携带其所需的路径、约束条件和报告格式。启动操作会立即返回；子智能体的报告会在协调器后续的回合中送达。
- **网络工具的域名白名单应保持收敛，切勿扩大范围。** 某个 roster 智能体的 `web_search` / `web_fetch` 调用受其自身 `allowed_domains` / `blocked_domains` 的限制，同时也受调用它的所有智能体的相应列表，以及协调器当前的列表所约束（白名单取交集，黑名单取并集）。确保每个 roster 智能体的白名单都在协调器的白名单范围内——如果白名单不相交，工具仍然存在，但每次调用都会因 `url_not_allowed` 而失败。参见 `shared/managed-agents-tools.md` § 网络搜索与网页抓取设置。
- **限制：** roster 中最多包含 1–20 个条目（至多有一个 `self`；每个 roster 智能体可以被多次启动），仅允许一级委派（roster 成员不得再拥有自己的 `multiagent`），且每个会话最多同时运行 25 个线程——如果长时间会话需要更多线程，请归档已完成的线程（详见下文的“中断与归档线程”）。

以下各节是关于 roster、线程、事件以及客户端侧处理的参考；平台指南请参阅 `https://platform.claude.com/docs/en/managed-agents/multiagent-orchestration.md`。

---

## 在协调器上声明 roster

`multiagent` 是 `agents.create()` / `agents.update()` 中的一个 **顶级字段**——**不是** `tools[]` 中的一项。`agents` 列表中最多可包含 1–20 个 roster 条目。`sessions.create()` 不涉及任何变更——roster 会从协调器的配置中解析得出。

```python
orchestrator = client.beta.agents.create(
    name="工程负责人",
    model="claude-opus-5-5",
    system="您负责协调工程工作。将代码评审委托给评审员，并将测试编写委托给测试代理。",
    tools=[{"type": "agent_toolset_20260401"}],
    multiagent={
        "type": "协调器",
        "agents": [
            reviewer.id,                                            # 纯字符串 - 最新版本
            {"type": "agent", "id": test_writer.id, "version": 4},  # 固定版本
            {"type": "self"},                                       # 协调器自身
        ],
    },
)

session = client.beta.sessions.create(agent=orchestrator.id, environment_id=env.id)
```

| 名单条目 | 形式 | 备注 |
|---|---|---|
| 字符串简写 | `"agent_abc123"` | 引用已存储代理的最新版本。 |
| 代理引用 | `{type: "agent", id, version?}` | 省略 `version` 可在协调器保存时固定为最新版本。 |
| 自身 | `{type: "self"}` | 协调器可以生成自身的副本。 |
| 咨询顾问 | `{type: "advisor", model}` | 会话主线程可在回合中咨询的模型。每个名单最多一个。详见下文“咨询顾问”部分。 |

如果会话是使用 `agent_with_overrides` 创建的（参见 `shared/managed-agents-core.md` -> 为会话覆盖代理配置），这些覆盖仅适用于**协调器及其“自身”副本**。通过 ID 引用的名单代理始终使用其创建时的配置——覆盖不会传播到它们。

协调器线程会获得用于管理名单的委托工具：`list_agents`（查看名单）和 `send_to_agent`（向成员分配任务或发送消息）。名单中最多可包含**20个不同代理**；协调器可为每个代理生成**多个副本**。**仅允许一级委托**——且该限制会被强制执行，而非静默扁平化：若名单中某个代理本身也带有 `multiagent.agents` 名单，则创建或更新操作会因验证错误而失败。

**推理地域必须在名单中保持一致。** 当代理设置推理地域（`model.inference_geo`——参见 `shared/managed-agents-core.md` § 锁定推理地域）时，协调器的设置以及名单中每位成员的设置要么全部相同，要么全部未设置。名单中的地域不一致会导致 400 验证错误，无论是在保存代理时，还是在会话创建时通过 `model` 覆盖更改任何代理的地域设置时都会发生。

---

## 线程

会话级别的事件流即**主线程**——它显示协调器的追踪信息，以及子代理活动的精简视图（线程状态转换和跨线程消息，而非每个子代理的工具调用）。可通过各线程端点深入查看特定子代理：

| 操作 | HTTP | SDK（`client.beta.sessions.threads.*`） |
|---|---|---|
| 列出线程 | `GET /v1/sessions/{sid}/threads` | `.list(session_id)` |
| 获取单个线程 | `GET /v1/sessions/{sid}/threads/{tid}` | `.retrieve(thread_id, session_id=...)` |
| 归档 | `POST /v1/sessions/{sid}/threads/{tid}/archive` | `.archive(thread_id, session_id=...)` |
| 列出线程事件 | `GET /v1/sessions/{sid}/threads/{tid}/events` | `.events.list(thread_id, session_id=...)` |
| 流式传输线程事件 | `GET /v1/sessions/{sid}/threads/{tid}/stream` | `.events.stream(thread_id, session_id=...)` |

每个 `SessionThread` 都包含 `id`、`status`（`running` | `idle` | `rescheduling` | `terminated`）、`agent`（代理配置的解析快照——`id`、`name`、`model`、`system`、`tools`、`skills`、`mcp_servers`、`version`——但顾问线程的 `agent` 是一个仅含两字段的顾问表单：`{"type": "advisor", "model": ...}`——参见 § 顾问）、`parent_thread_id`（主线程为 `null`，但仍会列入列表）、`archived_at`，以及可选的 `stats`/`usage`。按线程统计的 `usage.list_cost` 数值**不**会累加为会话总和——会话数值还包含了会话运行时长，且各项数值均独立取整；以会话层级的 `usage.list_cost` 为准。**会话状态会聚合各线程状态**——只要有任何线程处于 `running` 状态，`session.status` 即为 `running`。最多允许**25 个并发线程**（顾问线程除外——参见 § 顾问）。在消费单一线程的流时，遇到 `session.thread_status_idle` 即应停止，并像处理会话层级的空闲状态一样检查其 `stop_reason`。

**会话预算为所有线程共享的上限**——不存在单一线程的单独上限。每条线程的用量按其所使用的服务模型计费，当共享上限被触及时，各线程会独立暂停（`stop_reason: budget_reached`）；一条线程可以暂停，而另一条线程仍可完成其正在进行的请求。处于 `requires_action` 状态的线程在会话层级的预算约束中具有更高优先级。详情请参阅 `shared/managed-agents-core.md` 的 § 会话预算。

---

## 多代理事件（在会话流上）

| 事件 | 载荷要点 | 含义 |
|---|---|---|
| `session.thread_created` | `session_thread_id`、`agent_name` | 新建了一条线程。 |
| `session.thread_status_running` | `session_thread_id`、`agent_name` | 该线程开始执行任务。 |
| `session.thread_status_idle` | `session_thread_id`、`agent_name`、**`stop_reason`** | 该线程正在等待输入——或因达到会话共享预算而暂停（`stop_reason: budget_reached`）。请检查 `stop_reason`（其结构与 `session.status_idle.stop_reason` 相同）。 |
| `session.thread_status_rescheduled` | `session_thread_id`、`agent_name` | 该线程在发生可重试错误后重新调度。 |
| `session.thread_status_terminated` | `session_thread_id`、`agent_name` | 该线程结束——已完成工作并自行终止（顾问咨询线程——参见 § 顾问）、已被归档，或遇到了不可恢复的错误。 |
| `agent.thread_message_sent` | `to_session_thread_id`、`to_agent_name`、`content` | 当前线程向另一条线程发送了消息。在主流上：协调器向某个代理发送了任务或后续指令。 |
| `agent.thread_message_received` | `from_session_thread_id`、`from_agent_name`、`content` | 消息从其他线程到达当前线程。在主流上：代理向协调器发送了报告或问题。 |

> **方向是相对于发出事件的线程而言的**，而非相对于协调器。同一项委派任务，在主流上表现为 `agent.thread_message_sent`，而在子线程的流上则为 `agent.thread_message_received`。一旦你读取的是子线程的流，就不要把 `_received` 理解为“子代理已完成”。

---

## 预览子代理的文本输出

每条线程的流都接受与会话层级流相同的 `event_deltas[]` 参数，因此你可以实时查看子代理的文本生成过程：

```
GET /v1/sessions/{sid}/threads/{tid}/stream?event_deltas%5B%5D=agent.message
```

**预览仅限于单一线程范围。** 子线程的预览仅在其自身的流上推送，绝不会同步到会话层级的流；后者始终只显示主线程的预览内容。因此，要实时观察子代理的输出，必须打开其专属线程的流——无论你如何设置，会话流都不会显示相关内容。> 警告：**仅显示普通助手文本的预览。** 子代理对协调者的回复通过 `agent.thread_message_sent` 事件传递，且不会被预览。因此，如果一个工作节点只负责汇报结果，即使在正确的线程上正确启用了预览功能，也不会有任何增量内容流式传输。要让子代理的回复产生实时预览，其提示中必须先要求它将答案以普通助手消息的形式写入自己的线程，然后再向协调者汇报。每个连接运行一个累积器，并在收到 `session.thread_status_idle` 时退出读取循环。关于启用、累积和整合的详细说明，请参阅 `shared/managed-agents-events.md` -> 实时预览。

---

## 顾问

`{"type": "advisor", "model": "<model id>"}` 这一名册条目为会话的**主线程**配备了一位顾问：该模型可在回合中途提供战略指导（规划方案、突破瓶颈、在完成前审查工作）。此条目仅有两个字段——`type` 和 `model`——并且可以与其他任何名册形式并存；即使名册中没有其他条目也完全有效。顾问还作为服务器工具出现在消息 API 中（`advisor_20260301`——参见 `shared/tool-use-concepts.md` -> 顾问）；但在托管代理的界面中，其配置与交付方式有所不同：名册条目**不包含 `max_uses`、`max_tokens` 或 `caching` 字段**，且建议是通过线程事件而非 `advisor_tool_result` 块来传递的。

```python
agent = client.beta.agents.create(
    name="后端工程师",
    model="claude-sonnet-5-5",
    system="您负责端到端地实现后端功能。",
    multiagent={
        "type": "coordinator",
        "agents": [{"type": "advisor", "model": "claude-opus-5-5"}],
    },
)
```

（Claude Opus 5.5 是默认的顾问选择。它属于“已屏蔽”类型的顾问——代理会在服务端读取其建议，但客户端只会看到 `[{"type": "redacted"}]`；详情请参阅下文的“明文与屏蔽交付”。若需客户端可读的建议，则只有当代理自身使用的模型为 `claude-opus-4-8` 或更低版本时，才能选用明文顾问，例如 `claude-opus-4-8`。而使用 Claude Opus 5.5、Claude Opus 5、Claude Sonnet 5.5、Claude Fable 5.1 或 Claude Mythos 5.1 的代理只能搭配屏蔽型顾问，因此无法获得客户端可见的建议——配对表参见 `shared/tool-use-concepts.md`。）

**规则：**
- **每份名册最多包含一条顾问条目。** 此条目占用保留名 `anthropic.advisor`——如果一份名册中还有一位成员名为 `anthropic.advisor`，则会返回 400 错误。在响应中，无论提交时的位置如何，顾问条目都会**最后**被回显。
- **配对在保存代理时进行验证：** 顾问模型必须达到最低能力要求，且代理自身的模型能力不得高于其顾问（两者能力相等也可配对）。配对无效时返回 400 错误。有效的配对关系与消息 API 中顾问工具的执行者与顾问对应表一致（参见 `shared/tool-use-concepts.md`）。
- **只有主线程可以咨询顾问。** 顾问并非名册中的普通代理：对协调者的 `list_agents` 工具不可见，也无法通过 `send_to_agent` 调用，且名册中的其他代理也不能直接咨询它。

**咨询流程：** 每次咨询都会由平台创建一条名为 `anthropic.advisor` 的临时线程，完成后该线程会自行终止；建议会以 `agent.thread_message_received` 事件的形式传递至主线程。典型事件顺序如下（保留名称在生命周期事件中作为 `agent_name` 传递，在交付事件中作为 `from_agent_name` 传递）：

1. `session.thread_created`
2. `session.thread_status_running`
3. `agent.thread_message_received`——顾问的建议
4. `session.thread_status_idle`（`stop_reason: end_turn`）
5. `session.thread_status_terminated`

咨询过程中不会产生 `agent.tool_use` 或 `agent.thread_message_sent` 事件，并且**不能保证建议的传递一定先于顾问线程的空闲或终止事件**——请勿将这些事件视为“建议已送达”的标志。**明文与屏蔽交付。** 客户端能否读取建议内容由顾问模型的策略决定，这与 Messages 顾问工具的结果变体一致：在该工具中返回明文的模型，在此处也会交付可读的文本内容；而返回屏蔽结果的模型则会在所有客户端界面上将消息内容设为 `[{"type": "redacted"}]`，但代理仍可在服务器端读取完整的建议内容。顾问的思考过程绝不会对外暴露。客户端自身无法发送 `redacted` 类型的消息块——包含此类消息块的事件将被视为 400 错误。

**失败与中断。** 一次失败的咨询，或通过携带顾问线程 `session_thread_id` 的 `user.interrupt` 事件主动中止的咨询，都不会导致代理回合失败：代理将在显示一条通用提示后继续执行。如果在咨询过程中发生会话级别的 `user.interrupt`，整个会话将按常规流程终止（包括主线程在内的所有线程），顾问线程也将随之结束且不提供任何建议。

**线程、计费与缓存。** 顾问线程 **不受 25 个并发线程上限的限制**。它们会出现在会话的线程列表中，其 `agent` 字段被设置为已配置的顾问形式（`{"type": "advisor", "model": ...}`），并且 `parent_thread_id` 被设置为主线程的 ID。咨询按照顾问模型的费率计费；其 token 数量会同时计入顾问线程的用量和会话的总用量。顾问侧的提示缓存是自动生效的，无需额外配置。

**移除顾问：** 更新代理的座席列表，将其从列表中移除；若顾问是座席列表中唯一的条目，则将座席列表清空，设置为 `"multiagent": null`。

---

## 子代理线程中的工具权限与自定义工具

当子代理需要您的客户介入时（例如因工具调用而暂停等待批准——`always_ask`，或 `auto` 模式下未作出决定——或者处理自定义工具的结果），该请求会以 `session_thread_id` 标识发起线程的方式 **跨线程转发至主线程**，因此您只需监听会话流即可。请使用 `user.tool_confirmation`（携带 `tool_use_id`）或 `user.custom_tool_result`（携带 `custom_tool_use_id`）进行回复，并 **原样回传来自原始事件的 `session_thread_id`**（SDK 参数类型及文档注释均要求如此）。服务器也会根据工具使用 ID 进行路由，因此这一回传更多是一种双重保障而非必要条件——但仍请一并提供。

```python
for event_id in stop.event_ids:
    pending = events_by_id[event_id]
    confirmation = {
        "type": "user.tool_confirmation",
        "tool_use_id": event_id,
        "result": "allow",
    }
    if pending.session_thread_id is not None:
        confirmation["session_thread_id"] = pending.session_thread_id
    client.beta.sessions.events.send(session.id, events=[confirmation])
```

同样的模式也适用于 `user.custom_tool_result`。

**多代理会话中的 `auto` 模式。** 只有您在主线程上发出的 `user.message` 事件才能促使服务器在 `auto` 模式下允许原本会被拒绝的工具调用；子代理线程中的任何内容都不具备这种效力（您的客户不会在该线程中发布消息，协调器发给子代理的消息也不携带此类信息）。对于服务器在 `auto` 模式下拒绝的工具调用，**不会**进行跨线程转发——相关事件及错误工具结果仅会出现在子代理自身的线程流中，而子代理将继续运行。

---

## 中断与归档线程- **在未指定 `session_thread_id` 的情况下调用 `user.interrupt` 会中断会话中的所有未归档线程，包括主线程**——这并非仅针对主线程的停止操作。若要定向中断某一线程，请传入 `session_thread_id`。
- **当目标是因 `requires_action` 而阻塞的子线程时**，中断会以错误结果关闭每个待处理的工具调用（返回消息：“工具执行在完成前被中断，请重试。”），并直接重新发出 `session.thread_status_idle` 事件，同时设置 `stop_reason: end_turn`——此时不会对模型进行采样。对于已处于空闲状态的线程，中断操作则无效果——但有一种例外：在自托管环境中，如果某个工作节点未能成功处理其申领的任务（例如内存存储挂载失败），该会话会保持“空闲”状态；此时调用 `user.interrupt` 会将该任务重新入队，以便下一个工作节点再次尝试处理（参见 `shared/managed-agents-self-hosted-sandboxes.md` 第 4.2 节“内存存储 -> 故障排除”）。
- **归档要求线程处于空闲状态，而 `requires_action` 状态也被视为空闲**——因此，处于待处理工具调用状态的线程可以直接归档。只有正在运行的线程才需要先被中断。

---

## 易错点

- **不要在 `sessions.create()` 或 `tools[]` 中配置 rosters。** `multiagent` 是顶级代理字段；请先更新协调器，再创建引用该协调器的新会话。
- **不要假定存在共享上下文。** 各线程之间共享文件系统，但不共享对话历史或工具。如果协调器需要子代理对某项内容采取行动，必须在委派消息中明确说明（或将其写入磁盘）。
- **深度大于 1 属于验证错误。** 如果某个代理本身也包含 `multiagent.agents` rosters，则创建或更新操作将失败——只有会话的协调器才能进行委托。
  
有关 Python 之外的各语言绑定，请参阅 WebFetch 文档 `https://platform.claude.com/docs/en/managed-agents/multiagent-orchestration.md`（详见 `shared/live-sources.md`）。