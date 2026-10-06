---
name: "threads"
description: "读取并管理用户的 Threads 账户：包括个人资料、帖子、信息流、已保存的帖子、活动、洞察数据、社交图谱、搜索结果、热门趋势，以及通过 URL 或 ID 查看特定帖子。可根据请求调整信息流排序，并发布帖子。"
icon: "threads"
metadata: { "包含在提示中": 真 }
---
# Threads CLI

## 用途
使用 `threads-cli` 附带的命令行工具读取和管理已认证的 Threads 数据。在支持 Threads 技能的任何场景下，均可使用读取、`dear-algo-whisper` 以及经批准的 `publish-post` 调用。

如需处理 Threads 收件箱和消息线程，请使用单独的 `threads_messages` 技能。

## 账号关联
在执行任何其他命令之前，请先通过运行 `threads-cli accounts` 确认用户的 Threads 账号已连接。如果该命令返回账号信息（一个或多个带有 `id` 的账号），则表示账号已连接，可正常继续操作。在本次会话的剩余时间内缓存此结果，无需在每次执行命令前都重新检查。如果后续任何命令因身份验证或账号问题而失败，请再次运行 `threads-cli accounts` 以重新确认账号关联状态。

如果该命令失败，或返回空结果表明未关联账号，则表示账号尚未连接。可通过运行 `threads-cli connect-url` 获取连接 URL（输出为包含 `connect_url` 字段的 JSON），然后告知用户，并代入该 URL：

> 您的 Threads 账号尚未连接。要进行连接，请访问 ``[Meta 账号中心](`connect_url`)`` 并关联您的 Threads 账号。

如果用户要求断开 Threads 账号的连接，请运行 `threads-cli disconnect-url`（输出为包含 `disconnect_url` 字段的 JSON），并引导其访问该 URL：

> 要断开 Threads 账号的连接，请访问 ``[Meta 账号中心](`disconnect_url`)`` 并移除已关联的账号。

请始终从命令输出中读取这些 URL，切勿将其硬编码。

## 工具使用
使用 `exec` 运行以下命令：

```sh
threads-cli <目标> [选项]
```

目标：
- `accounts`
- `profile`
- `activity-feed`
- `liked-media`
- `saved-posts`
- `insights-overview`
- `post-insights`
- `top-posts`
- `user-profile`
- `profile-threads`
- `profile-replies`
- `profile-media`
- `followers`
- `following`
- `post`
- `fetch-post-comments`（别名：`comments`）
- `fetch-post-likers`（别名：`likers`）
- `feedback-hub-overview`
- `feedback-hub-tab`
- `trends`
- `search`
- `feed`
- `publish-post`（隐藏；需明确确认）
- `dear-algo-whisper`

### 帖子输出
`feed` 和 `post` 使用与 `instagram-cli` 相同的紧凑型 `social_posts_v1` 格式。应以过滤后的结果为准，不得自行补全被省略的字段。对于仅用于提示错误的帖子行，将予以省略，并将其消息作为 `provider_error` 返回。

### 全局选项
- `--account-id <threads_account_id>` **（除 `accounts` 外的所有命令均需）** — 选择要操作的 Threads 账号。该值必须是 `accounts` 响应中当前认证用户自身的 `id` 字段。务必先调用 `accounts`。如果返回多个账号，请询问用户希望使用哪一个。
- `--retries <N>` — 对临时性失败进行重试（默认：0）。此选项不适用于 `dear-algo-whisper` 或 `publish-post`，因为重试可能导致变更操作重复执行。

## 命令

### Accounts
列出与当前认证用户关联的 Threads 账号。

```sh
threads-cli accounts
```

### Profile
获取当前用户自己的 Threads 个人资料。

```sh
threads-cli profile --account-id <threads_account_id>
```

### Activity Feed
获取用户的活动通知。

```sh
threads-cli activity-feed --account-id <threads_account_id>
threads-cli activity-feed --account-id <threads_account_id> --first 20
threads-cli activity-feed --account-id <threads_account_id> --category-filter text_post_app_mentions
threads-cli activity-feed --account-id <threads_account_id> --after <cursor>
```

类别筛选器：`text_post_app_conversations`、`text_post_app_following`、`text_post_app_private_follow_requests`、`text_post_app_mentions`、`text_post_app_replies`、`text_post_app_user_follows`、`text_post_app_quote_posts`、`text_post_app_reposts`。

