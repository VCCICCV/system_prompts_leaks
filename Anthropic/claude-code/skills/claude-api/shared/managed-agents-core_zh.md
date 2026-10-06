# 托管代理——核心概念

## 架构

托管代理围绕四个核心概念构建：

| 概念 | 端点 | 定义 |
|---|---|---|
| **代理** | `/v1/agents` | 一个持久化、带版本的对象，定义了代理的能力和角色：模型、系统提示、工具、MCP 服务器、技能。**必须在启动会话之前创建。** 详见下文的“代理”部分。 |
| **会话** | `/v1/sessions` | 与代理的有状态交互。通过 ID 引用预先创建的代理，并指定环境和初始指令。产生事件流。 |
| **环境** | `/v1/environments` | 用于定义容器资源配置的模板。 |
| **容器** | 无 | 代理的**工具**在此执行的隔离计算实例（如 Bash 脚本、文件操作、代码）。代理循环并不在此运行——它在 Anthropic 的编排层中运行，并通过工具调用来操控容器。 |

```
                       +-------------------------------------+
                       |  Anthropic 编排层                   |
代理（配置）------->|  （代理循环：Claude + 工具调用）    |
                       +--------------+----------------------+
                                      | 工具调用
                                      v
环境（模板）--> 容器（工具执行环境）
                                 |
                         会话 -+
                                 +-- 资源（文件、代码库、内存存储——在启动时挂载）
                                 +-- Vault ID（MCP 凭证引用）
                                 +-- 对话（事件流输入/输出）
```

> **代理的创建是先决条件。** 会话通过 ID 引用预先创建的代理——`model`/`system`/`tools` 均存储于代理对象中，不会出现在会话中。每个流程都从 `POST /v1/agents` 开始。

---

## 会话生命周期

```
重新调度 -> 运行 <-> 空闲 -> 终止
```

| 状态         | 描述                                                        |
| -------------- | ----------------------------------------------------------- |
| `idle` | 代理已完成当前任务，正在等待输入。它可能在等待通过 `user.message` 输入继续工作，也可能因等待 `user.custom_tool_result` 或 `user.tool_confirmation` 而被阻塞，或者因达到会话预算上限而暂停。附加的 `stop_reason` 提供了代理停止工作的更多原因信息。 |
| `running` | 会话已开始运行，代理正在积极处理任务。 |
| `rescheduling` | 会话在发生可重试错误后正在进行（重新）调度，准备由编排系统接管。 |
| `terminated` | 会话已结束，处于不可逆且无法使用的状态——**无论是正常完成还是因不可恢复的错误导致**。终止状态本身并不意味着失败；需获取会话详情以区分两者。 |

- 当会话处于 `running` 或 `idle` 状态时，可以发送事件。消息按顺序排队并依次处理。例外情况：当会话因预算耗尽而暂停（`stop_reason: budget_reached`）时，仅接受**结算事件**——即解决已在进行中的工作的事件（如 `user.tool_confirmation`、`user.tool_result`、`user.custom_tool_result`、`user.interrupt`），而不启动新工作——详见“会话预算”章节。
- 代理在收到新事件时从 `idle` 切换到 `running`，完成后又返回 `idle`。
- 错误会以 `session.error` 事件的形式出现在事件流中，而非作为状态值。每个会话在 Anthropic 控制台中都有一个实时跟踪视图，网址为 `https://platform.claude.com/workspaces/{workspace}/sessions/{session_id}`。创建会话后请立即打印此 URL，以便用户能够实时查看工具调用和消息的推送。**`{workspace}` 是 API 密钥所属的工作空间**——只有当该组织使用默认工作空间时才使用 `default`。会话响应中**不包含**工作空间字段，且控制台也没有与工作空间无关的会话路由，因此对于非默认工作空间，请使用该工作空间的 ID（可在控制台地址栏中查看，或将其作为配置项与 API 密钥一同提供）。如果将指向其他工作空间中会话的 `default` 链接打开，页面会显示“未找到会话”——页面上的“搜索工作空间”按钮可以定位到该会话，但不会自动跳转。

### 内置会话功能

