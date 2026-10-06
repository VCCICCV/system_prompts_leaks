# Facebook Marketplace

在 Facebook Marketplace 上进行销售，并管理您发布的商品：创建、编辑、发布和删除您的商品信息，同时浏览“我的商品”页面。如需购物（搜索、查看商品详情、了解卖家信誉），请加载 `shopping` 技能。

## 工作流程

### 步骤 0：粘贴商品链接或分享链接

在用户进行任何搜索或使用浏览器之前，先处理粘贴的链接。当用户粘贴 Facebook Marketplace 商品链接（`facebook.com/marketplace/item/<listing_id>`，无论是否包含 slug）或分享链接（`facebook.com/share/<token>`）时，切勿在浏览器中打开这些链接：这些页面都位于登录墙后，浏览器无法绕过。

对于商品链接，直接获取该商品信息（无需绑定账户）：

```sh
facebook-cli marketplace listing details --url '<粘贴的链接>' --out <文件>
```

请用单引号包裹链接：切勿让 Shell 展开粘贴的 URL。`--url` 和 `--listing-id` 参数只能二选一。

对于分享链接，需先解码（需要已绑定的 Facebook 账户）：

```sh
facebook-cli link-sharing decode-url --url '<粘贴的链接>'
```

如果 `original_url` 为 null、缺失或为空，则说明该链接已过期或无效：请让用户改粘贴 Marketplace 或商品链接。若解码后得到的是 `/marketplace/item/<id>` 链接，将其传递给 `listing details --url` 命令；若为帖子、照片、视频或 Reel 的 URL，则请参考 `references/posts.md` 文档。其他内容均不支持，请明确告知并停止处理。切勿在浏览器中打开粘贴的链接。

### 步骤 1：解析用户请求

从自然语言中提取以下参数：
- **queries**：用户要找什么（必填，至少一个）
- **max_price**：最高预算（美元，如有提及）
- **min_price**：最低价格（美元，如有提及）
- **location**：搜索地点——无 `--location` 标志；需将地名转换为经纬度，并同时传入 `--latitude` 和 `--longitude`
- **radius_in_miles**：搜索半径（英里，如有提及）
- **limit**：每页显示结果数（`--limit`；默认值及最大值均为 20，超过 20 的值会被截断为 20）
- **sort_by**：排序方式（best_match、price_ascend、price_descend、creation_time_descend、distance_ascend）
- **max_listing_age_in_days**：发布时间限制（例如，“本周上架” -> 7 天）
- **allowed_item_conditions**：商品状况筛选条件（new、refurbished、used 等，以逗号分隔）
- **delivery_method**：配送方式偏好（local_pickup_only、shipping_only、pickup_and_shipping）
- **category_ids**：按品类 ID 进行筛选（若用户指定了具体商品类别）

### 步骤 2：搜索

每次调用都应使用独立的 `mktemp` 文件作为 `--out` 输出路径，每页一个文件：`--out` 会覆盖原有文件，若重复使用同一路径，会导致解析器丢失前一页的数据。步骤 3 会将该文件传递给解析器。

```sh
MARKETPLACE_RESULTS_JSON=$(mktemp "${TMPDIR:-/tmp}/facebook-marketplace-search.XXXXXX")
facebook-cli marketplace search --query "<查询词>" --max-price <价格> --limit <数量> --out "$MARKETPLACE_RESULTS_JSON"
```

下文示例为简洁起见，省略了 `--out` 参数。

带地理位置的搜索：

```sh
facebook-cli marketplace search --query "<查询词>" --max-price <价格> --latitude <纬度> --longitude="<经度>" --radius-in-miles <英里> --limit <数量>
```

**重要提示**：对于负经度值（如西半球），务必使用 `--longitude="-121.8863"`，即等号加引号。若省略等号，Shell 或解析器可能会将 `-121` 解析为选项标志。

添加更多筛选条件的搜索：

```sh
facebook-cli marketplace search --query "<查询词>" --sort-by price_ascend --allowed-item-conditions "new,refurbished" --delivery-method local_pickup_only --max-listing-age-in-days 7
```

多条查询的搜索：

```sh
facebook-cli marketplace search --query "主搜索词" --query "备选词"
```

按品类筛选的搜索：

