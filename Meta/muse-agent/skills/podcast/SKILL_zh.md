---
name: "generate_podcast"
description: "创作并发布音频内容：制作一集播客、简报或旁白式摘要，并以单声道或多声道形式输出为MP3文件。如需逐字朗读所提供的文本，请使用TTS功能。"
metadata: { "包含在提示中": 真 }
---
# 生成播客

## 目的
生成音频内容——包括播客、音频简报、旁白摘要或任何口语类音频。此技能适用于所有生成的音频，而不仅限于播客。“podcast-helper”脚本负责语音合成、目录管理、封面图生成及发布。

## 工作流程

### 1. 计划
- **主题与切入角度**
- **发言人姓名及声音** — 默认优先选择 Meta AI 的声音（`MAI_01` 和 `MAI_03`，分别为“Warm”和“Smooth”）；这是推荐的、具备制作级音质的声音选项，大多数情况下都是合适的选择。默认使用 Warm（`avocado_v2:MAI_01`）和 Smooth（`avocado_v2:MAI_03`）。如果主题、语气或用户需求指向其他目录中的声音——例如特定口音、角色适配，或需要更具辨识度的发言人——则可从目录中另行选择。完整的声音目录位于 `/opt/hatch/skills/voice-selector/voice_source.json`，可通过条目的显示名称（`name`）进行选择，并传入其 ID（如 `avocado_v2:MAI_01` 这种带冒号的形式即可）。支持任意数量的发言人。（“MAI”仅为内部 ID 前缀——向用户介绍时应称“Meta AI 声音”或使用显示名称，切勿使用“MAI”。）
- **主持人姓名** — 为每位发言人取一个真实的名字（如 Alex、Jordan）。这些是节目内的人物名，由您自行命名，与声音的显示名称无关：切勿以“Warm”或“Smooth”等声音名称来称呼发言人。**请务必记住为每个声音分配的姓名**——将对应关系记录在 `~/MEMORY.md` 中（例如：“播主 Alex = avocado_v2:MAI_01”），并在为新主持人命名前，先查看该文件，确认是否已为该声音使用过某个名字。每次再使用该声音时都沿用同一名称，使同一个声音对用户而言始终代表同一角色。对于**系列节目**，每期都应沿用相同的主持人姓名和声音分配，以保持节目的一致性——可在定时任务的代码中固定这些设置（参见“调度”部分）。
- **目标时长** — 5–15 分钟；约 130 字/分钟
- **避免重复过往内容** — 在确定主题与切入角度之前，请先运行 `podcast-helper manifest read`，查看近期每期节目的 `topics` 字段（即该期已涵盖内容的简要概述）。尽量让新一期聚焦于全新素材或真正新颖的视角，而非重复已有内容。这一点对于系列节目尤为重要。

### 2. 编写脚本
请完整写出对话内容。每句话前标注发言人的名字，后加冒号：  
```
Alex: 欢迎收听本期播客。今天我们来聊聊异步 Rust。
Jordan: 很棒的话题。我们先从为什么异步很重要说起吧。
Alex: 最大的优势是零成本的抽象……
```

将脚本保存到一个文件中（例如 `/tmp/script.txt`）。

**脚本会直接输入到 TTS 模型中——请以口语化的形式撰写。** 文本将按原样被朗读出来，请遵循以下规范：

- **数字**：请全部用文字写出——“十万个”、“四十二美元五十美分”、“百分之三点五”
- **时间**：如“上午十点”、“下午三点半”、“中午”、“午夜”
- **年份**：如“二〇二五年”、“一九九九年”
- **缩写**：请完整拼出——“美国”、“应用程序接口”（可发音的首字母缩略词如“NASA”可以保留）
- **网址**：简化表述——用“example dot com 的链接”代替原始 URL
- **电子邮件**：如“john at company dot com”
- **列表/数据**：转换为自然的句子，最多列出 3–5 项
- **标点符号**：用逗号表示自然停顿。如果句子过长，应将其拆分为两个完整的句子，切勿截断成片段。
- **切勿包含**：舞台提示（如“（停顿）”）、音效、标记、表情符号、引用编号（如[1]）、数学符号，以及视觉分隔符（如“---”）。