### Liked Media
获取您点赞过的帖子。此功能使用共享的互动查询，并支持 `--since`、`--until`、`--sort-order`、`--limit` 和 `--after` 参数。

```sh
threads-cli liked-media --account-id <threads_account_id>
threads-cli liked-media --account-id <threads_account_id> --limit 20
threads-cli liked-media --account-id <threads_account_id> --since 2026-03-01 --until 2026-03-20
threads-cli liked-media --account-id <threads_account_id> --sort-order asc
threads-cli liked-media --account-id <threads_account_id> --after <cursor>
```

### 已保存的帖子
获取您的已保存帖子。
此命令使用共享互动查询，并支持 `--since`、`--until`、`--sort-order`、`--limit` 和 `--after` 参数。

```sh
threads-cli saved-posts --account-id <threads_account_id>
threads-cli saved-posts --account-id <threads_account_id> --limit 20
threads-cli saved-posts --account-id <threads_account_id> --since 2026-03-01 --until 2026-03-20
threads-cli saved-posts --account-id <threads_account_id> --after <cursor>
```

### 数据概览
获取账号级别的数据概览（浏览量、点赞数、引用数、回复数、转发数、流量来源、用户画像）。

```sh
threads-cli insights-overview --account-id <threads_account_id> --start-date 2026-03-13 --end-date 2026-03-20
threads-cli insights-overview --account-id <threads_account_id> --start-date 2026-03-13 --end-date 2026-03-20 --sections followers
```

可选的 `sections` 参数值：`summary`、`views`、`interactions`、`followers`、`demographics`、`all`。

类型化的数据后端会使用通过 `--account-id` 指定的 Threads 账号。如果账号绑定失败，请不要通过 `/gq` 重试；请重新运行 `accounts` 命令，并传入其中一个返回的账号 ID。

### 单个帖子的数据
获取特定帖子的数据。

```sh
threads-cli post-insights --account-id <threads_account_id> --post-id 3856993780407305605
```

### 热门帖子
获取在指定日期范围内浏览量最高的帖子，以及平台评选出的点赞数前三的热门帖子。`--count` 参数用于控制浏览量最高帖子的数量，上限为 50。

```sh
threads-cli top-posts --account-id <threads_account_id> --start-date 2026-03-13 --end-date 2026-03-20
threads-cli top-posts --account-id <threads_account_id> --start-date 2026-03-13 --end-date 2026-03-20 --count 5
```

### 其他用户的个人资料
根据数字形式的 Threads 用户 Facebook ID 获取其他用户的个人资料。该命令不接受用户名或个人资料链接。

```sh
threads-cli user-profile --account-id <threads_account_id> --user-id 12345678
```

### 个人主页动态
根据数字形式的 Threads 用户 Facebook ID 获取其主页动态。`--limit` 可作为 `--first` 的兼容别名使用。

```sh
threads-cli profile-threads --account-id <threads_account_id> --user-id 12345678
threads-cli profile-threads --account-id <threads_account_id> --user-id 12345678 --first 10 --after <cursor>
```

### 个人主页回复
获取某用户的回复内容。

```sh
threads-cli profile-replies --account-id <threads_account_id> --user-id 12345678
threads-cli profile-replies --account-id <threads_account_id> --user-id 12345678 --first 10 --after <cursor>
```

### 个人主页媒体
获取某用户的媒体帖（图片、视频、轮播图）。

```sh
threads-cli profile-media --account-id <threads_account_id> --user-id 12345678
threads-cli profile-media --account-id <threads_account_id> --user-id 12345678 --first 10 --after <cursor>
```

### 关注者
获取某用户的关注者列表。

```sh
threads-cli followers --account-id <threads_account_id> --user-id 12345678
threads-cli followers --account-id <threads_account_id> --user-id 12345678 --first 10 --after <cursor>
```

### 关注的人
获取某用户的关注列表。

```sh
threads-cli following --account-id <threads_account_id> --user-id 12345678
threads-cli following --account-id <threads_account_id> --user-id 12345678 --first 10 --after <cursor>
```

### 通过 URL 或 ID 获取帖子
获取特定帖子。如需获取其回复，请使用 `fetch-post-comments` 命令。

