---
name: "instagram"
description: "查看 Instagram 个人资料、关注者、帖子、评论、点赞、快拍、动态、已保存内容及账号洞察。回答有关帖子、短视频和 Instagram 链接的问题。根据要求管理兴趣与个人资料信息，并发布快拍、短视频、帖子或轮播帖。"
icon: "instagram"
metadata: { "包含在提示中": 真 }
---
# Instagram 命令行工具

## 用途
读取、管理并解答有关 Instagram 数据的问题。

## 分析帖子和短视频

在处理 Instagram 帖子或短视频的内容之前，请先使用 `muse.read` 阅读 [Instagram 内容相关问题](references/content-questions.md)。说明您已核实的内容，区分猜测与已确认的事实，并指出无法验证的部分。

## 账号关联

除 `accounts`、`connect-url`、`disconnect-url` 和 `--help` 外，所有 `instagram-cli` 命令均需指定 `--account-id`。对于这些命令，请使用上下文中可用的、通过 `instagram-cli accounts` 获取的用户 `user_fbid`。若无可用的 `user_fbid`，请先运行 `instagram-cli accounts`。在发生身份验证或账号错误后，或当用户表示已连接、断开连接或切换账号时，请再次运行 `accounts`。

如果任务中返回了多个账号但未进行选择，或者所选关联账号与请求的发布目标不一致，请在执行相关命令前先确定要使用的账号。若任务中已明确指定了账号，则无需再次询问，直接沿用该选择。在实时对话中，由客服人员向用户确认未解决的账号选择；子代理则将未解决的账号选择反馈给其上级代理；独立工作者应在最终回复中报告未解决的账号选择。切勿随意选择账号。

如果某个命令提示 Instagram 尚未连接，请将其指定的 `--account-id` 与最新获取的 `accounts` 结果进行核对。若所选账号对应的 `user_fbid` 发生变化，请使用新的 `user_fbid` 重新尝试。仅当 `accounts` 返回空列表，或命令以新验证的所选账号 `user_fbid` 报告其未连接时，才将 Instagram 视为未关联状态。若处于未关联状态且任务需要账号，请运行 `instagram-cli connect-url`，并在回复中使用其提供的 `connect_url`：

> 您的 Instagram 账号尚未连接。如需连接，请访问 ``[Meta 账号中心](`connect_url`)`` 并关联您的 Instagram 账号。

对于帖子或短视频的链接，无需用户提供账号。相关内容请参考 Instagram 内容相关问题。

读取公开个人资料无需连接 Instagram。若未关联任何账号，可通过网络搜索或浏览器任务来解答公开个人资料相关问题。如仍需通过浏览器查看，请在条件允许时调用 `browser.spawn_task`。只有在确实需要通过浏览器查看且无法调用 `browser.spawn_task` 的情况下，子代理才会将已知的个人资料网址或用户名以及未解决的问题反馈给其上级代理。
原生的 `profile`、`user-profile` 和 `post` 命令仍要求关联账号。除非用户主动要求连接，否则请勿针对仅需公开信息的任务发送连接链接。

若用户要求断开 Instagram 关联，请运行 `instagram-cli disconnect-url`，并向用户发送其提供的 `disconnect_url`：

> 如需断开您的 Instagram 账号，请访问 ``[Meta 账号中心](`disconnect_url`)`` 并移除关联的账号。

请使用相应命令返回的 URL，切勿硬编码。

## 相关技能
- 使用 `instagram_messages` 技能进行收件箱、对话及消息的读取、私信搜索与发送，以及热门联系人查询。请遵循其单独的消息连接与发送规则。
- 使用 `social.search` 进行通用帖子的发现。
- 使用本技能获取直接的 `account-insights` 报告，并解释或重述所提供的指标数值。使用 `social_content_performance` 进行性能对比、趋势分析，或为用户自己的账号与帖子提供内容建议。使用 `social_competitor_analysis` 进行竞争对手、同行或类别的比较。当性能对比中的所有账号均属于用户本人时，请使用 `social_content_performance`。如果 `social_content_performance` 或 `social_competitor_analysis` 不可用，
请针对它们支持的部分，使用本技能的文档中所列的读取功能。
`account-insights` 仅用于用户已关联的账号。对于竞争对手，请在符合账号关联规则的前提下，使用公开的 `profile` 和 `posts` 数据。不得以用户的账号数据替代竞争对手的指标，也不得自行填补缺失值。请明确说明哪些请求的指标或对比仍无法获取。

## 执行命令