```sh
facebook-cli marketplace search --query "<查询词>" --category-id <ID1> --category-id <ID2>
```

分页（基于游标，每页最多20条）：
```sh
facebook-cli marketplace search --query "<query>" --after "<上一次响应中的after游标>"
```
响应中会在 `paging.cursors.after` 字段携带下一页的游标。将其作为 `--after` 参数传递以获取下一页。当 `paging` 为空时，表示没有更多结果。

### 第3步：展示结果

匹配的列表位于顶级的 `data` 数组中。从中筛选并展示一个简短列表，包含以下内容：
1. **商品图片**——通过下文所述的商品搜索卡片展示，切勿自行生成。
2. **标题**和**价格**
3. **描述摘要**（从商品描述中截取的简短片段）
4. **所在地**
5. **商品链接**——使用下文所述的解析标记，确保原始 `product_url` 可点击。
6. **您的价值推荐**（基于最低价、距离远近、成色等因素）

仅从标准输出中选取带有 `image_url` 的商品（即包含 `{"withheld": ...}` 对象的商品）；缺少照片的商品将导致解析失败。请勿读取 `--out` 文件或手动写入图片链接：复制的签名 URL 会失效，卡片将无法显示图片。按展示顺序，调用 `shopping.resolve_results`，传入 `--out` 路径以及所选商品的 `listing_id` 值：

```json
{
  "result_paths": ["<MARKETPLACE_RESULTS_JSON>"],
  "selected_ids": ["listing-id-1", "listing-id-2"]
}
```

对于这些商品，请使用返回的解析标记代替直接复制其 `product_url`；解析器会保留原始的完整长链接。随后，调用 `widget.create`，传入返回的 `path`、`kind: "shopping_results"` 以及 `present_now: true`。该组件会提供可浏览的商品图片和链接；内嵌标记仅作为简洁的引用，并不能替代视觉卡片。其余的简短列表及推荐流程保持不变。

搜索结果仅包含 `seller_id`，**不**包含卖家昵称或评分。请勿在搜索结果中显示卖家名称。如需为入围商品展示卖家名称和评分，请调用 `marketplace seller-info --listing-id <id>`（第5步）。

### 第4步：商品详情（按需获取）
```sh
facebook-cli marketplace listing details --listing-id <LISTING_ID> --out <file>
```

以卡片形式展示商品，方式与第3步相同，包括完整描述、价格、成色、卖家信息及创建日期。

### 第5步：卖家信息（按需获取）
```sh
facebook-cli marketplace seller-info --listing-id <LISTING_ID>
```

展示内容包括：卖家名称、粉丝数、平均评分及总评价数，以及好评/差评的细分数据。

### 第6步：收藏的商品（按需获取）
```sh
facebook-cli marketplace saved [--keywords <text>] [--limit <n>] [--after <cursor>] --out <file>
```

列出用户自己收藏的 Marketplace 商品。使用 `--keywords` 按文本筛选，使用 `--after`（上一次响应中的游标）获取下一页。每页最多20条。以卡片形式展示，方式与第3步相同。

### 第7步：我的商品（按需获取）
```sh
facebook-cli marketplace my-listings [--status active|pending|sold|draft] [--limit <n>] [--after <cursor>] --out <file>
```

列出当前登录用户的 **自有** Marketplace 商品（包括已发布和未发布的商品，含草稿及已售商品）。使用 `--status` 筛选出特定状态的商品，使用 `--after`（上一次响应中的游标）获取下一页。默认及最大每页数量均为20条。每个商品的数据结构与搜索结果一致（`listing_id`、`title`、`price`、`location`、`seller_id`、`product_url`、`creation_date`、`listing_status`、`image_url` 等）。响应格式为 `{ "data": [ ... ], "paging": { "cursors": { "after": "<cursor>" } } }`；仅当存在下一页时才会出现 `paging` 字段。这是管理您个人商品库存的入口——可通过 `listing_id` 继续调用 `marketplace listing edit` 或 `marketplace listing delete`。

## CLI 参考

