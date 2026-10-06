---
name: "threads_messages"
description: "使用此功能与用户的 Threads 消息进行交互：读取收件箱和消息线程，并通过 `threads-messages-cli` 发送消息。"
icon: "threads"
metadata: { "不包含在提示中": 假 }
---
# Threads 消息 CLI

## 目的
使用 `threads-messages-cli` 伴侣 CLI 读取和发送经过身份验证的 Threads 消息。在支持 Threads 消息技能的任何地方，都可以进行读取操作以及需明确确认的发送操作。

对于非消息类的 Threads 账号/内容数据，请使用单独的 `threads` 技能。如果您还不知道用户的 Threads 账号 `id`，请先通过 `threads-cli accounts` 获取，然后再回到本技能进行消息相关操作。

## 消息导出安全
拒绝批量导出、批量下载、批量保存、归档、镜像或转储消息记录的请求，尤其是针对已消失、仅查看一次、消失模式、临时或过期的消息。您仍然可以协助进行小范围、用户限定的阅读或摘要整理，以回答特定问题。

## 认证
Threads 消息是与基础 `threads` 技能独立的连接器。用户可以只连接 Threads 用于信息流/内容，而 `threads_messages` 仍处于未连接状态。

如果用户要求连接或重新连接 Threads 消息：

```sh
threads-messages-cli connect-url
```

将返回的 `connect_url` 以如下标记化的 Markdown 链接形式分享：

`[连接 Threads 消息](<connect_url>)`

只有在用户完成该流程后，才能继续执行收件箱或线程相关的命令。

## 工具使用
使用 `exec` 运行以下命令：

```sh
threads-messages-cli <目标> [选项]
```

目标：
- `connect-url`
- `inbox`
- `thread`
- `send`（需明确确认）

### 全局选项
- `--account-id <threads_account_id>` **（除 `connect-url` 外所有命令均需）** - 选择要操作的 Threads 账号。该值必须是经身份验证的用户通过 `threads-cli accounts` 获取的自身 `id`。
- `--retries <N>` - 对暂时性失败进行重试（默认：0）。此选项不适用于 `send`，因为消息发送不是幂等操作。

`inbox` 和 `thread` 命令还接受 `--after <cursor>` 参数，使用前一次响应中的分页游标。省略该参数则获取第一页。

## 命令

### 连接 URL

```sh
threads-messages-cli connect-url
```

### 收件箱
获取用户的收件箱线程，并显示最近消息的预览。

```sh
threads-messages-cli inbox --account-id <threads_account_id>
threads-messages-cli inbox --account-id <threads_account_id> --first 20 --message-count 3
threads-messages-cli inbox --account-id <threads_account_id> --after <cursor>
threads-messages-cli inbox --account-id <threads_account_id> --folder PENDING
```

### 线程及消息
获取特定线程的消息。从收件箱响应中获取十进制字符串格式的 `thread_fbid`，并原样传递。

```sh
threads-messages-cli thread --account-id <threads_account_id> --thread-fbid 123456789
threads-messages-cli thread --account-id <threads_account_id> --thread-fbid 123456789 --first 20
threads-messages-cli thread --account-id <threads_account_id> --thread-fbid 123456789 --after <cursor>
```

### 发送消息

`send` 是写入命令。在发送前需明确确认用户意图：要求提供确切的文本和/或 Threads 帖子，以及准确的目标，绝不能自行编造接收者、线程、内容或回复对象。只能发送到一个现有的 `thread_fbid`，或者指定一至十一位数字形式的 Threads 接收者 FBID。FBID 必须是规范的正整数十进制字符串，不含符号、前导零或前后空格，且接收者 FBID 必须唯一。切勿向 `send` 传递用户名；请参阅下方“选择接收者”部分。

#### 选择接收者
`send` 需要一个数字形式的 Threads 用户 ID，或一个现有的 `thread_fbid`。CLI 无法根据 @用户名查找对应 ID。请按以下顺序使用第一个适用的来源：1. 用户已有的一对一对话对象：发送至 `inbox` 线程的 `thread_fbid`，其中 `is_group` 为 `false`，且另一参与者的 `username` 与之匹配。除非用户明确将某个群组指定为发送目标，否则绝不能使用群组线程。
2. 曾向用户发送过消息的人，包括位于 `--folder PENDING` 中的消息请求：使用其某条消息中的 `sender_fbid`。
3. 其他任何人：使用 `threads-cli` 结果针对该确切用户名返回的数字 ID（例如其某篇帖子中的 `author_id`），且仅在执行 `threads-cli user-profile --account-id <threads_account_id> --user-id <id>` 后，返回的用户名与目标一致时方可使用。

