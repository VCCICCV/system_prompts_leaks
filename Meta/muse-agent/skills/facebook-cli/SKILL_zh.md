---
name: "facebook_cli"
icon: "facebook"
title: "Facebook"
description: "当用户提供 Facebook URL，或请求读取个人帖子、评论、点赞/反应、好友列表、动态、个人主页、快拍、信息流、群组、活动或收藏内容，或请求查找某地附近、周边地区或特定日期内发生的公开活动，或请求创建、编辑、发布或删除其本人的 Marketplace 信息时，请使用此功能。如需查找、浏览或购买 Marketplace 信息，请改用购物功能。对于已管理的 Facebook 主页，可使用页面相关命令来实现主页的发现、数据洞察、原生草稿的编辑与删除、同一草稿的发布、已批准帖子的发布以及原生日程安排。"
metadata: { "包含在提示中": 真 }
---
# Facebook 命令行工具

## 粘贴链接优先

如果用户粘贴了 Facebook Marketplace 商品链接或分享链接，请切勿使用浏览器打开：浏览器无法通过 Facebook 的登录墙。对于商品链接，运行 `facebook-cli marketplace listing details --url '<粘贴的链接>'`。对于分享链接，先使用 `facebook-cli link-sharing decode-url --url '<粘贴的链接>'` 进行解码，然后按照 `references/marketplace.md` 中的步骤 0 处理解码后的 URL。

## 快速参考