使用 `exec` 来运行 `instagram-cli` 命令。

请严格按照文档中规定的标志名称使用。未知的标志可以被忽略，不会引发错误。

对于 `fetch-post-comments` 和 `fetch-post-likers`，即使只处理一个帖子也必须使用 `--post-ids`。对于 `save-post`、`unsave-post` 和 `media-understanding`，即使只处理一项也必须使用 `--media-ids`。单数形式的标志 `--post-id` 和 `--media-id` 均不支持。

请复用上下文中已有的结果。当脚本需要进一步处理某个结果时，应将首次调用的输出重定向到 `~/workspace/` 目录下的文件，并从该文件中读取，而不要在同一任务中使用相同参数重复执行已成功完成的命令。

Instagram API 实施了速率限制。如果某次请求返回 `429 Too Many Requests`，则应停止当前任务中的所有 `instagram-cli` 调用，包括由脚本发起的调用。如果是针对帖子或短视频的链接，则可继续采用 Instagram 内容相关问题中的备选方案；否则，请报告已完成的工作及剩余部分。

在脚本中，请检查每次 `instagram-cli` 调用的退出状态。遇到首个 429 错误时即应停止，且不应将失败的调用视为空结果。

如果批量读取操作反复返回 HTTP 500 错误，则应停止对该批量读取的所有调用，包括来自脚本的调用。仅凭 HTTP 500 错误本身并不能判定为速率限制。请说明已读取的数据量及剩余待读取的部分。

对于需覆盖整个列表的请求，应在分页仍有进展时逐页获取数据；在尝试获取已用于该列表的游标之前应停止；此外，当某一页显示还有更多页面但未新增任何内容时也应停止。这些检查应在分页脚本内部执行。若在完成全部请求范围之前停止，请说明已读取的数据量及剩余待读取的部分。

当某项请求需要对超过 50 个账号或帖子分别进行 `user-profile`、`profile` 或 `post` 调用时，应先从最近的 50 个开始，处理完这 50 个后停止，并报告其结果以及后续批次还需处理的数量。此类调用在达到几十个之后会受到 Instagram 的速率限制。

仅在用户明确要求进行更改时才执行写入操作，包括更新兴趣、发布内容及修改个人资料等。

`instagram-cli` 不支持关注、取消关注、移除粉丝、屏蔽、点赞、评论、回复评论、删除帖子、归档帖子、置顶帖子或编辑帖子说明等功能。

### 帖子输出格式
`posts`、`feed` 和 `post` 返回与 `social.search` 相同的精简帖子集合。请直接读取 `posts[]`，切勿猜测提供方的 GraphQL 路径，亦勿通过 `jq` 进行管道处理：

```json
{
  "format": "social_posts_v1",
  "count": 1,
  "posts": [{
    "rank": 1,
    "post_id": "...",
    "url": "https://www.instagram.com/...",
    "platform": "instagram",
    "post_created_at": {"utc":"...","user_local":"...","user_timezone":"..."},
    "username": "account",
    "post_caption": "..."
  }],
  "next_cursor": "...",
  "has_next_page": true
}
```

在展示帖子发布时间时，优先使用 `post_created_at.user_local`。响应中保留了 `created_at` 字段以保持兼容性。提供方响应中不存在的字段将被省略。信息流中的某条内容可能缺少 `url` 或 `created_at` 字段。在引用、标注日期或添加链接之前，请先执行 `post --id <post_id>` 并使用返回的记录。有效的部分响应会在可选的 `provider_error` 中包含提供方给出的原因，切勿将其描述为完整的页面。如果必需的帖子数据缺失，CLI 将报错，并在提供方已给出原因时一并说明；若无提供方原因，则会报告 schema 不匹配，而非返回空的帖子列表。
帖子行可以包含作者的 `username`、`author_name`、`author_bio`、
`follower_count` 和 `verified`。请使用这些返回的字段，而不要针对相同信息进行单独的个人资料查询，除非请求需要缺失或已刷新的个人资料信息。

`posts`、`feed` 和 `post` 响应可以在 `likes` 和 `comments` 中包含每条帖子的计数，但不会公开每条帖子的浏览量、触达人数、收藏数或分享数。请勿将账号级别的总计分配给单个帖子。

### 全局选项
- `--account-id <user_own_fbid>` 用于选择账号。请使用“账号关联”中获取的用户自身 `user_fbid`，切勿传递其他用户的 ID。
- `--retries <N>` 用于重试临时性失败。默认值为 0。对于 `post-story`、`post-feed`、`set-profile-picture` 和 `profile-banner`，非零值不受支持。
- `--after <cursor>` 提供上一次响应中的分页游标，但 `own-stories-archive` 使用 `--max-id`。省略游标则获取第一页。