**请使用完整、自然的对话式句子——而非标题式表达。** 主持人说的每一句话都应是带有主语和谓语的完整语法句，就像人们日常说话一样。避免使用简短无动词的片段或类似新闻快讯的陈述；这是一场对话，而不是新闻稿或比分播报。偶尔为了强调而使用一句简短的话（如“难以置信”）是可以的，但绝不能把多个截断的片段连在一起。

- 避免：`梅西今晚送出第二记助攻。阿根廷二比一领先。全场比赛结束。英格兰队心碎了。`
- 推荐：`这是梅西今晚送出的第二记助攻，帮助阿根廷以二比一领先。终场哨响时，英格兰队只能黯然神伤。`

当需要呈现数据或比分时，应将其融入自然的句子中（如“英格兰队仅控球百分之四十四”），而不是直接抛出孤立的片段（如“英格兰，控球百分之四十四”）。

### 3. 生成

**编写主题摘要用于去重。** 在生成之前，请将本期节目的主要话题和关键要点以简短的项目符号形式写入一个文件（例如 `/tmp/topics.md`），并在运行时通过 `--topics-file` 参数传入。摘要应简洁明了，只需列出几个要点，标明主题及具体故事、嘉宾或角度，无需完整逐字稿。该摘要会与节目一同保存，并在后续检索时由 `manifest read` 命令的 `topics` 字段读取，以避免重复内容。示例：  
```
- 异步 Rust 的基础：Future、执行器、零成本抽象
- Tokio 与 async-std 的权衡
- 常见陷阱：在异步上下文中阻塞、忘记使用 .await
```

**默认行为：仅生成，不发布。** 只有在用户明确要求发布，或播客目录中已存在 RSS 订阅源（即 `feed` 对象中 `feed_url` 不为空，表明此前已发布过内容并希望继续添加新剧集）时，才添加 `--publish` 参数。仅包含 `spotify_show_url`（来自个人“保存至 Spotify”功能）的 `feed` 对象**不属于** RSS 订阅源，**不得**触发 `--publish`；发布属于公开行为，需用户明确同意。

始终使用 `--cover-prompt` 参数，提供一段简短的节目主题描述，以便 podcast-helper 自动生成封面图。

**切勿使用 `--cover-image` 参数。** 发布环节仅接受自动生成的封面或内置的默认封面，因此若使用 `--cover-image` 参数发布，将会失败。请改用 `--cover-prompt`；如果用户提供了图片文件，则应根据该图片生成一张封面，并说明已代为处理。该参数标志仍会被解析，未来会重新启用，只是目前无法直接用于发布流程。

**语言：** 除非用户明确要求使用其他语言，否则标题、描述、播客文案和脚本均应使用当前对话的语言撰写。`podcast-helper` 默认将合成语音设置为请求中的 `JARVIS_PRESENTATION_LOCALE`；仅当用户明确指定时才使用 `--language` 参数（例如 `--language es`、`--language pt`，或地区代码如 `pt_BR`）。不同语言的合成质量差异较大——英语质量最高；其他语言可能存在发音不准、口音等问题，且不同语音对同一语言的处理效果也有所不同（如果某段语音听起来不自然，可尝试从 `voice_source.json` 中选择其他 `--speaker` 语音）。请告知用户非英语音频可能不够完美。详情请参阅 `tts` 技能的“语言”部分。

**单集生成：**  
```sh
podcast-helper generate \
  --script /tmp/script.txt \
  --title "深度解析异步 Rust" \
  --description "Alex 和 Jordan 探讨 Rust 中的异步模式" \
  --speaker Alex=avocado_v2:MAI_03 \
  --speaker Jordan=avocado_v2:MAI_01 \
  --cover-prompt "异步 Rust 编程，齿轮与闪电" \
  --topics-file /tmp/topics.md
```

