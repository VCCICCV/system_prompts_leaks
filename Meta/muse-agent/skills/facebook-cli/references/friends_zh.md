# Facebook 好友

按姓名或个人资料字段列出并搜索您的 Facebook 好友。

## 命令

### 列出/搜索好友

```bash
# 列出第一页的密友（每页 20 人）
facebook-cli me friends

# 按姓名搜索所有好友
facebook-cli me friends --name "Sarah"

# 按城市筛选
facebook-cli me friends --city "旧金山"

# 按工作单位筛选
facebook-cli me friends --work "Meta"

# 组合多个筛选条件（默认为 AND，即所有条件都必须满足）
facebook-cli me friends --city "纽约" --work "谷歌"

# 使用 OR 组合筛选条件（只要有一个条件满足即可）
facebook-cli me friends --city "西雅图" --education "斯坦福大学" --filter-mode OR

# 未来一周内生日的好友（未指定天数时的默认行为）
facebook-cli me friends --birthday-within-days

# 未来 30 天内生日的好友（按生日临近程度排序）
facebook-cli me friends --birthday-within-days 30

# 获取下一页（使用上一次响应中的 paging.cursors.after 作为游标）
facebook-cli me friends --after <游标>
```

| 标志 | 必需 | 描述 |
|------|----------|-------------|
| `--name` / `-n` | 否 | 按姓名搜索所有好友（不区分大小写，支持子字符串匹配） |
| `--city` | 否 | 按当前居住城市筛选（不区分大小写，支持子字符串匹配） |
| `--hometown` | 否 | 按家乡筛选（不区分大小写，支持子字符串匹配） |
| `--work` | 否 | 按雇主名称或职位名称筛选（不区分大小写，支持子字符串匹配） |
| `--education` | 否 | 按学校名称筛选（不区分大小写，支持子字符串匹配） |
| `--filter-mode` | 否 | 多个筛选条件的组合方式：`AND`（默认，所有条件都必须满足）或 `OR`（只要有一个条件满足即可） |
| `--json-query` | 否 | 用于 SocialGraphSearch 的 JSON 查询（FindPeople 格式）。采用 Social RAG 搜索进行兴趣/主题匹配。**不分页**——返回单一结果集，无分页游标，因此 `--after` 无效；`--limit`（最大 20）仍会限制结果数量。 |
| `--birthday-within-days` | 否 | 仅返回未来 N 天内生日的好友（1–365 天）。**若仅指定该标志而未传入数值，则默认为 7 天（即未来一周）**（如 `--birthday-within-days`）；完全省略该标志则返回常规好友列表。新增 `birthday_date` 字段，并按生日临近程度升序排列。尊重隐私（隐藏生日的好友将被排除）。启用该选项后，其他筛选条件将被忽略。 |
| `--limit` | 否 | 每页最多显示的好友数量（最大 20；超过此值将在服务器端被截断） |
| `--after` | 否 | 分页游标——传入上一次响应中的 `paging.cursors.after` 值以获取下一页 |

**输出：** JSON 格式，包含 `data` 数组及 `paging` 对象。`data` 中的每位好友包含 `friend_id`、`name`、`profile_url`，以及可选的 `match_context`（含 `groups`、`pages`、`profile_details`，说明匹配原因）。使用 `--birthday-within-days` 时，每位好友还包含 `birthday_date`（格式为 `YYYY-MM-DD`），且结果按生日临近程度升序排列。未使用任何筛选条件时，返回第一页（20 人）的密友；可通过 `--after` 进行分页以查看更多。使用 `--json-query` 时，采用 SocialGraphSearch 实现更丰富的匹配。

**分页：** 结果采用游标分页（仅支持向前翻页）。`paging.cursors.after`（存在时）为下一页的游标；若不存在，则表示已无更多结果。请通过 `--after` 传递该游标。（注意：游标仅对其来源的筛选模式有效——不要在切换筛选条件后重复使用来自其他模式的 `after`。）**例外：** `--json-query`（SocialGraphSearch）**不分页**——它返回单一结果集，无分页游标，因此 `--after` 无效；`--limit`（最大 20）仍会限制返回的结果数量。

