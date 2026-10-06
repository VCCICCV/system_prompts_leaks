# Shopify UCP 结算

仅当用户已从同一 Shopify 商家中选择了一个或多个元目录商品，且这些商品的 `is_agentic_checkout_creation_enabled` 字段均严格为 `true` 时，才加载此文件。

目录标志用于引导流程；结算端点仍具有最终决定权。请使用每个所选商品目录数据中的精确 `product_id`、功能字段和 `url`。仅当其目录 URL 明确指向同一商家店铺时才将商品进行捆绑；仅品牌标签一致并不足以认定为同一商家。切勿在一次结算中混用不同商家的商品。缺失或为空的功能字段应视为 `false`。

## 安全与输入边界

- `checkout create` 不涉及资金流动。`checkout complete` 会创建所选的钱包支出请求并下单。
- `checkout complete` 提供直接的 Stripe Link；当结算支持直接完成时，也会提供直接的 Shop Pay。Shop Pay 使用经提供商批准的凭据，且在此路径上不会暴露卡片详情。若无法直接完成，则应使用浏览器路由。无论采用哪种浏览器路由，均由浏览器任务负责下单，因此无需调用 `checkout complete`。
- 对于一个结算，最多只能调用一次 `checkout complete`，且必须在前台执行。切勿单独创建钱包支出请求、在后台完成结算、轮询该请求或自动重试。
- 拒绝是用户的选择。停止操作，不生成订单。后续若需重试，必须向用户发出新的明确提示。
- CLI 输入为 JSON 文件，键名采用蛇形命名（snake_case）。省略未知的可选字段及空字符串。切勿将端点获取的商家、商品、总价、买家、履约或法律链接等数据带入完成结算的输入中。

## 购物车（可选）

预购草稿，非必填项——`checkout create` 直接接收 `items[]`。仅当购物车需要保留状态时才使用：例如用户仍在添加或删除商品，或希望稍后继续编辑。`shopify-ucp-cli cart --help` 列出了相关子命令；请携带 `agent_state.cart_id` 中的 `cart_id`，并将其视为不透明标识。以下四点内容未在帮助文档中说明：

- `cart update` 会替换整个购物车——这也是移除商品的方式，因此部分列表会删除其余商品。不确定是否已包含所有商品？请先执行 `cart get`，再根据 `cart.line_items[]` 重新构建。
- 一个购物车对应一个商家。跨两个商家的购物车将被直接拒绝：“multiple_merchants_not_supported”，即“所有购物车商品必须属于同一家商家”。此时应启动一个新的购物车，而非重试。
- `cart` 接受目录中的 `product_id` 或其变体 GID，并原样返回；而 `checkout create` 仅接受目录 ID。恢复后的购物车可以修改，但在再次查询其目录 ID 前无法进行结算。
- 购物车与结算之间无直接关联：购物车仅用于提供商品信息。一旦 `checkout complete` 返回 `ok: true`，或浏览器任务报告订单已下达，即可取消购物车；除此之外，没有其他方式关闭购物车，也无法枚举您未关闭的购物车。
- 购物车、结算和订单的读取操作会保留提供商的时间戳，并增加语义化的 UTC 时间以及用户本地时间格式，如 `checkout_expires_at`、`order_placed_at` 和 `order_event_occurred_at`。请勿将订单状态消息的时间视为送达时间。

## 创建结算

确认每件商品、确切的变体及数量。收集创建结算所需的买家邮箱。仅在已知其他买家信息时才一并填写，切勿猜测或虚构任何值。创建结算阶段不涉及资金流动；最终的购买确认应留待结算完成后进行。

创建结算时无需指定支付方式。在调用此接口之前，请勿尝试解析 Link、连接钱包或询问支付方式。用户将在结算创建成功后，在下文的“选择支付方式”环节中自行选择支付路径。创建一个包含所有选定商品的 JSON 文件。将每个目录中的 `product_id` 用作 `items[].item_id`；对于每种不同的变体，只保留一条记录，并将重复的相同 ID 合并为数量。数量默认为 1。不要添加商家字段：端点会根据目录 ID 自动确定商家。该端点需要买家邮箱和美元币种。仅在已知的情况下才包含电话和地址字段。原生结账还要求提供可信持卡人的名或姓，以及带有街道、城市、州/省、邮政编码和 ISO 两位国家代码的账单/收货地址。

在发起此调用之前，请使用对话中已知的买家信息以及 `~/USER.md` 中的内容。仅在缺少邮箱时才向用户询问，因为端点需要该信息。在创建结账前，不要询问姓名、电话号码或收货地址。