**定时/周期性生成**——通过传递与定时任务 ID 匹配的 `--series-id`，使同一系列的各集共享同一封面：  
```sh
podcast-helper generate \
  --script /tmp/script.txt \
  --title "晨间新闻 - 2026 年 6 月 2 日" \
  --description "今日要闻" \
  --speaker Alex=avocado_v2:MAI_03 \
  --speaker Jordan=avocado_v2:MAI_01 \
  --series-id daily-news \
  --cover-prompt "晨间新闻简报，日出与报纸" \
  --topics-file /tmp/topics.md
```

使用 `--series-id` 后，封面仅在首集生成一次，并在该系列后续各集中重复使用。

如果用户要求发布，或者播客目录中已存在 RSS 源，请运行 `podcast-helper manifest read`，并检查是否存在带有非空 `feed_url` 的 `"feed"` 对象（仅包含 `spotify_show_url` 的 `feed` 不计入）：

```sh
podcast-helper generate \
  --script /tmp/script.txt \
  --title "深度解析异步 Rust" \
  --description "Alex 和 Jordan 探讨 Rust 中的异步模式" \
  --speaker Alex=avocado_v2:MAI_03 \
  --speaker Jordan=avocado_v2:MAI_01 \
  --cover-prompt "异步 Rust 编程" \
  --topics-file /tmp/topics.md \
  --publish --feed-title "每日与 Alex"
```

返回包含 `path`、`duration_secs`、`slug`、`cover`，以及（若已发布）`feed_url`、`episode_url` 和 `subscription_links` 的 JSON 数据。

### 4. 在聊天中交付
始终以纯文本形式在可播放链接上方显示标题：  
```
{title}
[{title}](sandbox://workspace/podcasts/{slug}/{slug}.mp3)
```

首次提及发布、个人订阅源或订阅时，简要说明该订阅源是什么：即一个可供添加到播客应用的个人 RSS 播客订阅源，未来发布的各集将自动同步至此。首次说明后，后续可使用更简短的表述。

如果该集已发布，需告知用户其已发布并将出现在他们的订阅源中。请注意，已发布的各集均为公开内容。如果结果中 `published: false` 且存在 `publish_error`，则表示音频已在本地生成，但发布失败：请将 `publish_error` 原封不动地反馈给用户，切勿声称音频已推送至订阅源。若同时出现 `blocked: true`，则适用第 5 步中的内容审核拦截规则，无需进行后续发布操作——不得重试、修改脚本或尝试其他发布路径。若 `retryable: true`，则表示内容审核未能执行：请告知用户稍后可使用完整的第 5 步 `podcast-helper publish` 命令重新尝试发布。在重试前，请先运行 `podcast-helper manifest read`，并在已有目标订阅源的情况下，务必完全沿用其现有的 `feed_title`，切勿为现有订阅源随意创建或更改标题。发布完成后，展示相应的后续操作：
1. **如果尚未发布且未被屏蔽：**“需要我将这条内容发布到您的个人播客订阅源吗？这是一个 RSS 订阅源，您可以将其添加到 Apple 播客、Overcast、Pocket Casts 或大多数其他播客播放器中，未来发布的剧集会自动同步到这里。请注意：已发布的剧集是公开的——任何拥有链接的人都可以收听。”
2. **如果尚未设置定时任务：**“每天生成一集新内容吗？”

### 5. 发布（若未在步骤 3 中完成）
```sh
podcast-helper publish \
  --slug {slug} \
  --feed-title "Daily with Alex" \
  --feed-description "每日新闻与科技资讯"
```

返回包含 `feed_url`、`episode_url` 和 `subscription_links` 的 JSON 数据。

