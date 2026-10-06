# Meta 广告——选择合适的工具与参数

**本文件仅涉及读操作。** 写操作——创建、更新、激活、删除、连接、上传——均在 `references/writes.md` 中，同时也列出了每种操作会破坏的内容；选择写操作及其影响范围是同一个决策。

请对照本次对话中成功的 `meta-ads-cli list-tools --names-only` 结果，逐一确认此处列出的每个工具名称，并在使用前通过 `meta-ads-cli describe-tool --name <tool>` 检查所选工具的详细信息。在同一对话中，可复用之前成功发现并描述过的工具。旧版 CLI 的列表、状态输出或目录无效。服务器目录按工具粒度进行访问控制。切勿使用单纯的 `list-tools` 或 `call-tool` 探测来发现工具。下方列出的某个工具名称可能对该用户不可见；而对该用户可见的名称也可能不在下方。真正持久不变的是错误的“模式”——这些模式会在各个目录版本间反复出现。

## 首先明确对象归属哪个系统

只有在确认请求属于 Meta 广告后，才进入此流程。不要仅因用户提到“目录”、“信息流”、“产品”或“受众”就进入该流程；这些名词也用于其他产品。广告的所有权需由明确的广告场景、特定于广告的对象、之前的对话，或一次只读查询来确定，该查询会将提供的名称/ID 解析到 Meta 广告账户内。

即使没有先前的广告上下文，特定的目录或产品信息流名称/ID 也会触发这种只读查询。同样，当明确指定了某个信息流，并伴随上传或刷新计划请求时，例如“把我的 Shopify 夜间信息流改为每小时刷新”，也会触发此类查询。在搜索定时任务、钩子、跟踪项、提醒或本地脚本之前，请先执行广告查询；这些都不是广告信息流是否存在的确凿证据。先解析出必要的父级对象，再用相应的广告列表/读取工具查找唯一匹配项。找到唯一匹配即可确认广告所有权。如果没有匹配或存在多个合理候选，则只需提一个简短的澄清问题，且无需任何写操作。切勿仅凭对象名称中的“Shopify”等词推断其归属。像“更新我的信息流”这样的泛化请求不足以触发广告写操作。

一旦在 Meta 广告中确定了某个目录、产品信息流、产品集、像素/数据集、自定义受众、广告系列、广告组、广告或创意素材，后续关于该对象的所有读写操作都应在此界面完成。通用的信息流或电商工具不能作为缺失广告工具的替代方案。务必先检查实时广告目录；若所需功能不存在，应明确告知。

几乎所有失败情况可归结为以下三种模式：

- **分析层级因工具而异。** 随着上线策略的变化，不支持的值可能会被拒绝、映射或返回无用的行。请使用实时 Schema，切勿将空的分析结果视为账户无数据的证据。
- **被拒绝的参数会导致调用失败**，因此它绝不能证明广告主不存在该对象。
- **猜测的实体 ID 不会报错**——它会返回*用户提问下另一个对象的数据*，这看起来像是答错了，而非调用失败。

**请勿调用 `ads_agent`。** 如果目录中提供了该工具，其描述会说明它是首选工具，并取代了本文件所介绍的一系列 `ads_get_*` / `ads_insights_*` 调用。它是一种代理间委托的原型，但出于非政策层面的原因，它超出了本文讨论范围：它返回的是*合成的答案*而非原始数据行，因此你从中报告的每一个数字都是未实际获取的，无法与任何数据核对。`references/evidence.md` 中的每一条证据规则都假定你已亲眼看到数据。请自行构建调用链。

## 产品概念与操作类问题

对于 Meta Ads 的概念、设置、规格说明或通用操作类问题，请先调用 `ads_get_help_article`，并基于其返回结果进行解答。请勿依据模型记忆、浏览器搜索或通用的账户/实体查询来作答。对于纯粹的定义类问题，如“生命周期预算是什么意思？”，无需进行账户信息检索。政策类问题应按照 `references/policy.md` 中的检索顺序处理。

## 效果与诊断

