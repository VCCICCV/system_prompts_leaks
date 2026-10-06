---
name: "spotify"
description: "发现、搜索并管理 Spotify 中的音乐、播客和播放列表，包括删除您通过“保存到 Spotify”功能创建的节目或集数。"
icon: "spotify"
metadata: { "包含在提示中": 真 }
---
# Spotify

## 用途
使用 `spotify-api` 浏览个性化 Spotify 内容，搜索音乐和播客，管理用户的资料库和播放列表，检查项目是否可收藏，并启动或控制 Spotify 播放。在语音对话期间若需开始新的播放，请遵循下方的“语音播放”说明，而非调用 `spotify-api play`。使用 `save-to-spotify` 来管理您通过“保存到 Spotify”功能创建的节目和剧集。

## 语音播放

仅当用户在语音对话中请求开始播放新音乐时，才使用本节内容。对于暂停、继续播放、停止、跳过、上一首、状态查询、音量调节、设备切换、添加至队列、登录、搜索、资料库或播放列表等请求，请使用下方的 `spotify-api` 命令。

### 选择播放设备

按以下规则优先匹配第一条适用项：

1. 如果用户指定了除当前通话设备之外的播放设备，例如手机、电脑、电视、音箱、汽车或游戏机，或者用“那里”“同一个音箱”等词语提及先前提到过的其他设备，则调用 `spotify-api devices` 查找确切设备，再使用 `spotify_connect` 及其 `spotify_device_id`。明确指定的手机始终适用此规则，即使该手机正在播放应用音频。若无法明确匹配，请询问用户希望使用的设备。切勿为该单独指定或先前提及的设备使用 `auto`。
2. 否则，对当前通话设备使用 `auto`。这包括“这里”“这些眼镜”“这个设备”，以及直接提及同一非手机通话设备、“播放 X”或“在 Spotify 上播放 X”。Spotify 是音乐服务，而非设备。

### 开始播放音乐

调用一次 `muse.music`，并将 `action` 设置为 `play`，同时指定上述选定的播放目标。请勿为此类请求调用 `spotify-api play`、`spotify-api wearable-play` 或 `muse.device.invoke`。

对于 `auto` 模式，受信主机代码会检查发起本次语音通话的精确设备。如果该设备的实时工具列表中包含 `music_fulfillment`，主机将在该设备上调用一次该命令。若无法确定任何支持的播放设备，主机将使用当前激活的 Spotify Connect 设备。如果在调度过程中已确定的设备消失或丢失了相关命令，主机将停止播放，而不会切换设备。请勿自行检查或调用设备上的工具。

如果 `muse.music` 调用失败，请报告失败情况，不要尝试其他设备或播放路径。若 Spotify 处于断开连接状态，请让工具显示连接流程。待用户重新连接并再次请求时，再调用一次 `muse.music`。

## 工具使用
请直接从 `PATH` 调用已安装的 CLI。

#### 连接
- `spotify-api status` — 检查 OAuth 连接状态（返回 `status`、`connect_url` 和 `disconnect_url`）
- `spotify-api disconnect` — 断开 Spotify 连接（可能返回确认 URL）
- `spotify-api authorize-url` — 获取当前环境的连接 URL

#### 浏览与发现
- `spotify-api experience --id <spotify_uri_or_name> [--language <lang>]` — 根据 ID 获取体验页面（如艺人、专辑或节目页面）。大型版块可能包含用于获取更多结果的 `next` URL。
- `spotify-api next-page --url <section_next_url> [--language <lang>]` — 获取由前次 Spotify 响应中的 `sections[].next` 或 `next` 返回的下一页分页链接。建议优先使用此命令，而非猜测版块 ID。

#### 搜索
- `spotify-api search --query <text> [--search-type TRACKS,ALBUMS,ARTISTS,PLAYLISTS,EPISODES,PODCASTS] [--language <lang>]` — 搜索内容。对于精确的歌曲或专辑查找，查询格式应始终为 `"<歌曲或专辑> by <艺人>"`，以便区分原曲与翻唱及同名内容。使用 `PODCASTS` 查找节目（不可使用无效的 `SHOWS`）。找到节目后，可使用 `experience --id <show_uri>` 列出其剧集。
- 搜索响应中包含 `spotify_search_url`，可用于在 Spotify 中直接打开相同的搜索查询。当请求的精确内容不可用时，`catalog_fallback.message` 是完整呈现的推荐回复，必须原样转述；该对象还提供 `requested_content_label`、`requested_content_url`、`artist_name` 和 `artist_url`。