公开发布的内容需经过审核。如果输出中出现 `"blocked": true`，则表示该集未成功发布：音频文件和目录条目仍保留在本地，但无法进入公共订阅源。请原样向用户传达返回的 `error` 信息并停止执行——不要重试、不要修改脚本绕过审核，也不要切换其他发布路径。保存到用户自己的 Spotify 播客不会受到此次审核的影响。
发布时若上传了新的节目封面，该封面将统一应用于整个节目的所有剧集——其封面会在播客目录中被替换为新封面。如果是系列节目，则保留系列封面；而通过 `update-cover` 更改过封面的单集，在发布时仍会保留原有封面。来自其他订阅源的剧集则会保留各自的封面。
发布完成后，务必再次展示收听链接并确认发布状态：
```
{title}
[{title}](sandbox://workspace/podcasts/{slug}/{slug}.mp3)

已发布至您的个人订阅源，即您可在播客应用中订阅的 RSS 订阅源，未来发布的剧集会自动同步到这里。
已发布的剧集是公开的——任何拥有订阅链接的人都可以收听。
```

### 6. 订阅（发布后）
首次发布到某个订阅源时，根据输出的 JSON 数据展示订阅链接：
```
订阅您的个人订阅源：

RSS 订阅源地址（复制后粘贴到任意播客播放器）：
`{subscription_links.rss}`

或直接在播客应用中打开：
- [在 Apple 播客中打开]({subscription_links.apple_podcasts})
- [在 Overcast 中打开]({subscription_links.overcast})
- [在 Pocket Casts 中打开]({subscription_links.pocket_casts})

这是您生成剧集的 RSS 订阅源，添加到播客应用后，未来发布的剧集会自动同步到这里。
```

**重要提示——如何呈现订阅源地址：**RSS 订阅源地址仅供用户**复制粘贴**，切勿点击。对于任何已发布的 `https://` 链接（订阅源地址、剧集链接），请始终以代码格式显示，无论是行内 `` `https://…` `` 还是用代码块包裹。切勿以裸露的 URL 形式输出（如 `https://…`，在原生客户端中会自动转换为可点击的超链接），也切勿使用 `[label](https://…)` 的 Markdown 格式；这两种形式在原生客户端上可能无法正常打开，且容易引导用户点击而非复制。请勿提供直接的剧集下载链接。唯一可点击的链接应为本地的 `sandbox://` 收听链接，以及播客应用的协议链接（如 `podcast://`、`overcast://`、`pktc://`）。
对于同一订阅源的后续剧集，请跳过此步骤——用户已订阅。

## 订阅源管理
除非用户明确要求使用单独的订阅源，否则**始终使用同一个订阅源**。发布前，请运行 `podcast-helper manifest read`。`feeds` 列出了当前虚拟机上的所有订阅源，`feed` 是最近一次用于发布的订阅源；如果其中任一字段的 `feed_url` 不为空，则应完全沿用该订阅源的 `feed_title`。
当用户同时维护多个节目时，订阅源的选择取决于标题：使用与现有订阅源匹配的 `--feed-title` 进行发布会追加到该订阅源，而使用其他标题则会创建一个新的订阅源。因此，向现有节目追加内容时，请逐字符复用标题；切勿将同一标题用于不同的节目。在首次发布时（播客目录中尚无 `feed_url`——仅包含 `spotify_show_url` 的 `feed` 对象仍被视为无 RSS 源），请根据用户的 Muse 昵称为其选择一个个人播客标题，例如“今日与 Alex”、“新闻与 Alex”。

## 添加到您的 Spotify（个人，可选）
当用户明确要求将其生成的节目添加到自己的 Spotify 时，请使用 `podcast-helper save-to-spotify` 将其添加到用户**本人**的 Spotify 账户中。这是将节目保存到用户个人的 Spotify 资料库或播客系列中——这并非上述公开可订阅的 RSS 源发布，也不使用 `podcast-helper --publish` 命令。仅在用户明确请求时才执行此操作，切勿主动提供。