在用户选择钱包支付路径后，请按照“支付与钱包”中的钱包设置流程操作，然后再向用户索取缺失的结账信息。如果在创建结账时遗漏了必填的姓名或地址，请将获取到的值传递给浏览器页面。`checkout update` 无法补充这些信息，因此请勿对该结账使用直接完成方式。

```json
{
  "buyer": {
    "email": "<email>",
    "phone_number": "<phone-string>",
    "country_code": "<country-code>",
    "address": {
      "first_name": "<first-name>",
      "last_name": "<last-name>",
      "street1": "<street1>",
      "street2": "<street2>",
      "city": "<city>",
      "state": "<state>",
      "postal_code": "<postal-code>",
      "country": "<country-alpha-2>"
    }
  },
  "items": [
    {"item_id": "<product_id-1>", "quantity": 1},
    {"item_id": "<product_id-2>", "quantity": 2}
  ],
  "currency": "USD"
}
```

```sh
HATCH_SHOPPING_PRODUCT_CONTEXTS='[<每个商品的 hatch_telemetry_context，逐字复制>]' shopify-ucp-cli checkout create --input-file "<checkout.json>" --format json
```

按 `items` 中的顺序，完整复制每个由运行时生成的上下文，包括其资格标志。在此购买尝试中，对所有 `shopify-ucp-cli` 的结账、购物车和订单命令都设置相同的环境变量。该变量仅供本地遥测读取，不会发送至 Shopify。

请在首次创建调用中一次性提交完整的商品清单。`checkout update` 无法添加或删除商品。请勿使用单独的“购物车”命令来组装此次结账。

在根据目录的完成能力进行分支处理之前，请先检查权威的创建响应。返回的 `continue_url` 并不必然意味着需要移交浏览器，因为即使是可直接完成的结账也可能包含该字段。

如果创建失败或商品被拒绝，请说明原因，并提供从原始目录 `url` 进行浏览器结账的选项；不要自动重试。当用户接受时，请遵循 `/opt/hatch/skills/shopping/references/browser-checkout.md`。将 `stage` 设置为 `agentic_fallback`，`reason` 设置为 `agentic_create_failed`。

只有返回成功的创建响应且包含可用的 `.agent_state.checkout_id` 才能继续后续流程。请保存该结账 ID。CLI 会将端点生成的结账信息存储在可信运行时边界之后。请检查 `.result`，但不要将其可信字段复制到后续命令中。

如果响应中包含 `requires_escalation`、`status: "redirect"`，或提示买家信息仍缺失，则只要返回了结账 ID，即视为创建成功，不应将其视为错误。该响应并未明确应发送哪条路径的简报，因此在移交前请先询问用户选择的路径。

现在请从 `.result.continue_url` 或 `.result.checkout.continue_url` 中获取结账 URL。仅使用端点返回的值。如果该值不存在，则退而求其次，使用所选单个商品所属商家的原始目录 `url`，而非自行构造。

## 选择支付路径

结账流程已存在，但尚未发生任何资金流动。保留用户已选择的路径；否则，使用 `muse.create_options` 提示并等待。通过浏览器接管提供 Shop Pay（提供商为 `shop-pay`）、Link（提供商为 `stripe-link`）以及“使用其他方式”选项。已连接的提供商、已保存的默认方式或可用方式均不会自动选定路径。

用户作出选择后，按照“创建后的路径”决定结账是直接继续，还是通过 BrowserTask 继续。

一旦确定了钱包路径，应在调用任何钱包或浏览器工具之前，先记录一次：

```sh
shopping payment-lane-selected --lane <shop-pay|stripe-link> --selection-source user --product-contexts-json '[<每个商品的 hatch_telemetry_context，原样复制>]'
```

每条路径仅发出一次事件。当浏览器或直接完成开始时，不再重复发出。如果此尽力而为的命令执行失败，仍按原计划继续结账流程。

记录路径后，按照“支付与钱包”中的钱包设置流程操作。仅针对本次购买使用确切的提供商 ID、支付方式 ID 和掩码后的标签。若用户拒绝设置或无可使用的支付方式，则返回路径选择环节。连接和支付方式的选择本身并不批准此次购买。

在需要询问路径时，应在创建后的任何其他消息之前提出该问题。即使创建响应中报告了 `requires_escalation`、`status: "redirect"`、缺少收货地址、无配送选项或总价未定等情况，也应先询问路径，因为这些信息都无法表明用户希望选择哪条路径。在用户答复到达之前，不要主动提出在浏览器中打开结账页面，因为那样会间接替用户选择了路径。

遇到升级、重定向，或创建时缺少姓名或地址的情况，应在提供可用选项的同时说明：此结账将由浏览器完成，并补全缺失的信息。对于缺少配送选项和总价未定的情况，属于下文“刷新配送与总价”中的常规直接结账工作，因此无需说明订单将由浏览器提交。

