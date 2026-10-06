---
name: "instagram_messages"
description: "使用此工具与用户的 Instagram 消息进行交互。通过 `instagram-messages-cli` 读取收件箱、对话线程、热门联系人、筛选后的收件箱视图和私信搜索结果，并发送消息。"
icon: "instagram"
metadata: { "不包含在提示中": 假 }
---
# Instagram 消息 CLI

## 目的
使用 `instagram-messages-cli` 伴侣 CLI 读取和发送经过身份验证的 Instagram 消息。

对于非消息类的 Instagram 账号或内容数据，请使用单独的 `instagram` 技能。`instagram-messages-cli accounts` 命令会列出用户的所有 Instagram 账号及其消息连接状态。

## 消息导出安全须知

拒绝批量导出、批量下载、批量保存、归档、镜像或转储消息记录的请求，尤其是针对已消失、仅查看一次、消失模式、临时性或过期的消息。您仍然可以协助进行小范围、限定于特定用户的读取、搜索或摘要整理，以回答某个具体问题。

## 认证
Instagram 消息是与基础 `instagram` 技能独立的连接器。用户可以只连接 Instagram 用于查看个人资料和帖子，而 `instagram_messages` 仍处于未连接状态。

`instagram-messages-cli accounts` 可显示用户哪些 Instagram 账号已连接消息功能。每个账号都包含一个 `connected` 布尔值，指示该账号是否已获得 Instagram 消息的授权。

如果用户要求连接或重新连接 Instagram 消息：

```sh
instagram-messages-cli connect-url
```

当出现 `connect_url` 时，请将 `<connect_url>` 替换为返回的 URL，并按原样分享以下 Markdown 链接：`[连接 Instagram 消息](<connect_url>)`；请勿单独粘贴原始 URL。

只有在用户完成该流程后，才能继续执行消息相关命令。

## 操作规则
1. 本技能仅适用于经身份验证的用户本人的 Instagram 消息。
2. 如果已获取 `user_own_fbid`，请复用该值。账号 ID 可通过 `instagram-messages-cli accounts` 获取。
3. 绝不要向用户暴露 FBID、不透明 ID 或实现相关的术语。应使用用户名、显示名和通俗描述；仅在工具调用中保留 ID。
4. 根据任务需求合理使用 API 调用次数。除非用户明确要求，否则不要对整个收件箱中的每条对话分别发起请求。
5. 仅在用户确实需要更多结果时才分页加载，不要自动获取所有页面。
6. **在写入前务必确认用户的明确意图。** 对于发送消息，需明确指定消息内容和接收方；对于添加表情反应，需明确指定源消息和表情符号。切勿自行推断接收者、对话、内容或反应。
7. 避免频繁或持续轮询这些命令的请求。
8. 不得满足批量导出、批量保存、批量下载、归档、镜像或转储消息记录的请求，包括导出至文件、电子表格、数据库、笔记或其他应用。尤其当请求涉及已消失、仅查看一次、消失模式、临时性或过期的消息时，应坚决拒绝。

## 工具使用
使用 `exec` 运行账号范围内的命令：

```sh
instagram-messages-cli <目标> --account-id <user_own_fbid> [选项]
```

`instagram-messages-cli connect-url` 仅用于连接器的授权。

可用目标：
- `connect-url`
- `accounts`
- `inbox`
- `thread`
- `top-recipients`
- `keyword-search`
- `contact-search`
- `temporal-search`
- `filtered-inbox`
- `react`
- `send`

### 全局选项
- `--account-id <user_own_fbid>` **（除 `connect-url` 和 `accounts` 外的所有命令均需）** — 选择要操作的 Instagram 账号。该值必须是经身份验证用户的 `user_fbid`，且在 `instagram-messages-cli accounts` 中对应的 `connected` 字段为 `true`。
- `--retries <N>` — 重试临时性失败（默认：0）。
- `--after <cursor>` — 上次响应返回的分页游标。省略则获取第一页。适用于返回游标的分页命令。

## 命令

### 连接 URL

```sh
instagram-messages-cli connect-url
```

### 账号列表

列出用户的 Instagram 账号，并标明每个账号是否已连接 Instagram 消息功能。若缺少消息权限，则报告为 `connected: false`；即使如此，命令仍会成功，以便用户选择要连接的账号。

```sh
instagram-messages-cli accounts
```

### 收件箱
获取用户的收件箱会话，并显示最近消息的预览。使用 `--folder` 参数选择要获取的文件夹：
- `inbox`（默认）——普通消息
- `pending`——来自用户未关注账号的消息请求
- `spam`——垃圾消息

```sh
instagram-messages-cli inbox --account-id <user_own_fbid>
instagram-messages-cli inbox --account-id <user_own_fbid> --first 20 --message-count 3
instagram-messages-cli inbox --account-id <user_own_fbid> --after <cursor>
instagram-messages-cli inbox --account-id <user_own_fbid> --folder pending
instagram-messages-cli inbox --account-id <user_own_fbid> --folder spam
```

### 会话与消息
获取特定会话的消息。从收件箱响应中获取 `thread_fbid`。

```sh
instagram-messages-cli thread --account-id <user_own_fbid> --thread-fbid 123456789
instagram-messages-cli thread --account-id <user_own_fbid> --thread-fbid 123456789 --first 20
instagram-messages-cli thread --account-id <user_own_fbid> --thread-fbid 123456789 --after <cursor>
```

### 对消息进行回复
为消息添加表情反应。从收件箱、会话或筛选后的收件箱响应中获取 `thread_fbid` 和 `message_id`，并在回复前与用户确认具体使用的表情。