- **上下文压缩**：当接近最大上下文长度时，API 会自动压缩会话历史，以维持对话的连续性。
- **提示缓存**：重复出现的历史令牌会被缓存，从而减少处理时间和成本。
- **扩展思考**：默认开启；`agent.thinking` 事件用于指示思考进度，但不包含实际的思考内容。

### 会话操作

| 操作 | 说明 |
|---|---|
| 列出/获取 | 分页列出会话，或按 ID 获取单个会话资源 |
| 更新 | 可以覆盖 `title`、`metadata` 以及会话本地的 `agent.tools` 和 `agent.mcp_servers`（参见 § 在会话进行中更新代理配置）。`budget` 只能更改或移除（参见 § 会话预算）。`vault_ids` 仅支持创建，更新请求会遭到拒绝。 |
| 归档 | 会话变为只读状态，且不可恢复。 |
| 删除 | 永久删除会话、事件历史、容器及检查点。 |

这些是运维和检查类的调用，通常由终端而非应用代码发起。在 Shell 中（参见 `shared/anthropic-cli.md`）：

```sh
ant beta:sessions list --transform '{id,title,status,created_at}' --format jsonl
ant beta:sessions retrieve --session-id "$SID"
ant beta:sessions:events stream --session-id "$SID"   # 实时查看事件
ant beta:sessions archive  --session-id "$SID"
ant beta:sessions delete   --session-id "$SID"
```

---

## 会话

会话是环境中的一个正在运行的代理实例。

### 会话对象

API 返回的关键字段：

| 字段           | 类型     | 描述                                         |
| --------------- | -------- | --------------------------------------------------- |
| `type`          | string   | 始终为 `"session"`                             |
| `id`            | string   | 唯一会话 ID                                   |
| `title`         | string   | 人类可读的标题                               |
| `status`        | string   | 可能的值：`idle`（空闲）、`running`（运行中）、`rescheduling`（重新调度中）、`terminated`（已终止） |
| `created_at`    | string   | ISO 8601 格式的时间戳                         |
| `updated_at`    | string   | ISO 8601 格式的时间戳                         |
| `archived_at`   | string   | ISO 8601 格式的时间戳（可为空）               |
| `environment_id`| string   | 环境 ID                                      |
| `agent`         | object   | 代理配置                                     |
| `resources`     | array    | 附加的文件、代码库和内存存储                 |
| `metadata`      | object   | 用户提供的键值对（最多 8 个键）              |
| `usage`         | object   | 累计使用情况：包括 token 数量、`server_tool_use`（网络搜索/获取请求次数）、`list_cost`（按公开列表价格计算的消耗，格式为 `{amount, currency}`，其中金额以最小单位的整数字符串表示——如分），以及 `active_seconds`（至少有 1 个线程在运行的时间；并发线程的重叠时间只计算一次——与 `stats.active_seconds` 不同，后者是按线程分别累加时间） |
| `budget`        | object   | 会话的支出上限（如果创建时设置了该上限）——参见 § 会话预算 |
| `stats`         | object   | 计时统计信息——`stats.active_seconds` 是按线程分别累加的时间，与 `usage.active_seconds` 不同 |

### 创建会话

**没有代理，会话就没有意义。** 会话通过 ID 引用预先创建好的代理。请先通过 `agents.create()` 创建代理，然后再引用它：

```ts
// 1. 创建代理（可复用且支持版本管理）
const agent = await client.beta.agents.create(
  {
    name: "编程助手",
    model: "claude-opus-5-5",
    system: "你是一个有用的编程助手。",
    tools: [{ type: "agent_toolset_20260401"}],
  },
);
// 2. 创建一个引用它的会话
const session = await client.beta.sessions.create(
  {
    agent: agent.id,  // 字符串简写表示最新版本。或者：{ type: "agent", id: agent.id, version: agent.version }
    environment_id: environmentId,
    title: "Hello World Session",
  },
);
```

> 提示：**在控制台中实时查看。** 在开发过程中，打印一个链接，这样您就可以直接点击进入会话界面，而无需解析事件流：`https://platform.claude.com/workspaces/{workspace}/sessions/{session.id}`。只有当 API 密钥属于组织的默认工作空间时，才使用 `{workspace}` 的 `default`；否则，请替换为该工作空间的 ID（会话响应中不包含此信息——请从控制台地址栏中读取，或将其设为配置项）。在为本地迭代的用户生成的代码中加入这条 `print`/`console.log` 语句。