| 用户询问的内容 | 调用工具 |
|---|---|
| 标准投放效果概览，或账户/广告系列/广告组/广告的对比与排名——无论请求多少指标 | 在问题所指层级调用 `ads_get_ad_entities` |
| 某一指标变动的原因，包括被描述为“突然”的下降或上升；CPC/CPM/每次成效成本/ROAS/点击率/CVR 的趋势或时间序列 | `ads_insights_performance_trend` |
| 账户是否存在已检测到的异常或异常信号，且未指定某单一指标随时间的变化情况 | `ads_insights_anomaly_signal` |
| 竞价竞争力——质量或出价排名，广告为何未充分触达，受众重叠情况 | `ads_insights_auction_ranking_benchmarks` |
| 账户与同类广告主或行业相比的表现如何 | `ads_insights_industry_benchmark` |
| 哪种优化目标或出价策略更适合其业务与转化链路 | `ads_insights_advertiser_context` |
| 尚未创建的广告系列所需预算，以及每次成效的成本——“我该为此投入多少？” | `ads_budget_estimate` |
| 预算或出价设置是否限制了投放量——“是否有受预算限制的情况？”、“还有哪些可扩展的空间？” | `ads_insights_budget_scaling_analysis` |
| 预算结构是否影响投放效率——“是否应该合并？”、“广告组是否过多？” | `ads_insights_budget_liquidity_analysis` |
| 不同预算水平会带来怎样的效果——“在每个预算档位预期会怎样？” | `ads_insights_budget_allocation_simulation` |
| 账户级机会分值及其建议 | `ads_get_opportunity_score` |
| 近期账户变更、编辑历史、审计日志，以及谁在何时进行了哪些修改 | `ads_account_get_activity_logs` |
| 账户或投放错误，广告未投放的原因 | `ads_get_errors` |
| 可查询的字段、细分维度或筛选运算符 | `ads_get_field_context` |
| 指标的定义或公式解释 | `ads_get_metric_definition` |

请根据意图匹配工具，而非逐字对照。当一个问题确实涉及多个意图时，可以调用多个工具，但应优先选择最能直接解答该问题的单一工具。

### 专业读取是首要调用
当上表中的某一行与用户问题匹配时，请优先调用该工具，再考虑其他通用或邻近的 Ads 工具。如果用户请求中已提供 `ad_account_id` 或实体 ID，则这些信息已可用：在所选工具的实时架构支持的情况下，请直接传递。切勿仅为了获取上下文而将 `ads_get_ad_accounts`、`ads_get_ad_entities`、`ads_get_field_context`、机会分值、异常检测、趋势分析、扩量分析或其他分析工具作为前置步骤。

专业工具之间不可互换：
- “该指标为何随时间下降/上升？”属于效果趋势分析，即使用户使用“突然”一词。“是否存在异常？”则属于异常信号分析。
- “不同预算会带来什么效果？”或“是否值得增加预算？”属于预算分配模拟。当前投放受限属于扩量分析；预算分散、合并及 ABO/CBO 结构问题则属于预算流动性分析。
- “与行业或同类广告主相比如何？”属于行业基准分析，而非内部实体间的对比。
- Meta Ads 的定义、设置步骤或产品行为应通过帮助文档读取，而非依赖浏览器搜索或凭记忆作答。仅当用户请求第二个独立结果、确实缺少必要ID，或主工具的成功输出指明了某个特定的证据缺失时，才添加另一项广告数据只读调用。切勿预先进行过度扩展。邻近模拟或工具的失败并不意味着主专业能力不可用。
如果主专业调用成功并满足了用户意图，则停止调用广告数据，并直接生成回复。切勿将简短或看似合成的结果视为调用趋势、实体、领域上下文或账户发现工具以进行信息补充的许可。尤其要注意，成功的异常检测结果已回答了异常检测请求；只有在用户同时询问某指定指标随时间变化的原因时，才添加趋势调用。

### 分析层级