对于 `fetch-post-comments` 和 `fetch-post-likers`，分页是按 `post_groups[]` 条目进行的。若要为 `has_next_page` 为 true 的条目获取下一页，只需将该条目的 `media_id` 作为 `--post-ids` 传入，并将 `end_cursor` 作为 `--after` 传入。

## 命令

可用命令：
- `accounts`
- `connect-url`
- `disconnect-url`
- `profile`
- `current-interests`
- `posts`
- `followers`
- `following`
- `close-friends`
- `feed`
- `activity-notifications`
- `user-profile`
- `tagged-posts`
- `post`
- `fetch-post-comments`
- `fetch-post-likers`
- `saved-posts`
- `saved-collections`
- `create-saved-collection`
- `rename-saved-collection`
- `save-post`
- `unsave-post`
- `own-stories`
- `own-stories-archive`
- `stories-tray`
- `story-media`
- `location-search`
- `media-understanding`
- `account-insights`
- `recently-liked-posts`
- `recently-commented-posts`
- `update-interests`
- `post-feed`
- `post-story`
- `set-profile-picture`
- `update-bio`
- `profile-banner`

### Accounts
列出与当前用户关联的 Instagram 账号。

```sh
instagram-cli accounts
```

### Profile
获取 Instagram 个人资料简介及关注者/粉丝数量。省略 `--username` 和 `--profile-url` 可获取用户本人的个人资料；否则，请提供用户名或个人资料 URL 以获取他人的个人资料。当仅需关注者和粉丝数量时，可使用此命令，无需单独获取关注列表或粉丝列表。

使用 `--username` 时，每次调用仅传入一个用户名。从 `profiles[]` 中读取结果，包括 `bio` 和 `website`。对于 `profile` 和 `user-profile`，仅凭 `profiles` 列表为空，并不能判定该账号名是否可用或已被占用，也不能说明工具执行失败。

```sh
instagram-cli profile --account-id <user_own_fbid>
instagram-cli profile --account-id <user_own_fbid> --username <username>
instagram-cli profile --account-id <user_own_fbid> --username @<username>
instagram-cli profile --account-id <user_own_fbid> --profile-url https://instagram.com/<username>
```

### 更新简介
更新用户的 Instagram 个人简介。`--bio` 为必填项，且最多可包含 150 个字符。传入空字符串可清空简介。

```sh
instagram-cli update-bio --account-id <user_own_fbid> --bio "<bio>"
```

### Muse 个人资料横幅
在用户的 Instagram 个人资料上添加或移除 Muse 横幅。该横幅会显示在用户个人资料简介下方，呈长条状。此操作不会更改用户的头像或简介。

```sh
instagram-cli profile-banner --account-id <user_own_fbid> --action add
instagram-cli profile-banner --account-id <user_own_fbid> --action remove
```

请勿提供媒体或个人资料 URL。仅当结果中包含 `success: true` 时才视为完成；`success: false` 表示更新未成功。自动重试功能已禁用；遇到不确定的失败后，请勿自行认定操作成功，也不要静默重复该变更操作。

### 当前兴趣
获取用户感兴趣或不感兴趣的话题。显示话题及其类型，不区分是推断出的兴趣还是明确设置的兴趣。
```sh
instagram-cli current-interests --account-id <user_own_fbid>
```

### 更新兴趣
将某个话题标记为“感兴趣”，以便用户在 Instagram 上看到更多相关内容；或标记为“不感兴趣”，以减少相关内容的展示。
```sh
instagram-cli update-interests --account-id <user_own_fbid> --text "<topic>" --interested true
instagram-cli update-interests --account-id <user_own_fbid> --text "<topic>" --interested false
```

### 帖子
省略 `--username` 和 `--user-id` 参数可获取用户本人的帖子；否则，提供一个或多个用户名或一个或多个用户 ID 以获取其他用户的帖子。同一命令中请勿混用用户名和用户 ID。
将上一次响应中的 `next_cursor` 作为 `--after` 参数传递；首次请求时省略 `--after`。
可通过 `--since`、`--until`、`--sort-order asc|desc`、`--limit` 以及重复或以逗号分隔的 `--post-types` 参数（如 POST、REEL、STORY 或 HIGHLIGHT）来缩小结果范围。