**会话创建参数：**

| 字段           | 类型     | 必填 | 描述                                    |
| --------------- | -------- | ---- | --------------------------------------- |
| `agent`         | 字符串或对象 | 是   | 三种形式：字符串简写 `"agent_abc123"`（最新版本）；固定引用 `{type: "agent", id, version}`；或 `{type: "agent_with_overrides", id, version?, ...}`，仅针对该会话覆盖 `model`/`system`/`tools`/`mcp_servers`/`skills`——详见“为会话覆盖代理配置”章节 |
| `environment_id`| 字符串   | 是   | 环境 ID                                 |
| `title`         | 字符串   | 否   | 人类可读名称（显示在日志和仪表板中）      |
| `resources`     | 数组     | 否   | 文件、GitHub 仓库或内存存储，在容器启动时附加。内存存储仅能在会话创建时添加（无法通过 `resources.add()` 添加）。 |
| `initial_events`| 数组     | 否   | 创建时发送的事件，按顺序处理——将创建与首次发送合并为一次调用。详见下文“通过 `initial_events` 初始化会话”。 |
| `vault_ids`     | 数组     | 否   | Vault ID（`vlt_*`）——带有自动刷新功能的 MCP 凭证，以及在出站时被替换的 `environment_variable` 秘密。参见 `shared/managed-agents-tools.md` 中的“Vaults”部分。 |
| `budget`        | 对象     | 否   | 会话支出的硬性美元上限：`{type: "limit", max_list_cost: {amount, currency}}`。**仅限创建时设置**——之后可以更改或移除，但不能新增。详见“会话预算”章节。 |
| `metadata`      | 对象     | 否   | 用户提供的键值对                      |

#### 使用 `initial_events` 初始化会话

如果创建会话时未提供 `initial_events`，会话将注册为 `idle` 状态且不会开始任何工作；沙箱将在会话首次需要时才被 provision。如果传入**非空**的 `initial_events` 数组，则会在同一调用中启动代理循环——会话将**直接以 `running` 状态创建**，不会经过 `idle` 状态。如果客户端等待 `idle -> running` 的状态转换来判断工作是否开始，将会永远等待；请改查创建响应中的 `status` 字段。

```python
session = client.beta.sessions.create(
    agent=AGENT_ID,
    environment_id=ENVIRONMENT_ID,
    initial_events=[
        {"type": "user.message", "content": [{"type": "text", "text": "Review the auth module."}]},
    ],
)
```

- **仅接受 `user.message` 和 `user.define_outcome` 类型的事件**，最多 50 条。工具结果类事件（`user.tool_confirmation`、`user.tool_result`、`user.custom_tool_result`）会被拒绝，因为此时尚未开始代理回合；`user.interrupt` 也会被拒绝，因为没有可中断的回合。与计划部署的 `initial_events` 不同，会话的 `initial_events`**不接受** `system.message`。
- 每个事件都会在创建响应返回之前按列表顺序进行验证并持久化，并由服务器分配 ID——效果等同于在创建后立即通过发送事件端点发布这些事件。每条事件的内容规则与该端点相同。
- **创建响应中不会回显这些事件。** 如果需要其服务器分配的 ID，请使用 `sessions.events.list(session.id)` 重新读取。
- **验证是全有或全无机制：** 只要有一个事件验证失败，整个请求就会被拒绝，且不会创建会话。空列表等同于省略该字段。
- 拒绝情况：超过一条 `user.define_outcome` -> 400；`user.define_outcome` 缺少 `rubric` -> 400；整列事件中来自文件的 `document` 内容块超过 100 个 -> 400；请求体超过 32 MB -> 413。

因此，基于 outcome 的会话只需一次调用——在 `initial_events` 中传入一条 `user.define_outcome` 即可，无需先创建会话再发送该事件（参见 `shared/managed-agents-outcomes.md`）。

**代理配置字段**（传递给 `agents.create()`，而非 `sessions.create()`）：