必须且仅能指定 `--url` 或 `--post-id`（别名 `--id`）中的一个。

当用户提供 Threads 的永久链接或分享链接（`threads.com` 或 `threads.net`）时，请优先使用 `--url`，并原样传递原始 URL。系统会解析 WWW 链接并强制执行访问权限检查。
```sh
threads-cli post --account-id <threads_account_id> --url https://www.threads.com/@carnage4life/post/DdU_q-9mLbt
threads-cli post --account-id <threads_account_id> --url https://www.threads.com/share/HCmFh1x9l/
```

当您已知帖子的数字 FBID 时，请使用 `--post-id`：

```sh
threads-cli post --account-id <threads_account_id> --post-id 3856993780407305605
```

### 获取帖子评论
获取一个或多个帖子媒体 ID 的评论。必要时可使用 `--since`、`--until`、`--sort-order`、`--limit` 以及重复或以逗号分隔的 `--author-id` 筛选条件。

```sh
threads-cli fetch-post-comments --account-id <threads_account_id> --post-ids 3856993780407305605
threads-cli fetch-post-comments --account-id <threads_account_id> --post-ids 3856993780407305605,3856993780407305606 --limit 20 --after <cursor>
threads-cli fetch-post-comments --account-id <threads_account_id> --post-ids 3856993780407305605 --since 2026-03-01 --until 2026-03-20 --author-id 12345678
threads-cli comments --account-id <threads_account_id> --post-ids 3856993780407305605
```

### 获取点赞用户
获取对一个或多个帖子媒体 ID 点赞的用户列表。必要时可使用 `--since`、`--until`、`--sort-order`、`--limit` 以及重复或以逗号分隔的 `--reactor-id` 筛选条件。

```sh
threads-cli fetch-post-likers --account-id <threads_account_id> --post-ids 3856993780407305605
threads-cli fetch-post-likers --account-id <threads_account_id> --post-ids 3856993780407305605 --limit 20 --after <cursor>
threads-cli fetch-post-likers --account-id <threads_account_id> --post-ids 3856993780407305605 --since 2026-03-01 --until 2026-03-20 --reactor-id 12345678
threads-cli likers --account-id <threads_account_id> --post-ids 3856993780407305605
```

### 反馈中心概览
获取帖子的互动汇总信息（点赞数、转发数、引用数）。

```sh
threads-cli feedback-hub-overview --account-id <threads_account_id> --post-id 3856993780407305605
```

### 反馈中心标签页
获取分页显示的点赞、转发或引用某帖子的用户列表。

```sh
threads-cli feedback-hub-tab --account-id <threads_account_id> --post-id 3856993780407305605 --tab-type like
threads-cli feedback-hub-tab --account-id <threads_account_id> --post-id 3856993780407305605 --tab-type repost --first 10
threads-cli feedback-hub-tab --account-id <threads_account_id> --post-id 3856993780407305605 --tab-type quote --after <cursor>
```

### 趋势
获取 Threads 上的热门话题。

```sh
threads-cli trends --account-id <threads_account_id>
threads-cli trends --account-id <threads_account_id> --first 10
```

### 搜索
按关键词搜索 Threads，或深入查看某一趋势。

```sh
threads-cli search --account-id <threads_account_id> --query "AI news"
threads-cli search --account-id <threads_account_id> --query "AI news" --recent 1
threads-cli search --account-id <threads_account_id> --query "trending topic" --trend-fbid 987654
threads-cli search --account-id <threads_account_id> --query "AI" --first 10 --after <cursor>
```

使用 `--recent 1` 可获取“最新”标签页的结果，而非“热门”结果。

### 动态
获取排序后的动态（“为你推荐”或“关注”）。

“为你推荐”动态：  
```sh
threads-cli feed --account-id <threads_account_id> --variant for_you
```

“关注”动态：  
```sh
threads-cli feed --account-id <threads_account_id> --variant following
threads-cli feed --account-id <threads_account_id> --variant following --sort-by recent
```

对于所有动态类型，均可使用 `--after` 进行分页：  
```sh
threads-cli feed --account-id <threads_account_id> --variant for_you --after <cursor>
```

### 发布帖子