### 通过 `--birthday-within-days` 查看即将到来的生日使用 `--birthday-within-days N` 可以查找未来 N 天内（1–365天）生日的朋友。这是回答“谁的生日快到了？”、“这周有谁生日？”或“我该给谁送生日祝福？”的理想工具。它采用隐私保护的生日查询机制，因此隐藏生日的朋友将被排除在外。

**默认时间窗口：** 当用户询问即将到来的生日但未指定具体时间范围时，只需传递该标志而无需设置值——默认为未来7天（一周）。只有在用户明确指定期限时才传递具体数字（例如，“本月”→ `30`，“未来90天”→ `90`）。

```bash
# 未来一周内的生日（默认，无需设置值）
facebook-cli me friends --birthday-within-days

# 指定时间窗口内的生日
facebook-cli me friends --birthday-within-days 30
```

```json
{
  "data": [
    {
      "friend_id": "123456789",
      "name": "简·史密斯",
      "profile_url": "https://facebook.com/jane.smith",
      "birthday_date": "2026-06-14"
    }
  ],
  "paging": { "cursors": { "after": "<cursor>" } }
}
```

`birthday_date` 的格式为 `YYYY-MM-DD`。结果按日期由近及远分页返回，每次一页（每页20条）——可通过 `--after` 参数配合 `paging.cursors.after` 获取更多结果。当设置了 `--birthday-within-days` 时，其他过滤条件（`--name`、`--city`、`--hometown`、`--work`、`--education`、`--json-query`）将被忽略。

### 通过 `--json-query` 进行社交图谱搜索

使用 `--json-query` 可以按兴趣、运动、话题等条件搜索好友。首先通过 `facebook-cli me` 获取自己的 Facebook ID，然后将其作为 `friend_by` 过滤条件：

> **无分页：** `--json-query` 返回的是单一结果集，不含 `paging` 游标，因此 `--after` 无效。`--limit` 参数（最大20）仍会限制返回的结果数量；若需获取更多结果，请调整查询条件而非依赖分页。

```bash
facebook-cli me friends --json-query '{
  "intent": "FindPeople",
  "target": "users",
  "filters": {
    "friend_by": ["YOUR_FB_ID"],
    "topic": ["hiking"]
  }
}'
```

**重要提示：** 兴趣类筛选应使用 `topic` 键（而非 `interests`）。`friend_by` 的值必须是通过 `facebook-cli me` 获取的用户 Facebook ID。

#### 支持的过滤条件

| 过滤条件 | 说明 | 示例 | 匹配上下文类别 |
|----------|------|------|----------------|
| `friend_by` | 查看者的 Facebook ID（必填） | `["YOUR_FB_ID"]` | — |
| `topic` | 兴趣/话题/爱好 | `["hiking"]`、`["seattle seahawks"]` | 群组、主页 |
| `sports` | 体育团队/活动 | `["pickleball"]`、`["soccer"]` | 主页 |
| `company` | 雇主 | `["Meta"]`、`["Google"]` | 个人资料详情 |
| `job` | 职位 | `["engineer"]` | 个人资料详情 |
| `college` | 大学 | `["Stanford"]` | 个人资料详情 |
| `school` | 任何学校 | `["MIT"]` | 个人资料详情 |
| `location` | 当前城市 | `["Seattle"]` | 个人资料详情 |
| `hometown` | 籍贯 | `["Mumbai"]` | 个人资料详情 |
| `movies`、`music`、`tv_shows`、`books`、`games`、`podcasts` | 娱乐内容 | `["Inception"]` | 主页 |

**用户意图与过滤键的对应关系：**
- “喜欢徒步的朋友” / “对烹饪感兴趣的朋友” → `topic`
- “踢足球的朋友” / “热衷篮球的朋友” → `sports`
- “在 Meta 工作的朋友” → `company`
- “在西雅图的朋友” / “住在纽约的朋友” → `location`
- “曾就读斯坦福的朋友” → `college`
- “看过《绝命毒师》的朋友” → `tv_shows`