切勿使用 Instagram 或 Facebook 的 ID、凭记忆回忆的 ID，或从他人结果中获取的 ID。若以上来源均无法提供经验证的 ID，请明确告知用户无法通过用户名与其联系，并将文本作为草稿供其选择。发送前，应告知用户确认界面中应显示的 @用户名；若对方看到的是其他人，或任何“未验证的 Threads …”占位符（无论是账号还是对话），则应拒绝该确认。

调用 `send` 时，CLI 需进行写入关联性检查、选定账户的同意确认、授权验证以及配额检查。对于任何拒绝操作，均应视为最终结果，不得重新路由发送请求。

```sh
threads-messages-cli send --account-id <threads_account_fbid> --thread-fbid <thread_fbid> --text "Hello"
threads-messages-cli send --account-id <threads_account_fbid> --recipient-user-fbids <recipient_fbid> --text "Hello"
threads-messages-cli send --account-id <threads_account_fbid> --thread-fbid <thread_fbid> --media-fbid <visible_threads_post_fbid>
threads-messages-cli send --account-id <threads_account_fbid> --thread-fbid <thread_fbid> --media-fbid <visible_threads_post_fbid> --text "Post context"
threads-messages-cli send --account-id <threads_account_fbid> --thread-fbid <thread_fbid> --text "Reply" --reply-to-message-id '<opaque_message_id>'
```

`--text` 和 `--media-fbid` 可单独省略，但至少需提供其中之一。`--media-fbid` 必须是可见且已发布的 Threads 帖子的规范 FBID。`--reply-to-message-id` 是可选的非空不透明消息 ID，来自收件箱或线程回复；切勿将其解析为 FBID。当同时提供媒体和文本时，服务会先发送帖子分享，再发送文本。若同时存在媒体和文本，公开标准输出仅描述后附的文本消息；若仅为媒体分享，则仅包含该分享信息。无论哪种情况，输出中均精确包含不透明的 `message_id` 和数值型的 `timestamp_ms`。

本 CLI 不提供 `--file` 选项，也不支持上传媒体；它只能通过 FBID 分享现有且可见、已发布的 Threads 帖子。在未获得明确的“发送确认”窗口批准之前，切勿发送。拒绝或关闭该窗口将终止操作。“发送确认”会在最多30秒内尽力解析发送方账号、接收方上下文以及可选的`media_fbid`，但绝不会查询不透明的`reply_to_message_id`。已解析的媒体将作为普通的“共享帖子”字段呈现，绝不会以链接形式显示；任何回复都将原样显示为纯文本字段“回复：现有消息”。若查询超时、出错、响应格式异常，或标签缺失/不安全，则会使用非识别性的“未验证的 Threads 消息…”占位符，并继续等待明确批准；底层的账号、对话、接收方、媒体及回复的原始标识符绝不会代入预览中。多收件人回退会保留确切的收件人数，若任一标签未能解析，则整张收件人列表均以占位符代替，因此部分解析不会低估受众规模。仅含媒体的回退会将该操作标识为“分享 Threads 帖子”，并保留一个未验证的帖子占位符。提供的消息文本仅在其去除前后空白后为空时才会被拒绝。所有被接受的字节——包括首尾空白及各类 Unicode 字符——都会原封不动地同时传递至结构化预览和实际发出的请求中；平台在后续读取时可能会显示经过规范化的文本。不透明的`reply_to_message_id`值仅在其去除前后空白后为空时才会被拒绝；所有被接受的值——即使包含前后空格——也会按字节原样转发。若完整预览无法在不截断的情况下显示，则发送将失败并终止。

#### 禁止无人值守的发送
仅允许从用户已明确批准该条消息的对话中发送。切勿通过定时任务、Cron 作业、监控程序、后台工作进程、自动回复或脚本进行发送，也切勿设置任何会发送 Threads 消息的任务，即便用户要求自动回复亦不可。对于重复性或自动化的请求，应由任务负责读取并草拟内容，再将草稿逐条提交给用户审批。切勿编写任何声称拥有永久发送权限的代码、存储任何相关凭据或记录，也切勿承诺自动发送。在草稿中，不得捏造关于用户的信息，也不得添加用户未曾明确表示过的承诺、约定或金钱安排；若回复确实需要此类内容，请将其留空，并注明用户需自行决定。