响应中返回的帖子数量可能超过 `--limit` 的限制。每个帖子均归属于其对应的 `username`，该用户名可能与请求的账号不同。若仅需获取帖子和 Reels，请使用 `--post-types POST,REEL`。请勿根据返回帖子的顺序推断个人主页的排列顺序或置顶状态。
```sh
instagram-cli posts --account-id <user_own_fbid>
instagram-cli posts --account-id <user_own_fbid> --after <cursor>
instagram-cli posts --account-id <user_own_fbid> --username <username>
instagram-cli posts --account-id <user_own_fbid> --username <username>,<username>
instagram-cli posts --account-id <user_own_fbid> --user-id <FBID>
instagram-cli posts --account-id <user_own_fbid> --user-id <FBID>,<FBID>
instagram-cli posts --account-id <user_own_fbid> --username <username> --since 2026-01-01 --until 2026-02-01 --sort-order asc --limit 25 --post-types POST,REEL
```

### 粉丝
获取用户本人的粉丝列表。`--count` 参数为可选，默认值为 200。
读取返回的 `users[]` 列表。当 `has_more` 为真时，可将上一次响应中的 `next_max_id` 作为 `--after` 参数传递，以获取下一页数据；首次请求时省略 `--after`。
对于“粉丝”和“关注”的返回数据，`users[]` 中包含 `id`、`username`、`full_name` 和 `profile_pic_url`。请通过 `id` 来比较列表成员关系，无需对每个账号单独进行资料查询。
```sh
instagram-cli followers --account-id <user_own_fbid>
instagram-cli followers --account-id <user_own_fbid> --count 25
instagram-cli followers --account-id <user_own_fbid> --after <cursor>
```

### 关注
获取用户本人的关注列表。不支持获取其他用户的关注列表。`--count` 参数为可选，默认值为 200。
读取返回的 `users[]` 列表。当 `page_info.has_next_page` 为真时，可将上一次响应中的 `page_info.end_cursor` 作为 `--after` 参数传递，以获取下一页数据；首次请求时省略 `--after`。
```sh
instagram-cli following --account-id <user_own_fbid>
instagram-cli following --account-id <user_own_fbid> --count 25
instagram-cli following --account-id <user_own_fbid> --after <cursor>
```

### 密友
获取用户当前的 Instagram 密友列表。
这是用户在 Instagram 上自行设置的密友列表，未经过排序或推断。若需获取用户互动最频繁的人员推断列表，请改用 `instagram_messages` 技能中的 `top-recipients` 命令。
```sh
instagram-cli close-friends --account-id <user_own_fbid>
```

### 动态
关注动态：
```sh
instagram-cli feed --account-id <user_own_fbid> --variant following
```
密友动态：
```sh
instagram-cli feed --account-id <user_own_fbid> --variant favorites
```
`--variant` 参数为必填项，支持的值为 `following` 和 `favorites`。此接口无法获取经过排序的首页时间线。若省略 `--variant`，将返回错误。对于所有信息流变体，将上一次响应的 `next_cursor` 作为 `--after` 参数传递。首次请求时省略 `--after`。

```sh
instagram-cli feed --account-id <user_own_fbid> --variant following --after <cursor>
```

### 活动通知
获取 `notifications` 和 `priority_notifications`，且不将其标记为已读。使用 `page_info.end_cursor` 作为 `--after` 进行分页。

```sh
instagram-cli activity-notifications --account-id <user_own_fbid> [--after <cursor>]
```

### 其他用户的个人主页
通过用户 ID、用户名或个人主页 URL 获取其他用户的资料。如果之前已从工具输出中获得用户 FBID，建议优先使用 `--user-id`。  
使用 `--username` 时，每次调用仅传入一个用户名。结果可在 `profiles[]` 中读取，包括 `bio` 和 `website`。

```sh
instagram-cli user-profile --account-id <user_own_fbid> --user-id <user_fbid>
instagram-cli user-profile --account-id <user_own_fbid> --username @<username>
instagram-cli user-profile --account-id <user_own_fbid> --profile-url https://instagram.com/<username>
```

### 被标记的帖子
获取某用户被标记的帖子。省略 `--user-id` 可获取该用户本人被标记的帖子；否则，请将目标用户的 FBID 作为 `--user-id` 传递。  
可通过 `--since`、`--until`、`--sort-order` 和 `--limit` 缩小结果范围。

