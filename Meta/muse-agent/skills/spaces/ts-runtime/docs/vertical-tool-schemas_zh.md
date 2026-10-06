# `ctx.tool` 垂直领域模式

事实来源：

- 合约：`sdk/src/verticals.ts`（`TOOL_*_RESULT_SCHEMA`、选项类型、`SpaceToolClient`），从 `sdk/src/index.ts` 中重新导出。
- 后端映射：`worker/src/web_search.ts`（MASE 的 `vertical_data` → 类型化结果）。

## 当前已落地 vs. 后续工作

本文档记录的是**当前已落地并完成映射的内容**：

- **天气** — 包含丰富化的当前状况、每日/逐小时预报及预警信息。
- **金融** — 最新行情（含盘口变化）及分时段的 OHLCV 历史数据。`finance(query)` 返回匹配证券的**数组**；`finance_ticker(symbol)` 返回解析后的**单个**证券（字段相同）。`ctx.tool.finance` 已被**弃用**，并从指南中移除——请使用 `finance_ticker` 获取单只股票的行情/价格/历史数据（按证券代码），而其他需求（如将公司名称解析为股票代码、比较多只证券以及分析等）则使用 `web_search`。该方法仍保留以确保现有代码继续运行，但未来将被删除。
- **体育** — 每场比赛的扁平 `items` 列表，附加标准化的 `sport`/`league`/`season` 范围信息、明确的主队/客队标识，以及解析后的 `player_statistics` 和 `team_statistics`。按运动类型区分的 `games` 联合类型仍为**后续工作**（见第3节）。
- **网络搜索** — `web_search(query)`：通用网络搜索，**无垂直领域过滤**——与代理的浏览器搜索执行相同的普通搜索。仅返回**网络**结果——这是下文原则1的例外：**不含 `vertical_data`**；所有字段均从普通网络结果映射到 `results[]` 中（见第4节）。

仍然适用的设计原则：

1. **结构信息位于 `vertical_data` 中。** 下文的类型化合约即我们从 MASE 每条结果的 `vertical_data` 有效载荷中映射的内容。显示文本（如 `summary`、`excerpt`）仅供展示，不应解析，请直接读取结构化字段。
2. **每项观测均为可空/可选。** MASE 可能省略某些字段，或仅以文本形式提供，因此构建器会将缺失数据降级为 `null`，并在有效载荷为空时输出符合模式的占位值，而非抛出异常。
3. **上限在合约中定义。** 大量列表设有 `.max(...)` 上限（如 `forecast_hourly` ≤48，`finance history.points` ≤400）；构建器会按上限截取，以避免上游数据过长导致 `.parse()` 失败。

### 尚未对接后端的请求参数

`ToolSearchOptions.until`、`ToolWeatherOptions.hourly_hours` 以及 `ToolFinanceOptions.interval` 选择器目前并非后端请求字段；后端会返回完整的预报或蜡烛图序列。两种运行时环境均在客户端应用模型选定的变体：`hourly_hours` 限制逐小时预报长度，`interval` 选择特定蜡烛图集，而 `since`/`until` 则用于限定金融历史数据的时间范围。

---

## 1. 天气

### 请求

```ts
export interface ToolWeatherOptions extends ToolSearchOptions {
  /** 将上游逐小时预报序列限制在最多此数量的点数，≤48。 */
  readonly hourly_hours?: number;
}
```

### 结果（`TOOL_WEATHER_RESULT_SCHEMA`）

`location`、`summary`、`conditions`、`forecast_days`、`forecast_hourly?`、`alerts?`、`sources`。

**已支持（已落地）的字段：**

- `conditions`: `temperature`（温度）、`unit`（单位）、`description`（描述）、`feels_like`（体感温度）、`high`（最高温度）、`low`（最低温度）、
  `humidity_percent`（湿度百分比）、`precipitation_chance`（降水概率，0–100）、`precipitation_amount`（降水量，格式化字符串，如“0.37 in”）、
  `wind`（风速）、`uv_index`（紫外线指数）、`air_quality_index`（空气质量指数）、`air_quality_description`（空气质量描述）、
  `sunrise`（日出时间）、`sunset`（日落时间）。