| 字段         | 类型     | 必填 | 描述                                    |
| ------------- | -------- | ---- | --------------------------------------- |
| `name`        | 字符串   | 是   | 人类可读名称（1–256 个字符）            |
| `model`       | 字符串或对象 | 是   | Claude 模型 ID（纯字符串，或包含 `id`、`speed`、`effort` 和 `inference_geo` 的对象）。支持所有 Claude 4.5 及以上模型。详见“代理模型的 Effort 设置”和“推理地域的固定设置”章节。 |
| `system`      | 字符串   | 否   | 系统提示——定义代理的行为（最多 10 万字符） |
| `tools`       | 数组     | 否   | 包括三种类型：(1) 预置的 Claude Agent 工具（`agent_toolset_20260401`），(2) MCP 工具（`mcp_toolset`），以及 (3) 客户端自定义工具。最多 128 个。 |
| `mcp_servers` | 数组     | 否   | MCP 服务器连接——标准化的第三方能力（如 GitHub、Asana）。最多 20 个，名称需唯一。详见 `shared/managed-agents-tools.md` 中的“MCP Servers”部分。 |
| `skills`      | 数组     | 否   | 自定义的最佳实践上下文，具有渐进式披露机制。最多 20 个。详见 `shared/managed-agents-tools.md` 中的“Skills”部分。 |
| `description` | 字符串   | 否   | 代理的描述（最多 2048 个字符）          |
| `multiagent`  | 对象     | 否   | `{type: "coordinator", agents: [...]}`——该代理可委派工作的代理人名单。详见 `shared/managed-agents-multiagent.md`。 |
| `metadata`    | 对象     | 否   | 任意键值对（最多 16 个，键 <=64 个字符，值 <=512 个字符） |

### 会话预算

**会话预算**是在会话创建时设置的可选硬性支出上限。平台会持续按照**公开列表价格**（即会话的**列表成本**）对会话所消耗的一切进行计价，一旦总成本达到上限，便不再发出新的模型请求。达到预算的会话会**暂停并进入 `idle` 状态，停止原因是 `budget_reached`**——它不会被终止；历史记录和沙箱会被保留，修改或移除预算后，暂停的工作会自动恢复。