`publish-post` 是一个隐藏的写入命令。它会在一次授权操作中发布一条普通的 Threads 帖子，并返回帖子 ID。没有公开的草稿步骤，也没有可在不同命令间传递的创建句柄。在没有媒体的情况下，文本必须至少包含一个非空白字符。所有被接受的文本字节，包括前导或尾随的空白以及 Unicode 字符，在确认和发布过程中都会被完整保留。一条帖子可以包含一到二十个按顺序排列的本地图片或视频（来自 Hatch 工作区），还可以选择回复某条特定帖子，并设置允许哪些用户进行回复。如果文件不在工作区内，需先将其复制到工作区；切勿使用 `/tmp` 路径。

请先运行 `threads-cli accounts`，并将所选账户的数字 `id` 原封不动地作为 `--account-id` 参数传递。若返回多个账户，请询问用户选择哪一个。账户 ID 和回复目标均为规范的正十进制 FBID，不含符号、前导零或前后空白；切勿从 URL 或用户名中推导出这些 ID。

```sh
threads-cli publish-post --account-id <threads_account_fbid> --text "Hello Threads"
threads-cli publish-post --account-id <threads_account_fbid> --text "Caption" --media-item '{"file":"/workspace/Launch photo.jpg","alt_text":"Description"}'
threads-cli publish-post --account-id <threads_account_fbid> --text "Mixed media" --media-item '{"file":"/workspace/first.jpg","alt_text":"First image"}' --media-item '{"file":"/workspace/second.mp4","cover":"/workspace/second-cover.jpg"}'
threads-cli publish-post --account-id <threads_account_fbid> --text "Reply video" --media-item '{"file":"/workspace/video.mp4","cover":"/workspace/video-cover.jpg"}' --reply-to-post-fbid <post_fbid> --reply-control mentioned_only
```

回复控制的取值只能是 `everyone`、`accounts_you_follow`、`mentioned_only`、`parent_post_author_only` 或 `followers_only`。旧的拼写如 `following`、`mentioned` 和 `followers` 均无效，且不得进行静默转换。

`--media-item` 可重复指定，最多二十次。每个 JSON 对象必须包含 `file` 字段，还可选填 `cover` 和 `alt_text` 字段。支持的媒体文件格式为 JPEG、PNG、静态 WebP、MP4 和 MOV，每份文件大小不超过 100 MB。媒体类型由文件扩展名推断得出。所有 MP4/MOV 文件都必须附带一张 JPEG、PNG 或 WebP 格式的封面；图片则不允许指定封面。文件的顺序即为展示顺序，且每张封面与其对应的视频文件保持关联。若仅发布纯文本，则无需指定任何媒体项。指定一项时发布单张图片或一段视频；指定两项至二十项时则按指定顺序发布一张轮播图。

审核通过后，CLI 会通过其受限制的输入通道读取所有声明的文件，并以分块上传的方式将其上传至 Threads 的私有预发布端点。单个媒体项连同其指定的文本、替代文字及回复设置一同暂存，随后 `publish-post` 命令仅接收系统返回的私有创建 ID。对于轮播图，CLI 会将每个按序排列的媒体项连同空文本、无回复设置及 `is_carousel_item=true` 标记一同暂存，验证每个独立的创建 ID 合法性，最后发布一个 `media_type=CAROUSEL` 类型的父级帖子，其中包含按序排列的子项、指定文本及可选的回复设置。相关路径和创建 ID 均为内部使用，对外不可见。

CLI 在提交发布前会对整条帖子进行完整性校验。一次 `content.post` 审核涵盖账户信息、人工回复上下文、回复控制、指定文本以及所有按序排列的媒体内容。确认页面会从经过身份验证的工作区预览资源中显示每项内容，文件名采用安全的命名方式，并可选显示指定的替代文字；同时，每个视频的封面将以相邻图片附件的形式紧随其后显示。因此，最多二十个媒体项可能产生四十个视觉附件。确认环节绝不会代入原始标识符；缺失或不合规的文件名将被替换为 `Threads image` 或 `Threads video`。

文件名在去除首尾空白后必须非空，长度不得超过 255 个 Unicode 标量字符，且不得包含斜杠、反斜杠、控制字符、仅由数字组成的文件名主体、十六进制摘要或服务商/上传者标识。替代文字在去除首尾空白后也必须非空，长度不得超过 1024 个 Unicode 标量字符，且不得包含控制字符。超出长度将导致操作失败。获批后，每次写入均不重试。若发生子级失败或重复句柄，将立即终止，且不会创建后续子级或发布父级。任何后续尝试都将视为新的调用，需重新获得明确批准。创建 ID 仅接受来自私有暂存响应的值。