洞察类工具接受`analysis_level`参数，而`ads_get_ad_entities`则接受`level`参数。切勿在这两类参数之间传递字面值：实体层级使用小写，而暴露层级参数的洞察类工具则声明大写的、工具特有的取值。部分洞察类工具根本不暴露层级参数。应遵循各选定工具的实际Schema，在无`analysis_level`时予以省略，并按每个层级分别发起调用，而非将多个层级合并于一次调用中。
对于单纯的账户KPI或标准投放概览，应在所请求的时间窗口内，以`level=ad_account`调用`ads_get_ad_entities`。对于诸如“为何我的账户每次效果成本突然上升？”之类的分析性问题，仅在该工具实际支持的层级上调用。空的分析结果并非账户范围内的无数据状态：在声称任何内容缺失之前，应先通过`ads_get_ad_entities`按所请求的实体层级读取确切的请求时间窗口。若该层级未定义相关指标，则应下探一级读取，并明确说明范围更窄，而非将其重新归为账户总计。

**预算类工具具有阶段特异性。** 对于尚未创建的提案，请使用`ads_budget_estimate`；针对现有广告系列的假设情景模拟，请使用分配模拟；对于当前投放上限的扩量分析，请使用扩量分析工具；而对于碎片化分析，请使用流动性分析工具。切勿互相替代。对模拟结果应标注其假定预算下的预测值；当投放分析未返回任何结果时，应明确指出缺失的内容，而非单纯描述查询过程。

### 读取`ads_get_ad_entities`- 选择与问题相匹配的层级：`ad_account` 获取总体视图，
  `campaign` 用于比较广告系列，`adset` 或 `ad` 用于深入分析。请在用户指定的精确层级上作答。
- 仅获取您的答案将展示的行。当符合条件的实体多于一条回复所能呈现的数量（约20条）时，“哪些”“每个”“每一种”或“全部”并不意味着获取所有结果：请按问题所关注的指标排序（例如 `amount_spent_descending`），并设置一个接近您将展示数量的 `limit`，从这些行中作答，并说明已展示的数量以及还有更多结果存在。只有在用户明确要求时，才以最大 `limit`（不超过1000）获取其余结果。“复数”形式的提问（如“哪些广告组……”）仍可能返回多条结果，除非您明确指出仅有一条符合条件。
- 仅请求问题所需的字段，而非全部字段。
- 保留用户的时间范围设定。若实时 Schema 将其命名时间窗口暴露为 `date_preset`，则使用该预设，而非自行计算日历日期；服务器负责处理包含性及广告账户的时区。对于明确的日历日期或无匹配预设的情况，请使用 `time_range`。进行比较时，优先使用 Schema 原生的比较方式，或分别查询各时间段。对于“自始至终”或“整个生命周期”，请使用 `maximum`，而非 `data_maximum`：后者可能导致结果与已不再保留的花费关联，从而报告每结果的花费和成本均为0。
- 将状态词视为查询条件，而非需显示的字段。若用户查询处于“活跃”“运行中”“上线中”“仍在投放”状态的对象，或排除“暂停”“停止”“关闭”的对象，请先调用 `ads_get_field_context` 获取 `effective_status` 的上下文信息，然后在每次适用的 `ads_get_ad_entities` 调用中传递其支持的过滤器。当前该过滤器的格式为：
  `"filtering":[{"field":"effective_status","operator":"IN","value":["ACTIVE"]}]`
  单纯在 `fields` 中请求 `effective_status` 并不能对结果行进行过滤。
- 仅在用户询问某现象发生的原因或希望获得细分视图时，才使用分组维度（如投放位置、年龄、平台）。
- 当不确定某字段是否存在于所需层级，或在响应的 `additional_info` 报告该字段不被支持时，请调用 `ads_get_field_context`（仅需传入 `field_names`）。切勿凭空捏造字段名：直接舍弃该字段，或使用工具确认过的字段重新查询。
- 在 `filtering` 中，每个条目的 `value` 均为数组，即使仅包含一个值。请通过 `ads_get_field_context` 确认字段与运算符的正确性。

