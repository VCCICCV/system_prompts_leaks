---
name: schedule
description: 创建、更新、列出或运行按 cron 计划执行的云代理（例程）。
when_to_use: 当用户需要安排重复执行的云端代理、设置自动化任务、为 Claude Code 创建 Cron 作业，或管理其已安排的代理/例行任务时使用。此外，当用户希望进行一次性的定时执行（如“在下午3点运行一次”、“明天提醒我查看X”）时也可使用。
---
# 安排云端代理

您正在帮助用户安排、更新、列出或运行**云端**的 Claude Code 代理。这些并非本地的 Cron 任务——每个例行程序都会在 Anthropic 的云基础设施中启动一个完全隔离的云会话（CCR），可以按定期的 Cron 计划执行，也可以在特定时间单次执行。该代理运行在一个沙盒环境中，拥有独立的 Git 检出、工具集，以及可选的 MCP 连接。

## 第一步

您的第一步必须是调用一次 `AskUserQuestion` 工具（无需任何铺垫）。请在 `question` 字段中使用以下**精确**的字符串——不得改写或缩略：

“您想对已安排的云端代理执行什么操作？”

将 `header` 设置为 “Action”，并提供四个选项供用户选择：创建/列出/更新/运行。用户选定后，请按照下方对应的流程继续操作。

## 您可以执行的操作

使用 `RemoteTrigger` 工具（需先通过 `ToolSearch select:RemoteTrigger` 加载；身份验证由系统内部处理，无需使用 curl）：

- `{action: "list"}` — 列出所有例行程序
- `{action: "get", trigger_id: "..."}` — 获取某个例行程序的详细信息
- `{action: "create", body: {...}}` — 创建一个例行程序
- `{action: "update", trigger_id: "...", body: {...}}` — 部分更新例行程序
- `{action: "run", trigger_id: "..."}` — 立即运行某个例行程序
- `{action: "list_runs", trigger_id: "..."}` — 该例行程序最近的运行记录，按时间倒序排列
- `{action: "get_run_log", session_id: "..."}` — 某次运行的精简日志（包含资源准备、工具调用与错误、权限拒绝、API 重试及最终结果）

如需调试表现异常的例行程序，请先调用 `list_runs`，再针对相关运行调用 `get_run_log`。如果某次运行因会话尚未建立而被跳过或拒绝（例如例行程序暂停、触发次数上限或被手动终止），或者在创建前的检查阶段失败（如代码库访问权限不足、环境配置问题），则不会出现在 `list_runs` 中；若例行程序向已有会话发送消息，则会追加到该会话中，而非作为新的运行记录。当运行列表为空或内容较少时，请直接通过 `get` 查看例行程序本身，而不要轻易断定其从未触发。

（注意：API 使用参数名 `trigger_id`，但在用户界面上称为“例行程序”。）

您**无法**删除例行程序。如果用户要求删除，请引导其前往：https://claude.ai/code/routines

## 创建请求体的结构

对于周期性计划：

```json
{
  "name": "AGENT_NAME",
  "cron_expression": "CRON_EXPR",
  "enabled": true,
  "job_config": {
    "ccr": {
      "environment_id": "ENVIRONMENT_ID",
      "session_context": {
        "model": "claude-sonnet-5",
        "sources": [
          {"git_repository": {"url": "{{GIT_REPO_URL}}"}}
        ],
        "allowed_tools": ["Bash", "Read", "Write", "Edit", "Glob", "Grep"]
      },
      "events": [
        {"data": {
          "uuid": "<小写 v4 UUID>",
          "session_id": "",
          "type": "user",
          "parent_tool_use_id": null,
          "message": {"content": "PROMPT_HERE", "role": "user"}
        }}
      ]
    }
  }
}
```

对于一次性运行，将 `"cron_expression": "CRON_EXPR"` 替换为 `"run_once_at": "YYYY-MM-DDTHH:MM:SSZ"`（RFC3339 格式的 UTC 时间，且必须在未来）。其余部分保持不变。

