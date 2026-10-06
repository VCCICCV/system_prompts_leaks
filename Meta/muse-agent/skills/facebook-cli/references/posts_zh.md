# Facebook 帖子

## 读取帖子

```bash
facebook-cli post read --post-id <帖子ID>
facebook-cli post read --url '<规范的Facebook链接>'
```

必须且只能提供一个非空的 `--post-id` 或 `--url`。ID 可以是数字形式，也可以是简化的 PFBID。输出采用标准化的 `social_posts_v1` 格式，包含帖子的正文、永久链接及可用的媒体信息。

## 从 Facebook 链接中读取内容

在读取内容之前，请先解析 Facebook 的分享封装链接，例如 `/share/<token>/` 或 `/share/p/<token>/`：

```bash
facebook-cli link-sharing decode-url \
  --url 'https://www.facebook.com/share/p/<分享令牌>/'
```

响应中会包含 `original_url`。如果该字段为 null、缺失或为空，则报告无法解析该链接并停止处理，切勿将分享令牌用作 ID。解码后的 URL 仅用于识别内容类型，并不能确认当前账号是否可访问该内容。帖子读取器本身不会对分享链接进行解码。

对于已经是规范格式的 URL，可跳过解码步骤。将完整的原始 URL 或解码后的 `original_url` 传递给 `post read --url`，适用于以下 `facebook.com`、`www.facebook.com`、`m.facebook.com`、`mbasic.facebook.com`、`web.facebook.com` 或 `touch.facebook.com` 上的 HTTP(S) URL 模式：

| 内容 | 规范 URL 模式 |
| --- | --- |
| 帖子 | `/<个人主页>/posts/<帖子ID>` 或 `/posts/<帖子ID>` |
| 群组帖子 | `/groups/<群组>/posts/<帖子ID>` 或 `/groups/<群组>/permalink/<帖子ID>` |
| 帖子永久链接 | `/story.php?story_fbid=<帖子ID>` 或 `/permalink.php?story_fbid=<帖子ID>` |
| 照片 | `/photo.php?fbid=<照片ID>` 或 `/photo/?fbid=<照片ID>` |
| 视频或短视频 | `/reel/<视频ID>`、`/videos/<视频ID>` 或 `/<个人主页>/videos/<视频ID>` |
| 视频查询 URL | `/watch/?v=<视频ID>` 或 `/video.php?v=<视频ID>` |

```bash
facebook-cli post read --url '<来自解码器的原始URL>'
```

请保持 URL 完整，包括其查询参数；不要提取其中的 ID，也不要单独发起媒体解析请求。读取器会将支持的图片/视频 URL 解析为其所属的帖子。独立的媒体可能没有对应的帖子。如果读取失败，则应终止整个流程：报告失败，不要猜测 ID，也不要用媒体 ID 作为帖子 ID 重试，更不应声称仅凭该 URL 就能读取内容。

格式错误或不支持的 URL（包括未解码的分享链接）将返回 HTTP 400 错误。而那些媒体或帖子已不存在、为私密状态或不可读的支持 URL 则会返回 HTTP 404 错误。这两种情况均不会返回任何帖子内容。

### 其他实体

其他类型的实体应使用各自的读取命令：

- `/groups/<群组ID>`：使用 `groups posts --group-id <群组ID>` 来浏览群组帖子。如果群组路径中使用的是名称而非 ID，请先使用 `groups search` 进行搜索。
- `/marketplace/item/<商品ID>`：使用 `marketplace listing details --url '<解码后的完整URL>'`。
- 对于活动、游戏及其他无法识别的 URL，请报告该类型 URL 不受支持，切勿从 URL 中随意提取数字并传递给 `post read`。

## 浏览个人主页动态

如需查看某个个人主页的最新动态，可使用 `timeline fetch`：

```bash
facebook-cli timeline fetch --profile-id <个人主页ID>
```

## 读取评论或点赞等互动

```bash
facebook-cli post comments read --post-id <帖子ID>
facebook-cli post reactions read --post-id <帖子ID>
```

## 操作规则

1. 在呈现归一化的帖子、信息流或时间线结果时，如果存在 `post_caption` 和 `url` 字段，请一并包含。与 `header_text` 匹配的信息流标题，或由 `media_summary` 生成的标题，属于描述性的提供方文本，而非作者的原话。当这些字段缺失时，不得虚构链接或文本。
2. 仅在用户并非针对特定人物提问（例如浏览信息流）时，才显示发布者的姓名；若上下文已明确作者身份，则无需显示。
3. 在总结某人的动态时，每项陈述都应以具体帖子为依据。不得凭空捏造或推断未被实际帖子所证实的活动。
4. 如有归一化的 `media_summary`、`media_ocr` 或 `video_transcript` 字段，请使用它们来提供媒体内容背景。
5. 媒体相关字段是预先计算好的，并非所有帖子都具备。除非上述字段之一予以确认，否则不得声称某帖子包含特定的媒体内容。
6. Facebook 的视频和 Reels 帖子可能还包含购物相关信息：`shoppable_products`（检测到的商品列表，每项包含 `product_name` 及可选的 `descriptive_name`、`category`、`item_type`、`brand_name`、`color`、`style`、`gender`、`prominence`）、`shoppability` 和 `shoppable_product_category`。请使用这些字段回答有关视频中展示或销售内容的问题，而不要仅凭标题或字幕推断商品。不含购物相关内容的帖子以及列表类接口（如“个人主页帖子”、“信息流”）均不包含这些字段——只有在“帖子读取”接口中才会返回。