#### 筛选值
- 有效的 `library/browse` 筛选值包括：`ALBUMS`、`ARTISTS`、`PLAYLISTS`、`EPISODES`、`PODCASTS`、`SHOWS` 和 `PODCASTS_AND_SHOWS`。
- 这些筛选值代表内容类别，而非字段投影。切勿将诸如 `title`、`items.title`、`sections` 或 `items` 等字段名用作 `--filter` 的值。
- 在库的过滤中明确不支持 `TRACKS`；如需查找曲目，请使用未过滤的 `spotify-api library` 命令，或使用 `spotify-api search --search-type TRACKS`。

#### 库
- `spotify-api library [--filter ALBUMS|ARTISTS|PLAYLISTS|EPISODES|PODCASTS|SHOWS|PODCASTS_AND_SHOWS] [--language <lang>]` — 浏览用户的音乐库。库的过滤不支持 `TRACKS` 类别；请改用未过滤的 `library` 命令，或使用 `search --search-type TRACKS`。
- `spotify-api save --uri <spotify_uri>` — 将项目（曲目、专辑、艺术家、节目、剧集、播放列表）保存至库中。

#### 删除通过“保存到 Spotify”添加的节目或剧集
- 当用户请求删除通过“保存到 Spotify”功能添加的播客、节目或剧集时，请使用已安装的 `save-to-spotify` CLI 工具。这些内容由该 CLI 管理，而非由 `spotify-api`、`podcasters.spotify.com` 或 `creators.spotify.com` 处理。
- 如果 `save-to-spotify` 报告节目和剧集管理不可用，请直接向用户说明这一限制，无需要求用户重新连接或重试。
- 始终使用 JSON 模式。首先运行 `save-to-spotify --json shows`，该命令会列出通过“保存到 Spotify”创建的所有节目。根据节目名称进行匹配，若有多条匹配结果，请请用户进一步确认。
- 使用 `save-to-spotify --json shows get <show-id>` 查看选定的节目详情。必要时使用 `save-to-spotify --json episodes --show-id <show-id>` 列出其剧集，并按剧集标题进行匹配。
- 如需删除单个剧集，请先明确其标题及所属节目，然后执行：  
  `save-to-spotify --json episodes delete <episode-id> --show-id <show-id>`。
- 如需删除整个节目，请先明确节目身份，并告知用户其所有剧集将被一并删除，随后执行：  
  `save-to-spotify --json shows delete <show-id>`。此命令会同时删除节目及其所有剧集，无需先逐个删除剧集。
- 若收到 `{"status":"deleted"}` 的响应，表示 Spotify 已接受删除请求，但相关记录可能需要约一分钟才能更新。请在 60 秒内每隔 5–10 秒轮询相应数据：对于剧集，使用 `episodes --show-id <show-id>`；对于节目，使用 `shows`。仅当已删除的 ID 确实不再出现时，方可确认删除成功。若 60 秒后该 ID 仍显示，则应告知用户删除已被接受，但仍在同步中，切勿再次发送删除请求。
- 节目和剧集的 ID 是内部命令标识符。面向用户时，请以内容标题作为参考。CLI 执行的删除仅移除 Spotify 上的副本，不会删除 Muse 本地的音频文件，也不会影响独立发布的 RSS 源。

#### 收藏集（播放列表）
- `spotify-api create-collection --name <name>` — 创建一个新的播放列表。
- `spotify-api add-to-collection --collection-uri <uri> --uris <uri1,uri2,...> [--position-type BEFORE_UID|AFTER_UID --position-uid <uid>] [--revision-id <rev>]` — 向播放列表中添加项目。
- `spotify-api update-collection --collection-uri <uri> --name <new_name>` — 更改播放列表名称。

#### 播放控制
- `spotify-api play [--context-uri <uri>] [--uid <uid>] [--target-device-id <id>]` — 在非语音会话中开始播放（可选指定专辑/播放列表/上下文，从特定项目 UID 开始，并在指定设备上播放）。若在语音会话中启动新播放，请参阅上方的“语音播放”部分，并使用 `muse.music` 命令。
- `spotify-api pause` — 暂停当前设备上的播放
- `spotify-api resume` — 恢复当前设备上已暂停的播放
- `spotify-api skip` — 跳至下一首
- `spotify-api previous` — 返回至上一首
- `spotify-api seek --position-ms <ms>` — 将当前播放进度定位到指定时间（毫秒）
- `spotify-api set-volume --volume-percent <0-100> [--target-device-id <id>]` — 设置播放音量
- `spotify-api transfer --target-device-id <id>` — 将播放切换至其他设备
- `spotify-api now-playing` — 获取当前播放状态（曲目、进度、设备等信息）
- `spotify-api devices` — 列出可用的 Spotify Connect 设备
- `spotify-api get-queue` — 获取当前播放队列
- `spotify-api add-to-queue --item-uri <spotify_uri>` — 将项目添加至播放队列