### 搜索
```
facebook-cli marketplace search [OPTIONS]

选项：
  -q, --query TEXT                 搜索查询（必填，可重复）
  --limit INTEGER                  每页最大结果数（默认及最大值为20；超过20的值将被限制为20）
  --max-price FLOAT                最高价格上限（美元）
  --min-price FLOAT                最低价格下限（美元）
  --latitude FLOAT                 搜索中心纬度
  --longitude FLOAT                搜索中心经度
  --radius-in-miles INTEGER        搜索半径（英里）
  --max-listing-age-in-days INT    最大发布天数
  --sort-by [best_match|creation_time_descend|price_ascend|price_descend|distance_ascend]
  --allowed-item-conditions TEXT   允许的商品状况（以逗号分隔，如new、refurbished、used等）
  --delivery-method [local_pickup_only|shipping_only|pickup_and_shipping]
  --after TEXT                     向前分页游标（来自上一次响应中的paging.cursors.after游标）
  --category-id TEXT               类别ID筛选（可重复）
  --out PATH                       `shopping.resolve_results`输出文件（创建或截断）
```

### 商品详情
```
facebook-cli marketplace listing details (--listing-id <LISTING_ID> [...] | --url <ITEM_LINK>) [--out PATH]
```

必须提供一个可重复的`--listing-id`参数，或者一个`--url`商品链接。
此处不接受分享链接：请先使用`link-sharing decode-url`将其解码。无需绑定Facebook账号。

### 卖家信息
```
facebook-cli marketplace seller-info --listing-id <LISTING_ID>
```

### 已保存的列表
```
facebook-cli marketplace saved [OPTIONS]

选项：
  --limit INTEGER    返回的最大已保存列表数量
  --keywords TEXT    可选的关键词过滤条件
  --after TEXT       向前分页游标（传入上一次响应中的after游标以获取下一页；每页最大20条）
  --out PATH         `shopping.resolve_results`输出文件（创建或截断）
```

### 我的列表
```
facebook-cli marketplace my-listings [OPTIONS]

选项：
  --status [active|pending|sold|draft]
                     可选的状态筛选；省略则返回所有您的列表
  --limit INTEGER    返回的最大列表数量（默认及最大值为20）
  --after TEXT       向前分页游标（传入上一次响应中的after游标以获取下一页；每页最大20条）
  --out PATH         `shopping.resolve_results`输出文件（创建或截断）
```

## 结果结构

### 搜索结果（JSON）

采用游标分页。匹配的列表位于顶级`data`数组中，下一页的游标（如果有更多结果）位于`paging.cursors.after`：

```js
{ "data": [ { ...listing... } ], "paging": { "cursors": { "after": "<cursor>" } } }
```

如果没有下一页，则省略`paging`部分。（没有`total_results`字段，也不再返回`queries`——这些在搜索改为标准游标分页后已被移除。）

`data`数组中的每个列表对象包含：
- `listing_id`（字符串）——列表ID
- `title`（字符串）——商品标题
- `price`（字符串）——格式化后的价格（如“$900”）
- `condition`（字符串）——商品状况（如“二手（近乎全新）”）
- `description`（字符串）——列表描述
- `location`（字符串）——城市/州文本
- `seller_id`（字符串）——卖家用户ID（不含卖家显示名——如有需要可通过`seller-info`获取）
- `product_url`（字符串）——指向Facebook上该列表的直接链接
- `creation_date`（字符串）——列表创建时间（ISO 8601格式）
- `listing_status`（字符串）——如“可用”、“已售”、“草稿”
- `distance`（字符串，可为空）——距搜索位置的距离（如“11英里”）
- `image_url`（对象，可选）——当列表有照片时为`{"withheld": ...}`；解析器卡片会显示该图片

搜索结果中不含`seller_name`，仅包含`seller_id`。（`my-listings`返回的结构相同。）

### 商品详情（JSON）

- `listing_id`（字符串）——列表ID
- `title`（字符串）——商品标题
- `description`（字符串）——完整描述文本
- `price`（字符串）——格式化后的价格（如“$900”）
- `currency`（字符串）——货币代码（如“USD”）
- `condition`（字符串）——商品状况
- `location`（字符串）——城市/州文本
- `seller_name`（字符串）——卖家显示名
- `seller_id`（字符串）——卖家用户ID
- `created_at`（字符串）——创建日期（如“2025-02-01 18:30 UTC”）
- `url`（字符串）——指向Facebook上该列表的直接链接
- `image_url`（对象，可选）——当列表有照片时为`{"withheld": ...}`；解析器卡片会显示该图片