```sh
instagram-cli tagged-posts --account-id <user_own_fbid>
instagram-cli tagged-posts --account-id <user_own_fbid> --after <cursor>
instagram-cli tagged-posts --account-id <user_own_fbid> --user-id <user_fbid>
instagram-cli tagged-posts --account-id <user_own_fbid> --user-id <user_fbid>,<user_fbid> --since 2026-01-01 --until 2026-02-01 --sort-order asc --limit 25
```

### 通过 ID 或 URL 获取帖子
通过媒体 ID 或完整的 Instagram 帖子/短视频 URL 获取单个帖子。请仅提供 `--id` 或 `--url` 中的一个。

```sh
instagram-cli post --account-id <user_own_fbid> --id <media_id>_<owner_id>
instagram-cli post --account-id <user_own_fbid> --id <media_id>
instagram-cli post --account-id <user_own_fbid> --url https://www.instagram.com/p/<shortcode>/
instagram-cli post --account-id <user_own_fbid> --url https://www.instagram.com/reels/<shortcode>
```

### 帖子评论
通过媒体 ID 获取一个或多个帖子的评论。可将多个 ID 以逗号分隔的形式放入 `--post-ids` 列表中。可通过 `--since`、`--until`、`--sort-order`、`--limit` 以及重复或以逗号分隔的 `--author-ids` 来缩小结果范围。  
评论内容可在 `post_groups[].comments[].comment_text` 中读取。

```sh
instagram-cli fetch-post-comments --account-id <user_own_fbid> --post-ids <media_id>
instagram-cli fetch-post-comments --account-id <user_own_fbid> --post-ids <media_id>,<media_id>
instagram-cli fetch-post-comments --account-id <user_own_fbid> --post-ids <media_id> --limit 25 --after <cursor>
instagram-cli fetch-post-comments --account-id <user_own_fbid> --post-ids <media_id> --since 2026-01-01 --until 2026-02-01 --author-ids <user_fbid>
```

### 帖子点赞者
通过媒体 ID 获取一个或多个帖子的点赞者。可将多个 ID 以逗号分隔的形式放入 `--post-ids` 列表中。可通过 `--since`、`--until`、`--sort-order`、`--limit` 以及重复或以逗号分隔的 `--reactor-ids` 来缩小结果范围。  
点赞者信息可在 `post_groups[].reactors[]` 中读取。

```sh
instagram-cli fetch-post-likers --account-id <user_own_fbid> --post-ids <media_id>
instagram-cli fetch-post-likers --account-id <user_own_fbid> --post-ids <media_id>,<media_id>
instagram-cli fetch-post-likers --account-id <user_own_fbid> --post-ids <media_id> --limit 25 --after <cursor>
instagram-cli fetch-post-likers --account-id <user_own_fbid> --post-ids <media_id> --since 2026-01-01 --until 2026-02-01 --reactor-ids <user_fbid>
```

### 收藏的帖子
获取用户的收藏帖子。若要获取某个收藏集中的帖子，请使用 `saved-collections` 中的 `collection_id` 作为 `--collection-id` 传递。  
在不指定 `--collection-id` 的情况下，`saved-posts` 支持 `--since`、`--until`、`--sort-order` 和 `--limit`。

```sh
instagram-cli 已保存的帖子 --账户ID <用户自己的FB ID>
instagram-cli 已保存的帖子 --账户ID <用户自己的FB ID> --之后 <游标>
instagram-cli 已保存的帖子 --账户ID <用户自己的FB ID> --自2026-01-01 --至2026-02-01 --排序方式 升序 --限制 25
instagram-cli 已保存的帖子 --账户ID <用户自己的FB ID> --之后 <游标> --收藏集ID "<收藏集ID>"
```

### 已保存的收藏集
获取用户的已保存收藏集（已保存帖子的文件夹/分类）。

```sh
instagram-cli 已保存的收藏集 --账户ID <用户自己的FB ID>
instagram-cli 已保存的收藏集 --账户ID <用户自己的FB ID> --之后 <游标>
```

### 管理已保存的帖子和收藏集

```sh
instagram-cli 创建已保存的收藏集 --账户ID <用户自己的FB ID> --名称 "<名称>"
instagram-cli 重命名已保存的收藏集 --账户ID <用户自己的FB ID> --收藏集ID <收藏集ID> --名称 "<名称>"

instagram-cli 保存帖子 --账户ID <用户自己的FB ID> --媒体ID <媒体ID>
instagram-cli 保存帖子 --账户ID <用户自己的FB ID> --媒体ID <媒体ID>,<媒体ID> --收藏集ID <收藏集ID>

instagram-cli 取消保存帖子 --账户ID <用户自己的FB ID> --媒体ID <媒体ID> --收藏集ID <收藏集ID>
instagram-cli 取消保存帖子 --账户ID <用户自己的FB ID> --媒体ID <媒体ID>
```

