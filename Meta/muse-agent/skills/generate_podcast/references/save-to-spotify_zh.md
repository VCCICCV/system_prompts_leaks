# 将生成的剧集添加到用户的 Spotify

这是 `generate_podcast` 技能“添加到您的 Spotify”功能的参考文档。这是一项**个人**保存操作——用户通过捆绑的 `save-to-spotify` CLI 将自己生成的剧集保存到**自己的** Spotify 资料库/播客中——并非 RSS 源发布。仅在用户明确请求时执行此操作；切勿主动提供。

## 上传时优先使用 `podcast-helper save-to-spotify`
实际保存应通过 `podcast-helper save-to-spotify --slug <slug> [--show-id <id> | --new-show "<title>"] [--title ...] [--summary ...] [--image ...]` 命令完成。该辅助工具会通过下方的 `save-to-spotify` CLI 进行上传，并将生成的 `spotify_show_url` 和 `spotify_show_id` 记录到清单 feed（以及 Postgres 目录）中，以便在播客库页面上显示该播客链接。它会返回 `episode_id`、`episode_uri`、`spotify_show_id` 和 `spotify_show_url`。对于读取、状态查询和删除等步骤，请直接使用原生 CLI。本文档其余部分介绍的是该底层 CLI。

## 工具使用
始终调用 PATH 上的捆绑封装程序；切勿自行安装 Spotify CLI、下载二进制文件或设置其他路径。如果发现 `save-to-spotify` 缺失，应将其报告为 Muse 安装问题。每次调用时都使用 `--json` 参数，以确保结果可被机器解析；该参数为全局标志，请置于子命令之前（`save-to-spotify --json <command>`）。

```sh
save-to-spotify --json <command> [options]
```

可用命令：
- `save-to-spotify --json shows`
- `save-to-spotify --json shows create --title "<title>" --summary "<desc>"`
- `save-to-spotify --json shows get <show-id>`
- `save-to-spotify --json shows delete <show-id>`
- 上传操作仅可通过 `podcast-helper save-to-spotify` 执行；除非该辅助工具已确认并同步其持久化的上传凭证，否则原生封装程序会拒绝此类操作。
- `save-to-spotify --json episodes --show-id <show-id>`
- `save-to-spotify --json episodes status <episode-id> --wait`
- `save-to-spotify --json episodes delete <episode-id> --show-id <show-id>`
- `save-to-spotify --json timeline set --episode-id <id> --from-file <timeline.json>`

`update` 和 `token` 命令已被封装程序禁用，请勿使用。

## 认证（共享 Spotify 连接——在设置中连接）
该工具复用用户现有的 **Spotify** 连接，其后端与 `spotify` 播放技能相同。启用了门控机制的用户使用由 authd 管理的公共 PKCE；现有代理路径仍作为灰度发布期间的回退方案。连接与断开均在**设置 → 连接 → Spotify** 中进行；该工具不会提供连接链接，也不持有任何令牌，且没有 `auth` 子命令。连接状态可根据命令本身推断：若 `shows`（或任何命令）因“请在设置中连接 Spotify”或令牌错误而失败，请告知用户**前往设置 → 连接 → Spotify 进行连接**，然后重试。切勿向用户索取 Spotify 密码、客户端密钥或令牌。

## 上传流程
1. `shows` — 列出现有节目（这同时也会确认 Spotify 已连接；若出现令牌错误，请引导用户前往“设置 → 连接 → Spotify”）。询问是复用现有节目还是创建新节目（`shows create` 或 `upload --new-show`），切勿自动默认选择。
2. 执行 `podcast-helper save-to-spotify ...` — 首先与用户确认标题、节目名称和简介（此操作为对其账户的授权写入）。助手在启动前会获取一份持久性收据；只有当封装层明确报告未发生外部尝试时，才可安全重试确定性的预检拒绝。其他任何失败都会使收据状态不确定，并阻止盲目重试。
3. `episodes status <episode-id> --wait` — 轮询直至状态变为 `READY`。处理过程由服务器端执行，可能需要几分钟。
4. 在状态变为 `READY` 后，可选执行 `timeline set --episode-id <id> --from-file timeline.json`。
5. 告知用户内容已上传至其 Spotify 账户，可能需几分钟才会在应用中显示。使用节目和集数的标题，并以通俗语言说明准备状态（“已就绪”或“仍在处理中”）；切勿引用 CLI 的原始标识符或 JSON 数据。