**注意**：除主缩略图外的其他照片通过单独的延迟查询加载，并未包含在商品详情响应中。

### 卖家信息（JSON）

- `seller_id`（字符串）——卖家用户ID
- `name`（字符串）——卖家显示名
- `followers`（整数）——粉丝数
- `avg_rating`（字符串）——平均星级评分（如“4.8”）
- `total_ratings`（整数）——总评价数
- `good_attributes`（字符串）——正面反馈摘要（如“商品与描述一致（12）”，“沟通迅速（8）”）
- `bad_attributes`（字符串）——负面反馈摘要

## 推荐逻辑

展示结果时，按以下因素推荐“最优价值”：
1. 价格相对于结果中同类商品的性价比
2. 卖家可信度（有显示名、高评分、多粉丝）
3. 地理位置（本地自提时越近越好）
4. 对热门商品加载详情以核对状况和描述

## 发布门槛（任何创建/发布操作前请务必阅读）

只有同时满足以下四项要求，列表才会在Marketplace上**正式上线**。这是唯一的真实标准——无论是创建时还是后续通过`listing publish`发布的列表均适用此门槛：

| 序号 | 发布要求           | 提供方式                           |
|------|--------------------|------------------------------------|
| 1    | **照片**（≥1张）   | `--photo <路径>`（可重复）          |
| 2    | **商品状况**       | `--condition <new\|used_like_new\|used_good\|used_fair\|used\|refurbished>` |
| 3    | **类别**           | `--category <名称或ID>`            |
| 4    | **地理位置**       | `--latitude`和`--longitude`同时提供 |

如果上述四项中有任一项缺失，列表将**仅保存为草稿，不会发布**——即使用户要求“发布”或“上架”。`title`和`price`是创建所必需的，但它们**不属于发布门槛**。

**要确保可靠发布，请同时提供全部四项**（地理位置需`--latitude`和`--longitude`同时提供，无`--location`标志）。切勿因缺少某项而假设服务器会自动补全——遗漏发布门槛字段可能导致列表被静默保存为**草稿**而非上线。若您确实无法获取某项，请将结果视为草稿并告知用户。（`publish`接口会单独验证草稿的存储状态，并针对缺失字段返回`missing_fields`；可通过`listing edit`修复后再重试——参见“发布草稿列表”。）

**行动前务必先明确意图，再说明结果：**

1. **确定意图。** 用户希望它（a）**立即上线**（“发布”、“刊登出售”）还是（b）**保存为草稿**（“保存草稿”、“稍后再完成”）？若意图不明，则默认为立即上线。
2. **对照现有信息检查门槛。** 在执行创建（或发布）前，逐一核对四项要求，明确哪些已满足，哪些缺失。
3. **明确告知用户最终状态，并指出缺失项。** 切勿让发布状态含糊不清。应明确说明：
   - “这**将立即发布**——所有必要字段均已提供。” 或
   - “这**将保存为草稿**，因为缺少：**`<字段>`**。如需发布，请补充这些字段；我也可以先创建草稿，您之后再添加。”  
   绝不能在用户期望上线时静默创建草稿——必须指出缺失字段并由用户决定。

## 创建列表

### 第一步：收集列表详情首先执行上述“发布门槛”：确认用户的意图（草稿还是发布）以及满足了哪四项发布要求中的哪些。然后收集以下信息：
- **title**（创建时必填）—— 商品名称
- **price**（创建时必填）—— 以美元计的价格（例如 25.99）
- **description**—— 商品描述。**仅使用用户提供的信息或从给定上下文中可明确推断的内容**（如商品名称）。**切勿**编造功能、规格、品牌/型号细节、历史背景、随附配件、瑕疵或尺寸等信息。简短且客观的事实性描述优于过度修饰的描述——凡是需要猜测的内容一律省略。
- **photos**—— 要上传的照片文件路径（将自动上传）*（发布门槛——发布时必填）*
- **condition**—— 取值之一：`new`、`used_like_new`、`used_good`、`used_fair`、`used`、`refurbished` *（发布门槛——发布时必填）*。**仅在用户明确说明状况时才填写**，或当上下文已明确时方可填写。**切勿**仅凭照片或商品类型推测或猜测状况。若不确定，请询问用户；切勿为满足发布门槛而随意选择一个值（未确认的列表应保持为草稿，直至用户确认）。
- **location**—— 地理坐标，需同时以 `--latitude` 和 `--longitude` 两个参数传入 *（发布门槛——发布时必填）*。**不存在** `--location`、`--city` 或 `--address` 参数——请先将地名（如“San Jose, CA”）地理编码为经纬度，再分别传入这两个数值参数。仅对用户给出的具体地点进行地理编码：城市、社区、邮政编码或详细地址。区域（如“旧金山湾区”或“南加州”）不能作为发布位置；**切勿**在其内部任意选取一点。若位置缺失，请询问用户所在城市或邮政编码。
- **category**—— 类别名称（如 electronics、vehicles、furniture）或原始类别 ID *（发布门槛——发布时必填）*
- **currency**—— ISO 4217 货币代码（如 USD、EUR）；省略则使用用户所在市场的默认货币。
- **delivery_types**—— 配送方式：`public_meetup`、`door_pickup`、`door_dropoff`