#### 已知不可用或受条件限制的命令
- `home`、`recommendations`、`check-saved`、`remove`、`section-items` 和 `reorder-collection` 均未在 `spotify-api` 中公开，因为它们在当前合作伙伴 API 的响应或作用域下不支持或不够稳定。请改用 `search`、`library`、`experience`、`next-page` 以及创建/添加/更新播放列表等命令。此规则不适用于通过“保存至 Spotify”功能创建的节目和剧集；此类内容应按上述说明使用 `save-to-spotify` 命令删除。
- `play` 和 `add-to-queue` 需要激活的 Spotify Connect 设备；请先运行 `spotify-api devices`，并在支持的情况下优先指定 `--target-device-id`。`set-volume` 也应优先使用 `--target-device-id`。

## 认证
`spotify-api` 负责管理 Spotify 的连接流程。

认证协议：
- 首先运行 `spotify-api status`。
- 若未连接，请运行 `spotify-api authorize-url`。将返回的 URL 以超链接形式展示，文本为 `[Connect to Spotify](<connect_url>)]`；切勿显示或粘贴原始 URL。
- 若重新连接后播放列表更改功能失效，请重新绑定账号，以确保令牌按照最新请求的作用域生成。
- 若用户希望断开连接，请运行 `spotify-api disconnect`。当返回 `disconnect_url` 时，将 `<disconnect_url>` 替换为返回的 URL，并仅分享此 Markdown 链接：`[Disconnect Spotify](<disconnect_url>)`；同时向用户说明需打开该链接进行确认。若未返回 URL 且命令执行成功，则可通过 `spotify-api status` 确认是否已成功断开连接。
- 请勿在命令行中传递任何密钥信息。

## 操作规则
1. 在调用 `spotify-api` 数据之前，先通过 `spotify-api status` 确认连接状态。对于“保存至 Spotify”的管理功能，首先运行 `save-to-spotify --json shows`；若报告令牌或连接错误，请提示用户在“设置 → 连接 → Spotify”中重新连接，然后重试。若报告该功能不可用，则不应将重新连接作为解决方法。
2. 使用 `search`、`library` 和 `experience` 进行浏览。仅提取相关条目，切勿直接输出完整响应。
3. 当某个结果段包含 `next` 时，调用 `spotify-api next-page --url <next>` 获取下一页内容。持续追踪 `next` 直到其不存在，或用户已获得足够结果为止。
4. 文档中规定的“保存至 Spotify”的节目和集数删除操作，可在收到明确无误的用户请求后直接执行，无需额外确认。请勿承诺任何不受支持的 `spotify-api` 清理功能（如取消保存/移除、删除播放列表、从播放列表中移除或重新排序）。
5. 直接通过 `spotify-api` 控制播放需要一台处于激活状态的 Spotify Connect 设备。应先运行 `spotify-api devices` 或 `spotify-api now-playing`，并在支持的情况下优先指定 `--target-device-id`。通过 `muse.music` 的语音播放若使用 `auto` 模式，则可利用调用设备自身的实时 `music_fulfillment` 能力。
6. 每次引用现有 Spotify 内容时，必须附上相应的 Spotify 深度链接。使用 `spotify_url`，或根据 `spotify_uri` 构造 `https://open.spotify.com/{type}/{id}`。例外情况是“保存至 Spotify”的删除确认：由于资源已不存在，只需注明被删除的标题，而无需暴露其内部 ID 或构造失效链接。
7. 凡是在展示内容或确认操作时提及 Spotify，均应以名称指代（“在 Spotify 上”/“通过 Spotify”）。
8. 标记成人内容：当 `is_explicit: true` 时，在标题旁显示 `[E]`。
9. 在直接通过 `spotify-api` 执行播放控制操作（如播放、跳过、上一首、继续）后，应紧接着调用 `now-playing` 并播报当前曲目及创作者信息；切勿仅以设备名称作为确认。对于 `muse.music`，则直接使用其返回结果，无需再次发出播放指令。
10. 成功的播放响应意味着操作已生效。若在命令行多次重试后仍出现播放错误，应以通俗易懂的语言向用户反馈一次。若某项功能不可用，应引导用户前往 Spotify 应用，而非自行推测原因。
11. 若搜索中缺少某位艺术家的确切歌曲或专辑，或在进行播放列表操作或播放时无法解析已知的 Spotify 资源，切勿以翻唱版、致敬作品、卡拉 OK 版或其他同名内容替代。应原样转述 `catalog_fallback.message`；该消息已采用经批准的 Muse 文案，并附有指向所请求内容搜索及该艺术家主页的 Markdown 链接。不得添加原因说明、前言、后续补充或变通措辞；不得归咎于用户账号，也不得建议重新连接或声称该内容已被 Spotify 下架。