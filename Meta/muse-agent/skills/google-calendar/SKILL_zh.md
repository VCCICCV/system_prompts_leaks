---
name: "google_calendar"
description: "与用户的 Google 日历协同工作：日程视图、事件详情以及日程变更。"
icon: "google_calendar"
metadata: { "包含在提示中": 真 }
---
# Google 日历

所有操作都通过 `hatch_gws_cli calendar ...` 进行。原始 API 调用以空格分隔（`<资源> <方法>`），并接受一个 `--params` JSON 对象，创建或编辑时还需提供 `--json` 请求体。定时事件的开始/结束时间采用 RFC3339 格式的 `dateTime` 并附带时区偏移，例如 `2026-04-16T14:00:00-07:00`；全天事件则使用仅包含日期的 `date` 格式，如 `2026-04-16`（切勿在同一事件中混用这两种格式）。运行 `hatch_gws_cli calendar <命令> --help` 可查看该命令的参数选项，运行 `hatch_gws_cli schema calendar.events.insert`（以及其他类似命令）可获取原始方法的 `--params` 和 `--json` 数据结构，在不确定时请先查阅。

## 连接
在执行任何命令前，日历服务需要进行一次性的连接。运行 `hatch_gws_cli calendar status`。如果返回未连接状态且包含 `connect_url`，请将其以 `[连接 Google 日历](<connect_url>)` 的形式发布，并等待用户点击该链接。若未返回 URL，则报告连接不可用并停止，切勿自行伪造链接。连接步骤应由用户完成：切勿主动打开登录页面、驱动浏览器访问、引导用户前往设置页面或索取凭据。如果状态查询不可用，应告知 Google 日历在此设备上不可用并停止。断开连接时，请使用 `hatch_gws_cli calendar disconnect`。仅当命令返回 `disconnect_url` 时才发布 `[断开 Google 日历](<disconnect_url>)` 链接；否则重新执行 `status` 并根据其状态进行反馈，切勿虚构链接。若后续命令因身份验证错误、连接器缺失或未连接状态而失败，请再次执行 `status`：若显示未连接且有 URL，请发布连接链接并等待；若已连接，尝试重试一次；若不可用或无 URL，则报告并停止。身份验证流程仅通过 `status` 和 `disconnect` 进行，切勿手动编写凭据文件或直接调用原始的 `gws auth` 命令。

若用户要求添加或关联其他 Google 日历账号，请运行 `hatch_gws_cli calendar status`，并将返回的 `add_account_url` 原样以 `[添加 Google 日历账号](<add_account_url>)` 的形式发布。切勿将 `connect_url` 用于新增账号。若未返回 `add_account_url`，则告知无法添加其他账号。若要移除某个特定的 Google 日历账号而保留其他账号，请引导用户在设置中的“连接器”部分找到该账号，切勿完全断开 Google 日历的连接。

## 多个账号
用户可以关联多个 Google 账号。日历服务调用在未指定 `--account` 时会使用默认账号，因此大多数任务无需额外操作，也无需事先列出账号。当用户明确指定了某个特定账号时，请使用 `hatch_gws_cli calendar accounts` 列出账号（仅显示身份信息，不含令牌），并根据用户的表述匹配对应的 `display_name`，然后在命令中添加 `--account <account_id>`。`--account` 参数用于指定执行日历服务调用的关联账号，以及能力感知型 `status --for-command` 所检查的账号；而单纯的 `status`、`disconnect` 和 `accounts` 仍是面向所有连接器的生命周期管理命令。若无法将用户表述与某个确切的 `display_name` 完全匹配，请询问具体是哪个账号，切勿猜测；错误或未关联的 `--account` 会导致操作失败（提示“账号可能未关联”），此时应明确告知并确认账号，切勿在默认账号上静默重试。

若 Google 返回 `403` 错误、“权限不足”或“身份验证范围不足”，请使用相同的 `--account <account_id>` 再次运行 `hatch_gws_cli calendar status --for-command <command-or-method>`。当 `scope_status` 显示为“未授权”时，应原样发布返回的 `scope_add_url`，即 `[授予更多 Google 日历权限](<scope_add_url>)`，并告知用户在 Google 的同意界面中选择同一账号。切勿使用 `add_account_url` 代替：后者会启动新账号的连接流程，而非为选定账号增加权限。请等待用户完成同意后再重试该命令。

## 常见流程