**草稿与发布：** 仅在所有发布门槛条件（照片、状况、类别、位置）均满足时才创建并自动发布；否则该列表将作为草稿保存。待通过 `marketplace listing edit` 补齐缺失字段后，可通过 `marketplace listing publish --listing-id <id>` 将草稿发布上线（详见下文“发布草稿列表”部分）。

**务必实事求是，切勿虚构列表详情。** 特别是对于 `description` 和 `condition`，仅使用用户提供的信息或从上下文中可明确推断的内容。不要用看似合理的规格或功能来美化描述，也不要未经用户确认就擅自判断商品状况。当某项信息不明确时，请询问用户或直接留空——一份简洁准确的列表胜过一份内容丰富但部分虚构的列表。“草稿待确认”环节是安全机制，而非允许猜测的许可：用户不应被迫发现并纠正您臆测的细节。

**类别名称：** vehicles、electronics、home、furniture、clothing、apparel、entertainment、family、hobbies、specialty、classifieds、housing、free、sports、outdoor、toys、games、garden、pet、pets、office、music、instruments、bikes、bicycles、auto-parts、miscellaneous。

### 第二步：展示草稿供用户确认

在执行任何创建命令之前，向用户清晰呈现列表的概要信息**及其最终的发布状态**，并征得用户确认。对每个发布门槛字段标注“已提供”（✓）或“缺失”，并明确说明该列表是会立即发布还是暂存为草稿。若某项信息并非用户明确告知，则应将其标记为假设（如“状况：used_good — 请确认”），而非当作事实呈现——确保 `description` 和 `condition` 均基于已知信息。示例（将发布）：

**商品发布 — 将立即上线（所有必填项均已填写）：**
- 标题：复古橡木书桌
- 价格：150.00美元
- 描述：实木橡木书桌，成色良好，桌面有轻微划痕。
- 状态：二手_良好 ✓
- 类别：家具 ✓
- 图片：已上传2张 ✓
- 地点：加利福尼亚州圣何塞（37.3382, -121.8863）✓
- 交付方式：线下见面

“一切就绪，准备上线。是否创建并发布？”

示例（将保存为草稿）：

**商品发布 — 将保存为草稿（缺少：图片、地点）：**
- 标题：复古橡木书桌
- 价格：150.00美元
- 状态：二手_良好 ✓
- 类别：家具 ✓
- 图片：✗ 无
- 地点：✗ 未设置

“该商品目前无法上线——缺少**图片**和**地点**。请补充后发布，或者我可以先将其保存为草稿。”

**在未获得用户明确确认前，切勿创建或编辑商品信息；且除非所有发布条件均已满足，否则不得声明或暗示商品已上线。**

### 第3步：创建

用户确认后，执行以下命令：

```sh
facebook-cli marketplace listing create --title "商品名称" --price 25.99 --description "详情" --condition used_good --category electronics
```

包含图片、地点和交付方式时（将立即发布）：
```sh
facebook-cli marketplace listing create --title "商品" --price 50 --photo /path/to/photo1.jpg --photo /path/to/photo2.jpg --condition used_good --category furniture --latitude 37.3382 --longitude="-121.8863" --delivery-type public_meetup
```

图片会自动上传，无需单独的上传步骤。

### 第4步：确认