## 删除流程
通过“保存到 Spotify”功能创建的节目和集数只能通过该 CLI 删除。它们不会在 `podcasters.spotify.com` 或 `creators.spotify.com` 中管理，且 `spotify-api` 也无法删除这些内容。
1. 执行 `save-to-spotify --json shows`，将此视为用户“保存到 Spotify”节目的权威清单。按标题匹配；若有多个节目匹配，请询问用户具体指哪个。
2. 执行 `save-to-spotify --json shows get <show-id>`，验证所选节目的标题和集数。
3. 对于单个集数，执行 `save-to-spotify --json episodes --show-id <show-id>`，匹配请求的标题，确认删除后，再执行  
   `save-to-spotify --json episodes delete <episode-id> --show-id <show-id>`。
4. 对于整个节目，确认节目标题并告知所有集数将被移除，然后执行  
   `save-to-spotify --json shows delete <show-id>`。节目命令会一并删除其下的所有集数，无需先逐个删除各集。
5. 将 `{"status":"deleted"}` 视为删除已被接受，但不等于 Spotify 的列表已同步更新。每隔 5–10 秒轮询一次，最长持续 60 秒：删除集数后查询 `episodes --show-id <show-id>`，删除节目后查询 `shows`。仅当被删除的 ID 不再出现时，才确认删除成功。
6. 若 60 秒后该项目仍列于列表中，请告知用户删除请求已被接受，但可能还需约一分钟才能完全生效，切勿再次发起删除操作。否则应报告被删除的标题，而非内部 ID。同时说明此举仅移除 Spotify 上的副本，不会删除 Muse 本地的音频文件或独立发布的 RSS 源。

支持的音频格式：`.mp3`、`.m4a`、`.wav`、`.ogg`。封面图片：**仅限 JPEG 或 PNG，大小不超过 1 MB** — 其他格式（如 `.webp`）会被拒绝，提示“不支持的图片扩展名”；请先转换格式（例如使用 `ffmpeg -i cover.webp cover.jpg`），并将转换后的文件传入 `--image` 参数。

**就绪状态与重试机制。** 返回的 `episode_uri` 表示 Spotify *已接受*上传；达到 `READY` 状态则是服务器端的另一项独立处理步骤。Spotify 在后台负载较高时，偶尔会返回 `503` 错误，或使集数保持在 `NOT_READY` 状态。若数分钟后仍未变为 `READY`，则属于 Spotify 侧的延迟（非 Muse 问题）：请告知用户内容已接受，但仍在 Spotify 侧处理中，并建议稍后再试，而非无期限地持续轮询。

## 规则
1. 始终使用 `save-to-spotify --json`；切勿直接安装或调用 Spotify CLI。
2. 连接仅在设置中完成：如果出现令牌相关错误或“未连接”错误，请告知用户前往“设置 → 连接 → Spotify”进行连接，然后重试——本工具不提供任何 `auth` 子命令或连接入口。
3. 在执行 `upload` 之前，请确认标题、目标节目和摘要——该操作会写入用户的账户。
4. 执行 `upload` 后，请轮询 `episodes status --wait` 直到状态变为 `READY`，然后再设置时间线或告知用户内容已上线。
5. 仅保存由用户生成或提供的音频内容；尊重第三方权利。
6. 不得使用 `update` 或 `token`（已禁用）。
7. 封面图片必须为 JPEG/PNG 格式，且大小不超过 1 MB（请先将 `.webp` 等其他格式转换为上述格式）。
8. 返回的 `episode_uri` 表示上传已被接受；如果返回 `NOT_READY` 或 `503` 状态且始终无法变为 `READY`，则表明 Spotify 端存在阻塞——应向用户提示此情况并稍后重试，切勿无限循环。
9. **切勿暴露 Spotify 的内部标识符。** 节目 ID、集数 ID 以及 `spotify:show:` / `spotify:episode:` URI 均为不透明的句柄，仅用作后续 CLI 命令的参数。切勿在确认信息、上传摘要、就绪状态更新或错误说明中打印、回显或提及这些标识符。请以节目和集数的标题来指代它们。
10. 每次删除前均需确认。删除一个节目将同时删除该节目的所有集数。
11. 切勿引导用户前往 Spotify 的创作者网站处理相关内容；请使用上述 CLI 提供的清单和删除命令。
12. 切勿仅凭 `{"status":"deleted"}` 就断言某项内容已消失。请通过有限次数的读取循环验证其确已不存在，或者明确告知用户：被接受的删除操作仍在传播中，且已超过一分钟的超时时间。