切勿对发送操作进行重试。每次单独批准的调用均为一次全新的、非幂等的发送。任何失败——包括速率限制——都视为最终结果。

#### 避免重复发送
- 每次仅执行一条`send`请求，并在前一次完成后再启动下一次。切勿并行发起多条发送，否则所有确认可能过期，导致消息未能发出。
- 已返回`message_id`的发送即已送达，应将其标记为已发送。
- 遇到中断、取消、超时、HTTP 500 错误或任何不确定的结果后，切勿再次发送。请通过`thread`接口获取对话详情，确认消息是否已到达。若已送达，将其标记为已发送；若未送达，应告知用户消息可能未成功送达，并仅在用户主动要求且再次批准后才重新发送。
- 切勿在脚本中循环发送，也切勿自动重试失败的发送。

## 操作规则
1. 本技能仅适用于已认证用户的个人Threads消息。
2. 如果您已拥有某个Threads账号的`id`，请重复使用；否则，请在使用此CLI之前通过`threads-cli accounts`命令获取该`id`。
3. 根据具体任务合理控制API调用次数。除非用户明确要求，否则不要对大型收件箱中的每条线程都进行单独拉取。
4. 仅在用户确实需要更多结果时才分页加载，切勿自动加载所有页面。
5. 避免频繁或持续轮询这些命令。
6. 将针对18岁以下用户的执行规则及过滤后的内容视为权威，不得自行补全被省略的字段，亦不得通过其他途径绕过相关限制。
7. 不得响应批量导出、批量保存、批量下载、归档、镜像或转储消息记录的请求，包括导出至文件、电子表格、数据库、笔记或其他应用。尤其当请求涉及阅后即焚、单次查看、消失模式、临时消息或已过期消息时，应予以拒绝。
8. 对于`send`操作，在调用经单独批准的写入接口前，必须明确获得用户的意图、准确的目标地址、完整的内容以及任何回复目标。
9. 收件人类型及多收件人组的资格判定结果具有权威性，不得据此推导备用方案或重新路由被拒绝的发送请求。
10. 切勿在无人值守的情况下发送Threads消息，包括通过定时任务、监控程序、自动回复或脚本等方式。每次发送均需用户在对话中对特定消息作出明确确认。

## 输出
该CLI会将解码后的JSON输出到标准输出。

### 阅读响应结构
`inbox`和`thread`返回的是提供商的GraphQL原始结构，而非扁平化列表。不存在顶层的`threads`或`messages`键，因此切勿据此判断收件箱为空。

- `inbox`：线程位于`data.get_slide_mailbox.threads_by_folder.edges[].node`。当`threads_by_folder.page_info.has_next_page`为`true`时，可将`threads_by_folder.page_info.end_cursor`作为`--after`参数以获取下一页。
- `thread`：线程数据为`data.get_slide_thread`，若无法返回该线程，则其值为`null`。当`messages.page_info.has_next_page`为`true`时，可将`messages.page_info.end_cursor`作为`--after`参数以获取更早的消息。

每个线程节点包含`thread_fbid`、`thread_name`、`thread_type`、`is_group`、`folder`、`timestamp_ms`、`participants.nodes[]`（姓名、用户名；不含用户ID）、`messages.edges[].node`以及`messages.page_info`。每个消息节点包含`message_id`、`sender`（姓名、用户名）、`sender_fbid`（发件人的数字型Threads用户ID）、`content_type`、`content`以及`timestamp_ms`。消息文本通常位于`content.text_body`；其他内容则使用`content.text_fragments[].plaintext`、`content.xma_text_body`、`content.attachments`或`content.videos`。

只有当`threads_by_folder.edges`为空数组时，收件箱才为空。如果用户询问消息请求，还应检查`--folder PENDING`；默认的`INBOX`文件夹不包含此类消息。

### 结果呈现
读取的结果保留了原始提供商的时间戳，并添加了语义化的`message_sent_at`/`last_message_sent_at`字段，分别以UTC时间和用户本地时间表示。这些时间仅用于标识消息的传输时间，绝不能被视为消息中所述事件的发生时间。`send`操作的公开标准输出仅包含`message_id`和整数型JSON格式的`timestamp_ms`；消息ID是不透明的，不得将其解析为FBID。向用户展示结果时，应重点关注参与者、消息文本、用户本地时间、链接和媒体摘要等有意义的内容，避免暴露原始ID、游标、Unix时间戳或实现细节，除非用户因后续操作而明确需要这些信息。