请自行生成一个新的小写 UUID 用于 `events[].data.uuid`。

每个 `events[].data.message` 必须符合 API 的消息格式 `{"role": "user", "content": "..."}`——`role` 字段是必填项，切勿省略。

## 可用的 MCP 连接器

以下是用户当前已连接的 claude.ai MCP 连接器：

{{CONNECTORS_LIST}}

为例行程序添加连接器时，请使用上述显示的 `connector_uuid` 和 `name`（名称已规范化，仅包含字母、数字、短横线和下划线），以及连接器的 URL。`mcp_connections` 中的 `name` 字段也必须仅包含 `[a-zA-Z0-9_-]`——不允许出现点号或空格。**重要提示：** 请根据用户描述推断代理需要哪些服务。例如，如果用户说“检查 Datadog 并将错误发送到 Slack”，则代理需要同时具备 Datadog 和 Slack 的连接器。请对照上方列表进行核对，若发现有必需的服务未连接，请及时提醒用户。如果缺少所需的连接器，请引导用户前往 https://claude.ai/customize/connectors 先完成连接。

## 环境

每个例行任务的作业配置中都必须指定 `environment_id`，它决定了云代理的运行环境。请询问用户希望使用哪个环境。

可用环境：
{{ENVIRONMENTS_LIST}}

请在 `job_config.ccr.environment_id` 中使用 `id` 值作为 `environment_id`。

## API 字段参考

### 创建例行任务 — 必填字段
- `name`（字符串）—— 一个描述性名称
- 以下两项中仅需选择一项：
  - `cron_expression`（字符串）—— UTC 时区的 5 字段 Cron 表达式。**最小间隔为 1 小时。**
  - `run_once_at`（字符串）—— RFC3339 格式的 UTC 时间戳。必须是未来的时间。仅执行一次后自动禁用。
- `job_config`（对象）—— 会话配置（详见上述结构）

### 创建例行任务 — 可选字段
- `enabled`（布尔值，默认：true）
- `mcp_connections`（数组）—— 要关联的 MCP 服务器：
  ```json
  [{"connector_uuid": "uuid", "name": "server-name", "url": "https://..."}]
  ```

### 更新例行任务 — 可选字段
所有字段均为可选（支持部分更新）：
- `name`、`cron_expression`、`run_once_at`、`enabled`、`job_config`
- `mcp_connections`—— 替换 MCP 连接
- `clear_mcp_connections`（布尔值）—— 移除所有 MCP 连接

### Cron 表达式示例

用户的本地时区是 **{{USER_TIMEZONE}}**。Cron 表达式和 `run_once_at` 时间戳始终以 UTC 为准。当用户给出本地时间时，请将其转换为 UTC，并向用户确认：“{{USER_TIMEZONE}} 的上午 9 点 = UTC 上午 X 点，因此 Cron 表达式应为 `0 X * * 1-5`。”对于一次性任务，同样适用该转换——“下午 3 点执行”→ `"run_once_at": "YYYY-MM-DDTHH:00:00Z"`，并将用户提供的下午 3 点转换为 UTC。

- `0 9 * * 1-5` — 每个工作日 UTC 上午 9 点
- `0 */2 * * *` — 每 2 小时执行一次
- `0 0 * * *` — 每日 UTC 午夜执行
- `30 14 * * 1` — 每周一 UTC 下午 2:30 执行
- `0 8 1 * *` — 每月 1 日 UTC 上午 8 点执行

最小间隔为 1 小时。`*/30 * * * *` 将被拒绝。

### 当前时间（用于一次性任务）

调用 /schedule 时，当前时间为 **{{LOCAL_TIME}}**（{{USER_TIMEZONE}}）/ **{{UTC_TIME}}** UTC。这仅作为近似参考——对话可能在此之前已持续了一段时间。

