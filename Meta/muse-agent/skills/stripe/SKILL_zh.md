---
name: "stripe"
description: >-
  管理商家的 Stripe 客户、产品、优惠券、促销代码及支付事务，
  发票和定期订阅。用于计费方案变更、支付链接等。
  重复刷卡收费的退款、争议与拒付、账户余额，
  支付款项，并通过 Stripe 的官方 MCP 服务器对处理费用进行对账。
icon: "connectorStripe"
metadata: { "不包含在提示中": 假 }
---
# Stripe

使用已安装的 `stripe` CLI。首先运行 `stripe status`。如果显示 `not_connected`，请运行 `stripe authorize-url`，并将仅返回的 `connect_url` 提供给用户。

OAuth 通过 authd 使用动态客户端注册和 PKCE。Stripe 为每个虚拟机颁发一个公共客户端，因此不会有任何共享凭据进入 Muse，且绝不能在聊天中请求凭据。

运行 `stripe list-tools` 来查看实时的提供商目录和架构，然后通过以下命令调用已公布的工具：

```text
stripe call-tool --name <tool> --arguments-json '<json-object>'
```

OAuth 连接会标识商户账户。切勿向用户索取该已连接账户的 `acct_...` ID。如果其他工具需要 `stripe_context`，请先使用空的参数对象调用已公布的 `list_available_accounts_or_orgs` 工具，并使用其返回的上下文。如果有多于一个上下文符合所请求的实时或测试模式，请列出它们的名称并询问用户使用哪个账户；切勿要求用户粘贴账户 ID。切勿自行构造上下文，也不要在未从 Stripe 获取上下文的情况下调用与账户相关的工具。

`list-tools` 仅暴露经过审核的 Stripe 工具，并包含每个工具的 `hatch_permission`、`hatch_action` 和 `hatch_permission_label`。未知或新引入的提供商工具在审核通过前均不可用。对于 `stripe_api_write`，返回的架构会列出已审核的 `stripe_api_operation_id` 及其特定操作的权限覆盖；未列出的操作 ID 均不可用。读取权限遵循用户的连接器设置；Stripe API 的写入操作及反馈需要逐项批准。对于敏感操作，Stripe 还可能要求通过提供商 URL 进行二次确认。切勿自动重试失败或超时的写入操作，因为其副作用可能已经执行完毕。