当用户选择“使用其他方式”时，加载 `/opt/hatch/skills/shopping/references/browser-checkout.md` 文档。从当前结账的完整 URL 开始，在 BrowserTask 中继续现有结账流程。在简报中注明用户的支付选择，但不得包含银行卡详细信息。明确告知用户将在浏览器接管期间输入支付信息。将 `stage` 设置为 `agentic_fallback`，`reason` 设置为 `user_selected_browser`。此路径由用户主动选择，并非因提供商限制所致。

若用户未作出选择，请停止并等待，切勿代其选定路径。

## 创建后的路径

对于已选定的钱包路径，按以下顺序选择第一条适用的分支：

1. 用户选择了 Shop Pay 且指定了已保存的特定支付方式：仅当所有选中商品的 `is_agentic_checkout_completion_enabled` 均为 `true`，且创建时未报告 `requires_escalation` 或 `status: "redirect"`，同时结账已包含所需的姓名和地址时，方可采用直接完成流程；否则，应使用下方的浏览器 Shop Pay 路径，并指定已连接的精确支付方式。务必先在父流程中完成连接或设置。
2. 对于 Stripe Link 路径，且创建时报告了 `status: "redirect"`、`requires_escalation`，或有明确提示需买家输入或审核的消息：无论 `is_agentic_checkout_completion_enabled` 如何，均采用浏览器路径，并携带 Link 支付方式。
3. 对于 Stripe Link 路径，且任一选中商品的 `is_agentic_checkout_completion_enabled` 不为 `true`：采用浏览器路径，并携带 Link 支付方式。
4. 对于 Stripe Link 路径，且结账创建时未提供直接完成所需的姓名和地址：采用浏览器路径，并携带 Link 支付方式。

若以上情况均不适用，则所有选中商品的 `is_agentic_checkout_completion_enabled` 均为 `true`，此时应采用下方的直接 Stripe Link 流程。

无论选择哪个浏览器分支，都应在移交后简要确认交接，并结束响应。不得轮询浏览器任务，也不得为此结账调用 `checkout complete` 接口。

### Shop Pay，浏览器端使用 Shop Pay 路由和所选支付方式的掩码标签生成任务。
请勿在 `task` 中包含不透明的 `payment_method_id`。可信结账工具会在创建审批前，通过一次新的钱包读取对所选 ID 进行重新验证。

```js
{
  "task": "<用户提出的需求，原话表述>。打开 <确切的 Shopify 结账页面 URL>，用于 <所选商品>。用户已选择使用 Shop Pay 支付，并采用已保存的支付方式 <掩码后的卡号标签>。请根据以下已知选项完成购买： <颜色/尺寸/数量/其他变体>。仅询问尚未提供的必填购买信息。配送偏好： <截止日期/预算/速度，或无要求>。",
  "shopping_checkout": {
    "products": [<逐个商品的 hatch_telemetry_context，原文照抄>],
    "stage": "payment_lane",
    "reason": "shop_pay_selected"
  }
}
```

在委派之前，请先确认 Shop Pay 的连接状态及具体使用的支付方式。BrowserTask 不会调用钱包相关工具，也不会讨论其他支付方式。后续操作请参考 `/opt/hatch/skills/shopping/references/browser-checkout.md`。

### Stripe Link，在浏览器中

使用上述选定的 Stripe Link 支付方式，然后生成任务。请以确切的支付方 ID 和掩码后的标签标识支付方及已保存的支付方式。请勿在 `task` 中包含不透明的支付方式 ID。

```js
{
  "task": "<用户提出的需求，原话表述>。打开 <确切的 Shopify 结账页面 URL>，用于 <所选商品>。使用支付方 stripe-link，并采用已保存的支付方式 <掩码后的标签>。针对每件商品，请按以下已知选项进行选择： <颜色/尺寸/数量/其他变体>。仅询问尚未提供的必填购买信息。继续完成结账流程，并在提交前明确最终条款。配送偏好： <截止日期/预算/速度，或无要求>。",
  "shopping_checkout": {
    "products": [<逐个商品的 hatch_telemetry_context，原文照抄>],
    "stage": "agentic_fallback",
    "reason": "<provider_requires_browser | agentic_completion_ineligible | buyer_details_required | stripe_link_unavailable>"
  }
}
```

请从第一个匹配的“创建后路由”条件中选择原因。初始浏览器路由时，请勿使用创建后的回退原因。

后续操作请参考 `/opt/hatch/skills/shopping/references/browser-checkout.md`。

### Stripe Link，直接完成

使用上述选定的 Stripe Link 支付方式，然后继续直接流程。

## 使用已选钱包