**不同层级的转化指标互不相同，且不可互相替代。**
`conversions` 不是账户层级的字段，而当账户拥有多种结果类型时，`cost_per_result` 在 `ad_account` 层级上也不存在。在该层级上可用的是 `cost_per_conversion`，因此当广告主询问账户层面的单次转化成本时，请始终使用此字段，切勿暗中替换为 `cost_per_result`，反之亦然。若要回答账户范围内的“每次结果成本”问题，请在 `campaign` 层级查询，按各广告单元返回的结果类型分组，仅在同一组内进行比较，并明确说明这些结果并非账户层级的汇总数据。`results` 是广告对象根据目标定义的结果，而非所有转化指标的别名，因此切勿擅自将其作为替代。
  
**`ads_get_errors` 接受 `entity_ids`，而非 `ad_account_id`。** 若要检查某个广告账户，请将账户 ID 以字符串形式放入 `entity_ids` 中。

### 如何解读机会得分

请将该得分视为账户层级指标——切勿将其归因于某个广告系列、广告组或广告。请在一句话中直接给出得分，并立即进入改进建议部分（“机会得分为74/100。主要改进建议：”）。无需解释得分的含义或高低的意义，也无需添加任何安抚性表述。得分即为原样返回的整数值；“100分制”仅为单独的量表标注。切勿写成“74/100”、“得分74%”或“74分”——它既不是百分比、分数，也不是计分点数。
按 `opportunity_score_lift` 从高到低对推荐进行排序，并将该值称为 **points**（而非“impact”）。使用 `lift_estimate` 表示预期收益，用 `recommendation_content.body` 指明需要变更的内容。若推荐包含 `url`，则将其作为执行变更的入口；若包含 `recommendation_signature`，则说明该变更可通过程序化方式应用。仅按原样引用 `lift_estimate` 和 `opportunity_score_lift`——若某条推荐没有提升数值，则应以定性方式描述收益，而不凭空捏造。

当推荐涉及具体对象时，应通过 `ads_get_ad_entities` 确保上下文明确，使建议指代实际的广告系列及其当前预算，而非泛化的抽象概念。若评分无待处理的推荐，也不应止步于此：应结合所获取的各对象的投放表现指标，从这些数据中解答用户的核心问题。

## 目录与电商

### 每种对象类型对应一个工具

不存在“列表工具”和“详情工具”可供选择。应根据用户询问的对象类型来确定使用哪个工具，然后将已有的 ID 作为 `entity_id` 传入。读取单个实体时，返回结果即为一页一行的数据。

| 用户询问的内容 | 使用工具 | 作为 `entity_id` 传入 |
|---|---|---|
| 账户或商家下有哪些目录 | `ads_catalog_list_catalogs` | 商家 ID；若省略，则列出查看者可访问的所有目录——切勿传入广告账户 ID，因为该账户无法限定此读取范围 |
| 某个目录的元数据或设置 | `ads_catalog_list_catalogs` | 该目录的 ID |
| 目录中的商品 | `ads_catalog_list_products` | 该目录的 ID |
| 某个商品的详细信息或属性 | `ads_catalog_list_products` | 该商品的 ID |
| 某个商品组内的商品 | `ads_catalog_list_products` | 该商品组的 ID |
| 目录中的商品组 | `ads_catalog_list_product_sets` | 该目录的 ID |
| 某个商品组（名称、筛选条件、数量） | `ads_catalog_list_product_sets` | 该商品组的 ID |
| 包含某个商品的商品组（反向查询） | `ads_catalog_list_product_sets` | 该商品的 ID |
| 目录下的数据源 | `ads_catalog_list_product_feeds` | 该目录的 ID |
| 某个数据源的配置或调度 | `ads_catalog_list_product_feeds` | 该数据源的 ID |
| 某个数据源的上传会话 | `ads_catalog_get_product_feed_upload_sessions` | — 需传入 `product_feed_id`，而非 `entity_id` |