#### 带匹配上下文的响应

使用 `--json-query` 时，结果中会包含 `match_context`，用于说明每位好友为何符合匹配条件。这些上下文字符串包含 XML 格式的数据，需要解析后才能展示。
```json
{
  "friend_id": "123456789",
  "name": "简·史密斯",
  "profile_url": "https://facebook.com/jane.smith",
  "match_context": {
    "groups": [
      {
        "id": "111222333",
        "context": "<GROUP_NAME>徒步爱好者</GROUP_NAME><GROUP_DESCRIPTION>一个讨论徒步和背包旅行相关话题的开放群组...</GROUP_DESCRIPTION>"
      }
    ],
    "pages": [
      {
        "id": "444555666",
        "context": "<PAGE_NAME>家庭烹饪小贴士</PAGE_NAME><PAGE_CATEGORY>厨房/烹饪</PAGE_CATEGORY><PAGE_DESCRIPTION>适合家庭主妇的简单食谱...</PAGE_DESCRIPTION>"
      }
    ],
    "profile_details": [
      {
        "source_id": "777888999",
        "context": "就职于Acme公司"
      }
    ]
  }
}
```

**解析上下文字符串：**
- **groups**：从`<GROUP_NAME>...</GROUP_NAME>`中提取群组名称，从`<GROUP_DESCRIPTION>...</GROUP_DESCRIPTION>`中提取描述。
- **pages**：从`<PAGE_NAME>...</PAGE_NAME>`中提取页面名称，从`<PAGE_CATEGORY>...</PAGE_CATEGORY>`中提取类别。
- **profile_details**：纯文本，不含XML标签（如“就职于Acme公司”、“目前位于华盛顿州西雅图市”、“就读于州立大学”）。

**呈现结果：** 向用户展示匹配上下文时，提取可读的名称并自然呈现：
- “简·史密斯——‘徒步爱好者’、‘越野跑者’群组的成员”
- “约翰·多伊——关注‘家庭烹饪小贴士’、‘厨师餐桌’页面”
- “亚历克斯·李——就职于Acme公司”

### 您的身份

```bash
facebook-cli me
```

**输出：** 包含`ok`、`name`和`profile_id`的JSON——即您的Facebook姓名和个人主页ID。这为代理提供了当前Facebook用户的身份信息。您可以将该个人主页ID用于其他命令，例如`profile info --profile-id`（获取完整个人资料详情）或`timeline fetch --profile-id`（获取您的动态）。

## 操作规则

1. 使用`me friends`按姓名或个人资料字段搜索好友。没有通用的好友搜索接口——`me friends`是唯一的好友查找方式。
2. 直接使用专用的筛选标志（`--city`、`--hometown`、`--work`、`--education`）来筛选好友，无需先按姓名搜索再手动过滤结果。
3. 对于基于兴趣的好友搜索（例如，“哪些朋友喜欢徒步？”），请使用`--json-query`并搭配以`topic`为筛选键的FindPeople模板。请勿通过扫描好友的帖子、动态、加入的群组或点赞的页面来推断其兴趣。
4. 当有多个好友符合条件时，请列出他们的显著特征（姓名、城市、工作单位、个人主页链接），并请用户进一步确认。
5. 结果中务必包含每位好友的个人主页链接（`profile_url`）。
6. 在完成身份确认后，继续执行原始请求。如果用户要求“给我看莉兹的帖子”，而您已列出符合条件的好友，待用户选定其中一人（或当仅有一个符合昵称“莉兹”的人时，如“莉兹·泰勒”），则应直接获取并展示其帖子，不要仅停留在列出好友的阶段。
7. 按姓名搜索时，应将常见昵称扩展为其全名并同时进行搜索。例如，“Liz”→也搜索“Elizabeth”；“Mike”→也搜索“Michael”；“Bob”→也搜索“Robert”；“Bill”→也搜索“William”；“Teddy”→也搜索“Theodore”。在向用户展示前，将两次搜索的结果合并后再呈现。