```python
session = client.beta.sessions.create(
    agent=AGENT_ID,
    environment_id=ENVIRONMENT_ID,
    budget={
        "type": "limit",
        "max_list_cost": {"amount": "2500", "currency": "USD"},  // 小单位：`"2500"` 表示 25.00 美元
    },
)
```- `type` 始终为 `"limit"`。`max_list_cost.amount` 是以**货币的最小单位（分）表示的金额，采用不带前导零的整数字符串**，且大于 0——例如，`"2500"` 表示 25.00 美元，`"50"` 表示 50 美分。使用字符串而非数字，因此不会进行浮点数四舍五入；诸如 `"25.00"` 之类的十进制形式将被拒绝。`max_list_cost.currency` 为大写 ISO-4217 代码——**仅支持 `USD` 这一种货币。**
- **计入列表成本的项目包括：** 按各调用模型的列表价格计算的模型 Token 数量、每 1,000 次网络搜索计费 10 美元，以及按每小时 0.08 美元计费的会话运行时长。列表成本*并非*您的合同价——在享受协商折扣的情况下，当按列表价格计算的总费用达到上限时，会话即触发限额，而实际计费支出可能更低。
- **执行机制为请求前的拦截：** 在每次模型请求之前，平台都会检查已消耗的列表成本是否已达到上限；若已达到，则暂停该线程。跨越上限的那次请求仍会完成，因此最终数值最多可能超出上限一个模型请求（每个运行中的线程）。请将预算视为对新增工作的约束，而非精确的停止点。
- 报告的 `list_cost` 会**四舍五入到最接近的美分**，而执行阶段则比较精确金额——四舍五入可能导致报告值与实际金额相差最多半美分，因此，即使某次会话的报告 `list_cost` 等于其上限，该会话也可能尚未被暂停。请以 `stop_reason: budget_reached`（或 `user.message` 中的 400 错误）作为达到预算上限的标志，而非报告的数值。
- **仅限创建时设置。** 向未设置预算的会话添加预算会导致 400 错误。更新操作仅允许两种变更：**修改上限**（新值可高于或低于旧上限，但必须严格大于已消耗的列表成本，否则返回 400 错误：“budget.max_list_cost 必须大于会话已消耗的列表成本”），或**移除预算**（设置 `budget: null`——`session.updated` 事件中直接携带 `budget: null`，无需单独的标志）。由于会话暂停时已消耗的成本通常略高于旧上限，因此应以会话报告的 `usage.list_cost` 为准来设定新上限，而非沿用旧的 `max_list_cost`。**移除后不可恢复**：一旦移除预算，便无法重新添加；如需继续设置上限，请通过变更来实现。
- **达到上限时，仅接受结算类事件**——即用于处理已在进行中的任务，而非启动新任务的事件，例如：`user.tool_confirmation`、`user.tool_result`、`user.custom_tool_result`、`user.interrupt`。当会话因预算已达上限而暂停（所有线程均处于暂停状态）时，发送的 `user.interrupt` 请求会被接受但忽略——该中断不会出现在事件列表中，也不会产生任何影响。如需继续，请提高或移除预算。任何试图启动新工作的操作（如 `user.message`）都会导致 400 错误，并明确指出是“列表成本”问题。没有任何事件能够恢复会话运行，只有变更或移除预算才能解除限制。
- **多智能体场景：** 所有线程共享同一预算，无单线程上限。各线程独立暂停；每条线程的费用按其所调用的模型分别计算。待处理的工具调用优先于预算上限：若会话中一条线程处于 `requires_action` 状态，另一条线程处于 `budget_reached` 状态，则会话层面仍显示 `requires_action`——请照常响应（结算类事件不受阻）。
- **未设列表价格的模型无法纳入预算管理：** 若已设置预算的会话在其代理（或任何名册中的代理，包括顾问使用的模型）中调用了未定价的模型，则创建请求会返回 400 错误。若正在运行的已设预算会话在使用过程中引入了此类模型，则更改预算的操作将被拒绝——需先移除预算方可继续。
- 达到预算上限时的流式行为及 `session.usage` 事件，请参阅：`shared/managed-agents-events.md` 第节“达到会话预算”。
- 定时部署也可设置预算——预算会复制到每次启动的会话中，且更新语义有所不同（可清空并重新添加）：请参阅 `shared/managed-agents-scheduled-deployments.md` 第节“部署预算”。> **这与 Messages API 的任务预算不是一回事。** 会话预算是在单个会话中由平台强制执行的、以美元计价的硬性上限。而 Messages API 中的 `task_budget` 是一种建议性的、以 token 计价的预算，用于指导模型在单个代理循环中的运行节奏。

---

## 代理

**所有托管代理流程都由此开始。** 代理对象是一种持久化、带版本的配置——您只需创建一次，之后每次启动会话时都通过 ID 引用它。没有代理，就没有会话。

### 代理对象

API 是**扁平的**——`model`、`system`、`tools` 等字段均为顶级字段，而非嵌套在 `agent:{}` 子对象中。

| 字段              | 类型     | 必填 | 描述                                        |
| ------------------ | -------- | ---- | ------------------------------------------- |
| `name`             | string   | 是   | 人类可读的名称                              |
| `model`            | string 或 object | 是   | Claude 模型 ID——可以是纯字符串，也可以是 `{id, speed?, effort?, inference_geo?}` |
| `system`           | string   | 否   | 系统提示                                    |
| `tools`            | array    | 否   | 代理工具集 / MCP 工具集 / 自定义工具         |
| `mcp_servers`      | array    | 否   | MCP 服务器连接                              |
| `skills`           | array    | 否   | 技能引用（最多 20 个）                      |
| `description`      | string   | 否   | 代理的描述                                  |
| `multiagent`       | object   | 否   | 协调人名单——参见 `shared/managed-agents-multiagent.md` |
| `metadata`         | object   | 否   | 任意键值对                                  |

### 生命周期：创建一次，多次运行，原地更新

代理是一种**持久化资源**，而不是每次运行的参数。推荐的使用模式如下：