请通过 `podcast-helper` 进行上传（而非直接使用原生的 `save-to-spotify` 命令行工具）：该辅助工具会通过内置的 `save-to-spotify` 命令行工具完成上传，并将生成的 Spotify 播客链接记录到播客目录中，以便在播客库页面中显示。有关底层命令行工具、授权流程及规则，请参阅 `references/save-to-spotify.md`。1. **列出播客 / 检查连接：** 运行 `save-to-spotify --json shows` — 该命令会列出已有的播客，并确认 Spotify 已成功连接。如果因令牌问题或“未连接”错误而失败，请提示用户前往 **设置 → 连接 → Spotify**（共享的 Spotify 连接）进行连接，然后重试。本工具没有专门的 `auth` 子命令或连接入口。在创建新播客时，应询问用户是复用现有播客还是新建一个，切勿自动默认选择。
2. **上传并发布**：使用 `podcast-helper` 工具（需先与用户确认标题、目标播客和简介 — 此操作会直接写入用户的账号）。该工具会根据播客的 slug 从播客目录中读取音频、标题和封面；如需覆盖封面，请单独指定（仅支持 JPEG/PNG 格式，且文件大小不超过 1 MB — 如为 `.webp` 等其他格式，请提前转换）：  
   ```sh
   podcast-helper save-to-spotify \
     --slug {slug} \
     [--show-id <id> | --new-show "<播客标题>"] \
     [--title "{标题}"] [--summary "{简介}"] [--image {封面}]
   ```
   返回的 JSON 中包含 `episode_id`、`episode_uri`、`spotify_show_id` 和 `spotify_show_url`。
3. **等待处理就绪：** 使用 `save-to-spotify --json episodes status <episode-id> --wait` 命令（请使用步骤 2 中返回的 `episode_id`）。处理过程在服务器端进行，通常需要几分钟；返回 `episode_uri` 即表示已成功接收。若状态长时间停留在 `NOT_READY`（Spotify 有时会返回 503 错误），则属于 Spotify 侧的延迟，请告知用户正在处理中，稍后再重试，而非无限轮询。
4. 提示用户内容已上传至其 Spotify 账号，可能需要几分钟才会在应用中显示。由于播客链接已记录，播客库页面上的 Spotify 选项将直接链接到该播客。向用户说明播客及各集的状态时，请以标题为准，并用通俗易懂的语言概括处理进度。Spotify 播客 ID、集 ID 以及 `spotify:show:` 和 `spotify:episode:` URI 仅为 CLI 内部使用的标识符，可用于后续命令，但切勿在面向用户的反馈中直接展示。

要执行删除操作，请使用原生的 `save-to-spotify` 命令行工具，而不是 `podcast-helper`、`spotify-api`、`podcasters.spotify.com` 或 `creators.spotify.com`。首先运行 `save-to-spotify --json shows`，根据节目标题找到目标节目，并通过 `shows get <show-id>` 查看其详细信息。然后使用 `episodes --show-id <show-id>` 列出该节目的所有集数，再用 `episodes delete <episode-id>` 删除指定的某一集。若要删除整个节目，请先确认所有相关集数都将被删除，然后运行 `shows delete <show-id>`；该命令会同时删除所有集数。当收到 `{"status":"deleted"}` 的响应时，即表示删除成功，随后请在最多60秒内轮询相关的 `shows` 或 `episodes --show-id` 列表，确认目标项目已彻底移除；否则应告知用户 Spotify 仍在同步删除状态，且不应重复执行删除操作。删除 Spotify 上的副本不会影响本地音频文件或 RSS 源。完整的操作流程请参阅 `references/save-to-spotify.md`。

## 封面图

封面图由 `podcast-helper` 在生成过程中自动处理。您可以通过以下两个参数进行控制：

- **`--cover-prompt`**：提供一段简短的描述，用于说明该集的主题（例如：“晨间新闻播报，日出与城市风光”）。`podcast-helper` 会调用 `media-generation` 工具生成一张正方形图标风格的图片。此参数必须始终提供。
- **`--series-id`**：用于将共享同一封面的各集归为一组。对于定期更新的节目，可使用对应的 Cron 任务 ID。若不指定该参数，每集都会生成独立的封面。

**封面图的处理逻辑如下：**

