---
name: "ticketmaster"
description: "在Ticketmaster上搜索活动及带价格的座位信息。返回即时购买链接，引导用户前往Ticketmaster结算页面；该工具本身无法完成购票。"
metadata: { "不包含在提示中": 假 }
---
# Ticketmaster

## 目的
搜索活动并获取智能的Ticketmaster座位推荐。

对于面向用户的活动票务搜索或购买流程，请先阅读  
`/opt/hatch/skills/booking/SKILL.md`、  
`/opt/hatch/skills/booking/references/tickets.md`，以及  
`/opt/hatch/skills/booking/references/presentation.md`。这些文件定义了路由、展示、浏览器结账及承诺规则。本文件用于Ticketmaster CLI接口规范。在预订流程中，请勿调用`seat-view-carousel`、生成HTML或使用`widget.create`。请使用简洁的Markdown门票表格，并可选使用经过验证的Markdown图片。
请勿为门票选择或结账决策调用`create_options`。

## 工具命令

```sh
ticketmaster <子命令> [选项]
```

#### search-events
按关键词（必填）、地点和日期范围搜索活动。支持`--country-code`（默认：US）、`--page`（从0开始计数）和`--sort`（例如`date,asc`、`relevance,desc`、`name,asc`）。`--start-date`和`--end-date`接受ISO 8601格式；仅输入`YYYY-MM-DD`时，CLI会将其标准化。

```sh
ticketmaster search-events --keyword "Taylor Swift" --city "Los Angeles" --size 10
ticketmaster search-events --keyword "Lakers" --start-date 2026-05-01 --end-date 2026-06-01 --state-code CA --sort date,asc
```

#### event-details
获取由`search-events`返回的特定活动ID的完整详情。

```sh
ticketmaster event-details --event-id vvG1IZ_AKnSEae
```

#### top-picks
按价格或质量排序获取座位推荐。默认`--selection Any`返回标准票、转售票及白金票。使用`--selection Standard`可排除转售票。当可用时，返回`venue_map_url`及每个推荐的`snapshot_image_url`。请保留这些URL，以便用于验证过的Markdown链接或图片。

```sh
ticketmaster top-picks --event-id vvG1IZ_AKnSEae --quantity 2 --sort listprice
ticketmaster top-picks --event-id vvG1IZ_AKnSEae --quantity 4 --sections "MEZZ,ORCH" --sort quality
ticketmaster top-picks --event-id vvG1IZ_AKnSEae --quantity 2 --price-min 50 --price-max 150 --limit 10
ticketmaster top-picks --event-id vvG1IZ_AKnSEae --quantity 2 --selection Standard --sort listprice
ticketmaster top-picks --event-id vvG1IZ_AKnSEae --quantity 2 --areas "10,11" --ticket-type-id 000000000001
```

#### seat-view-carousel
根据top-picks JSON生成轮播图HTML组件。以完整的top-picks JSON输出作为`--input`参数。将原始HTML输出到标准输出。

```sh
ticketmaster seat-view-carousel --input '<top-picks JSON>' --title "湖人队 vs 雷霆队" --subtitle "Paycom Center · 5月13日 · 晚上8:30"
```

## 输出
各命令返回来自Ticketmaster的JSON数据。请勿期望在接口响应外再包裹一层`ok`字段。

CLI会将下划线命名的查询参数映射为Ticketmaster API使用的驼峰命名参数。响应体为Ticketmaster的原始数据，除非本CLI为本地渲染添加了明确的便利字段。

**search-events**：原始Ticketmaster Discovery搜索响应。活动信息位于`_embedded.events[]`中；分页信息位于`page`字段。对于每项活动，使用`id`作为后续调用的活动ID，`name`作为标题，`url`作为公开的Ticketmaster页面链接，`dates.start.localDate`、`dates.start.localTime`及`dates.start.dateTime`用于时间信息，`_embedded.venues[0]`提供场馆/城市/州信息，`priceRanges[]`则包含Ticketmaster提供的搜索层级定价信息。每项活动可能包含`tmol_available`字段；若其值为`false`，则会省略`url`及`tmMarketPlace`出口链接，而`box_office_url`将包含经处理的场馆链接或为`null`。若未提供`tmol_available`，则表示是否可售未知。

**event-details**：原始Ticketmaster Discovery活动对象。可直接读取以下字段：`id`、`name`、`url`、`dates.start.*`、`dates.timezone`、`dates.status.code`、`priceRanges[]`、`classifications[]`、`images[]`、`_embedded.venues[]`、`_embedded.attractions[]`以及`_links`。定时事件读取会添加 `event_starts_at` 和 `event_ends_at` 字段，分别以标准 UTC 时间和用户本地时间两种格式呈现，并附带由运行时生成的 `retrieved_at` 字段。在响应时优先使用用户本地时间格式；仅包含日期的事件字段仍保持为日期类型。

**top-picks**：原始 Ticketmaster Top Picks 响应，新增 CLI 便捷字段，以便在可用时进行轮播展示。

Top-picks 包含 `picks[]`、`_embedded.offer[]`、`eventDetails` 和 `page`。座位级定价信息位于 `_embedded.offer[]` 中，以 `offerId` 为键；每个精选项通过 `picks[].offers[]` 引用相关优惠。请将首个引用的匹配优惠作为主要显示价格。`_embedded.offer[].totalPrice` 是**每张票**的价格，而非所请求数量的总价。请将该价格乘以所请求的数量，并将计算出的整桌/整场金额作为主要价格展示；仅在必要时单独标注每张票的价格。切勿将单张票的 `totalPrice` 标注为双人或整桌总价。如果 `eventDetails.allInclusivePricing` 为 true，则说明显示的每张票价格及计算出的整桌总价已包含费用，并在存在时显示 `eventDetails.listingsDisclaimer`。`faceValue` 仅可作为补充信息展示，不得作为主要价格。

