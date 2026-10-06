# 浏览器结账

当“购买”工作流通过商品或结账页面发起购买时，请使用浏览器结账。借助本参考，您可以启动并继续 BrowserTask。

## 启动浏览器任务

调用 `browser.spawn_task`，传入准确的商品或结账页面 URL，以及本次购买已做出的所有选择：

```json
{
  "task": "从<准确的商品或结账页面 URL>购买<商品列表>。使用以下选项：<变体、数量、配送详情、支付方式及其他要求>。仅询问尚未提供的必填项或结账选项。继续完成结账流程，并在提交前移交最终的完整条款。"
}
```

如果选择了钱包支付方式，请附上其准确的提供方 ID；若同时选择了已保存的支付方式，还需附上该方式的脱敏标识。请勿在 `task` 中包含不透明的支付方式 ID。如已发生支付拒绝或结账失败，也应在任务中注明。若未选择任何支付方式，则可省略此项。运行时会将支持该结账流程的支付提供方自动添加到 BrowserTask 的交接信息中。

请勿要求 BrowserTask 通过结账按钮推断可用的支付提供方。当用户选择“关联支付”且附加了诸如“存在时”或“可用时”之类的条件时，应将 `stripe-link` 作为所选提供方传递，但不要将该条件转化为商家必须提供 Stripe Link 按钮的要求。浏览器结账允许通过浏览器接管实现“使用其他方式”的选项，并将其与其他符合条件的支付方式一并呈现。若用户选择了此选项，请明确说明将在浏览器接管期间输入支付信息，且无需提供卡片详细信息。

## 添加商品目录路由信息

某些由 `shopping product-details` 返回的商品会包含 `hatch_telemetry_context`。对于这些商品，在浏览器任务中添加 `shopping_checkout` 字段，并原样复制每个商品的完整 `hatch_telemetry_context` 至 `products` 中，不得修改。这些信息用于记录为何采用了浏览器结账，但不会影响实际的结账流程。

根据启动浏览器任务的具体情况设置 `stage` 和 `reason`：

| 情况 | `stage` | `reason` |
|---|---|---|
| 代理式结账创建不可用，且未调用 `checkout create` | `checkout_start` | `agentic_creation_ineligible` |
| 用户在调用 `checkout create` 之前选择了浏览器结账 | `checkout_start` | `user_selected_browser` |
| Shop Pay 必须在浏览器中完成 | `payment_lane` | `shop_pay_selected` |
| `checkout create` 调用失败 | `agentic_fallback` | `agentic_create_failed` |
| 用户在调用 `checkout create` 之后选择了浏览器结账 | `agentic_fallback` | `user_selected_browser` |
| 所选支付提供方要求使用浏览器结账 | `agentic_fallback` | `provider_requires_browser` |
| 代理式结账无法完成 | `agentic_fallback` | `agentic_completion_ineligible` |
| 浏览器需收集必要的买家信息 | `agentic_fallback` | `buyer_details_required` |
| Stripe Link 无法用于代理式结账 | `agentic_fallback` | `stripe_link_unavailable` |

对于仅通过浏览器发现的商品，无需添加 `shopping_checkout`。

示例：

```js
{
  "task": "<自包含的浏览器结账任务>",
  "shopping_checkout": {
    "products": [<每个商品目录项的完整 hatch_telemetry_context>],
    "stage": "checkout_start",
    "reason": "agentic_creation_ineligible"
  }
}
```

## 继续购买流程

按照 `browser.spawn_task` 返回的确认信息执行操作，使用 `browser.steer_task` 继续同一任务。在钱包设置或确认阶段，请勿用新任务替换当前任务。