| 场景 | 行为 |
|------|------|
| 单集节目 | 根据 `--cover-prompt` 为每集单独生成独特的封面 |
| 定期/定时节目 | 首集生成封面后，后续具有相同 `--series-id` 的各集将复用该封面 |
| 已发布节目 | 新发布的节目会用新的节目封面替换原有各集的封面；如果是系列节目，或者通过 `update-cover` 更改了封面，则继续使用该节目的封面 |

**备用方案**：若未提供 `--cover-prompt`，则使用内置的默认封面（路径为 `/opt/hatch/skills/generate_podcast/default-cover.jpg`）。系统也会优先尝试使用用户的头像作为封面，但目前无法将头像封面发布到播客源中——这也是为何每次向播客源推送内容时都必须提供 `--cover-prompt` 的另一个原因。

您无需自行调用 `media.generate_image` 来生成封面图，`podcast-helper` 会在内部自动完成这一过程。

目前暂不支持 `--cover-image` 参数：发布时仅接受自动生成的封面或内置的默认封面。虽然该参数仍会被接受以便日后重新启用，但如果使用它发布节目将会失败，因此请勿使用。

### 更改封面

当用户要求更改某集的封面时——无论是提供新的提示词、调整现有封面，还是直接上传图片——请通过 `podcast-helper` 重新生成封面，以确保目录中的引用指向新文件：

```sh
podcast-helper update-cover \
  --slug {slug} \
  --cover-prompt "山间湖面上的星空夜景"
```

该命令会生成一张新的封面图片并保存至新路径，同时更新该集的 `cover` 引用指向该新路径，并在聊天界面中输出该新路径。请将此新路径展示给用户。

硬性规则：

- **切勿覆盖当前封面文件。** 不要将新图片通过 `cp`、移动或重命名的方式覆盖旧的封面路径，不要为封面调用 `media.generate_image`，也不要在更改了文件内容后再次提供旧的路径。已发布的预览和库中的节目行会通过路径引用旧文件，客户端也会按路径进行缓存——如果静默地改写文件内容，就会导致已展示的内容发生变化（或显示过时的封面），而这正是本流程旨在避免的问题。
- **用户提供的图片仅作为提示，而非实际文件。** 在 `--cover-prompt` 中描述该图片，并在生成时进行合成（与 `--cover-image` 的规则相同）；发布时仍只使用合成后的封面图。
- **系列剧集各自独立。** `update-cover` 只会更改本地目录中该集的封面；节目的整体封面、其他剧集以及未来基于 `--series-id` 的生成仍会沿用共享的系列封面。发布已编辑的剧集时，节目的封面也会一并保留，且公开订阅源中显示的是该封面（一个订阅源只有一个封面）。
- **已发布的订阅源会保留之前的封面，且发布操作拒绝重复。** `podcast-helper publish` 会拒绝发布已发布的剧集，因为发布操作会在订阅源中追加一条新的剧集条目（从而在每个订阅者的订阅源中复制该集），而目录中的对应行却会遗忘原始信息。因此，不存在仅推送封面变更的发布路径：目前尚无仅更新封面的功能，已发布节目的封面修改也仅限于本地目录和聊天记录。

## 定时任务
当用户请求创建一个定期更新的播客、音频简报或定时播放的音频内容时：

1. **立即生成第一集**——不要只设置定时任务就让用户无内容可听。应立即生成并交付第一集，以便用户立刻有内容可听。
2. **询问是否发布**——主动提出将其发布到播客订阅源，方便用户在自己喜欢的播客应用中订阅收听。若用户同意，则发布第一集并提供订阅链接。
3. **随后创建定时任务**——设置好定期的执行计划。

对于定期更新的播客，使用 `cron` 工具创建定时任务。**务必设置 `timeout_secs: 1800`**。