```
+- 设置（仅一次）---------+     +- 运行时（每次调用）-+
| agents.create()        |     | sessions.create(             |
|   -> 保存 agent_id    | ---> |   agent={type:..., id: ID}   |
|     在配置/环境/数据库 |     | )                            |
+------------------------+     +------------------------------+
```

**反模式：** 在每个脚本运行的开头都调用 `agents.create()`。这样会导致孤立的代理对象不断累积，每次调用都会产生创建延迟，并且破坏了版本管理机制。如果在每个请求或每个定时任务中都看到 `agents.create()`，那就是错误的——应将其移到一次性设置阶段，并持久化保存 ID。

> **推荐做法：将代理和环境定义为文件，并通过 `ant apply` 进行同步。** 分工是：**控制平面使用 CLI，数据平面使用 SDK**——代理和环境是比较静态的资源，通过 `ant` 来管理（版本控制的文件，可手动或从 CI 同步）；而会话则是动态的，由您的应用通过 SDK 驱动。关于文件结构、`claude-lock.json` 以及 CI 流程，请参阅 `shared/anthropic-cli.md` 中的“版本控制的托管代理资源”部分。本文档其他地方提到的 SDK 中的 `agents.create()` 调用是代码层面的等效操作——当需要以编程方式创建时使用它，但对于人工维护的部分，优先使用文件和 `ant apply`。

### 代理模型的运行强度

可通过将 `model` 作为对象传递来设置运行强度：`{"id": "claude-opus-5-5", "effort": "high"}`。`effort` 接受一个强度级别字符串（`low`、`medium`、`high`、`xhigh`、`max`）或类似 `{"type": "high"}` 的对象。创建或更新响应会以对象形式返回该设置，并用默认值填充未指定的 `model` 字段。

> 警告：**会话级别的 `model` 覆盖会完全替换代理的 `model` 对象，因此代理自身的 `effort` 设置不会被继承。** 若要以特定强度运行会话，请在覆盖的 `model` 对象中设置 `effort`。如果指定的强度不被模型支持，则会返回 400 错误；若 `model` 覆盖中未指定 `effort`，则会按该模型的默认强度运行。同样的对象形式在快速模式下携带 `speed` 参数：`{"id": "claude-opus-5-5", "speed": "fast"}`。

### 锁定推理地域（`inference_geo`）

`model` 对象还接受 `inference_geo` 参数，用于锁定为代理模型请求提供服务的地域：`{"id": "claude-opus-5-5", "inference_geo": "us"}`。该参数可取值为 `"us"` 或 `"global"`——与消息 API 中 `inference_geo` 作为顶级请求参数不同，在这里它始终嵌套在 `model` 内部，而不会出现在顶层。若未设置，则每次模型请求都将遵循该请求发出时工作空间的默认推理地域。

- **各阶段均进行验证**：在保存代理、基于代理创建会话以及会话每轮处理时，都会根据工作空间的 `allowed_inference_geos` 列表检查该锁定设置。如果后续工作空间的允许列表被缩小导致该锁定不再有效，则无法再基于该代理创建新会话，且**正在运行的会话将拒绝继续处理**——锁定设置不具有追溯效力（工作空间依赖这些设置以确保合规性）。
- 如果为不支持地域锁定的模型设置 `inference_geo`，将返回 400 错误。
- **会话生命周期内固定不变**：锁定设置在会话期间不可更改。可在代理上设置，也可在创建会话时通过 `model` 覆盖来为单个会话设置或清除该设置（参见“为会话覆盖代理配置”一节）。
- **多代理名单必须地域一致**：协调器的锁定设置与名单中每位成员的设置要么全部相同，要么全部未设置——详见 `shared/managed-agents-multiagent.md`。
- 类似于 `effort`，会话级 `model` 覆盖中的 `inference_geo` 设置**会被应用**；由于覆盖会完全替换 `model` 对象，因此若覆盖中*省略*了 `inference_geo`，则会清空该会话的代理地域锁定设置。

### 版本管理

每次调用 `POST /v1/agents/{id}`（更新）都会创建一个新的不可变版本，版本号为从 1 开始的连续整数，每次更新递增。代理的历史记录采用追加方式——无法编辑过往版本。