- `forecast_days[]`: `date`（日期）、`summary`（天气概况）、`high`（最高温度）、`low`（最低温度）、`precipitation_chance`（降水概率）、
  `precipitation_amount`（降水量）、`wind`（风速）。
- `forecast_hourly[]`（≤48条，当数据源提供逐小时预报时存在）：`time`（ISO时间）、`temperature`（温度，可能为空——今日逐小时预报中未提供该字段）、
  `description`（天气描述）、`precipitation_chance`（降水概率）、`precipitation_amount`（降水量）、`wind`（风速）。若指定`hourly_hours`，
  则返回的逐小时预报点数将被限制在该值以内；若不指定，则保持默认的最大48条。
- `alerts[]`: 当前生效的预警标题（由`{ event, severity }`对象映射而来）。

上游传入的`vertical_data`中，各项指标以`{ value, unit }`对象形式呈现（如`temperature`、`wind_speed`、`feels_like`、`precipitation_*`以及每日最高/最低气温）；
`humidity`（湿度）、`uv_index`（紫外线指数）、`air_quality_index`（空气质量指数）为纯数值；
`air_quality_description`（空气质量描述）、`sunrise`（日出时间）、`sunset`（日落时间）则为字符串。

> 通过KES天气Thrift接口、NLQ及MSL/WWW解码器工作实现上线（D107777759、D107957780、D107729843、D108000766）；已与P2371491675进行实时验证。
> （此前关于“`feels_like`和降水量需待KES后续支持”的说明已不再适用——相关字段现已上线。）

---

## 2. 财经

### 请求参数

```ts
export interface ToolFinanceOptions extends ToolSearchOptions {
  /** 选择用于映射到`instruments[].history`的K线周期。省略则仅返回行情信息。 */
  readonly interval?: "1m" | "30m" | "1d" | "1w" | "1mo";
  // `since`/`until`（继承自父类）用于限定历史数据窗口范围——详见下文。
}
```

### 返回结果（`TOOL_FINANCE_RESULT_SCHEMA`）

针对`instruments[]`中的每一项，包含：`name`（名称）、`symbol`（证券代码）、`summary`（摘要）、`price`（价格）、`currency`（币种）、
`change`（涨跌幅）、`change_percent`（涨跌幅百分比）、`market_status`（市场状态）、`as_of`（更新时间）、`url`（链接），以及可选的`history`（历史数据）。

**已支持（已上线）的字段：**

- `change` ← `entity.attributes.change`；`change_percent` ←
  `entity.attributes.percentChange`；`as_of` ←
  `entity.attributes.lastUpdatedAt`（从Unix时间戳转换为ISO格式）。
- `history`（仅在请求了特定周期时返回）：
  `{ interval, since, until, points: [{ date, open?, high?, low?, close, volume? }] }`，
  按时间顺序由早至晚排列，最多返回400条记录。对于日内周期（`1m`和`30m`），每条记录的`date`均为完整ISO时间戳，确保同一天内的K线能够区分；而`1d`/`1w`/`1mo`则使用`YYYY-MM-DD`格式。
- **历史数据窗口（`from`/`to`）** 使用继承来的`since`/`until`参数，在客户端侧对K线数据进行裁剪：`until`默认为**今天**；`since`默认为基于所选周期的回溯区间，其终点为`until`（`1m`约1天、`30m`约1周、`1d`约3个月、`1w`约1年、`1mo`约5年）。落在`[from, to]`范围之外的K线将被舍弃，且`history.since`/`history.until`会反映实际生效的查询窗口。边界按日期粒度计算（`YYYY-MM-DD`）。

`vd.candles`是`entity`的**同级顶级字段**，分为`{ daily, weekly, monthly, thirty_minute, one_minute }`五个子字段；每根K线包含`{ open, high, low, close, volume, timestamp }`（timestamp为Unix时间戳）。其中，`interval`对应关系为：`1m→one_minute`、`30m→thirty_minute`、`1d→daily`、`1w→weekly`、`1mo→monthly`。调用方只能选择其中一个周期的数据；`30m`并非由`1m`派生而来。若需查看当日或当前交易时段的图表，请选择`1m`；若需要更粗粒度的多日盘中图表，则选择`30m`。