使用收藏集保存帖子时，既会将帖子保存到收藏集中，也会将其添加到该收藏集中。仅通过收藏集取消保存时，只会将其从该收藏集中移除；若不指定 `--collection-id`，则会在全局范围内取消保存。

### 我的动态
获取用户当前发布的 Instagram 动态。

```sh
instagram-cli 我的动态 --账户ID <用户自己的FB ID>
```

### 动态存档
获取用户的 Instagram 动态存档。可使用上一次响应中的 `next_max_id` 作为 `--max-id` 进行分页；首次请求时无需指定 `--max-id`。当某条动态的视频 URL 不为空时，请使用视频 URL 而非图片 URL。

```sh
instagram-cli 我的动态存档 --账户ID <用户自己的FB ID>
instagram-cli 我的动态存档 --账户ID <用户自己的FB ID> --最大ID <游标>
```

### 动态托盘
获取用户的动态托盘，其中包含其关注账号的可用动态。`--count` 为可选参数，默认值为 200。响应中仅包含媒体 ID。如需获取特定动态的永久链接，请将其 ID 传递给 `story-media`。

```sh
instagram-cli 动态托盘 --账户ID <用户自己的FB ID>
instagram-cli 动态托盘 --账户ID <用户自己的FB ID> --数量 25
```

请仅对用户明确请求的动态调用 `story-media`，默认情况下不要批量获取托盘中所有动态的媒体信息。

### 动态媒体
根据 ID 获取一条或多条动态的媒体元数据，包括永久链接和发布时间。可在 `--ids` 参数中以逗号分隔列出多个 ID。

```sh
instagram-cli 动态媒体 --账户ID <用户自己的FB ID> --ID <动态ID>
instagram-cli 动态媒体 --账户ID <用户自己的FB ID> --ID <动态ID>,<动态ID>
```

### 地点搜索
使用必填的 `--search-query` 搜索 Instagram 上的地点。还可同时传入 `--latitude` 和 `--longitude` 来缩小搜索范围。

```sh
instagram-cli 地点搜索 --账户ID <用户自己的FB ID> --搜索词 "<查询>"
instagram-cli 地点搜索 --账户ID <用户自己的FB ID> \
  --搜索词 "<查询>" --纬度 <纬度> --经度 <经度>
```

### 用于发布的媒体文件
用于 `发布动态`、`发布动态消息` 和 `设置头像` 的媒体文件及封面文件必须位于 `~/workspace/` 目录下。在使用来自 `~/workspace/` 之外的文件之前，需先将其复制到 `~/workspace/` 中，不得使用 `/tmp` 路径。

### 发布动态
可发布一张 JPEG、PNG、静态 WebP、MP4 或 MOV 格式的文件，建议采用竖屏 9:16 比例，分辨率为 1080 x 1920 像素。请勿对媒体进行黑边处理。`--file` 参数为必填项，且文件大小不得超过 100 MB。直接发布的视频需要提供一张与之匹配的 JPEG、PNG 或静态 WebP 格式的 `--cover` 封面。带有贴纸或文字的视频会自动生成封面。

使用 `post-story --draft` 时，请同时指定 `--file` 和 `--editor-json`，以处理所有可见的 Story 文本及受支持的原生贴纸。首先运行 `instagram-cli post-story --help`，并遵循其详尽的参数、文本、贴纸、几何布局和媒体格式说明。在草稿模式下，渲染内容会保存至 `~/workspace/instagram/stories/` 目录，但不会发布。后续的每次修订均基于同一份干净的媒体素材进行渲染。请记录每次生成的草稿输出及其封面路径。

使用 `--editor-json` 编辑的 Story 在发布前需经用户确认。若待确认，实时对话中的代理会内联显示预览并请求确认；子代理会将预览返回给父代理，而独立的工作进程则通过正常的结果传递机制返回预览。仅当委托或独立任务已包含用户对渲染后草稿的确认时，方可从该任务中发布编辑后的 Story。

获得确认后，再次执行相同的命令，但不带 `--draft` 参数。此时将在 `~/workspace/instagram/stories/` 目录中渲染已批准的编辑版本，并直接发布该输出。发布成功后，请按精确记录的路径删除所有先前的草稿及其封面。保留最后一次用户批准的草稿以及最终发布的渲染结果，切勿使用通配符。