**被这些工具取代的 `ads_catalog_get_*` 已被移除。**
`ads_catalog_get_catalogs`、`_get_details`、`_get_products`、
`_get_product_details`、`_get_product_sets`、`_get_product_set_details`、
`_get_product_set_products`、`_get_product_product_sets`、
`_get_product_feed_details` 以及 `ads_catalog_search_product` 均已不再存在。调用这些接口不会报错，但会返回一条成功的响应，其内容仅为说明该工具已被移除的提示语。请勿将此提示视为产品功能状态的声明，也切勿因看到该提示而告知广告主某项目录功能不可用；请改用上表中列出的相应工具重试。

### 健康、诊断与事件源

这三类功能名称相似，常被混淆，但彼此不可互换。

| 用户询问的内容 | 使用工具 | 作用范围 |
|---|---|---|
| 目录级诊断、数据源错误、商品质量问题 | `ads_catalog_get_diagnostics` | 单个目录 |
| 动态广告投放健康状况、目录与动态广告的匹配度、动态广告覆盖范围 | `ads_catalog_get_dynamic_ads_health` | 单个目录，限定在动态广告范围内 |
| 目录下像素或数据集的事件源健康状况 | `ads_catalog_event_source_get_health` | 目录下的单个事件源 |
| 目录关联了哪些事件源——“哪些像素与此目录相连？” | `ads_catalog_event_source_get`，传入 **目录** ID | 从目录指向其事件源 |
| 某个事件源关联了哪些目录——“这个像素连接了哪些目录？” | `ads_catalog_event_source_get_catalogs`，传入 **事件源** ID | 从事件源指向其所属目录 |仅凭“健康”一词就指向 `ads_catalog_get_diagnostics`（全目录范围）。
只有当问题限定于某个像素或数据集事件源时，才使用 `_event_source_get_health`；而只有在明确针对动态广告时，才使用 `_get_dynamic_ads_health`。

**这两个事件源相关工具的读取方向相反，且其名称并未明确指出这一点。** `ads_catalog_event_source_get` 接受一个目录并返回其所有事件源；`ads_catalog_event_source_get_catalogs` 则接受一个事件源并返回其所属的所有目录。将目录 ID 作为 `event_source_id` 传递，正是本文件开篇所提到的“猜测 ID”导致的错误——这并不一定会报错，只是回答了另一个不同的问题。将“与我的目录关联的像素”视为 `ads_catalog_event_source_get` 的结果；它会返回所有已连接的类型，并以 `source_type` 标记（`PIXEL`、`APP`、`OFFLINE_CONVERSION_DATA_SET`）。CAPI 是对像素或应用事件源的一种增强，而非一种独立的事件源类型，因此不应将其作为单独的类型报告。

### 链式调用

大多数原先的两步链式调用现已成为单次调用，因为反向查找均使用相同的 `entity_id`：从商品到其商品集是 `ads_catalog_list_product_sets(entity_id=<product_id>)`，从目录到其信息流则是 `ads_catalog_list_product_feeds(entity_id=<catalog_id>)`。不要在反向查找已连通的两个步骤之间插入额外的列表调用——获取某个商品的集合无需在其前先进行商品列表查询。

**但这并不意味着可以省略对问题所涉及对象的解析，而这恰恰是最危险的做法。** `entity_id` 没有默认值。如果广告主未指定目录，而您在此对话中也尚未持有该目录的 ID，请先调用 `ads_catalog_list_catalogs`——“我有哪些信息流？”这一问题并未指定任何目录。传递未经解析的 ID 并不会导致失败：系统会成功返回该目录的信息流，而广告主则可能在毫无错误提示的情况下，误将其他目录的数据当作自己的数据来读取。若返回多个目录，请在读取之前先确认具体是哪一个。

当您是从 SKU 而非 ID 出发时，仍需进行链式调用：先通过 `ads_catalog_list_products(entity_id=<catalog_id>, filter={"retailer_id":{"eq":"ABC-001"}})` 解析 SKU，再使用返回的 `product_id`。务必执行这两个步骤——切勿仅停留在查询阶段，直接将列表交给用户。

### 参数

几乎所有的错误都源于两种习惯：在工具需要目录或集合 ID 时却传入广告账户 ID，以及将 ID 复数化。