> 目前集成尚未公开`market_status`字段，因此该字段映射为`null`。
> **月线数据目前尚未开放（今日为空）**，因此当`interval: "1mo"`时，`history`中的`points`将为空，直至该功能上线。此功能通过财经NLQ及WWW解码器工作实现上线（D107985840、D108000766）；月线数据的相关进展已在D107789956中跟踪。

---

## 3. 体育

目前已支持的是**扁平化的赛事列表**，其中包含了上游解码器在`vertical_data`顶层输出的标准化赛事范围（`sport`/`league`/`season`）以及明确的`home`/`away`标识：

```ts
export const TOOL_SPORTS_DATA_RESULT_SCHEMA = z.object({
  summary: z.string(), // 仅用于展示
  items: z.array(
    z.object({
      title: z.string(),
      summary: z.string(),       // 仅用于展示（简短正文摘录）
      url: nullableString.optional(),
      sport: nullableString.optional(),   // 规范化标识符，例如 "basketball"
      league: nullableString.optional(),  // "NBA" | "NFL" | "MLB" | …（当存在歧义时省略）
      season: z.object({                  // 简单标签或结构化对象
        label: nullableString.optional(), // 例如 "2025-26" | "2026 REG"
        year: nullableNumber.optional(),  // 例如 2025（可从整数或 "2026" 转换而来）
        type: nullableString.optional(),  // "REG" | "PST"
        name: nullableString.optional(),  // "常规赛" | "2026年世界杯"
        start_date: nullableString.optional(), // 如有提供，则为 YYYY-MM-DD 格式
        end_date: nullableString.optional(),
      }).nullable().optional(),
      teams: z.array(z.string()).optional(), // 顺序不代表主客场
      home: nullableString.optional(),    // 主队名称（在上游已拆分时）
      away: nullableString.optional(),    // 客队名称（在上游已拆分时）
      score: nullableString.optional(),      // "主队-客队"比分，例如 "90-94"（来自 home/away.score）
      status: nullableString.optional(),     // "已结束" | "进行中" | "已安排" | …
      starts_at: nullableString.optional(),  // ISO 8601 UTC 格式（仅适用于旧版数据——见备注）
      player_statistics: z.array(z.object({  // 按球员统计，各运动项目通用
        player: z.string(),
        team: nullableString.optional(),
        position: nullableString.optional(),
        stats: z.record(z.string(), z.union([z.number(), z.string()])),
      })).optional(),                        // 当数据源未提供时不存在或为空
      team_statistics: z.array(z.object({
        team: z.string(),
        qualifier: nullableString.optional(),
        stats: z.record(z.string(), z.union([z.number(), z.string()])),
      })).optional(),
    }),
  ),
  sources: z.array(toolSourceSchema),
});
```

映射说明（`worker/src/web_search.ts`）：

- `sport`、`league` 和 `status` 读取规范化后的顶层 `vertical_data` 字段
  （D108265855 来源自 KES 的 `results` 数据块）。`sport` 统一转为小写以保证标识符的稳定性；对于较旧的数据，`status` 则回退到原始的 `event.attributes.status`。