Ticketmaster 可能同时返回驼峰命名和蛇形命名的重复字段。读取字段时请按以下优先级使用：
- 结算 URL：`pick.redirect_url || pick.redirectUrl`
- 座位图片：`pick.snapshot_image_url || pick.snapshotImageUrl || pick.snapshotURL`
- 价格：`pick.total_price || _embedded.offer[offerId].totalPrice`

CLI 可能为轮播展示新增以下便捷字段：`venue_map_url`、`snapshotURL`、`snapshot_image_url`、`redirect_url`、`total_price`、`face_value` 和 `currency`。

每个精选项可能包含：`{ type, selection, section, row, seats, area, quality, descriptions, listingDetails, offers, snapshotImageUrl, redirectUrl }`。

**seat-view-carousel**：旧版 HTML 输出，请勿在预订流程中使用。图片由 Ticketmaster 托管，例如 `https://app.ticketmaster.com/maps/geometry/...`。结算 URL 必须是有效的 HTTPS 地址；当返回时，带有预选座位的 `https://ticketmaster.evyy.net/...` 联盟跳转 URL 亦属有效。

请勿在预订流程中生成或渲染轮播组件或“立即购买”列表小部件。请将三至五个具体的票务组合置于预订技能的 Markdown 表格中。经验证的 `snapshot_image_url` 可以在表格下方以普通 Markdown 图片语法展示，并附上准确的区域标签。请将每个有效的结算 URL 以简洁的 Markdown 链接形式置于相应选项或下一步操作中。

## 认证
访问 Ticketmaster 服务无需进行用户设置。

## 操作规则
1. 在调用 `top-picks` 之前，先使用 `search-events` 查找活动 ID。
2. 当用户要求“便宜”的门票时，使用 `--sort listprice`；当用户想要“最佳”座位时，使用 `--sort quality`。
3. 搜索座位区域时，不要猜测区域名称。场馆对地面层、夹层、乐池、舞台前区和上层等区域的命名并不统一。请先执行一次范围较广的查询，使用 `--sort quality --limit 20`，且不添加 `--sections` 过滤条件。在后续的 `--sections` 查询中，再根据返回的区域名称和分区标签进行筛选。
4. 当存在有效的结账 URL（`redirect_url || redirectUrl`）且显示了价格时，为该选项附上一个简洁的 Markdown 链接：`[在 Ticketmaster 上查看](<redirect_url>)`。Ticketmaster 的联盟链接，如 `https://ticketmaster.evyy.net/...`，均视为有效结账 URL。如果结账 URL 不存在或无效，或者无法获取价格信息，则不得虚构结账链接，也不得将该座位标示为可直接购买。
   请将每个链接与对应的座位选项绑定，并在发送前核对其优惠和座位参数。切勿将一个选项的 URL 用于另一个选项；对于未经验证的链接，请予以省略。
5. 对于转售门票（`selection: "resale"`），如有 `listingDetails` 描述，请一并展示，并注明这些是经过验证的转售票。
6. 如果所选优惠包含 `limit` 对象，则应遵守其 `min` 和 `max` 数量限制。
7. 如果 `eventDetails.allInclusivePricing` 为真，则说明显示的价格已包含所有费用。如有 `eventDetails.listingsDisclaimer` 或 `eventDetails.importantInformation`，请一并展示。
8. 在 `top-picks` 返回结果后，请参照 `/opt/hatch/skills/booking/references/presentation.md` 中的“活动门票”部分进行呈现。
9. `search-events` 返回的 `priceRanges[]` 常常缺失或不完整。切勿仅凭 `search-events` 的结果就告知用户“无定价信息”。请使用 `top-picks` 获取当前定价。
10. 在 `event-details` 中使用 Discovery API 返回的原始 `id`（来自 `_embedded.events[]`），并在调用 `top-picks` 时优先使用该 ID。`top-picks` 接受 Discovery ID（例如 `vvG...`）或数字形式的几何/详情 ID（例如 `0900...`）；响应中可能会以 `eventDetails.id` 的形式回显数字 ID。
11. 如果某次搜索结果显示 `tmol_available: false`，则该活动不在 Ticketmaster 上销售。请明确告知用户，并提供非空的 `box_office_url`；切勿对该活动调用 `top-picks`。如果 `box_office_url` 为空，则说明 Ticketmaster Discovery 未提供官方购票链接。如果 `tmol_available` 缺失，则不得声称该活动不在 Ticketmaster 平台上。
12. 当用户选择门票时，请通过浏览器保持结账页面处于可用状态，并准备进入最终审核环节。询问用户是否拥有 Ticketmaster 账户并希望在结账前登录；如允许，可继续以访客身份操作。仅在遇到身份验证问题、自动化被阻止或其他实际限制导致主代理无法继续时，才移交完整的结账链接。
13. 如果响应中携带 `error` 字段，应静默丢弃该条数据：跳过相应活动或选项，不得将其展示，也不得暴露错误文本。
14. 切勿凭记忆或参考先前的报价报出价格。每次都需要重新调用 `top-picks`，并仅引用其最新结果。价格监控的定时任务必须在每次运行时获取最新的库存信息。