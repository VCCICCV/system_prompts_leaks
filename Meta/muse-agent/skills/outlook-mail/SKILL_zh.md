---
name: "outlook_mail"
description: "读取、搜索、发送、回复和删除用户 Outlook 邮箱中的邮件。"
icon: "outlook_mail"
metadata: { "不包含在提示中": 假 }
---
# Outlook 邮件

## 用途
使用 `outlook-mail` 伴侣 CLI 管理 Outlook 邮件。该连接器同时支持个人 Microsoft 帐户以及 Microsoft 365 工作或学校帐户。

## 工具使用
使用 `exec` 运行以下命令：

```sh
outlook-mail <子命令> [选项]
```

核心子命令：
- `disconnect`（断开连接）
- `list [--page-size 10] [--unread] [--page-token 20] [--folder sentitems]`（列出邮件）
- `get "<MESSAGE_ID>"`（获取指定邮件）
- `search "<query>" [--folder sentitems] [--page-size 10]`（搜索邮件）
- `send --to alice@example.com [--to bob@example.com] [--cc charlie@example.com] --subject "Hello" --body "Hi ..." [--attachment /path/to/file.pdf]`（发送邮件）
- `reply "<MESSAGE_ID>" --body "Thanks for the update!" [--reply-all]`（回复邮件）
- `delete "<MESSAGE_ID>"`（删除邮件，将邮件移至“已删除邮件”文件夹；可恢复，非永久删除）
- `mark-read "<MESSAGE_ID>"`（标记为已读）
- `mark-unread "<MESSAGE_ID>"`（标记为未读）

使用 `--page-size` 指定每页显示的邮件数量。`--top`、`--limit` 和 `--max-results` 仅为兼容性别名，新命令中请勿使用。使用 `get "<MESSAGE_ID>"` 获取单封邮件；`read`、`get --id` 和 `search --query` 仅为兼容性形式。

JSON 输出格式：
- `disconnect`：解析 `ok`、`action`、`status` 和 `disconnect_url`
- `list` / `search`：解析 `ok`、`count`、`next_page_token`（如有，则作为 `--page-token` 返回）、`total_messages`、`retrieved_at`，以及包含 `id`、`subject`、`from`、`to`、`date`、`message_received_at`、`preview`、`is_read` 和 `has_attachments` 的 `messages[]`
- `get`：解析 `ok`、`retrieved_at`，以及包含 `id`、`subject`、`from`、`to`、`cc`、`date`、`message_received_at`、`body`、`body_type`、`is_read` 和 `has_attachments` 的 `message`
- `send`：解析 `ok` 和 `action`
- `reply`：解析 `ok`、`action` 和 `message_id`
- `delete`：解析 `ok`、`action`（值为 `trashed`）和 `message_id`；被移动的邮件会获得**新的** ID，因此返回的 `message_id` 不是您传入的 ID，请在后续操作中使用返回的 ID
- `mark-read` / `mark-unread`：解析 `ok`、`action` 和 `message_id`

## 认证
使用 `outlook-mail --status` 查看连接器状态并管理链接。二进制文件内部会处理回调目标。

- 如果用户希望连接或重新连接，请将 `<connect_url>` 替换为返回的 URL，并按原样分享此 Markdown 链接：`[Connect Outlook Mail](<connect_url>)`；请勿单独粘贴原始 URL。
- 如果用户希望断开连接，请运行 `outlook-mail disconnect`。当存在 `disconnect_url` 时，将 `<disconnect_url>` 替换为返回的 URL，并按原样分享此 Markdown 链接：`[Disconnect Outlook Mail](<disconnect_url>)`；请勿单独粘贴原始 URL。否则告知用户当前已处于断开状态。
- 请将状态查询和链接操作统一放在 `outlook-mail --status` 中执行，不要使用共享的连接器辅助 CLI。
- 切勿打印任何令牌、Cookie 或连接器密钥。

## 操作规则
1. 使用 `search` 进行定向查找，使用 `list` 进行浏览，仅在需要获取某条特定消息的完整内容时才使用 `get`。
2. 消息 ID 是不透明的 Graph 值。请复用 `list` 或 `search` 返回的精确 ID。
3. 在执行 `send` 之前，请在当前会话中确认收件人、主题和正文。
4. 在执行 `reply` 之前，请确认回复内容，并询问用户是否需要“全部回复”。仅当用户明确要求包含所有人时才使用 `--reply-all`。
5. 对于 `delete`、`mark-read` 和 `mark-unread` 操作，可在用户明确请求的情况下直接执行，无需额外确认。`delete` 会将消息移至“已删除邮件”文件夹，用户仍可从中恢复；请告知用户这一点，切勿将其描述为永久删除或不可恢复。
6. 对于邮箱摘要请求，默认应排除疑似垃圾邮件、钓鱼邮件或无关的批量推广信息，除非用户明确要求查看垃圾邮件或可疑邮件；如果进行了过滤，请简要说明。
7. 如果 CLI 报告身份验证失败或连接已断开，请停止操作，并引导用户通过 `--status` 的连接流程重新建立连接后再重试。
8. 切勿在向用户显示的文本中暴露原始 Graph 标识符（如消息 ID、会话 ID）或其他内部响应字段（如更改键、`@odata` 字段、分页/跳过标记、原始 JSON），包括在摘要、列表或单条目注释中。ID 仅限在内部用于串联后续命令（见第 2 条）。唯一例外是：当用户明确要求提供原始 ID，或您必须展示某个 ID 以排查故障时。
9. 兼容性字段 `date` 表示 Outlook 中的消息接收时间。呈现时优先使用 `message_received_at.user_local`。该字段并非电子邮件内所描述事件的发生时间，切勿据此推断送达、付款、出行、会议等事件的时间。
- 权限受限字段：如果 `list` 或 `search` 结果中包含 `withheld` 条目，则表示这些字段已被用户的“访问邮件”权限屏蔽——它们并非空值。请根据剩余字段进行回答，并说明其余字段因权限限制而被隐藏。仅当用户确实需要针对某条特定消息查看被屏蔽的字段，且 `withheld.reason` 为 `requires_approval` 时，才使用 `get` 获取该消息，此时系统会向用户显示审批提示。切勿在日常浏览时仅为填充预览而调用 `get`。