目前仅支持提及、位置和链接这三种原生贴纸类型，其他类型的原生贴纸将被拒绝。对于位置贴纸，请先通过 `location-search` 获取搜索结果，并将其 `location_id` 填入传递给 `--editor-json` 的位置对象中。提及对象需提供 `user_fbid`，链接对象则需提供 `url`。

最终命令会自动上传渲染后的输出及其视频封面：

```sh
EDITOR_JSON='[
  {"type":"text","text":"<text>","style":"poster","x":0.5,"y":0.2},
  {"type":"mention","text":"@<username>","user_fbid":"<user_fbid>","style":"default","x":0.5,"y":0.4},
  {"type":"location","text":"<location_name>","location_id":"<location_id>","style":"default","x":0.5,"y":0.65},
  {"type":"link","text":"<link_text>","url":"<url>","style":"default","x":0.5,"y":0.82}
]'

instagram-cli post-story --draft --account-id <user_own_fbid> \
  --file <clean-media> --editor-json "$EDITOR_JSON"

instagram-cli post-story --account-id <user_own_fbid> --file <clean-media> \
  --editor-json "$EDITOR_JSON"
```

### 发布动态
将媒体发布到用户的动态中。`post-feed` 会根据提供的文件选择帖子类型：
- 如果提供一张 JPEG、PNG 或静态 WebP 图片，则发布为普通图片帖子。建议使用 4:5 的竖版尺寸，分辨率为 1080×1350 像素；正方形 1:1 尺寸（1080×1080 像素）亦可。
- 如果提供一个 MP4/MOV 文件，则发布为分享至动态和个人主页网格的短视频。请使用 9:16 的竖版视频，推荐分辨率为 1080×1920 像素，并同时提供一张尺寸匹配的 JPEG、PNG 或静态 WebP 封面图（`--cover` 参数）。
- 如果提供两张或更多 JPEG、PNG、静态 WebP、MP4 或 MOV 文件，则按提供的顺序发布为轮播帖。支持纯图片轮播、纯视频轮播以及图片与视频混合的轮播。所有内容应保持相同的宽高比和分辨率，建议使用 4:5 的比例，分辨率为 1080×1350 像素。

每个文件的大小上限为 100 MB。`--caption` 为可选参数，可包含话题标签和文本形式的@提及，且字符数限制为 2,200 个。仅包含空白字符的标题将被视为未填写。`--mentions` 接受一个用户标签的 JSON 数组，每个标签必须指定 `user_fbid`，可选的 `x` 和 `y` 坐标范围为 0 到 1，默认值为 0.5。对于轮播帖，命令行工具会将提供的用户标签应用到每一张图片/视频上。对于视频媒体，`--video-thumbnail-playback-offset-ms` 可选地指定用于最终缩略图的非负视频帧。每个视频都需要一张 JPEG、PNG 或静态 WebP 格式的封面图片（`--cover`），该封面将在帧提取完成之前显示。对于轮播帖，需按与视频文件相同的顺序，为每个视频重复指定一次 `--cover`。图片类内容无需封面。如果是仅包含图片的帖子或轮播帖，请勿使用上述任一视频相关选项。```sh
# 发布图片
instagram-cli post-feed --account-id <用户自己的FBID> \
  --file ~/workspace/<图片>.jpg --caption '<描述>' \
  --mentions '[{"user_fbid":"<被标记用户的FBID>","x":0.5,"y":0.5}]'

# 发布到动态的短视频
instagram-cli post-feed --account-id <用户自己的FBID> \
  --file ~/workspace/<短视频>.mp4 --cover ~/workspace/<封面>.jpg \
  --video-thumbnail-playback-offset-ms 1200 \
  --caption '<描述>'
```# 混合轮播。文件顺序即展示顺序，封面顺序跟随视频。
instagram-cli post-feed --account-id <user_own_fbid> \
  --file ~/workspace/<first>.jpg --file ~/workspace/<second>.mp4 \
  --cover ~/workspace/<second-cover>.jpg \
  --caption '<caption>'
```

在 `post-feed` 命令正在运行时，或该发布的计划任务仍在进行中时，请勿再次启动相同的发布操作，应等待命令或任务执行完毕。

发布时请勿使用 `--retries` 参数，因为自动重试可能会导致重复发布。此外，Instagram 还会对每个账号的每日发布次数设置上限。

### 设置个人头像
仅需提供一张正方形（1:1）的 JPEG 或 PNG 格式图片，建议分辨率至少为 320×320 像素，最大不超过 1080×1080 像素，文件大小不得超过 100 MB。

```sh
instagram-cli set-profile-picture --account-id <user_own_fbid> \
  --file ~/workspace/<profile>.jpg