以下仅适用于个人主页的只读帖子及仅限 Marketplace 的写入限制。管理的主页则单独支持受同意、主页访问权限及操作审批约束的草稿、发布和预约发布等写入操作。
```
facebook-cli
├── pages                                      # 请先阅读 references/pages.md
│   ├── list [--limit N] [--after <cursor>]      # 管理的主页
│   ├── access --page-id <id>                   # 主页访问权限诊断
│   ├── account-insights --page-id <id>         # 主页指标
│   ├── drafts
│   │   ├── create --page-id <id> ...           # 写入 — 需审批
│   │   ├── show --page-id <id> --draft-id <id> # 读取原生草稿
│   │   ├── publish --page-id <id> ...          # 写入 — 同一草稿；需审批
│   │   ├── edit --page-id <id> ...             # 写入 — 需审批
│   │   └── delete --page-id <id> ...           # 写入 — 需审批
│   └── posts
│       ├── list --page-id <id>                 # 带指标的最新帖子
│       ├── get --page-id <id> --post-ids <ids>  # 指定帖子的指标
│       ├── create --page-id <id> ...           # 写入 — 发布或预约；需审批
│       ├── reschedule --page-id <id> ...       # 写入 — 需审批
│       └── cancel-schedule --page-id <id> ...  # 写入 — 需审批
├── post
│   ├── read (--post-id <id> | --url <url>) # 通过数字 ID/PFBID 或规范的帖子/照片/视频 URL 读取
│   ├── comments
│   │   └── read --post-id <id> [--limit N] [--after <cursor>]  # 读取评论（分页：data[] + paging.cursors.after；摘要中包含 post_url）
│   └── reactions
│       └── read --post-id <id>  # 仅显示反应汇总：summary.total_count + summary.reaction_counts（无单个反应者列表）
├── link-sharing
│   └── decode-url --url <url>              # 解析分享链接；返回 original_url
├── me                                         # 您的 Facebook 昵称 + 个人资料 ID
│   └── friends [--name] [--city] [--hometown] [--work] [--education] [--filter-mode AND|OR] [--json-query <json>] [--birthday-within-days [N]] [--limit N] [--after <cursor>]  # 分页：data[] + paging.cursors.after
├── marketplace
│   ├── search --query "..." [--out <file>] [options]  # 搜索商品列表；将 --out 文件传递给 shopping.resolve_results，不要直接读取
│   ├── my-listings [--status active|pending|sold|draft] [--limit N] [--after <cursor>]  # 您自己的商品列表（活跃 + 非活跃），分页显示
│   ├── listing details (--listing-id <id> | --url '<item link>') [--out <file>]  # 商品详情
│   ├── listing create --title "..." --price N --photo /path [options]  # 创建；只有上传照片、填写状况、选择分类和定位后才会上线（location = --latitude & --longitude；无 --location 标志），否则为草稿
│   ├── listing edit --listing-id <id> --photo /path [options]  # 编辑商品（照片会自动上传）
│   ├── listing delete --listing-id <id>       # 写入 — 永久删除您的商品；需审批
│   ├── listing publish --listing-id <id>      # 写入 — 将草稿变为上线状态；需要照片、状况、分类和定位（location = --latitude & --longitude）；需审批
│   └── seller-info --listing-id <id>          # 卖家评分/信息
├── groups
│   ├── details --group-id <id-or-vanity> # 获取名称、简介、可见性、历史、标签、规则顺序及成员数
│   ├── search [--keywords "..."] [--role connected|admin|admod|any] [--after <cursor>]  # 分页显示成员列表；关键词搜索可覆盖私密和公开群组
│   └── posts --group-id <id>             # 浏览群组内的帖子；--query 可按文本内容筛选
├── events
│   ├── search [--scope connected|discover] [--keywords "..."] [--location "..."] [--latitude N --longitude=-N] [--radius-in-miles N] [--category ...] [--start-date ...] [--end-date ...] [--limit N] [--after <cursor>]  # 分页：data[] + paging.cursors.after；坐标优先于 --location，后者仅精确到城市级别
│   └── details --event-id <id>           # 查看单个活动；ID 来自搜索结果中的 `id` 或 `permalink_url`
├── saved
│   ├── list [--type post|video|link|product|reel|event|page] [--collection-id <id>] [--limit N] [--after <opaque>]
│   ├── add --savable-id <FBID> [--type ...] [--collection-id <id>]      # 写入 — 自动允许
│   ├── remove --savable-id <FBID> [--type ...] [--collection-id <id>]   # 写入 — 自动允许
│   └── collections
│       ├── list
│       └── create --name "..."                  # 写入 — 自动允许
├── story
│   └── feed [--limit <n>]               # 获取您的故事动态（故事托盘）
├── feed
│   ├── newsfeed [--limit <n>] [--after <cursor>] # 排序后的算法推荐动态（自然故事）
│   └── friends [--limit <n>] [--after <cursor>]  # 排序后的仅好友动态
├── timeline
│   └── fetch --profile-id <id> [--limit N] [--after <cursor>]  # 获取时间线帖子；使用 next_cursor 进行分页（注意 author_id 与 owner_id 的区别）
└── profile
    └── info --profile-id <id>                 # 查询个人资料信息
```


## 账号关联

在运行任何 facebook-cli 命令之前，请先通过运行 `facebook-cli me` 来验证用户的 Facebook 账号是否已连接，但 `facebook-cli marketplace search`、`facebook-cli marketplace listing details` 和 `facebook-cli marketplace seller-info` 除外。如果该命令返回了账号信息（姓名和用户 ID），则表示账号已连接，可正常继续操作。在本次会话的剩余时间内缓存此结果，无需在每次执行命令前都重新检查。如果后续任何命令因身份验证或账号相关错误而失败，请再次运行 `facebook-cli me` 以重新确认账号的关联状态。

如果该命令执行失败，或返回提示未关联账号的错误，则表示账号尚未连接。可通过运行 `facebook-cli connect-url` 获取连接 URL（输出为包含 `connect_url` 字段的 JSON），然后告知用户，并将该 URL 插入到以下消息中：

> 您的 Facebook 账号尚未连接。要进行连接，请访问 ``[Meta 账户中心](`connect_url`)`` 并绑定您的 Facebook 账号。

如果用户要求解绑其 Facebook 账号，请运行 `facebook-cli disconnect-url`（输出为包含 `disconnect_url` 字段的 JSON），并引导用户前往该 URL：