**更新时的 `version` 字段为可选。** 可以提供该字段以实现乐观并发控制，也可以省略以无条件应用更新：

| `version` | 行为 | 适用场景 |
|---|---|---|
| 提供（必须 ≥ 1）| 若与代理当前版本不符，则返回 409 错误——**即使您发送的字段值已与存储值完全一致**。请重新读取并重试。 | 交互式调用方；推荐的默认设置 |
| 省略 | 无条件应用更新。最新一次更新会静默覆盖任何并发更新，且双方调用方均不会收到错误提示。 | 手动编写的同步循环——例如，使用 `agents.update()` 将检入的代理定义推送到系统的 CI 脚本，此时循环本身负责维护代理状态 |

**更新语义。** 省略的字段将保留原值。标量字段（`model`、`system`、`name`、`description`）会被替换；其中 `system` 和 `description` 可以通过设置为 `null` 来清空，而 `model` 和 `name` 则不能。数组字段（`tools`、`mcp_servers`、`skills`）会被整体替换——设置为 `null` 或 `[]` 即可清空。**在您提供的 `model` 对象中，只有 `effort` 是例外：** 如果模型 `id` 未变，省略 `effort` 则保持原有级别；若更换了 `id`，省略 `effort` 则会重置为新模型的默认值。`model` 的其他字段会随对象一起被替换——**若仅提供 `model` 而未提供 `inference_geo`，则会清空代理的推理地域锁定设置。**

**版本管理的作用：**
- **可复现性**——将会话锁定到某个已知良好的配置：`{type: "agent", id, version: 3}`
- **安全迭代**——在不影响已在旧版本上运行的会话的情况下更新代理
- **回滚**——如果新的系统提示造成功能退化，可在调试期间将新会话重新锁定到先前版本

**`version` 字段为可选。** 可以省略（或使用字符串简写 `agent="agent_abc123"`），以便在创建会话时获取最新版本。也可显式传入（如 `{type: "agent", id, version: N}`），以实现可复现性的锁定。

**获取要固定的版本号：** `agents.create()` 和 `agents.update()` 都会在响应中返回 `version` 字段。请将其与 `agent_id` 一起存储。要获取现有代理的当前最新版本：`GET /v1/agents/{id}` -> `.version`。

**何时更新 vs 创建新代理：** 当概念上是同一个代理，只是行为有所调整（如优化了提示词、新增了工具）时，使用更新（`POST /v1/agents/{id}`）。当代理代表不同的角色或用途时，则创建新代理。经验法则：如果会给它起相同的名字，就选择更新。

### 代理相关端点

| 操作           | 方法   | 路径                                  |
| -------------- | ------ | ------------------------------------- |
| 创建           | `POST` | `/v1/agents`                          |
| 列表           | `GET`  | `/v1/agents`                          |
| 获取           | `GET`  | `/v1/agents/{id}`                     |
| 更新           | `POST` | `/v1/agents/{id}`                     |
| 归档           | `POST` | `/v1/agents/{id}/archive`             |

> 警告：**归档操作不可逆。** 归档后，代理将变为只读状态：已有会话仍可继续运行，但 **新会话无法引用该代理**，且无法取消归档。由于代理没有删除功能，归档即为最终生命周期状态。切勿将生产环境中的代理作为日常清理而随意归档，请务必事先与用户确认。

### 在会话中使用代理

可通过字符串 ID（使用最新版本）或通过指定明确版本的对象来引用代理：

```python
# 字符串简写形式——使用代理的最新版本
session = client.beta.sessions.create(
    agent=agent.id,
    environment_id=environment_id,
)

# 或者固定到某个特定版本（整数）
session = client.beta.sessions.create(
    agent={"type": "agent", "id": agent.id, "version": agent.version},
    environment_id=environment_id,
)
```

### 为单个会话覆盖代理配置

第三种 `agent` 形式 `agent_with_overrides` 可以在 **单个会话** 中替换代理的部分配置——例如尝试不同的模型或临时授予额外的工具，而无需对代理进行版本管理。传入 `id`（以及可选的 `version`；省略时默认为最新版本，与前两种形式一致），并可同时指定 `model`、`system`、`tools`、`mcp_servers`、`skills` 等字段：