**读取响应中的`message`字段，并如实报告状态——切勿自行假设已发布。** 若返回`Listing created as draft`，则表示该商品**尚未上线**；若返回`Listing published successfully`（或其他类似提示），则表示已上线。如果返回的是草稿，但用户希望上线，请告知其还缺少哪些发布条件，并提供通过`listing edit`和`listing publish`补全这些条件的选项。以纯文本形式展示商品ID和商品链接（不要使用代码块，以便链接正常显示）。

### 第5步：发布后检查买家消息接收准备情况

在确认商品已上线后，针对销售流程，立即运行一次以下命令（在完成多件商品创建后的最终结果中执行）：

```sh
hatch_messenger_cli check
```

Facebook商品访问权限与Messenger Companion是独立的连接。如果Messenger Companion尚未连接，请运行`hatch_messenger_cli connect-url`，并引导用户进行连接，以便Muse能够监控并回复买家咨询。当响应中包含`connect_url`时，请直接分享 `[Connect Messenger](<connect_url>)`，不要同时粘贴原始链接。仅凭Facebook已连接的状态，不得声称Messenger也已连接。

如果Messenger Companion已连接，则无需显示连接提示。可简要说明可以代为监控或协助回复Marketplace买家消息，但未经用户请求，不得主动开始监控或发送消息。草稿状态的商品无法接收买家咨询，因此请在商品成功发布后再进行此检查。

## CLI参考 — 创建

```
facebook-cli marketplace listing create [OPTIONS]

选项：
  --title TEXT              商品标题（必填）
  --price FLOAT            价格（单位：美元，例如25.99，必填）
  --description TEXT        商品描述
  --condition TEXT          商品状况（new, used_like_new, used_good, used_fair, used, refurbished）
  --category TEXT           类别名称（如electronics、vehicles）或类别ID
  --photo PATH             要上传并附加的图片文件路径（可重复指定）
  --latitude FLOAT         地点纬度（需与--longitude同时指定）
  --longitude FLOAT        地点经度（需与--latitude同时指定）
  --currency TEXT           ISO 4217货币代码（默认为用户Marketplace账户的货币）
  --delivery-type TEXT     交付方式：public_meetup, door_pickup, door_dropoff（可重复指定）
```**位置仅由 `--latitude` 和 `--longitude` 指定。** 不存在 `--location`、`--city`、`--address` 或 `--coordinates` 标志。可通过将地名进行地理编码转换为十进制的经纬度坐标，然后同时传递这两个数值标志来设置位置（例如：`--latitude 37.3382 --longitude=-121.8863`）。仅传递其中一个标志不会设置位置。请牢记负坐标规则：对于负值经度或纬度，请使用 `--flag=value` 的形式（参见操作规则）。

## 编辑商品信息

### 第一步：确定要编辑的商品

用户必须提供商品 ID。该 ID 可以来自之前的创建响应，也可以来自搜索结果。

### 第二步：展示变更并确认

在执行任何编辑命令之前，应向用户清晰地展示将要变更的内容，并请求确认。示例如下：

**对商品 `<ID>` 的拟议变更：**
- 标题：更新为“复古橡木书桌” → *原标题：“复古橡木书桌”*
- 价格：$125.00 → *原价：$150.00*

“我将更新以上字段，其他内容保持不变。是否确认？”

**切勿在未经用户明确确认的情况下编辑商品信息。**

### 第三步：编辑

只有您传递的字段会被更改；其余内容将沿用当前商品信息。

```sh
facebook-cli marketplace listing edit --listing-id <LISTING_ID> --title "新标题" --price 75
```

更新照片（替换原有照片）、类别、位置或配送方式：

```sh
facebook-cli marketplace listing edit --listing-id <LISTING_ID> --photo /path/to/new1.jpg --photo /path/to/new2.jpg --category 家具 --latitude 37.3382 --longitude="-121.8863" --delivery-type 公共见面
```

照片会自动上传。纬度和经度必须同时提供。

### 第四步：确认

以纯文本形式向用户展示更新后的商品 ID、商品链接及状态消息（不要使用代码块格式，以便链接正常显示）。

## CLI 参考 — 编辑