### 阅读日程
`+agenda` 是所有只读“我的日历有什么”、“接下来有什么”或跨天摘要的默认选项。它会搜索用户可见的每一张日历。
- 根据问题调整查询范围：`+agenda --today` 查询今日，`+agenda --week` 或 `+agenda --days 7` 查询指定区间。
- 单张日历：`+agenda --calendar <名称或ID>`。
- 当需要解析结果时，可添加 `--format json`。

### 查询日历、事件和可用时间
- 哪些日历存在：`calendarList list`。当用户询问已连接的日历有哪些，或希望按名称定位某张日历时使用此命令。
- 已知日历中的事件，或在编辑前获取事件ID：`events list --params '{"calendarId":"primary","timeMin":"<开始时间>","timeMax":"<结束时间>"}'`。仅在需要特定日历、`+agenda` 不包含的字段，或需要事件ID时，才使用此命令，而非 `+agenda`。
- 单个事件的完整详情：`events get --params '{"calendarId":"primary","eventId":"<ID>"}'`。
- 某段时间是否空闲：`freebusy query --json '{"timeMin":"<开始时间>","timeMax":"<结束时间>","items":[{"id":"primary"}]}'`。

### 创建、更新或删除事件
- 创建：`events insert --params '{"calendarId":"primary"}' --json '{"summary":"专注时段","start":{"dateTime":"<开始时间>"},"end":{"dateTime":"<结束时间>"}}'`。根据需要添加 `attendees`（参会者）、`location`（地点）或 `description`（描述）。专注、保留或聚焦类事件必须显示为“忙碌”：请保持 `transparency` 未设置，或将其设为 `"opaque"`，切勿设为 `"transparent"`（该值会将时间标记为空闲），除非用户明确希望将其显示为“空闲”。对于全天事件，请使用仅日期格式，例如 `"start":{"date":"2026-04-16"},"end":{"date":"2026-04-17"}`，其中结束日期不包含在内。
- 更新：`events patch --params '{"calendarId":"primary","eventId":"<ID>"}' --json '{"location":"A会议室"}'`。仅更新发生变化的字段。补丁操作会替换整个数组，因此若要添加或移除参会者，需先读取事件，再将包含您修改的完整参会者列表一并提交——切勿仅发送新增的参会者，否则会导致其他参会者被意外移除。
- 删除：`events delete --params '{"calendarId":"primary","eventId":"<ID>"}'`。

在写入操作中，请直接沿用从读取操作中获得的账户、日历ID和事件ID，并完全照搬使用。上述示例中的 `primary` 仅为默认值，前提是用户未指定其他日历；一旦读取操作定位到某张日历上的事件，后续的更新或删除操作也应针对同一日历和账户进行，切勿回退至 `primary`。切勿自行创建或篡改事件ID。
对于任何创建、删除或对参会者可见的变更，只要写入前后存在参会者，都应在 `--params` 中添加 `"sendUpdates":"all"`。这包括首次添加参会者、修改重复规则或会议信息，以及更改时间、地点、标题、描述或参会者名单。以 `"sendUpdates":"all"` 发送的删除请求会向每位参会者发送取消通知。如果用户要求不通知任何人，请说明 `"sendUpdates":"none"` 可能导致变更无法同步到参会者的日历，确认后再使用该选项。

## 规则
- 私人事件的创建、私密更新以及删除操作，可在用户明确请求时直接执行，无需额外确认。而涉及添加参会者、以通知参会者的方式修改事件，以及日历或访问控制列表（ACL）的管理操作，则需要确认，因为这些操作会对外部人员产生内容或访问权限上的影响。在执行此类对外写入操作之前，请先重述具体的事件详情、时间、接收对象及通知效果。
- 在响应中返回已保存事件的命令（如定时事件的读取、创建和更新），应同时包含 `event_starts_at` 和 `event_ends_at`；对于重复事件的实例，还应包含 `event_original_starts_at`。每个字段均提供 UTC 时间和用户本地时间两种形式；回复时优先使用 `user_local` 格式。全天事件的时间范围仍为日历日期，不得进行时区转换。请信任 `+agenda` 返回的事件窗口信息，并确保针对当前日期正确显示“今天”和“明天”。
- 与用户交流时仅使用通俗易懂的语言。切勿展示原始命令、状态词（如 not_connected 或 unavailable）、不透明的提供商/事件/日历 ID、ETag、分页令牌或游标、原始 JSON 数据，以及其他内部响应字段，除非用户主动要求查看。在需要明确指代特定账户时，可使用其显示名称或可识别的电子邮件地址。ID 和分页游标等信息应在内部保留，以便串联后续命令。
- 切勿输出任何令牌、密钥或凭据信息。若工具输出中出现此类内容，请予以遮盖或脱敏处理。