`ads_catalog_list_*` 系列读取接口均接受 `entity_id`（即待读取对象的 ID），以及 `limit` 和 `cursor` 参数。其余 `get_*` 工具则各自使用不同的 ID：

| 工具 | 必填参数 | 绝对不可传递 |
|---|---|---|
| `ads_catalog_list_catalogs` | 无——省略 `entity_id` 即可列出所有可访问的目录 | `ad_account_id`、`ad_account_ids`、`business_ids`（广告账户无法限定此读取范围；如需限定，请以单一企业或目录 ID 作为 `entity_id`） |
| `ads_catalog_list_products`、`_list_product_sets`、`_list_product_feeds` | `entity_id` | `catalog_id`、`product_set_id`、`after_cursor`、`cursor_after` |
| `ads_catalog_get_diagnostics`、`_get_data_sources`、`_get_dynamic_ads_health` | `catalog_id` | `after_cursor`、`cursor_after` |
| `ads_catalog_get_product_feed_upload_sessions` | `product_feed_id` | `catalog_id` |
| `ads_catalog_get_feed_rules` | `product_feed_id` | `feed_id`、`catalog_id` |
| `ads_catalog_event_source_get`、`_get_health` | `catalog_id` | — |
| `ads_catalog_event_source_get_catalogs` | `event_source_id` | `catalog_id` |

**`ads_catalog_list_products` 中的 `filter` 参数为可选；省略即可列出所有内容。** 过滤条件和字段名均依目录而定——请仅使用工具自身错误信息或目录 Schema 所确认的名称，切勿猜测 `product_type`、`product_feed_id`、`custom_label_0`、`images_fetch_status` 或 `description`。分页时，请严格按照上一次响应中 `cursor` 参数所返回的精确游标值进行传递；自行构建或重命名的游标将被拒绝。**`product_id` 是一个数字 ID，而不是 SKU 或产品名称。** 传入 `MB-001`、`ring` 或 `malla` 作为 `entity_id` 将被拒绝。请先通过 `ads_catalog_list_products(entity_id=<catalog_id>, filter={"retailer_id":{"eq":"MB-001"}})` 获取对应的数字 ID。

## 数据集、像素和信号

| 用户询问的内容 | 应使用（详细） | 不应使用（列表） |
|---|---|---|
| 单个数据集的设置或配置 | `ads_get_dataset_details` | `ads_get_datasets` |
| 单个数据集的事件质量或评分 | `ads_get_dataset_quality` | `ads_get_datasets` |
| 单个数据集的事件量或统计信息 | `ads_get_dataset_stats` | `ads_get_datasets` |
| 单个像素的事件配置 | `ads_pixel_event_read` | `ads_get_datasets` |
| 单个像素的参数设置 | `ads_pixel_parameter_read` | `ads_get_datasets` |
| 账户中的自定义转化 | `ads_get_customconversions` | — |

不要仅用 `ads_get_datasets` 回答“数据集 X 的健康状况如何”——该接口仅列出 ID，并不包含质量或统计信息。务必后续调用相应的详细信息工具。

| 工具 | 必需参数 | 绝对不能传递 |
|---|---|---|
| `ads_get_datasets` | `ad_account_id` 或 `business_id`——二者必选其一 | `ad_account_ids`（拒绝复数） |
| `ads_get_dataset_details`、`_quality`、`_stats` | `dataset_id` | `ad_account_id` |
| `ads_get_customconversions` | `ad_account_id` | — |
| `ads_pixel_event_read`、`ads_pixel_parameter_read` | `items` | 直接传递顶层的 `ad_account_id` 或 `pixel_id` |

**`ads_get_datasets` 必须限定作用域。** 只传入 `ad_account_id` 或 `business_id` 中的一个，是此处最常见的错误。

**`ads_pixel_event_read` 和 `ads_pixel_parameter_read` 接受的是列表，而非单个 ID。** `items` 是一项项读取请求的列表；每条记录要么指定具体的对象（如 `event_rule_id` 或 `parameter_id`），要么指定要查询的像素（`pixel_id`）。若要读取某个像素的事件配置，应传递一条包含该 `pixel_id` 的记录，而不能单独传入 `pixel_id`。

