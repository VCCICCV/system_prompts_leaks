---
name: "ghl"
description: "使用 HighLevel 的联系人、销售漏斗、预约、消息以及其更广泛的运营功能。"
icon: "📇"
metadata: { "不包含在提示中": 假 }
---
# HighLevel

在面向用户的消息中，始终使用产品名称 **HighLevel**。切勿在其前加“Go”前缀，也切勿将内部标识符 `ghl` 展开为显示名称。

使用 `/opt/hatch/bin/ghl status` 检查连接状态。若已断开连接，请运行 `/opt/hatch/bin/ghl authorize-url`，并分享返回的 `connect_url`。用户应通过 HighLevel 进行登录；切勿在聊天中索取凭据。

## 查找操作与子账户

运行 `/opt/hatch/bin/ghl list-operations` 获取固定的操作目录及其细粒度的权限方法。该目录涵盖 HighLevel 的 CRM、通信、日程安排、商务、内容、管理及 AI 等领域。

CRM/日历读取权限与会话/消息读取权限是分开的。联系人编辑、备注/任务、商机、预约、自动化、消息发送以及 CRM 删除均各自独立受控。即使消息读取权限被拒绝，也不会影响 CRM/日历的读取权限。

使用 `/opt/hatch/bin/ghl call-tool --name list_locations --arguments-json '{"query":"<子账户名称>"}'` 解析目标子账户。若有多个匹配项，请询问用户具体指哪个，并保留其 ID 以供后续执行。

使用 `call-tool --name describe_operation --arguments-json '{"operationId":"<操作ID>"}'` 获取精确的路径/查询/请求体字段及必填参数。`search_operations` 可搜索 HighLevel 的实时操作目录。若存在，则优先使用 `list-operations` 中固定的权限配置；对于尚未进入固定目录的未来提供方操作，在权限审核完成前仍需一次性写入授权。

`list-tools` 返回实时发现的工具 Schema。这些调用同样使用 `/opt/hatch/bin/ghl`。

## 读取或变更数据

```bash
/opt/hatch/bin/ghl execute-operation \
  --operation-id <操作ID> --location-id <子账户ID> \
  --params-json '{"path":{},"query":{},"body":{}}'
```

请根据 `describe_operation` 提供路径 ID 和请求体字段。创建/更新/插入（upsert）联系人时不可包含 `tags`，应使用 `add-tags` 或 `remove-tags`；插入商机时不可包含关注者管理相关字段，应使用单独的关注者添加/移除操作。仅接受分组后的 `path`、`query` 和 `body` 对象；必要时可省略空的分组。每次逻辑上的写入或删除操作，还需提供一个全新的 `--idempotency-key <UUID>`。可选的 `--reason` 参数用于说明操作原因。`--dry-run` 可预览解析后的请求而不更改 CRM 数据，且无需提供幂等键。试运行不会授权后续的实际写入。

默认情况下，写入和删除操作需按其所属领域的权限进行审批。消息相关的变更（包括发送与取消）每次调用均需审批。预约变更可通知联系人或触发已配置的自动化流程（`toNotify` 默认为 true）；活动与工作流的加入也可启动通信。请在请求中明确预期的通知行为。更新或删除记录前务必先进行读取确认，切勿猜测目标 ID。

CLI 不会自动重试变更操作。若提交后写入失败或结果未确认，请在再次尝试前检查记录或服务提供商的状态。针对同一逻辑变更，请始终使用相同的幂等键，切勿为应对不确定性而生成新的键。除非服务提供商确认送达，否则应将消息状态报告为“已接收”或“已计划”。

对于财务、发布、消息发送、访问控制及删除类操作，请视为重大变更：务必先描述操作，展示确切的预期参数，切勿跨子账户推断或复用标识符。尚未出现在 `list-operations` 中的未来操作，即使是读取或试运行，也需重新获得授权。