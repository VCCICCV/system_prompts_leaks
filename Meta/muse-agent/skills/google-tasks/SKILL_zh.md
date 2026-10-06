---
name: "google_tasks"
description: "管理用户的 Google 任务：包括任务列表、任务详情、创建、更新及标记完成。"
icon: "google_tasks"
metadata: { "不包含在提示中": 假 }
---
# Google Tasks

所有操作都通过 `hatch_gws_cli tasks <resource> <method>` 来执行。资源分为 `tasklists` 和 `tasks` 两种，因此任务相关的命令会重复使用 `tasks`：例如 `hatch_gws_cli tasks tasks list`（服务为 `tasks`，资源为 `tasks`，方法为 `list`）。参数以 `--params` JSON 对象的形式传递，创建和编辑操作则需附加 `--json` 请求体。以下流程给出了每项操作的完整命令，请按原样使用；如需查看某命令的选项，请运行 `hatch_gws_cli tasks <resource> --help`；如需了解某方法的 `--params` 或 `--json` 格式，请运行 `hatch_gws_cli schema tasks.tasks.insert` 等命令。

任务的 `due` 字段是一个符合 RFC3339 标准的日期时间戳，例如 `2026-04-16T00:00:00Z`，但 Google Tasks 只使用日期部分——不包含具体时间，也不支持提醒。默认任务列表为 `@default`。

## 连接
在执行任何命令之前，Tasks 需要先进行一次连接。运行 `hatch_gws_cli tasks status` 检查连接状态。如果未连接，将该命令返回的 `connect_url` 原封不动地以 `[Connect Google Tasks](<connect_url>)` 的形式发送给用户，并等待用户点击。请勿自行构造 URL、引导用户前往设置页面或要求用户提供凭据。

如需断开连接，运行 `hatch_gws_cli tasks disconnect`，并将返回的 `disconnect_url` 以 `[Disconnect Google Tasks](<disconnect_url>)` 的形式发送给用户。

如果某个命令提示认证失败或未连接，请重新运行 `status` 并按照其返回的链接完成连接。如果 `status` 也无法访问，则告知用户当前设备暂不支持 Google Tasks，并停止后续操作。认证流程仅通过 `status` 和 `disconnect` 完成，无需手动编写凭据文件或直接调用 `gws auth`。

## 常见操作流程

### 查看任务列表和任务
- 查看待办列表：`hatch_gws_cli tasks tasklists list`。当用户询问自己有哪些列表，或希望按名称选择某个列表时使用此命令。
- 查看某个列表中的未完成任务：`hatch_gws_cli tasks tasks list --params '{"tasklist":"@default","showCompleted":false}'`。除非用户指定了特定列表，否则使用 `@default`；若要显示已完成的任务，可设置 `"showCompleted":true,"showHidden":true`（已完成的任务会随时间从默认视图中消失）。
- 查看指定时间段内的待办任务：`hatch_gws_cli tasks tasks list --params '{"tasklist":"@default","dueMin":"<start>","dueMax":"<end>"}'`（采用 RFC3339 格式），适用于查询“本周有哪些待办事项”。
- 查看单个任务的详细信息或获取其 ID 以便编辑：`hatch_gws_cli tasks tasks get --params '{"tasklist":"@default","task":"<id>"}'`。

### 添加任务
`hatch_gws_cli tasks tasks insert --params '{"tasklist":"@default"}' --json '{"title":"买 groceries","notes":"牛奶、鸡蛋","due":"<date>"}'`。其中 `title` 为必填项；当用户提供备注和截止日期时，再一并添加。

### 标记任务为已完成或重新打开
- 标记为已完成：`hatch_gws_cli tasks tasks patch --params '{"tasklist":"@default","task":"<id>"}' --json '{"status":"completed"}'`。
- 重新打开：同上，但 `status` 设置为 `"needsAction"`。

### 编辑或调整任务时间
`hatch_gws_cli tasks tasks patch --params '{"tasklist":"@default","task":"<id>"}' --json '{"title":"...","due":"<date>"}'`。仅更新需要修改的字段。

### 整理任务
- 移动任务、将其设为子任务或调整顺序：`hatch_gws_cli tasks tasks move --params '{"tasklist":"@default","task":"<id>","parent":"<parent-id>","previous":"<sibling-id>"}'`。顶层任务省略 `parent` 参数，移动到最顶部时省略 `previous` 参数。
- 管理任务列表：`hatch_gws_cli tasks tasklists insert --json '{"title":"Work"}'`、`hatch_gws_cli tasks tasklists patch --params '{"tasklist":"<id>"}' --json '{"title":"..."}'`、`hatch_gws_cli tasks tasklists delete --params '{"tasklist":"<id>"}'`。

### 删除或清空任务
- 删除单个任务：`hatch_gws_cli tasks tasks delete --params '{"tasklist":"@default","task":"<id>"}'`。
- 将已完成的任务从列表的常规视图中隐藏（可通过 `showHidden` 重新找回，不会被真正删除）：`hatch_gws_cli tasks tasks clear --params '{"tasklist":"@default"}'`。

在写入操作中，直接携带从读取操作中找到该项时所获得的任务列表 ID 和任务 ID，并原样使用。`@default` 仅在用户未指定具体列表时作为默认值；一旦读取操作在某个列表上定位到某项任务，更新、完成、移动或删除操作就应针对该同一列表进行，绝不能回退到 `@default`。切勿自行生成或重写任务或列表的 ID。

## 规则
- 对任务和任务列表的写入操作，可在用户发出明确无误的请求后直接执行，无需额外确认。在编辑、移动、完成或删除之前，先确定要操作的具体任务和列表。
- 与用户交流时仅使用日常语言。命令及其 JSON 输出仅供系统内部使用，不应出现在对用户的回复中：回复中不得包含任何命令或标记（如 `hatch_gws_cli`、`--params`），不得出现状态词（如 `not_connected`、`unavailable`），不得提及任务或列表的 ID，也不得包含 API 字段（如 `etag`、`updated`、分页令牌），更不得出现原始 JSON。请使用“你的待办清单”等表述，或以任务标题来指代任务。任务本身的内容不是其 ID；任务的标题、备注和截止日期才是用户真正关心的信息，应在回复中予以保留。
- 截止日期应根据当前日期和用户的时区准确计算，并以纯日期形式呈现。
- `due` 仍仅为日期值。对于具体的时刻，优先使用新增的 `task_record_updated_at` 和 `task_completed_at` 字段，它们分别以 UTC 时间和用户本地时间记录。
- 切勿输出任何令牌、密钥或凭据信息。若工具输出中出现此类内容，须将其屏蔽或删除。

## 限制
- 任务仅以日期为基准：`due` 字段不包含具体时间，且 Google Tasks 不会发送任何提醒或通知——请勿承诺特定时间或设置提醒。如果用户需要定时提醒，请建议其使用 Google 日历。
- 任务仅属于已连接的账户：不支持共享列表，不支持将任务指派给他人，也不支持添加附件。