关于转化 API 的设置、操作指南及文档，请通过 `ads_get_help_article` 查询，而非本接口。目录事件源的健康状态归上文的目录部分处理。

## 受众与主页

| 用户询问的内容 | 应使用（详细） | 不应使用（列表） |
|---|---|---|
| 单个自定义受众的配置、规模或来源 | `ads_get_custom_audience` | `ads_get_ad_account_custom_audiences` |
| 哪些广告组使用了某个自定义受众 | `ads_get_custom_audience_adsets` | `ads_get_ad_account_custom_audiences` |
| 单个账户的受众库存 | `ads_get_ad_account_custom_audiences` | `ads_get_custom_audience` |
| 创建广告时所需的兴趣、地域或语言等目标定向对象 | `ads_targeting_search` | 在广告组创建时直接输入自由文本 |

如果用户以名称而非 ID 指定受众，请先通过列表接口解析出 ID，再调用详细信息工具。切勿仅停留在列表层面。

请根据作用域选择合适的主页列表工具：与广告账户关联的主页使用 `ads_get_ad_account_pages`，隶属于某个企业的主页使用 `ads_get_pages_for_business`，当前用户可访问的主页使用 `ads_get_user_pages`。

**上述大多数接口都以所读对象的 ID 为参数，且该 ID 并非广告账户 ID。** `ads_get_custom_audience` 和 `ads_get_custom_audience_adsets` 使用 `custom_audience_id`，完全不接受 `ad_account_id`。`ads_get_pages_for_business` 需要 `business_id`。`ads_get_ig_media` 需要 `ig_account_id`——请先通过 `ads_get_ig_accounts` 解析；它虽也接受 `ad_account_id`，但不能替代 `ig_account_id`。只有 `ads_get_ad_account_custom_audiences`、`ads_get_ad_account_pages` 和 `ads_get_ig_accounts` 以 `ad_account_id` 为键，而 `ads_get_user_pages` 则无需任何 ID。在创建或更新广告组之前，如果所选的兴趣、地理位置或语言仍需要平台 ID，请使用 `ads_targeting_search`。将所有搜索词批量合并为一次调用。查询时仅提供基础地名，并附上已知的国家以及语义正确的提示词：`city`、`subcity`、`neighborhood`、`region`、`zip`、`address`、`place` 或 `geo_market`；半径参数仅适用于地址/地点条件。将查询结果视为候选而非最终结果。需同时验证返回的地名、类型、所属国家及区域，以确保不会误选其他同名地点。原样传递返回的定向对象：将 `targeting_results` 用于 `targeting.interests`，`location_results` 用于 `targeting.geo_locations`，`locale_results` 用于 `targeting.locales`。内部保留原始 ID，且仅复用本账号与本次会话中的结果。若返回多个合理地点，应优先根据广告主的独特信息或经验证的主页/企业信息进行选择；否则应进一步确认。在写入数据前，请仔细检查 `unresolved_*` 字段及警告信息。未解析的精确位置并不意味着可以放宽地理范围，未解析的兴趣也不应纳入受众群体。提供的 ISO 国家代码已是标准形式，无需再进行查找。

本接口涵盖付费广告及与 Instagram 相关的功能——`ads_get_ig_accounts` 和 `ads_get_ig_media`。而关于自然流量帖子与 Reels 的互动指标、帖子内容，以及自有 Instagram 账号的管理，则属于 `instagram` 和 `instagram-messages` 技能范畴，不在本接口中。

## 实验

| 用户询问的内容 | 应使用的工具（详细） | 不应使用的工具（列表） |
|---|---|---|
| 单个 A/B 测试的设置、状态或结果 | `ads_experiment_abtest_get_test` | `ads_experiment_list_tests` |
| 单个提升测试的设置、状态或结果 | `ads_experiment_lift_get_test` | `ads_experiment_list_tests` |
| 单个账户的实验清单 | `ads_experiment_list_tests` | 任一详细工具 |
| 某账户是否具备创建实验的资格 | `ads_experiment_check_eligibility` | 任何列表或详细工具 |