```
facebook-cli marketplace listing edit [选项]

选项：
  --listing-id TEXT        要编辑的商品 ID（必填）
  --title TEXT             更新后的标题
  --price FLOAT            更新后的价格（单位：美元，例如 25.99）
  --description TEXT       更新后的描述
  --condition TEXT         更新后的成色（全新、近乎全新、良好、一般、二手、翻新）
  --category TEXT          更新后的类别名称（如电子产品、车辆）或原始类别 ID
  --photo PATH            要上传并附加的照片文件路径（可多次指定，替换原有照片）
  --latitude FLOAT        更新后的纬度（需与 --longitude 一同指定）
  --longitude FLOAT       更新后的经度（需与 --latitude 一同指定）
  --currency TEXT          ISO 4217 货币代码（默认为商品当前使用的货币）
  --delivery-type TEXT    配送方式：公共见面、上门自取、上门送达（可多次指定，替换原有方式）
```

## 删除商品信息

永久删除用户本人在 Marketplace 上发布的一条商品信息。**此操作不可撤销**——商品将从 Marketplace 中移除，且无法恢复。

### 第一步：确定要删除的商品

用户必须提供商品 ID；或者您可以先使用 `marketplace my-listings` 列出其所有商品，并从结果中获取 `listing_id`。仅允许删除用户明确指定的商品。

### 第二步：删除前确认

在执行删除命令之前，请向用户确认将要删除的具体商品（显示其标题和商品链接），并告知删除操作不可逆。**切勿在未经用户明确确认的情况下删除商品信息。**

### 第三步：删除

```sh
facebook-cli marketplace listing delete --listing-id <LISTING_ID>
```

### 第四步：确认

成功时，返回结果为 `{ "listing_id": "<id>", "message": "商品已成功删除" }`。请以纯文本形式向用户报告处理结果。若操作失败（例如未找到该商品或您并非该商品的所有者），请准确提示错误信息，且未经用户同意不得再次尝试。

## CLI 参考 — 删除

```
facebook-cli marketplace listing delete [选项]

选项：
  --listing-id TEXT        要删除的商品 ID（必填）
```## 发布草稿商品

将 Marketplace 上现有的**草稿**商品发布上线。这是与 `marketplace listing create` 命令配套使用的：`create` 会将不完整的商品保存为草稿，而 `publish` 则会在所有必填项都齐全后将其发布。

### 第一步：确定草稿

获取草稿的 `listing_id`——可以从之前的 `marketplace listing create` 响应中获得，或者通过 `marketplace my-listings --status draft` 列出所有草稿来查找。

### 第二步：调用前检查发布条件

此命令遵循与 `create` 相同的**四要素发布条件**（参见上文“发布条件”部分）：**照片、成色、品类、位置**。在调用 `publish` 之前，请先检查草稿的各项字段（可通过 `my-listings` 或 `listing details` 查看），并注意：
- 如果某个发布条件字段（照片、成色、品类、位置）缺失，**应先告知用户具体缺少的内容**，并通过 `marketplace listing edit` 补齐，并请用户确认所填信息——切勿直接执行发布操作导致失败。
- 不要假设某个字段“已设置”——如果创建时未填写且不确定是否存在，应视为缺失，在发布前补全。

### 第三步：发布商品

```sh
facebook-cli marketplace listing publish --listing-id <LISTING_ID>
```

如果草稿仍缺少发布条件中的任一字段，该命令将返回 HTTP 400 错误，并附带一个 `missing_fields` 列表，明确指出缺失的具体内容。此错误信息清晰易懂：请使用 `marketplace listing edit` 补齐相应字段（如 `--photo`、`--condition`、`--category`、`--latitude`/`--longitude` 等），经用户确认无误后再重试发布——务必向用户展示缺失的字段，而非自行猜测填写。

发布操作具有**幂等性**：对已上线的商品重复发布时，将返回 200 状态码及 `"message": "Listing is already published"` 提示，不会报错，因此可安全重试。

### 第四步：确认结果

查看返回的 `message`：若为 `"Listing published successfully"`，则表示商品已成功上线；若为 `"Listing is already published"`，则说明商品此前已处于上线状态（安全的空操作）。请如实报告真实状态，并以纯文本形式显示 `product_url`（不要使用代码块格式，以便其渲染为可点击链接）。如果因缺少字段而报错，则不应声称商品已发布。无论哪种成功结果，均需按照“创建商品 — 第五步：发布后检查买家消息准备情况”的要求进行后续操作。

