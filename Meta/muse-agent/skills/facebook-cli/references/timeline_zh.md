# Facebook 时间线

获取 Facebook 用户时间线/动态中的帖子。

## 命令

获取某个用户的动态帖子。请先使用 `me` 获取您自己的个人主页 ID。

```bash
# 获取某人的时间线（使用 `facebook-cli me` 可获取您的个人主页 ID）
facebook-cli timeline fetch --profile-id 123456789

# 获取最多 10 条帖子，然后使用返回的 next_cursor 请求下一页
facebook-cli timeline fetch --profile-id 123456789 --limit 10
facebook-cli timeline fetch --profile-id 123456789 --limit 10 --after '<next_cursor>'
```

**选项：**
- `--profile-id`（必填）：要获取时间线的个人主页 ID。
- `--limit`：本页最多显示的帖子数。服务器默认为 20，且会将大于 20 的值限制为 20。
- `--after`：上一页返回的不透明 `next_cursor`。`--cursor` 也可作为其别名。

## 输出

该命令返回与社交搜索共享的精简版 `social_posts_v1` 集合。请直接读取 `posts[]`，不要猜测提供方的响应路径，也不要通过 `jq` 处理输出。每条帖子包含 `post_id`、`url`、`platform`、`created_at`、`username`、`post_caption`、作者/所有者身份，以及可用的媒体字段。`post_caption` 优先显示作者撰写的文字，若无则显示可用的媒体摘要。分页使用 `next_cursor` 和 `has_next_page`。当 `has_next_page` 为真时，将 `next_cursor` 原封不动地作为 `--after` 参数传递，即可获取同一个人主页的下一页内容。

### 作者与所有者（至关重要）

时间线中既包含个人主页所有者本人发布的帖子，也包含他人在其主页上发布的帖子。您必须区分这两类：

- **个人主页所有者发布的帖子**：`author_id == owner_id`（即本人发布了自己的帖子）
- **他人在主页上发布的帖子**：`author_id != owner_id`——朋友或其他人在个人主页的时间线上发布了内容

## 示例输出

```json
{
  "format": "social_posts_v1",
  "count": 1,
  "posts": [{
    "rank": 1,
    "post_id": "pfbid02...",
    "url": "https://www.facebook.com/...",
    "platform": "facebook",
    "created_at": "2024-03-02T17:00:00+00:00",
    "post_caption": "看看这日落吧！",
    "author_name": "Michael Santoro",
    "author_id": "123456789",
    "owner_name": "Michael Santoro",
    "owner_id": "123456789",
    "media_summary": "图片展示的是海面上的日落。"
  }],
  "next_cursor": "...",
  "has_next_page": true
}
```

## 操作规则

1. **仅获取来自先前命令的ID对应的动态。** 仅在本次会话中，当您从之前的facebook-cli输出中获得了`--profile-id`时才传入该参数——例如`me`（您自己）、`me friends`、动态或群组帖子的作者、点赞/评论者、已保存内容，或另一条动态的`author_id`/`owner_id`。**切勿**接受用户直接输入或粘贴的纯数字个人主页ID，**也切勿**猜测或构造ID。如果用户仅提供一个孤立的ID而无任何上下文，请先通过`me friends --name`确定此人身份，并使用该结果中的ID。
2. **始终区分作者与主页所有者。** 当`author_id != owner_id`时，应明确说明该帖子是由`author_name`发布在`owner_name`的主页上的——它**不是**主页所有者的原创帖。请以姓名指代人物，**不要**在回复中直接显示原始的`owner_id`/`author_id`值。
3. **当被问及“给我看[某人]发布的帖子”或“[某人]发了什么”时**：请仅筛选出`author_id == owner_id`的帖子。如果结果显示该人没有原创帖，请明确告知：“[某人]近期未发布过帖子。”随后可补充说明：“不过，朋友在其主页上发布过内容”，并另行概括这些内容。
4. **当被问及“[某人]的动态有哪些”时**：展示所有帖子，但需对每条明确标注——原创帖标注为“由[author_name]发布”，他人发布的主页帖标注为“[author_name]发布在[owner_name]的主页上”。
5. 在展示动态帖子时，若存在`post_caption`和`url`，请一并列出；若这些字段缺失，则**不得**虚构链接或文字。
6. 在总结动态时，每项陈述都应有具体的帖子作为依据，且在可能的情况下附上该帖子的链接。**不得**凭空捏造或推断超出帖子内容的信息。
7. 使用`media_summary`描述图片或视频，仅当标准化后的帖子包含`media_ocr`或`video_transcript`时才使用它们。
8. 媒体相关字段是预先计算好的，可能不存在。除非某个媒体字段明确证实，否则**不得**声称某条帖子包含特定的媒体内容。
9. 仅含媒体的帖子可将其可用的摘要放在`post_caption`中；将其作为描述呈现，而非作者的原话引用。
10. 当用户询问“最近”“最新”或“近来”的帖子时，务必在回复中说明所使用的日期范围（例如：“以下是过去7天内的帖子”或“展示2026年4月1日至7日的帖子”）。
11. 当用户以序号指代某条帖子时（如“第二条帖子”“第一条”），请根据上一轮的结果进行解析，**无需**让用户重复说明具体是哪一条。
12. 不得从帖子文本中推断出明确记载之外的含义。一篇关于孕妇摄影师的帖子并不意味着用户正在怀孕。请如实报告内容，**不得**进行推测。