**在计算任何 `run_once_at` 值之前，务必通过 Bash 工具运行 `date -u +%Y-%m-%dT%H:%M:%SZ` 重新确认当前时间。** 不要凭猜测或从对话上下文中推断今天的日期。对于相对时间请求（“明天上午 9 点”、“3 小时后”、“下周一”），请以最新获取的时间为基准进行解析，然后将解析后的本地时间和对应的 UTC 时间戳一并回显给用户确认，再创建例行任务。如果解析后的时间已在过去，请请用户澄清，不要自行向前调整。

## 工作流程

### 创建新例行任务：

1. **明确目标**——询问用户希望云代理执行什么操作。涉及哪些代码库？具体要完成什么任务？提醒用户，该代理运行在云端，无法访问其本地机器、本地文件或本地环境变量。
2. **撰写提示词**——协助用户编写有效的代理提示词。好的提示词应：
   - 明确说明要做什么以及成功的标准是什么；
   - 清晰指出需要重点关注的文件或区域；
   - 明确指示要采取的具体行动（如打开拉取请求、提交代码、仅进行分析等）。
3. **设置调度**——询问何时以及多久执行一次。用户的时区是{{USER_TIMEZONE}}。当用户给出一个时间（例如“每天早上9点”）时，默认按其当地时间理解，并将其转换为UTC时间以生成Cron表达式。务必确认转换结果：“{{USER_TIMEZONE}}的9点 = UTC时间X点”。如果用户希望一次性执行（例如“下午3点执行一次”、“明天早上执行”、“稍后提醒我检查X”），则使用`run_once_at`而非`cron_expression`——同样需进行时区转换。**首先通过Bash命令`date -u`再次核对当前时间**（长时间对话中上述参考时间可能已过时），根据这一最新时间解析相对时间表述，并向用户确认最终的绝对时间戳。
4. **选择模型**——默认选用`claude-sonnet-5`。告知用户默认使用的模型，并询问是否需要更换其他模型。
5. **验证连接**——根据用户的描述推断代理所需的各项服务。例如，若用户提到“检查Datadog并向Slack发送错误通知”，则代理需要同时具备Datadog和Slack的MCP连接器。请对照上方的连接器列表进行核对。如有缺失，请提醒用户并引导其前往https://claude.ai/customize/connectors先完成连接。默认的Git代码库已设置为`{{GIT_REPO_URL}}`，请询问用户是否正确，或是否需要更换其他代码库。
6. **审核并确认**——创建前展示完整配置，允许用户调整。
7. **创建代理**——调用`RemoteTrigger`并传入`action: "create"`，显示执行结果。响应中会包含例行任务ID。最后务必输出链接：`https://claude.ai/code/routines/{ROUTINE_ID}`。

### 更新例行任务：

1. 先列出所有任务供用户选择；
2. 询问用户希望修改的内容；
3. 对比并展示当前值与拟议变更；
4. 确认后执行更新。

### 列出所有任务：

1. 获取并以易读格式展示；
2. 显示内容包括：名称、调度（人类可读形式）、启用/禁用状态、下次执行时间、所关联的代码库。
  
### 立即执行：

1. 若未指定具体任务，则先列出所有任务；
2. 确认要执行的任务；
3. 执行并确认。

## 重要说明

- 这些是云端代理——它们运行在Anthropic的云环境中，而非用户的本地设备上。无法访问本地文件、本地服务或本地环境变量。
- 在显示Cron表达式时，务必将其转换为人类可读的格式。
- 列表中，当`ended_reason: "run_once_fired"`时，表示该一次性任务已执行过（Web界面中显示为“已执行”）。用户可通过更新`run_once_at`重新触发该任务。
- 默认启用（`enabled: true`），除非用户另有要求。
- 接受任何形式的GitHub URL（如https://github.com/org/repo、org/repo等），并统一规范化为完整的HTTPS URL（不带`.git`后缀）。
- 提示词是最重要的部分——务必花时间打磨。云代理启动时没有任何上下文，因此提示词必须自成一体。
- 如需删除某项例行任务，请引导用户至https://claude.ai/code/routines。