请复用前面已确定的支付方及具体的支付方式。无需再次询问支付方式的选择。如果所选支付方式已不可用，请在完成或转交浏览器处理之前停止，并返回到“选择支付方式”环节。请勿擅自更换其他支付方式。因 Stripe Link 无法完成本次结账而改用浏览器路由时，应设置 `stage: "agentic_fallback"` 且 `reason: "stripe_link_unavailable"`。

## 刷新配送、折扣与总计

如果结账页面提供配送选项，请选择一项。若有多项且用户已明确表达配送偏好（如截止日期、预算或速度），或各选项差异不大，则请选择最符合需求的一项并告知用户；否则，请列出各选项及其价格与预计送达时间，由用户自行选择。在完成结算前，请更新可信报价：

如果结账页面要求为不同商品组分别选择配送方式，则不得尝试直接完成，而应在浏览器中继续操作，返回的结账页面 URL 应设置 `stage: "agentic_fallback"` 且 `reason: "provider_requires_browser"`。

```json
{
  "checkout_id": "<结账 ID>",
  "selected_delivery_option_id": "<配送选项 ID>"
}
```

```sh
HATCH_SHOPPING_PRODUCT_CONTEXTS='<与创建时相同的 JSON 数组>' shopify-ucp-cli checkout update --input-file "<update.json>" --format json
```

要应用促销码或优惠券代码，请在 `discount_codes` 中传递所需的完整集合。使用用户提供的代码，或在用户请求的促销搜索中找到的代码。不要自行生成代码，也不要在每次结账时都要求输入代码。省略 `discount_codes` 可保留结账当前已有的代码；使用空数组则清空所有代码。配送选项和折扣代码可以在一次调用中同时更新：

```json
{
  "checkout_id": "<checkout-id>",
  "selected_delivery_option_id": "<delivery-option-id>",
  "discount_codes": ["<promo-code>"]
}
```

仅需更新折扣时，只需提供 `checkout_id` 和 `discount_codes`。返回已应用的折扣信息（`result.checkout.rejected_discount_codes`）以及更新后的总金额；即使 `ok` 为 `true`，也可能存在被拒绝的代码。如果结账不涉及配送选项，则无需更改配送选择，但可以请求更新折扣。

## 审核并完成
请使用上述步骤中精确保存的支付方式。如果用户要求切换支付方式，请返回到“支付与钱包”页面，重新选择之前保存的支付方式，不要要求用户再次确认他们刚刚提出的切换请求。

显示已完成的报价，其中包含遮蔽后的支付方式、商品明细、最终总额以及配送选项。此报价应作为购买流程中的“购买审核”展示。对于 Stripe Link，需获取用户的明确授权并等待；对于 Shop Pay，则无需额外的聊天确认。`checkout complete` 请求的是钱包端的授权，该授权即为最终购买确认。钱包连接及此前的购买请求并不构成对本次报价的确认。随后，提交仅包含可信结账 ID、所选钱包提供商、所选支付方式 ID 以及（如有）所选配送选项 ID 的完成输入：

```json
{
  "checkout_id": "<checkout-id>",
  "wallet_provider": "<stripe_link-or-shop_pay>",
  "payment_method_id": "<selected-wallet-payment-method-id>",
  "selected_delivery_option_id": "<delivery-option-id>"
}
```

每次完成输入中都必须包含 `wallet_provider`：Stripe Link 使用 `"stripe_link"`，Shop Pay 使用 `"shop_pay"`。对于 Shop Pay，请使用 `wallet.list_payment_methods` 返回的确切工具 ID。Shop Pay 流程会将凭据保留在受信的支付服务端。由提供商 CLI 生成支付授权。若商家支持直接 Shop Pay，运行时会将该授权 ID 添加到所选凭据中，并提交给商家。如果无法进行直接 Shop Pay 结算，则应通过浏览器流程完成，而不要生成银行卡详情。运行时不会接收单独的买家身份令牌。

如果结账不涉及配送选项，则省略 `selected_delivery_option_id`。

```sh
HATCH_SHOPPING_PRODUCT_CONTEXTS='<与 create 相同的 JSON 数组>' shopify-ucp-cli checkout complete --input-file "<complete.json>" --format json
```

读取顶层的 `ok` 字段：`true` 表示订单已下单；`false` 表示未下单。切勿粘贴原始 `.result` JSON，也不要暴露内部 ID、API 字段、买家联系方式或收货地址等敏感信息。仅汇总并向用户展示可用的字段：订单状态、商品、商家、最终金额、预计送达时间及确认链接。如果在保存银行卡信息后完成失败，或报告了未知结果，切勿断言未下单，也不要自动重试或切换至浏览器结算。