```python
session = client.beta.sessions.create(
    agent={
        "type": "agent_with_overrides",
        "id": agent.id,
        "model": "claude-opus-5-5",   // 为本次会话替换代理的模型
        "system": None,           // 清除本次会话的系统提示词
    },
    environment_id=environment_id,
)
```

每个可覆盖字段均遵循三态规则：
- **省略** -> 会话将继承所引用代理版本的值。
- **`null`（或列表字段为 `[]`）** -> 会话将以该字段被清空的状态运行。此规则对 `system` 和 `skills` 完全适用。有三条例外：`model` 永远不可清空（`model: null` 将返回 400 错误 `agent_model_required`）；当会话的有效 `skills` 非空时，清空 `tools` 也会返回 400 错误（因为技能需要 `read` 工具）；当有效 `tools` 中仍包含引用代理某台服务器的 `mcp_toolset` 时，清空 `mcp_servers` 也会返回 400 错误——请在同一请求中同时覆盖 `tools` 以移除这些条目，然后再清空 `mcp_servers`。
- **指定一个值** -> 将**完全替换**代理的对应值。覆盖不会合并——`tools` 的覆盖必须列出会话应具备的所有工具。`model` 的覆盖也会完全替换代理的 `model` 对象：代理自身的 `effort` 不会被沿用，因此需在覆盖的 `model` 对象中设置 `effort`，以指定会话的特定推理强度（如果模型不支持该强度，则会返回 400 错误；若 `model` 覆盖中未指定 `effort`，则按该模型的默认推理强度运行）。`model` 覆盖中的 `inference_geo`**会被应用**——由于对象被完全替换，若覆盖中省略了它，将会清除代理的地理定位配置，使会话采用工作空间的默认推理地理区域。被覆盖的值将在会话创建时根据工作空间的 `allowed_inference_geos` 进行验证。

覆盖仅限于会话级别：它们**不会**修改代理资源或创建新的代理版本。响应中的 `agent` 对象反映了覆盖后的配置，而其 `id` 和 `version` 仍标识基础代理——因此您可以将会话追溯至其基础代理。在多代理会话中，覆盖适用于协调器及其 `{type: "self"}` 复制；通过 ID 引用的名册代理始终使用其创建时的原始配置（参见 `shared/managed-agents-multiagent.md`）。

### 在会话进行中更新代理配置

`sessions.update()` 可以在**现有**会话上更改 `agent.tools` 和 `agent.mcp_servers`（包括权限策略以及每种工具的 Web 设置——`allowed_domains` / `blocked_domains` 等，详见 `shared/managed-agents-tools.md` § Web 搜索与 Web 抓取设置）。更新后的域名列表将应用于会话剩余部分。这属于**会话级别的覆盖**，不会创建新的代理版本，也不会回传到代理对象。提供的数组是**完全替换**；如需添加单个工具，请先 `GET` 会话，修改后再 `POST` 回去。会话必须处于 `idle` 状态——若正在运行，请先中断。`vault_ids` 仅可在创建时设置：SDK 中虽有此更新参数，但 API 会拒绝（“暂不支持”），请在创建会话时附加保险库。

在代理配置字段中，只有 `tools` 和 `mcp_servers` 可以在会话创建后更改——若要使用不同于代理配置的 `model`、`system` 或 `skills`，请在创建时使用 `agent_with_overrides`（见上文）。`title`、`metadata` 和 `budget` 有各自的会话更新路径（参见 § 会话操作 / § 会话预算）。代理的模型配置（包括其 `inference_geo` 地理定位）及其配置的 `system` 字段在会话生命周期内是固定的；您仍可通过发送 `system.message` 事件，在回合之间**追加系统级上下文**（参见 `shared/managed-agents-events.md` § 在会话中添加系统上下文）。

```python
client.beta.sessions.update(
    session.id,
    agent={
        "tools": [
            {"type": "agent_toolset_20260401"},
            {"type": "mcp_toolset", "mcp_server_name": "linear"},
        ],
        "mcp_servers": [{"type": "url", "name": "linear", "url": "https://mcp.linear.app/sse"}],
    },
)
```

