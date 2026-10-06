# Facebook 活动

搜索 Facebook 活动并查看单个活动的详细信息。

## 命令

### 搜索活动

```bash
facebook-cli events search [--scope <范围>] [--keywords <文本>] [--location <地点>] [--latitude <纬度> --longitude=<经度>] [--radius-in-miles <距离>] [--category <类别>] [--start-date <YYYY-MM-DD>] [--end-date <YYYY-MM-DD>] [--limit <数量>] [--after <游标>]
```

**选项：**
- `--scope`（可选）：`connected`（默认）——您与之相关的活动（已报名、感兴趣、受邀、主办）；`discover`——来自推荐后端的热门附近活动
- `--keywords`（可选）：自由文本搜索，例如 `"music"`、`"farmers market"`
- `--location`（可选）：地点字符串，例如 `"Seattle"`。在发现模式下，将围绕该地点而非您的位置进行搜索。服务器会将其地理编码为**市中心**，因此无法精确到社区或具体地址。
- `--latitude` / `--longitude`（可选，仅限发现模式）：精确的 WGS-84 坐标搜索中心。必须同时提供或都不提供，单独使用会导致错误。坐标对优先于 `--location`。当您拥有真实坐标时，请务必使用此选项，其精度高于任何地点字符串。
- `--radius-in-miles`（可选，需配合坐标对）：以坐标为中心的搜索半径，默认为 25 英里。
- `--category`（可选）：活动类别名称，例如 `MUSIC_AND_AUDIO`
- `--start-date` / `--end-date`（可选）：日期范围（`YYYY-MM-DD`），开始日期不得晚于结束日期。
- `--limit`（可选）：每页最多显示的活动数（默认 10，最大 25）。
- `--after`（可选）：上一次响应中的分页游标（`paging.cursors.after`）。游标与其搜索范围绑定，不可跨范围重复使用。

**示例：**
```bash
# 我的即将举行的活动
facebook-cli events search

# 在热门附近活动中按关键词搜索
facebook-cli events search --scope discover --keywords "music" --limit 5

# 某地附近且在指定日期之后的活动（市中心级别精度）
facebook-cli events search --scope discover --location "Seattle" --start-date 2026-07-31 --limit 5

# 精确坐标附近的活动，半径 5 英里内（已知坐标时优先使用）
facebook-cli events search --scope discover --latitude 37.5072 --longitude=-122.2605 --radius-in-miles 5 --limit 5

# 下一页（传入上一次响应中的游标）
facebook-cli events search --scope discover --keywords "music" --after "<上一次响应中的游标>"
```

**选择地点：**
1. 如果当前上下文已包含用户的坐标（例如 `message_location` 标记），请直接将其传递给 `--latitude` / `--longitude`。不要先将其反向地理编码为地点名称——这样做会丢失本命令旨在利用的精度。
2. 如果用户提供了地点名称，请使用 `map.geocode` 进行地理编码并传递坐标；或者在城市级精度足够时，直接将地点字符串传递给 `--location`。
3. 如果没有任何地点信息，请询问用户。在发现模式下，若未提供地点，则系统会回退至服务器端的猜测，且响应中不会说明具体使用了哪个地点。

**负坐标：** 请使用 `--longitude=-122.2605` 的格式，即等号前后不留空格。如果写成 `--longitude -122.2605`，会被误认为另一个标志而报错 `error: unexpected argument '-1' found`。南半球的 `--latitude` 同理。**每个事件的响应字段：**
- `id`：事件的原始数字 ID（全局，非应用范围——与 `permalink_url` 中的 ID 一致）
- `name`：事件名称
- `permalink_url`：可分享的链接，始终为 `https://www.facebook.com/events/<id>/`
- `description`：事件描述（可能为空）
- `start_time` / `end_time`：带时区偏移的时间戳（例如 `2026-08-01T11:00:00-0400`）
- `location`：场地或地址文本（可能为空）
- `category`：类别名称（可能为空）
- `going_count` / `interested_count`：参与人数统计
- `timezone`：IANA 时区（可能为空）
- `privacy`：`public`、`private` 等（可能为空）
- `is_online`：该事件是否为线上活动
- `online_url`：线上活动的第三方托管网址。仅在详细信息响应中返回——搜索结果则完全不包含此字段，而非返回 `null`

响应采用分页机制：活动数据位于 `data` 下，当还有更多页面时，下一页的游标位于 `paging.cursors.after`。

### 获取事件详情

```bash
facebook-cli events details --event-id <id>
```

**选项：**
- `--event-id`（必填）：事件的原始数字 ID——即搜索结果中的 `id`，或嵌入在事件 `permalink_url` 中的数字

这是唯一会返回 `online_url` 的事件接口。如果 ID 格式错误，则返回 400；如果事件不存在或您无权查看，则返回 404（出于设计考虑，这两种情况无法区分）。

**示例：**
```bash
# 查看一个事件（ID 取自搜索结果）
facebook-cli events details --event-id 1679757800076464
```

**响应字段：** 与搜索结果相同，外加 `online_url`（当该事件没有第三方在线链接时为 `null`）。