若用户通过名称、状态、日期或目标来指代某个实验，而非直接提供 ID，请先列出所有实验以确定其 ID，再调用相应的详细工具。切勿仅停留在列表层面。

## 广告素材与媒体资源

| 用户询问的内容 | 应使用的工具 | 不应使用的工具 |
|---|---|---|
| 单个账户的广告素材库 | `ads_get_creatives` | `ads_get_creative_ads` |
| 哪些广告使用了某特定素材 | `ads_get_creative_ads(creative_id)` | `ads_get_creatives` |
| 预览或渲染某现有广告 | `ads_get_ad_preview` | `ads_get_creatives`、`ads_get_ad_images` |
| 账户上传的图片素材 | `ads_get_ad_images` | `ads_get_creatives` |
| 账户上传的视频素材 | `ads_get_ad_videos` | `ads_get_creatives` |

对于“哪些广告使用了我的素材 X”这类问题，务必使用 `ads_get_creative_ads(creative_id)` —— `ads_get_creatives` 返回的是素材对象，而非引用这些素材的广告。

对于已有对象，`ads_get_ad_preview` 接受 `ad_id` 或 `creative_id`，并可选指定 `ad_format`；请先通过 `ads_get_creatives` 确认其 ID。`ads_get_creative_ads` 则仅接受 `creative_id`。针对整个账户的预览请求并非单次调用——必须明确指定要预览的广告。

**`ad_format` 即为预览的投放版位**，同一广告在不同版位的呈现效果各异：`MOBILE_FEED_STANDARD`（默认）、`DESKTOP_FEED_STANDARD`、`INSTAGRAM_STANDARD`、`INSTAGRAM_STORY`、`INSTAGRAM_REELS`、`RIGHT_COLUMN_STANDARD`、`MESSENGER_MOBILE_INBOX_MEDIA`、`THREADS_STREAM`。当广告主询问广告在特定位置的展示效果时，应直接指定对应的版位，而非先预览默认格式后再描述差异。请按照 `references/response-style.md` 中的编码值规范，使用广告主表述的版位名称，如“Instagram Stories”、“Facebook 移动信息流”等。无论是创意组合是否健康、哪个创意表现最佳，还是创意表现为何发生变化，这些都是关于效果的问题，因此上述洞察工具能够提供相关分析。不过，这些工具并不能单独给出答案：数据会为广告进行排序，而具体的创意则能解释这些排序结果，所以请同时查阅 `ads_get_creatives`——参见 `references/analysis.md` 中的“创意是效果解答的一部分”。

## 公开广告库研究

`ads_library_search` 会查询 Meta 的公开透明数据集——涵盖 Facebook、Instagram、Messenger 和 Audience Network 上的所有在投广告，以及受监管类别的历史广告。这些是关于**任何**广告主的公开数据，并非用户自己的账户数据，因此不受账户范围规则的约束。

该功能可用于竞争性研究、市场中的品牌发现、将品牌名称映射到 Facebook 的 `page_id`、政治与议题类广告监测、住房/就业/信贷领域的透明度核查，以及获取公开的创意示例。请勿将其用于分析用户自身的创意、自身的效果或政策相关问题。

在投广告仅能证明某种投放模式当前正在使用。切勿在缺乏独立返回的效果证据的情况下，将其描述为“有效”“胜出”或“表现优异”。

该接口会返回 `estimated_total_count` 以及最多 `limit`（最大 50）条记录，每条记录包含 `id`、`page_id`、`page_name`、`ad_creative_link_title`、`ad_creation_time`、`ad_delivery_start_time`、`ad_snapshot_url` 和 `currency`。出于安全考虑，`ad_creative_body` 字段不会直接暴露；请引导用户通过 `ad_snapshot_url` 查看完整的创意内容。

至少需提供以下参数之一：`search_terms`、`page_ids` 或 `countries`。国家代码应采用 ISO-2 格式（如 US、GB、DE、IN）。请勿传递 `ad_account_id`、`ad_reached_countries` 或单个的 `country` 参数——这些均不被接受。
