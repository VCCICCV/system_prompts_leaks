---
name: "peloton"
description: "连接Peloton，浏览健身课程、查看课程表并预约锻炼。"
icon: "peloton"
metadata: { "不包含在提示中": 假 }
---
# Peloton

## 用途
连接到 Peloton，以浏览点播和直播健身课程、管理课程表并预约锻炼。

## 工具
直接通过 `PATH` 使用已安装的 CLI。

核心认证命令：
- `peloton status`
- `peloton authorize-url`
- `peloton disconnect`

### 课程浏览
- `peloton ride-archived [--limit 10] [--page 0] [--fitness-discipline cycling] [--duration 1800] [--sort-by popularity] [--difficulty-level beginner] [--has-closed-captions true] [--super-genre-id <ID>] [--browse-category cycling] [--class-type-id <ID>] [--instructor <ID>] [--desc true]`
- `peloton ride-live [--limit 10] [--schedule-type live] [--exclude-complete true] [--days 7] [--fitness-discipline cycling] [--super-genre-id <ID>] [--browse-category cycling] [--class-type-id <ID>] [--instructor <ID>] [--desc true]`
- `peloton search-class --query "<text>" --limit 10 --page 0 [--include-raw] [--include-schema]` — 通过自然语言或合作伙伴提供的关键词（如“坐姿力量”）搜索课程。始终传递 `--limit` 和 `--page`；除非用户要求不同的本地结果窗口，否则使用 `--limit 10 --page 0`。仅在检查响应结构时才使用 `--include-raw` 或 `--include-schema`。

响应结构：课程浏览命令返回一个可供展示的 `body.classes[]` 数组。`search-class` 还会返回本地窗口相关字段：`body.count`、`body.total`、`body.page`、`body.limit`、`body.has_more` 以及 `body.pagination_mode: "local_window"`。Peloton 搜索不支持 API 分页；`--limit` 和 `--page` 只是在客户端对返回的结果集进行切片。若需不同结果，请优化查询条件，而非尝试获取其他提供方的页面。原始提供方数组仅会在请求它们的调试命令中出现。

课程读取会保留 Peloton 的原始纪元时间字段，并在源字段存在时添加语义化的 `class_scheduled_start_at`、`class_starts_at`、`class_ends_at` 和 `class_originally_aired_at` 值，同时提供 UTC 和用户本地时间两种格式。请优先使用语义化字段。

#### 筛选器 ID
- `peloton metadata-mappings` — 返回所有筛选器分类：导师、课程类型、器材、运动门类、难度等级和内容形式。请使用返回的 ID 作为 `--super-genre-id`、`--class-type-id` 和 `--instructor` 筛选器的值，而非硬编码。

### 课程安排
- `peloton schedule-event --ride-id <ID> --scheduled-start-time <EPOCH>` — 安排一次点播课程
- `peloton schedule-event --join-token <TOKEN>` — 安排一节直播课程
- `peloton delete-scheduled-event --join-token <TOKEN>` — 删除已安排的课程
- `peloton reschedule-event --join-token <TOKEN> --scheduled-start-time <EPOCH>` — 更改课程时间

### 深层链接
- `peloton deeplink --class-id <ID>` — 生成用于在 Peloton 中查看该课程的链接

## 认证
Peloton 是一项基于 OAuth 的技能。对于常规的课程读取操作，直接调用请求的读取命令，不要添加 `peloton status` 的前置检查。如果某个命令报告“未连接”，或者用户要求管理连接：
1. 执行 `peloton status`。
2. 如果状态为“未连接”，则先完成连接流程。
3. 在执行 `peloton authorize-url` 之前，需征得用户的明确确认：
   - 明确告知服务提供商名称：“Peloton”；
   - 警告：此操作将存储 Peloton 的持久访问令牌。在您本地断开连接或在 Peloton 设置中撤销授权之前，您的助手将持续拥有访问权限。
   - 连接提示中绝不能显示原始的 OAuth 范围字符串或范围名称。
4. 只执行一次 `peloton authorize-url`。当返回 `connect_url` 时，将 `<connect_url>` 替换为该 URL，并按原样分享以下 Markdown 链接：`[Connect Peloton](<connect_url>)`；切勿单独粘贴原始 URL。
5. 在等待用户批准链接或回调完成期间，不得再次调用 `peloton authorize-url`；应使用 `peloton status` 来检查连接的待处理状态。
6. 回调完成后，再次执行 `peloton status`，仅当状态变为“已连接”时才继续操作。如果状态仍为“未连接”，则告知连接仍在等待或已失败，并询问是否需要生成新的链接。

凭证存储：
- 绝对不得打印 `client_secret`、`access_token` 或 `refresh_token`。

## 操作规则
1. 对于账号绑定，请遵循认证部分的规定。
2. 除非用户明确要求查看技术或调试信息，否则不得在面向用户的结果中显示原始的内部标识符、难度评分、星级评价、评价数量或课程 URL。应使用课程标题、教练姓名和时间作为备用文本。
3. 对于自然语言的课程搜索，调用 `peloton search-class`，参数设置为 `--limit 10 --page 0`。仅在用户请求其结构化筛选条件或浏览未被搜索过的已归档课程时，才使用 `ride-archived`。将返回的 `body.classes[]` 通过 `widget.create` 渲染，设置 `kind: "list"`，并在 `data` 中包含列表标题和 `items`。将每节课的标题放入 `title`，教练和 `duration_minutes` 放入 `subtitle`，如有相关课程时间则放入 `tertiary_title`。若存在 `thumbnail_url`，则将其复制到 `image_url`。对于有返回 HTTP(S) `deeplink_url` 的情况，使用 `type: "link"` 并将 `data.url` 复制自该链接；若无此类链接，则使用 `type: "generic"`. 将返回的 `embed_token` 纳入回复。不得以纯文本形式重复课程列表。若无课程，则明确告知；若小部件创建失败，则使用简洁的 Markdown 列表，列出标题、教练和时长。不得以文本形式输出课程 ID 或图片 CDN URL。
4. 对于其他课程浏览指令，同样采用上述列表呈现方式，使用 `body.classes[]`。仅在用户请求调试时才使用原始的服务提供商数组。不得自行构造课程 URL。
5. 显示课程时长时，请使用 CLI 提供的 `duration_minutes` 字段。
6. 对于直播课程，默认使用 7 天的时间窗口，即设置 `--days 7 --exclude-complete true`。
7. 在用户提出明确且无歧义的请求时，可直接进行课程的预约与取消操作，无需额外确认。
8. 当出现令牌过期错误时，系统会自动刷新令牌。若自动刷新后 API 调用仍然失败，请再次检查 `peloton status`，必要时重新绑定。