### 写入后安全性

1. 未经明确的“Hatch”确认窗口批准，切勿发布。拒绝或关闭该窗口将终止操作。
2. 切勿使用传输重试机制。子项或父项失败即为最终结果且具有不确定性，不得继续处理失败的轮播内容。任何后续尝试均视为新的调用，需重新获得明确批准。
3. 显示完整的帖子正文、准确的回复控件、所有按顺序排列的媒体项，以及上述提及的每个视频封面。尽最大努力在最多30秒内解析账户和回复标签；若查询超时、返回错误、数据格式异常，或标签缺失/不安全，则显示一个非识别性的“未验证 Threads…”占位符，并继续等待明确批准，切勿以原始标识符替代。切勿在常规文本中显示账户、回复、子项、创建时间或帖子的 ID，以及本地路径。
4. 帖子正文仅用于检测是否为空；对于非空内容，包括首尾空白及 Unicode 字符，其每一个字节均会在结构化的确认预览和已发送请求中完整保留。若全文无法在不截断预览的情况下容纳，则写入操作将被拒绝。
5. 此写入命令在支持 Threads 技能的任何场景下均可使用，但每次调用仍需单独获得明确确认。
6. 应妥善处理授权失败、账户选择失败及配额不足等情况，切勿通过其他途径或未公开的命令路径绕过这些限制。
7. 当确切的发布结果至关重要时，请读取已发布的帖子，并将其与已批准的快照进行比对。

### 亲爱的算法低语
向Threads的推荐算法直接发送一条消息，以调整用户信息流中显示的内容（例如：“少给我推荐政治内容”、“多给我推荐猫咪相关内容”）。此命令对所有支持的用户开放，且在收到明确的用户请求时，无需额外确认即可执行。

```sh
threads-cli dear-algo-whisper --account-id <threads_account_id> --message "show me less politics"
```## 操作规则
1. 本技能仅读取经身份验证的用户可访问的 Threads 数据——包括其本人账号以及用户提供的链接所指向的帖子——而非通用的 Threads 搜索功能。
2. **务必先调用 `accounts` 接口**以获取账号的 `id`，并将其作为 `--account-id` 参数。在同一次对话中，请缓存该值以供后续命令使用。如果 `accounts` 返回空结果，请引导用户前往 Meta 账号中心绑定 Threads 账号（详情请参阅“账号绑定”章节）。
3. **尽量减少 API 调用次数，避免触发限流。** Threads API 对调用频率有严格限制，过多的调用将导致 `429 Too Many Requests` 错误，且此类错误不会重试。请遵循以下原则：
   - **批量获取信息。** 在发起调用前，明确所需数据，切勿盲目获取。
   - **复用先前响应中的数据。** 如果已获取过 `accounts` 或 `profile` 数据，应直接从响应中提取 ID 和用户名，无需再次调用。
   - **避免重复分页。** 仅在用户明确要求查看更多结果时才进行分页（`--after`），切勿自动拉取所有页面。
   - **在同一对话中，除非前一次调用失败或用户明确要求刷新，否则不要使用完全相同的参数重复调用同一命令。**
4. 避免对这些命令进行持续或频繁的轮询式请求。
5. 对于账号、用户、帖子及其他实体的标识符，请使用数字形式的 Threads FBID。切勿从 Threads URL 中推导标识符，也不要在需要 FBID 的地方传递 URL。唯一例外是 `post --url` 命令，该命令原样接收原始 Threads URL，并由 WWW 自行解析。
6. 将 U18 内容过滤视为服务端的固有约束，切勿尝试重建被过滤的字段，也勿通过 `/gq` 端点重试，或以其他方式绕过服务端的过滤结果。
7. 使用各读取命令返回的结构化、已过滤的媒体结果，切勿自行重建被省略的提供方字段。

## 输出
CLI 会将解码后的 JSON 打印到标准输出。`publish-post` 命令仅输出
`{"post_id":"<string>"}`。请将该值视为仅供机器使用的流程句柄，切勿在面向用户的文案或审批预览中重复使用。