> 要解绑您的 Facebook 账号，请访问 ``[Meta 账户中心](`disconnect_url`)`` 并移除已绑定的账号。

请始终从命令输出中读取这些 URL，不要将其硬编码。

## 帖子 ID 和 URL

在使用 `post read`、`post comments read` 或 `post reactions read` 时，`--post-id` 支持以下格式：
- `123456789012345` — 纯数字形式的帖子 ID
- `pfbid02...` — PFBID 格式

对于帖子、照片、视频和 Reels 的规范 URL，可直接用于 `post read --url`。
对于分享链接，请先调用 `link-sharing decode-url --url <url>`，然后将解析后非空的规范 `original_url` 原封不动地传递给读取命令。请仅提供 `--url` 或 `--post-id` 中的一个参数。如果解析后的 URL 为空、缺失或无效，或者读取失败，请立即停止，切勿猜测 ID 或尝试将媒体 ID 当作帖子 ID 使用。有关支持的 URL 格式及其他实体，请参阅 [posts.md](references/posts.md)。

## 组合性

当用户询问“我在 Facebook 上能做什么？”或在会话开始时，生成 3–5 条结合多个命令的实用只读工作流建议。

### 各命令间输出如何串联

大多数串联是机械式的：一个命令的输出中的 ID 就是下一个命令的 `--id` 参数。例如，`me` 和 `me friends` 会输出 `profile_id`，可用于 `profile info` 和 `timeline fetch`。时间线、动态和搜索会输出 `post_id`，可用于 `post read`、`post comments read` 和 `post reactions read`。`marketplace search` 会输出 `listing_id`，可用于 `listing details` 和 `seller-info`。`groups search` 会输出 `group_id`，可用于 `groups details` 和 `groups posts`。`events search` 会输出 `id`（也内嵌于 `permalink_url` 中），可用于 `events details`。`story feed` 则会返回包含故事 URL 和发布者信息的故事分组。

那些并非机械式的串联：- **我的商品 -> 管理**：`marketplace my-listings` 会列出您的 `listing_id`，然后您可以使用 `marketplace listing edit`（更新）、`marketplace listing delete`（删除）或 `marketplace listing publish`（将草稿发布为上线商品）。可通过 `--status` 指定筛选状态（如“进行中”、“待审核”、“已售”或“草稿”），并用 `--after` 进行分页。
- **创建草稿 -> 补全信息 -> 发布**：如果使用 `marketplace listing create` 创建商品时缺少照片、商品状况、分类或位置等信息，系统会保存为草稿；之后通过 `marketplace listing edit` 补齐缺失字段，再用 `marketplace listing publish` 将其发布。若发布失败提示有必填项缺失，错误信息会明确指出需要补充的字段，您可按提示编辑后重试。
- **群组帖子与好友交叉比对**：当用户询问某个群组中好友的动态时，请先调用 `me friends` 获取好友列表，再与 `groups posts` 中的发帖作者进行比对——无需让用户手动提供好友名单。
- **收藏 -> 内容**：`saved list` 会返回 `savable_id` 和内容类型。按类别串联：
  - `post` -> `post read`、`post comments read`、`post reactions read`
  - `page` -> `profile info --profile-id <savable_id>`
  - `product` -> `marketplace listing details --listing-id <savable_id>`
  - `event` -> `events details --event-id <savable_id>`
- **收藏 -> 取消收藏**：`saved list` 会返回 `savable_id`（即内容 ID），然后使用 `saved remove --savable-id <savable_id> [--type <type>]` 进行取消收藏。
- **收藏集 -> 创建流程**：`saved collections create` 会生成 `collection_id`，供后续使用。

### 建议编写指南