- `season` 会将扁平字符串（→ `{ label }`）或结构化对象统一归一化为一种格式；`year` 可从整数 **或** 数字字符串转换而来，且若存在则保留 `start_date` 和 `end_date`。若缺失则为 null。
- `home` 和 `away` 直接读取 `vertical_data.home` 和 `vertical_data.away` 中明确指定的球队名称；上游数据以 `{name, score}` 形式传递。`score`（“主队-客队”）由双方得分推导得出，对于较旧的数据则回退到遗留的 `event.attributes.results` 数据块。`teams` 仍来自 `competitors`。
- `starts_at` 曾来自 `event.attributes.startDateUTC`。精简版解码器（D108695383）不再输出 `event.attributes`，因此该字段仅对旧版数据有效，其余情况均为 null。
- `player_statistics` 和 `team_statistics` 以文本形式由 WWW 解码器原样传递，并在此处解析为灵活的 `stats` 记录。两者采用相同的区块布局（经生产环境验证）：
  ```
  统计 - <label>:
    key: value
    key: value
  ```
  - **球员** — 标签中包含球队及限定词，例如 `"Ariel Hukporti (New York Knicks (Away))"` → 分离出 `player` 和 `team`。
  - **球队** — 标签格式为 `<Team> (<Qualifier>)`，例如 `"New York Knicks (Away)"` → 分离出 `team` 和 `qualifier`。  
    值可以是数字或带引号的字符串（如 `minutes: "1:52"`）。数据源（SportRadar）并非总是完整填充这些信息，因此当字段为空时会被省略。已在 NBA 生产流量中确认；若其他运动项目出现差异，需重新评估。

> **负载裁剪（D108695383）：** 解码器不再输出逐事件的原样 `summary`（约24 KB 的二进制大对象）或原始的 `event.attributes` 映射；消费者仅读取经过解析的小型字段。`summary` 字段仍保留，但现在引用的是简短的 `excerpt` 摘要（不再包含那个巨大的二进制大对象）；冗余的 `player_statistics_text` 原始回退值已被移除。

**暂未实现（不在本次变更中）：** 针对不同体育项目的 `games` 联合类型（按项目划分的周期/比分模型），以及标准化的跨体育项目的统计键——目前统计键仍按 feed 中的原样保留。这些功能将作为后续迭代逐步引入。

---

## 4. 网络搜索（`web_search`；仅限网络，无 `vertical_data`）

`web_search(query)` 是在动作中执行的通用网络搜索——这是“结构应置于 `vertical_data` 中”原则的**例外**：它**不传递任何垂直领域信息**（`verticals` 为空），与代理默认执行的浏览器搜索完全一致，且 MASE 对于普通网络结果**不返回任何 `vertical_data`**。因此，`buildWebSearchResult` 直接将 `summary.top[]` 中的网络条目映射为强类型的 `results[]` 数组（**不使用**基于 `vertical_data` 进行筛选的 `selectByVertical` 函数）。

### 结果（`TOOL_WEB_SEARCH_RESULT_SCHEMA`）

`results[]` 数组最多包含 20 条结果，按上游排序顺序排列，每条结果包含：

- `title`（标题）、`url`（链接）、`source`（发布者/主机名）、`snippet`（显示用摘要）。
- `published_at`（发布时间）——尽力推断，主要依据 URL 的路径段（如 `/YYYY/MM/DD/`）得出；仅供参考，并非权威。
- `last_updated_raw`（原始更新时间）——直接暴露原文的新鲜度表述（如“3小时前”），**从不解析**（无论正向还是反向都不可靠——存在来源过时的风险）。
- `favicon_url`（网站图标 URL）——来自外部源域的图标链接，Space 并不拥有该资源；仅以小型图标形式渲染，并提供 `onError` 回退机制，绝不能作为内容图像使用（如需真实图片，请使用 `media.generate_image` 或自行托管）。本工具无 `thumbnail` 字段——此类工具不会返回真实的页面缩略图。
- `rank`（排名）、`is_index_page`（是否为索引页，启发式判断：区分栏目/标签页/索引页与普通内容页）。

结果的 `vertical` 字段为空字符串（`""`），表示此搜索未指定任何垂直领域。

### 路由

`web_search` 在动作中运行，与代理的浏览器搜索采用相同的查询逻辑，适用于任何主题的网络查询。如需生成叙述性摘要，可在必要时调用 `ctx.inference.complete` 对结果进行总结。若需获取股票行情、价格或历史数据，请使用 `finance_ticker`；若需获取赛事比分、赛程或统计数据，请使用 `sports_data`。不要仅为执行搜索而创建一个 Space 任务——`spawnTask` 也会执行同样的搜索。请将任务专门用于代理循环在搜索之外所增加的操作，例如打开并阅读完整页面（`browser.open`）、跨多个站点浏览、多步骤研究或其他代理工具。