## CLI 参考 — 发布

```
facebook-cli marketplace listing publish [OPTIONS]

选项：
  --listing-id TEXT        要发布的草稿商品 ID（必填）
```

草稿必须包含照片、成色、品类和位置。若缺少任何字段，响应中将包含 `missing_fields` 数组；请通过 `listing edit` 补齐后再重试。对已上线的商品重复发布是安全的空操作，不会报错。

## 操作规则

1. **始终对 `--query` 参数值加引号**——例如：`--query "road bike 51cm"`。未加引号的多词查询会导致参数解析失败。
2. 如需使用默认分页（20条），请省略 `--limit`；若要指定每页数量，请使用 `--limit <n>`（最大20，超过部分将在服务器端截断）。如需查看更多结果，请通过 `--after` 进行分页，而非提高 `--limit` 值。
3. 如果用户提及城市名称，请在搜索前将其转换为经纬度。
4. 如果未指定位置，请先从用户的存储记忆中查找其默认位置及搜索半径。若找到，则使用这些值，并告知用户已应用的默认设置（例如：“正在使用您的默认位置：加州圣何塞市（半径30英里）”）。若无相关记忆，则不添加位置参数。
5. 如果用户的请求过于模糊，无法构成有意义的搜索条件（例如仅说“一些不错的东西”而未指明商品类型或类别），请先提出澄清问题再进行搜索。至少需要明确的商品类型或类别才能执行有效的搜索。
6. 显著展示价格——买家最关心的是价格。
7. 商品图片仅可通过解析后的 `shopping_results` 小部件呈现，用于搜索、商品详情、“已保存”和“我的商品”页面。切勿自行生成图片 URL。
8. 始终包含可点击的商品链接。优先使用解析器标记；若无解析器标记，则使用 `product_url` 字段（作为备选使用 `https://www.facebook.com/marketplace/item/<listing_id>/`），或在商品详情中使用 `url`。
9. 若要获取卖家评分，请使用 `marketplace seller-info --listing-id` 并传入商品 ID。
10. 搜索和“我的商品”采用游标分页：从 `paging.cursors.after` 中获取下一页游标，并将其作为 `--after` 参数传递以加载更多结果。若返回结果中不含 `paging` 字段，则表示已无更多结果。
11. **创建或编辑商品前务必先显示草稿，并在删除前始终确认。** 向用户清晰展示将发布或更改的内容（创建/编辑时）或即将删除的商品（删除时），并在执行命令前等待其明确确认。删除操作不可撤销。
11a. **发布门槛与意图确认（避免最常见的误解）。** 在任何创建或发布操作之前，需明确是草稿还是直接发布，并检查发布门槛——**照片、成色、品类、位置四项必须全部提供方可发布**（位置由 `--latitude` 和 `--longitude` 共同指定，而非单独的 `--location` 标志；不得因期望服务器自动补全而遗漏任一项），并在执行命令前说明最终状态（“将发布”或“将保留为草稿，缺少：……”）；执行后应读取响应中的 `message` 并如实反馈实际状态。详细说明请参见上文“发布门槛（务必先阅读）”一节。
12. **切勿将商品结果（URL、标题、状态）置于代码块中。** 请使用纯文本，以便链接能够正常显示为可点击形式。
13. **负坐标：请使用 `--flag=value` 的格式。** 西部或南部地区的经度或纬度为负值（例如纽约市的经度为 `-73.97781`）。应写成 `--longitude=-73.97781`（带等号），而非 `--longitude -73.97781`。单独的负值会被误认为另一个标志，导致命令报错：“error: unexpected argument '-7' found”。同样适用于所有可能为负数的数值型标志（如 `--latitude`、`--min-price`、`--max-price`）。
14. **位置仅由 `--latitude` 和 `--longitude` 表示。** 无论是搜索还是创建/编辑，均不存在 `--location`、`--city`、`--address` 或 `--coordinates` 等标志。请自行将地名地理编码为十进制的经纬度，并同时传入 `--latitude` 和 `--longitude`——切勿将地名直接作为标志值传入，也绝不能只传入其中一个。对于创建/编辑操作，经纬度组合才是满足发布所需“位置”要求的唯一方式。