```

### 媒体理解
获取一个或多个 Instagram 媒体 FBID 的描述信息，包括内容概要和语义分析。可通过逗号分隔的 `--media-ids` 列表批量查询。

```sh
instagram-cli media-understanding --account-id <user_own_fbid> --media-ids <media_id>
instagram-cli media-understanding --account-id <user_own_fbid> --media-ids <media_id>,<media_id>
```

### 最近点赞的帖子
获取用户最近点赞过的帖子。

```sh
instagram-cli recently-liked-posts --account-id <user_own_fbid>
instagram-cli recently-liked-posts --account-id <user_own_fbid> --limit 5
instagram-cli recently-liked-posts --account-id <user_own_fbid> --limit 10 --after <cursor>
instagram-cli recently-liked-posts --account-id <user_own_fbid> --since 2026-01-01 --until 2026-02-01 --sort-order asc
```

### 最近评论的帖子
获取用户最近评论过的帖子。

```sh
instagram-cli recently-commented-posts --account-id <user_own_fbid>
instagram-cli recently-commented-posts --account-id <user_own_fbid> --limit 5
instagram-cli recently-commented-posts --account-id <user_own_fbid> --limit 10 --after <cursor>
instagram-cli recently-commented-posts --account-id <user_own_fbid> --since 2026-01-01 --until 2026-02-01 --sort-order asc
```

### 账号洞察
在报告、解释或改写 Instagram 指标时，请参考 [账号洞察指标](references/account-insights-metrics.md) 中的相关定义。当当前上下文中缺少确切定义时，可使用 `muse.read` 查阅这些定义，包括在后续跟进时。先前的摘要不能替代这些定义。这同样适用于基于所提供数值生成的文案。除非请求中需要补充或刷新账号数据，否则请直接使用提供的数值。

`account-insights` 返回的是账号级别的汇总数据。默认时间范围为过去 30 天，也可通过 `--period last_7_days`、`last_30_days`、`last_90_days`、`this_month` 或 `this_year` 来选择其他时间范围，或者通过 `--start-date` 和 `--end-date` 自定义日期范围。成对出现的 `--start-time` 和 `--end-time` 参数以 Unix 秒为单位指定时间边界，优先级高于时间段和日期；支持的时间段优先于日期。仅指定日期的边界将采用 UTC 凌晨 0 点。呈现数据时，请使用一种支持的格式并明确说明其时间窗口，但响应结果中不包含该时间窗口。

某些指标可能缺失或为 `null`。否则，指标将以包含 `value` 字段的条目列表形式返回。如果存在 `dimension_values`，请先根据指标定义解读其中的代码，再报告细分数据。

请按 Instagram 的官方名称报告各字段：
- `accounts_reached`（或 `reach`）为“触达账号数”；
- `viewers` 为“观看人数”；
- `content_views` 为“浏览量”；
- `engaged_accounts` 为“互动账号数”；
- `total_interactions` 为“内容互动数”。

报告触达账号数、观看人数或互动账号数时，应明确指出这些数值为估算的账号数量。同一个人可能拥有多个账号。在报告和文案中始终使用“账号”这一单位，即使直接面向受众时亦然，并在包含这些数值的文本中注明报告的时间窗口。

浏览量、互动数、主页访问量和点击量均统计事件次数，应保留这些单位，不要将其总数换算为账号数或人数。当指标定义中明确说明为估算值时，应在报告中加以标注；其他指标则无需添加此类说明。

若收到的文案使用了错误的单位，请在最终发布文本中予以修正。单独的备注无法纠正误导性的文案标题。

## 输出
CLI 将 JSON 数据输出至标准输出。发布相关操作会使用上述标准化的集合格式，其他命令则保留其供应商特定的解码后 JSON 数据。向用户展示结果时，应聚焦于有意义的内容（如名称、文字、图片或视频、链接、文案）。

在提出问题或提供更多帮助之前，请先在回复中包含已找到的信息。切勿仅以提问、提供帮助或承诺后续工作作为结束。

切勿向用户暴露 FBID、不透明 ID、游标、Unix 时间戳或实现相关的术语。应使用用户名、显示名和通俗易懂的描述。仅在工具调用中保留 ID。