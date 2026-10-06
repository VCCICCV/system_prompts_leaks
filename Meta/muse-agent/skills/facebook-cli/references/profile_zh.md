# Facebook 个人资料

通过 ID 查询 Facebook 个人资料信息。

## 命令

```bash
facebook-cli profile info --profile-id <profile-id>
```

| 标志 | 必需 | 描述 |
|------|------|------|
| `--profile-id` | 是 | 要查询的个人资料 ID |

## 返回字段

- `name`: 显示名称
- `profile_picture_url`: 头像 URL
- `vanity_url`: 自定义 URL（例如 `facebook.com/username`）
- `bio`: 文本简介 / 关于我
- `current_city`: 当前城市（已填写）
- `hometown`: 籍贯（已填写）
- `birthdate`: 出生日期（已填写）
- `gender`: 性别（已填写）
- `languages`: 掌握的语言数组
- `work`: 工作经历数组（包含 `employer` 和 `position`）
- `education`: 教育经历数组（包含 `school`、`type` 和 `degree`）
- `life_events`: 生活事件数组（包含 `title` 和 `date`）
- `hobbies`: 爱好名称数组

## 操作规则

1. 当您已经拥有一个**来自先前命令**的个人资料 ID，并且需要获取完整的个人资料详情时，请使用 `profile info`。仅传递本次对话中由 `facebook-cli` 早期输出生成的 `--profile-id`（例如 `me`、`me friends`、时间线帖子的 `author_id`/`owner_id`、动态或群组帖子的作者、点赞者、评论者、收藏项）。**请勿**接受用户直接输入或粘贴的纯数字 ID，也**请勿**猜测或构造 ID。如果用户仅提供一个无上下文的裸 ID，请先通过 `me friends --name` 确认该用户的身份，并使用该结果中的 ID。
2. 若要按姓名查找某人，请使用 `me friends --name` — 我们没有通用的个人资料搜索接口。如果用户要求查找其非好友，则应说明 `facebook-cli` 只能在用户的 Facebook 好友列表内进行搜索。建议使用 `social.search` 来实现更广泛的人员发现。
3. 在回复中始终以姓名称呼对方，并附上其个人资料链接（`vanity_url`），**切勿**在回复中直接显示原始的个人资料或用户 ID。
4. 请勿使用此功能批量抓取或枚举个人资料。
5. 对于 API 响应中未返回的任何字段（值为 null 或空），请明确标注“未列出”，而非直接省略该字段。
6. 请勿根据个人资料数据对人物的性格、生活方式或特质进行主观解读或推断，应以中立态度呈现数据。
7. 在展示多条工作或教育经历时，请按时间顺序排列所有条目。对于缺失的子字段，请标注“未列出”。