`cron` 工具调用示例：  
```json
{
  "action": "add",
  "file_name": "daily-podcast__daily@13:00:00.md",
  "id": "daily-podcast",
  "enabled": true,
  "mode": "task",
  "timeout_secs": 1800,
  "schedule": {
    "kind": "daily",
    "timezone": "UTC",
    "time": "13:00:00"
  },
  "body": "生成一期关于<主题>的新播客节目。遵循 generate_podcast 技能的工作流程。每期保持相同的主播阵容：主持人 Alex (--speaker Alex=avocado_v2:MAI_01) 和 Jordan (--speaker Jordan=avocado_v2:MAI_03)。调用 podcast-helper generate 时，请使用 --series-id daily-podcast，并提供 --cover-prompt '<主题描述>'。在确定本期选题前，请先运行 podcast-helper manifest read，查看近期各集的 topics 字段，避免重复已报道的主题。将简要的主题摘要写入 /tmp/topics.md，并通过 --topics-file 传递给后续生成过程，以便后续剧集进行去重处理。生成完成后发布至订阅源。"
}
```

请确保定时任务的描述具体明确——包括选题方向、固定的主播阵容（包含姓名及对应的语音 ID，以保证每集使用相同的主播和声音）、与定时任务 ID 一致的 `--series-id`，以及明确要求在生成时实时检索最新信息的指令。

## 实用命令
针对特殊情况和手动操作，`podcast-helper` 提供了一些子命令：

- `podcast-helper manifest read` — 打印当前的播客目录
- `podcast-helper manifest add-episode --slug ... --title ... --duration-secs ... --path ... --chunk-count ...` — 添加一集
- `podcast-helper manifest update-episode --slug ... [--episode-url ...] [--feed-url ...]` — 更新一集
- `podcast-helper update-cover --slug ... --cover-prompt ...` — 用新生成的封面替换某集的封面（仅更新引用，不会覆盖原文件）
- `podcast-helper generate-slug --title "..."` — 根据今日日期生成一个带连字符的小写字符串作为 slug

## 收听已有集数
如果用户要求收听某个播客、收听他们的播客，或询问其集数，请使用 `podcast-helper manifest read` 命令读取播客目录，并为相关集数提供收听链接：

`[{title}](sandbox://workspace/podcasts/{slug}/{slug}.mp3)`

如果该信息源已发布，还应包含订阅链接（详见第6节的格式说明）。

## 操作规则
1. 使用位于 `/opt/hatch/skills/voice-selector/voice_source.json` 中的语音，通过其目录 `id` 进行引用。大多数情况下默认使用 Meta AI 的语音（目录 ID 为 `MAI_01` 和 `MAI_03`），但当话题、语气或用户请求需要时，可选择其他目录中的语音。切勿对用户称其为“MAI”，应称之为 Meta AI 语音或使用显示名称。
2. 如果某个片段生成失败，`tts synthesize-script` 将停止运行——不得交付部分剧集。大多数失败是暂时性的后端问题：请在有限退避策略下（约 5 分钟、10 分钟、30 分钟、1 小时）**稍后重试同一生成任务**，并保持**相同的说话人语音**。切勿为规避失败而更换不同的语音或 TTS 引擎。应安排重试任务（通过延迟唤醒或短周期 Cron 作业）而非阻塞执行，并告知用户将在合成恢复后交付剧集。**若在约 1 小时的重试后仍失败，则停止处理**——取消已安排的重试任务，上报错误，并建议用户稍后再试。（如果出现 `... not allowed to use voiceID ...` 错误，重试也无法解决——该语音 ID 不允许用于当前客户端；请从 `voice_source.json` 中选择其他语音并重新生成。身份验证或请求清除类错误同样需要修复，而非重试。）
3. 在新订阅源的第一集时，引导用户完成订阅；后续剧集则跳过。
4. **Cron 任务执行：** 严格按照 Cron 任务描述执行。若要求发布，则无需确认直接发布。
5. **切勿**将已发布的 `https://` 订阅源或剧集 URL 以裸链接或可点击链接的形式呈现——始终以可复制代码的形式展示（用反引号包裹）。只有 `sandbox://` 播放链接以及播客应用协议链接（如 `podcast://`、`overcast://`、`pktc://`）才可设置为可点击。
