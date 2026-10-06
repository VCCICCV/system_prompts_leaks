# Facebook 群组

获取群组详情、搜索群组，并浏览 Facebook 群组中的帖子。

## 命令

### 群组详情

```bash
facebook-cli groups details --group-id "<id-or-vanity>"
```

`--group-id` 接受数字形式的群组 FBID，或来自 Facebook 群组 URL 的自定义名称，例如  
`https://www.facebook.com/groups/<group-id-or-vanity>`。使用 `/groups/` 之后的路径段；请勿传递整个 URL。

**示例：**

```bash
# 通过数字 FBID 获取
facebook-cli groups details --group-id "<group-id>"

# 通过自定义名称获取
facebook-cli groups details --group-id "<group-vanity-name>"
```

**响应字段：**

- `name`: 群组名称
- `about`: 群组简介文本
- `visibility`: `OPEN`、`CLOSED` 或 `SECRET`
- `history`: 当可见时，显示本地化的创建与名称变更历史
- `tags`: 管理员选定的群组标签
- `rules`: 按管理员定义顺序排列的规则标题和描述对象
- `member_count`: 大致成员数

对于不存在的群组，以及对已关联 Facebook 账号不可见的群组，将返回相同的未找到错误。

### 搜索群组

```bash
facebook-cli groups search [--keywords <text>] [--role <role>] [--sort-by <sort>] [--status <status>] [--limit <n>] [--after <cursor>]
```

**选项：**
- `--keywords`（可选）：按名称或主题进行搜索。省略则列出您自己的群组
- `--role`（可选）：控制搜索范围：
  - `connected`（默认）——仅限您是成员的群组
  - `admin`——您是管理员的群组
  - `admod`——您是管理员或版主的群组
  - `any`——优先搜索已连接的群组（包括私密群组），然后是公开发现结果（需指定 `--keywords`）
- `--sort-by`（可选）：`MOST_RELEVANT`、`LARGEST`、`LAST_VISITED`、`RECENT_ACTIVITY`、`ALPHABETICAL`
- `--status`（可选）：`any`、`weekly_active`、`non_archived`（默认）、`archived`
- `--limit`（可选）：每页最多显示的结果数量（默认：使用 `--keywords` 时为 25，否则为 10；最大 25）
- `--after`（可选）：来自上一次响应中 `paging.cursors.after` 的游标。仅在未指定 `--keywords` 时支持

**示例：**  
```bash
# 列出我的群组
facebook-cli groups search

# 搜索关于徒步的群组
facebook-cli groups search --keywords "hiking"

# 我最大的几个群组
facebook-cli groups search --sort-by LARGEST --limit 5

# 我担任管理员的群组
facebook-cli groups search --role admin

# 搜索已连接及公开群组
facebook-cli groups search --keywords "hiking" --role any

# 在列出成员关系时查看下一页
facebook-cli groups search --role connected --after "<上一次响应中的游标>"
```

**响应：** 每个页面将结果保存在 `groups` 中；当还有下一页时，会提供 `paging.cursors.after`。

**组字段：**
- `group_id`: 数字型组 ID
- `name`: 组名称
- `member_count`: 成员数量
- `privacy`: `Public` 或 `Private`
- `group_url`: 该组的直接链接（例如 `https://www.facebook.com/groups/<group-id>/`）

### 在组中搜索帖子

```bash
facebook-cli groups posts --group-id <id> [--query <text>] [--sort-by <sort>] [--limit <n>] [--after <cursor>] [--min-timestamp <unix>] [--max-timestamp <unix>]
```

**选项：**
- `--group-id`（必填）：要搜索帖子的组 ID
- `--query`（可选）：用于按内容筛选帖子的文本查询
- `--sort-by`（可选）：`MOST_RECENT_ACTIVITY`（默认）、`MOST_REACTS`、`NEW_POSTS`
- `--limit`（可选）：最大返回结果数（默认 10，最大 25）
- `--after`（可选）：来自上一次响应的分页游标——传递它以获取下一页（每页最多 20 条）
- `--min-timestamp`（可选）：帖子时间的下限，Unix 时间戳（不包括该时间），仅返回晚于此时间的帖子
- `--max-timestamp`（可选）：帖子时间的上限，Unix 时间戳（包括该时间），仅返回等于或早于此时间的帖子

> **注意：** 分页功能（`--after`）仅支持 `MOST_RECENT_ACTIVITY` 和 `NEW_POSTS`（每页最多 20 条）。`MOST_REACTS` 不会返回分页游标，因此无法进行分页。

**示例：**  
```bash
# 某组的最新帖子
facebook-cli groups posts --group-id "<group-id>"

# 最受欢迎的帖子
facebook-cli groups posts --group-id "<group-id>" --sort-by MOST_REACTS --limit 5

# 最新帖子优先
facebook-cli groups posts --group-id "<group-id>" --sort-by NEW_POSTS --limit 5

# 搜索关于某个主题的帖子
facebook-cli groups posts --group-id "<group-id>" --query "recipe"

# 下一页（传递上一次响应中的游标）
facebook-cli groups posts --group-id "<group-id>" --after "<上一次响应中的游标>"
# 时间窗口内的帖子（Unix时间戳，单位：秒）
facebook-cli groups posts --group-id "<group-id>" --min-timestamp 1750000000 --max-timestamp 1752000000
```
