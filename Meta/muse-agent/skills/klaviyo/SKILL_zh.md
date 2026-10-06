---
name: "klaviyo"
description: >-
  读取并管理用户的 Klaviyo 电子邮件和短信营销活动、新闻通讯，
  流量、订阅者列表和受众细分。用于比较营销活动
  业绩与收入、寻找近期客户、规划受众定位，以及
  通过 Klaviyo 的官方 MCP 服务器管理营销内容和订阅。
icon: "klaviyo"
metadata: { "不包含在提示中": 假 }
---
# Klaviyo

使用已安装的 `klaviyo` CLI。连接后可访问完整的受支持目录。写入、发送和删除操作需经 Hatch 审批。设置将目录划分为八大功能：读取、营销内容、活动投放、工作流、受众与订阅、目录与优惠券、跟踪与集成，以及数据删除。编辑内容并不授予发送活动或删除数据的权限。每次审批仍会预览具体操作及其输入。创建活动及编辑活动/消息时，会显示所提交的受众、发件人、主题，以及如有则显示消息内容。活动发送/取消以及工作流状态/动作的审批还会显示可用的 Klaviyo 元数据；若查找失败，则保留原始输入。在可用时会显示受众 ID 和收件人数估算值，但不会推断受众名称或保证送达数量。

运行 `klaviyo list-tools` 可查看精简的已审核工具目录。这些工具名称及其对应的 Hatch 权限在连接前即可获取。运行 `klaviyo list-tools --name <tool-name>` 可查看某已审核工具的实时输入架构；为限制发现范围，提供方的输出架构被有意省略。请勿猜测工具名称或参数架构。

如果 `klaviyo status` 显示 `auth_status: not_connected`，请运行 `klaviyo authorize-url`，并仅分享返回的 `connect_url`。Klaviyo 要求用户具备所有者、管理员或经理角色。请勿在聊天中自行构造 OAuth URL 或请求令牌。

```text
klaviyo call-tool --help
klaviyo status
klaviyo list-tools [--name <tool-name>]
klaviyo call-tool --name <tool-name> --arguments-json '<JSON 对象>'
klaviyo account-details
klaviyo list-campaigns --channel <email|sms|mobile-push> [--page-cursor <cursor>]
klaviyo list-flows [--page-cursor <cursor>] [--page-size <1-100>]
klaviyo list-metrics [--page-cursor <cursor>]
```

该目录涵盖稳定的远程活动、工作流、受众、订阅、模板、图片、目录、事件、指标、报告、优惠券、标签、Webhook、表单、评价、推送令牌以及个人资料删除等工具。测试版及仅本地使用的工具不在其中。Klaviyo 的 MCP 支持发送活动；Hatch 的明确白名单控制此处可调用的工具。若 `list-tools --name` 报告某已审核工具不可用，请先检查连接状态，再判定其是否不受支持。请勿自行构建架构，亦勿绕过 CLI 直接调用 API。

- 活动：在执行 `send_campaign` 前，请核对受众、消息内容及计划时间。发送与取消共享一项活动投放权限，与内容编辑权限分开。成功请求会启动异步任务；在报告结果前，请查询 `get_campaign_send_job` 和 `get_campaign`。`cancel_campaign_send` 可在 Klaviyo 允许的情况下取消发送或将发送状态回滚至草稿，请核对状态。取消操作无法召回已送达的邮件。创建的草稿不会被发送。
- 工作流：`create_flow` 使用编码的工作流定义，请保留返回的 ID。`update_flow` 会同时更改工作流状态及其所有动作。请在变更前后分别查询 `get_flow`、`get_flow_action` 和 `get_flow_message`。使工作流或动作上线后，可能会开始发送消息。
- 受众：列出成员信息并不授予或撤销营销同意；如需变更同意，请使用订阅相关工具。添加列表成员、让个人资料订阅或创建事件都可能触发工作流。请在进行上述变更前检查相关工作流。
- 批量变更、合并与删除：请确认目标集合及其影响。个人资料删除为永久性操作。请保留返回的任务 ID，并在声称完成前使用对应的任务状态查询。任何不确定的写入或删除结果发生后，请在重试前检查当前状态。

`assign_template_to_campaign_message` 会返回一个全新的、归消息所有的新模板副本，其 ID 与源模板不同。请勿因这一替换而重复尝试；请使用 `include=campaign-messages` 参数通过 `get_campaign` 进行验证。在存在时，请保留返回的 Klaviyo UI 链接。