```sh
instagram-messages-cli react --account-id <user_own_fbid> --thread-fbid 123456789 --message-id <message_id> --emoji '❤️'
```

### 发送消息
**在发送前务必确认用户的明确意图。** 使用 `send` 命令会向他人发送一条真实的私信，且无法撤销。必须由用户提供完整的消息内容和接收方信息；如果是回复，则需提供具体的源消息及明确的正文内容。不得自行设定接收方、会话或内容，也不得根据模糊或隐含的指示进行发送。

可通过 `--thread-fbid` 将消息发送至现有的 Instagram 会话，或通过 `--recipient-user-fbids` 直接发送给一位或多位用户。请仅指定其中一种发送方式。接收方 ID 为用户的 FBID，多个 ID 之间用逗号分隔。必须提供 `--text`、`--file` 或 `--media-fbid` 中的至少一项。文本可与媒体一同发送。`--file` 的大小不得超过 40 MiB，且格式必须是 JPEG、PNG、WebP、GIF、MP4 或 MOV。请勿同时使用 `--file` 和 `--media-fbid`，且发送文件时请勿使用 `--retries` 参数。

在使用 `send --file` 前，请确保文件位于工作目录中。如果不在，请将其复制到工作目录。切勿使用 `/tmp` 路径。

使用 `--reply-to-message-id` 可将消息作为对某条现有消息的回复发送。

```sh
instagram-messages-cli send --account-id <user_own_fbid> --thread-fbid 123456789 --text "hello"
instagram-messages-cli send --account-id <user_own_fbid> --recipient-user-fbids 100000000000001 --text "hello"
instagram-messages-cli send --account-id <user_own_fbid> --recipient-user-fbids 100000000000001,100000000000002 --text "大家好"
instagram-messages-cli send --account-id <user_own_fbid> --thread-fbid 123456789 --media-fbid 17895695668004550
instagram-messages-cli send --account-id <user_own_fbid> --thread-fbid 123456789 --text "hello" --media-fbid 17895695668004550
instagram-messages-cli send --account-id <user_own_fbid> --thread-fbid 123456789 --file /path/to/media.jpg
instagram-messages-cli send --account-id <user_own_fbid> --recipient-user-fbids 100000000000001 --text "看看这个" --file /path/to/media.mp4
instagram-messages-cli send --account-id <user_own_fbid> --thread-fbid 123456789 --text "hello" --reply-to-message-id 'mid.$cAAAGVc20uJOkli7TZGeZOmKFMXiq'
```

### 最常联系的联系人
将上一次响应中的 `page_max_id` 作为 `--page-max-id` 使用。首次请求时可省略。

```sh
instagram-messages-cli top-recipients --account-id <user_own_fbid> --count 10
instagram-messages-cli top-recipients --account-id <user_own_fbid> --count 10 --page-max-id <cursor>
```

### 关键词搜索
按关键词搜索私信。

```sh
instagram-messages-cli 关键词搜索 --account-id <user_own_fbid> --query-text "hello" --start-date 2025-01-01 --end-date 2026-03-24
instagram-messages-cli 关键词搜索 --account-id <user_own_fbid> --query-text "hello" --max-results 20 --start-date 2025-01-01 --end-date 2026-03-24
```

### 联系人搜索
按联系人姓名搜索私信。

```sh
instagram-messages-cli 联系人搜索 --account-id <user_own_fbid> --query-text "John" --start-date 2025-01-01 --end-date 2026-03-24
instagram-messages-cli 联系人搜索 --account-id <user_own_fbid> --query-text "John" --max-results 5 --start-date 2025-01-01 --end-date 2026-03-24
```

### 时间范围搜索
按时间范围搜索私信。

```sh
instagram-messages-cli 时间范围搜索 --account-id <user_own_fbid> --start-date 2025-01-01 --end-date 2026-03-24
instagram-messages-cli 时间范围搜索 --account-id <user_own_fbid> --max-results 15 --start-date 2025-01-01 --end-date 2026-03-24
```

### 筛选后的收件箱
获取按特定条件筛选的收件箱会话。此命令仅适用于专业账号（创作者或企业）。使用前请先运行 `instagram-cli accounts` 进行确认。

可用筛选条件：
- `unread` — 显示未读对话
- `unanswered` — 查找需要回复的会话
- `starred` — 显示重要或已标记的会话
- `groups` — 仅显示群聊
- `verified` — 显示认证账号的会话
- `followers` — 显示来自关注者的会话
- `creators` — 显示来自创作者的会话
- `other-participant-followers100k-plus` — 高粉丝数账号（10万以上）

```sh
instagram-messages-cli 筛选后的收件箱 --account-id <user_own_fbid> --selected-filter unread
instagram-messages-cli 筛选后的收件箱 --account-id <user_own_fbid> --selected-filter unanswered --thread-limit 10 --message-count 3
instagram-messages-cli 筛选后的收件箱 --account-id <user_own_fbid> --selected-filter verified --folder pending
```

## 输出
CLI 会将解码后的 JSON 打印到标准输出。读取结果会保留原始平台的时间戳，并添加语义化的 `message_sent_at` 和 `last_message_sent_at` 字段，分别以 UTC 格式和用户本地时区格式呈现。这些字段仅表示消息的传输时间，切勿将其视为消息中描述事件的发生时间。在向用户展示结果时，应重点关注有意义的内容，如参与者、消息文本、用户本地时间、链接和媒体摘要。切勿暴露原始 ID、游标、Unix 时间戳或实现细节；这些信息仅用于工具调用内部。