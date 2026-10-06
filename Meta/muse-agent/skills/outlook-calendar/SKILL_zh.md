---
name: "outlook_calendar"
description: "查看、创建、更新和删除用户 Outlook 日历中的事件。"
icon: "outlook_calendar"
metadata: { "不包含在提示中": 假 }
---
# Outlook 日历

## 用途
使用 `outlook-calendar` 伴侣 CLI 管理 Outlook 日历事件。该连接器同时支持个人 Microsoft 帐户以及 Microsoft 365 工作或学校帐户。

## 工具
使用 `exec` 直接从 `PATH` 运行已安装的 CLI。

```sh
outlook-calendar --status
outlook-calendar disconnect
outlook-calendar list --page-size 10 [--time-min <RFC3339>] [--time-max <RFC3339>] [--query <text>] [--page-token <token>]
outlook-calendar get "<event_id>"
outlook-calendar create --summary "<title>" --start "<RFC3339-or-date>" --end "<RFC3339-or-date>" --timezone "<iana_tz>" [--attendee <email>] [--location <text>] [--description <text>] [--all-day]
outlook-calendar update "<event_id>" [--summary <title>] [--start <RFC3339>] [--end <RFC3339>] [--timezone <iana_tz>] [--location <text>] [--description <text>]
outlook-calendar delete "<event_id>"
```

对于 `list` 命令，使用 `--page-size` 指定返回结果的数量。`--top`、`--limit` 和 `--max-results` 仅为兼容性别名，新命令中请勿使用。`list` 使用 `--time-min` / `--time-max` 来设置时间范围；`--start` 和 `--end` 仅在 `create` / `update` 命令中为规范参数，而在 `list` 命令中仅作为兼容性别名接受。

JSON 输出格式：
- `disconnect`: 解析 `ok`、`action`、`status` 和 `disconnect_url`。
- `list`: 解析 `ok`、`count`、`next_page_token`（存在时作为 `--page-token` 返回）、`retrieved_at`，以及包含 `id`、`summary`、`description`、`start`、`end`、`event_starts_at`、`event_ends_at`、`location`、`is_all_day` 和 `attendees` 等字段的 `events[]`。
- `get`: 解析 `ok`、`retrieved_at` 和 `event`。
- `create` / `update`: 解析 `ok`、`action`、`event_id` 和 `web_link`。
- `delete`: 解析 `ok`、`action` 和 `event_id`。

## 认证
本技能使用由 CLI 管理的 Outlook 连接器，请勿手动编写认证文件。

首次使用或重新连接流程：
1. 运行 `outlook-calendar --status`。
2. 如果响应中包含 `connect_url`，请将 `<connect_url>` 替换为返回的 URL，并按原样分享此 Markdown 链接：`[连接 Outlook 日历](<connect_url>)`；请勿单独粘贴原始 URL。等待用户完成绑定。
3. 继续前请再次运行相同的状态查询命令。
4. 如果响应中包含 `disconnect_url`，则表示账户已连接。

对于断开连接请求，运行 `outlook-calendar disconnect`。当出现 `disconnect_url` 时，将 `<disconnect_url>` 替换为返回的 URL，并按原样分享此 Markdown 链接：`[断开 Outlook 日历连接](<disconnect_url>)`；请勿单独粘贴原始 URL。如果未出现，则说明账户已断开连接。

## 操作规则
1. 当您需要事件 ID 或用户询问日期范围时，请先使用 `list` 命令。
2. 所有带时间的值必须采用 RFC3339/ISO 8601 格式。仅在使用 `--all-day` 时才使用纯日期值。
3. 在创建事件时，以及在更新事件且开始或结束时间发生变化时，务必提供有效的 IANA 时区。
4. 在创建会发送邀请的事件之前，请先与用户确认。
5. 私密事件的更新可在用户明确请求的情况下进行。助手会在更新前读取当前事件；对于由组织者发起且包含参会者的会议，由于 Outlook 会发送会议更新邮件，因此需要确认。
6. 当前助手不支持在 `update` 命令中修改参会者。如果用户要求在现有事件中添加或移除受邀者，请说明这一限制，而不要输出不支持的 `--attendee` 参数。
7. 在用户提出明确且无歧义的请求时，可以直接执行 `delete` 操作，无需额外确认。
8. 事件 ID 是 Graph 返回的不透明值。请在后续操作中复用 `list` 或 `get` 命令返回的完整 ID。
9. 切勿使用共享连接器助手 CLI 来操作 Outlook 日历；状态查询和关联操作必须通过 `outlook-calendar --status` 进行。
10. 切勿在向用户展示的文本中暴露原始 Graph 标识符（如事件 ID、日历 ID）或其他内部响应字段（如更改键、分页/跳过标记、原始 JSON），包括在摘要、列表或单个条目注释中。这些标识符仅限于内部使用，用于串联后续命令（见第 8 条）。唯一例外是：当用户明确要求获取原始 ID，或为排查故障必须显示该 ID 时。
11. 对于有时段的事件，优先使用 `event_starts_at.user_local` 和 `event_ends_at.user_local`；原始 Graph 字段保留以保持兼容性。全天事件的值为日历日期而非具体时间点，且不得进行时区转换。
- 权限受限字段：如果 `list` 结果中包含带有 `withheld` 标记的条目，则表示该会议的组织者和参会者已被用户的“访问事件”权限屏蔽——该会议并非无参会者。请根据剩余字段回答，并说明权限已隐藏参会者名单。只有当用户确实需要查看某一特定事件的受限字段，且 `withheld.reason` 为 `requires_approval` 时，才使用 `get` 命令读取该事件，此时系统会向用户显示审批提示。切勿仅为填充参会者名单而在日常浏览时使用 `get` 命令。