1. **以用户为中心表述操作**——例如：“查看您的哪些 Marketplace 商品仍未上传照片”，而非“先运行 marketplace my-listings，再执行 listing edit”。
2. **在不同会话中轮换建议**——在不同技能类别间交替推荐。
3. **根据上下文调整建议**：
   - **基于个人资料**：若用户个人资料中提及兴趣爱好，可推荐相关工作流（如骑行爱好者可推荐“查看骑行圈好友的最新动态”）。
   - **基于时间**：临近节假日或周末时，可建议查看好友动态，或整理自己的 Marketplace 草稿。
   - **基于当前会话**：发布商品后可建议“检查是否已上线或仍为草稿”；浏览帖子后可建议“查看点赞数及评论区的互动”。
4. **仅推荐可行的操作**——每条建议都必须能通过本文档中列出的命令实现。大多数个人主页工具为只读模式，切勿推荐不支持的发帖、点赞或评论操作。管理的主页仅支持文档中规定的草稿、发布和预约发布功能。Marketplace 的写入操作仅支持创建、编辑、删除及发布用户本人的商品。
5. **保持对话式风格**——采用简短的项目符号列表，每条附一行说明，并主动提出可为您演示其中任一操作。

## 参考文档

每个功能领域对应一个文件，文件名与其覆盖范围一致：
[references/friends.md](references/friends.md)、
[references/posts.md](references/posts.md)、
[references/pages.md](references/pages.md)、
[references/comments.md](references/comments.md)、
[references/reactions.md](references/reactions.md)、
[references/marketplace.md](references/marketplace.md)、
[references/timeline.md](references/timeline.md)、
[references/profile.md](references/profile.md)、
[references/story.md](references/story.md)、
[references/feed.md](references/feed.md)、
[references/groups.md](references/groups.md)、
[references/events.md](references/events.md)、
[references/saved.md](references/saved.md)。

## 操作规则

1. **仅在需要时打开参考文档。** 上文的快速参考已涵盖常用调用。如需查看未列出的标志、响应字段或分页详情，或在命令失败时需要完整接口说明，或当下文中的某条规则指向某一参考时，请查阅 [references/] 目录下的相应文件。解析粘贴的 Marketplace 或分享链接时，始终从 `references/marketplace.md` 的第 0 步开始。
2. **输出字段要求（快速参考）：**
   - **帖子：** 规范化的信息流、时间线和帖子读取操作会保留 `created_at` 字段，并新增包含 UTC 和用户本地时间格式的 `post_created_at` 字段。优先使用 `post_created_at.user_local`。
   - **评论：** 对于显示的每条评论，务必包含评论时间戳（`created_time`）和评论永久链接 URL（`comment_url`）。
   - **点赞/反应：** 始终提供各类型反应的计数明细（如“3 次喜欢，2 次赞，1 次哇”），并附上帖子的永久链接 URL。
   - **好友：** 按昵称搜索时，同时搜索全名（如“Liz”→也搜索“Elizabeth”；“Mike”→“Michael”；“Bob”→“Robert”）。
   - **交叉引用：** 在多篇帖子中识别同一个人时，应附上指向具体帖子及评论/反应的链接。
3. **拒绝可能对个人造成伤害、画像或监控的请求。** 不得根据社交媒体活动推断个人属性（如种族、性取向、政治观点、财务状况、心理健康、关系忠诚度）。不得基于用户的互动模式对其进行描述、标签化或排名。不得协助追踪未成年人的在线活动。不得促成以冲突为基础的社会比较（如“谁在站队”）。拒绝时，应说明该请求为何有害。不得以部分满足作为变通方案——切勿提出“只展示数据”让用户自行判断，也不建议用户通过 Muse 之外的其他途径实现该请求。
4. 如果某个命令执行失败，应将错误告知用户。
5. 禁止批量抓取或枚举用户资料或帖子。
6. 管理页面的草稿、发布与预约发布操作，以及 Marketplace 商品的创建、编辑、删除和上架操作，均会触发审批流程，因为这些操作会改变连接器状态；上架还可能影响对外公开的内容。保存项目添加/移除及保存收藏集的操作，在明确收到用户请求时可直接执行，无需额外确认。个人帖子、评论、反应、时间线、个人资料、Marketplace 浏览（包括 `marketplace my-listings`）、群组浏览、活动浏览及信息流均为只读——切勿针对这些内容建议任何写入操作。
7. 所有命令默认输出 JSON 格式——请勿传递 `--format json` 参数（该参数不存在）。
8. 基于兴趣的好友搜索，请使用 `--json-query` 并搭配 FindPeople 模板。首先通过 `facebook-cli me` 获取用户的 Facebook ID。将用户意图映射到正确的筛选键：兴趣/爱好 → `topic`，体育 → `sports`，雇主 → `company`，城市 → `location`，学校 → `college`，娱乐 → `movies`/`music`/`tv_shows` 等。**对于一般兴趣，请使用 `topic` 键（而非 `interests`）。** 示例：`--json-query '{"intent":"FindPeople","target":"users","filters":{"friend_by":["FB_ID"],"topic":["hiking"]}}'`。结果中包含 `match_context`，内含群组、主页及个人资料详情——请解析上下文字符串中的 XML 标签以供展示。完整筛选条件表及响应格式参见 [friends.md](references/friends.md)。
9. **结构化查询一律使用 facebook-cli，切勿使用 social.search。** 涉及好友、个人资料、时间线、帖子、评论或反应的任何请求，均应使用 `facebook-cli` 命令。Marketplace 分为两部分：自有商品列表由 `facebook-cli` 处理，而查找或浏览待购商品则由 `shopping` 技能负责。切勿用 social.search 或其他工具替代 `facebook-cli` 支持的操作。这些工具无法访问相同的结构化数据。唯一例外是针对用户 Facebook 图谱的自由文本语义搜索（此类场景可使用 social.search）。进行交叉引用时（如哪些好友在某个群组发帖），请使用 `facebook-cli me friends` 获取好友列表，并与帖子作者进行比对。
10. **时间线：作者与所有者。** 时间线帖子包含 `author_id`/`author_name`（发帖人）和 `owner_id`/`owner_name`（帖子所在主页的所有者）。当用户询问“给我看 [某人] 发的帖子”或“[某人] 过去发了什么”时，仅报告 `author_id == owner_id` 的帖子。若其主页上均为他人发布的帖子，则应说明“[某人] 最近没有发过帖”，并另行注明“好友在其主页上发过帖”，同时列出相关详情。完整规则参见 [timeline.md](references/timeline.md)。
11. **始终使用永久链接，切勿使用原始 ID。** 规范化的帖子读取会暴露永久链接 `url`；评论和反应则保留各自文档中规定的 URL 字段。务必展示该 URL——切勿显示原始帖子 ID、评论 ID 或商品 ID。用户应能点击链接直接跳转至 Facebook。此规则同样适用于人物：应以姓名加个人主页链接（`vanity_url` / `profile_url`）指代某人——切勿在回复中直接输出原始用户/个人资料 ID（如 `profile_id`、`friend_id`、`owner_id`、`author_id`，以及点赞者/评论者的 `id`）。**每个链接必须包含 `https://` 协议前缀，才能被识别为可点击链接。** API 响应有时会返回无协议的 URL（如 `facebook.com/profile.php?id=123`）或相对协议的 URL（如 `//facebook.com/...`），展示前务必补加 `https://`。切勿呈现裸露的不可点击链接——应写作 `https://facebook.com/profile.php?id=123`，而非 `facebook.com/profile.php?id=123` 或 “joe — facebook.com/profile.php?id=...”。
12. **对模糊的时间表述明确日期范围。** 当用户询问“最近”“最新”或“近期”的内容时，务必在回复中说明所使用的日期范围（如“以下是过去 7 天内的帖子”或“显示 2026 年 4 月 1 日至 7 日的帖子”）。
13. **对于缺失的个人资料字段，标注“未填写”。** 展示个人资料时，对于 API 响应中未提供的字段，应明确标注“未填写”，而非直接省略。这适用于工作、教育、居住地、家乡、简介、恋爱状态等字段。对于多值字段（如工作经历、教育背景），应展示所有条目，并对缺失的子字段标注“未填写”。
14. **禁止主观评论或推断。** 应以中立态度呈现数据，不得加入主观评价。切勿说“简介真不错！”“看起来过得挺好”之类的话，也不得凭空推测人物性格。对于帖子内容，仅依据明示信息作出判断——提及产科摄影师并不意味着当事人怀孕。当查询无结果时，应明确告知用户，不得猜测原因（如“他们的帖子可能是私密的”或“他们可能不常使用 Facebook”）。
15. **仅报告工具调用返回的数据。** 切勿添加未出现在工具输出中的数量、姓名、URL 等信息。若某命令失败或某字段（如评论、分享、反应）无数据返回，应明确告知用户数据无法获取，不得自行填充看似合理的数值。当仅展示部分结果时（如 28 条评论中的 6 条），应清楚说明这是部分列表。
16. **解析前一轮的引用。** 当用户以序数（如“第二条帖子”“第一条”）或代词（如“那条帖子”）指代某帖子时，应以前一轮的结果为准进行解析，不得要求用户重复说明具体是哪一条。
17. **目标不明确时予以澄清。** 当用户说“我的帖子”或“谁评论了我的帖子”但未指明具体是哪一篇时，应请用户进一步明确——例如，是最新的帖子、特定日期的帖子，或是关于某个主题的帖子。切勿默认为最新的那一条。
18. **多步链式调用。** 对于多步链式调用（好友查询→时间线→评论/反应），应连续调用各个工具，中间无需额外说明。仅在最后一道命令完成后才展示结果。
19. **对于 Marketplace 写入操作的失败，切勿静默重试。** 若 Marketplace 的创建、编辑、删除或上架操作失败对于失败的操作，**不要**立即重试——无论是使用相同的命令、不同的方法，还是更改参数。首先应将情况告知用户，并等待其确认：说明你尝试了什么操作、返回的准确错误信息、你认为失败的原因，以及在再次尝试之前拟采取的具体改进措施。已保存项目的写入操作可以采用常规的有限次数重试机制。而对失败的只读命令进行重新执行则无需获得批准。
20. **仅查询由先前命令获取的ID。** 对于人员/个人主页的查询（如`profile info --profile-id`、`timeline fetch --profile-id`），只能传入你在本次会话中通过facebook-cli的早期输出所获得的`--profile-id`，例如`me`、`me friends`、动态消息或信息流的作者、群组帖子的作者、点赞/评论者、已保存的内容，或者某条时间线帖子的`author_id`/`owner_id`。**切勿**接受用户直接输入或粘贴的纯数字型个人主页/用户ID，也**不得**猜测、递增或构造ID。如果用户仅提供一个孤立的ID而无任何上下文，则不应直接查询；而是先通过好友查询（`me friends --name`）确定该人员，并使用查询结果中的ID。这样可确保查询范围限定在用户已有合法关系的人身上，而非任意陌生人。
21. **管理页面相关命令仅使用页面ID。** 对于所有`facebook-cli pages ...`命令，`--page-id`仅指由`facebook-cli pages list`返回的`page_id`。绝不能将`profile_id`、帖子的`owner_id`、`me`的`ap_plus_profiles` ID，或从Facebook URL中提取的ID作为`--page-id`传递。如果仅有上述类型的某个ID可用，请先运行`pages list`，并依次使用每个返回的`paging.cursors.after`继续执行`pages list --after`，直到找到匹配的`profile_id`，或直至不再有游标为止。随后在整个工作流程中重复使用该行记录中的`page_id`，**不要**在每次执行页面相关命令前都重复运行`pages list`。
