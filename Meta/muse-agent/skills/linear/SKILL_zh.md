---
name: "linear"
description: >-
  在 Linear 中阅读和管理软件问题、工单、项目及规划数据。
  用于回顾冲刺和周期，识别工单延期和发布障碍，
  并通过 Linear 更新问题状态、分配对象、优先级和里程碑
  官方MCP服务器。
icon: "linear"
metadata: { "不包含在提示中": false }
---
# Linear

使用已安装的 `linear` CLI。首先运行 `linear status`。如果报告为 `not_connected`，则运行 `linear authorize-url`，并将返回的 `connect_url` 仅分享给用户。

OAuth 使用通过 authd 实现的动态客户端注册和 PKCE。没有任何共享的客户端密钥会进入 Muse，且在聊天中绝不能请求凭据。首次连接时请求的是只读访问权限。在进行写操作之前，或在因缺少访问权限而失败后，运行 `linear status --for-command <permission>`。对于 Linear 的 `save_*` 工具，当参数中不包含 `id` 时使用对应的 `*.create` 权限，当包含 `id` 时使用对应的 `*.manage` 权限。如果状态命令返回 `scope_status: not_granted`，请原样复制 `scope_add_url` 并将其单独放在一行，格式为 `[Additional Linear access](<scope_add_url>)`，以便客户端渲染原生访问按钮，然后等待同意完成后再重试一次。OAuth 访问权限不能替代 Hatch 审批。切勿在聊天中构造范围 URL 或请求令牌。

运行 `linear list-tools` 以查看实时提供者目录和模式，然后按以下方式调用已公布的工具：

```text
linear call-tool --name <tool> --arguments-json '<json-object>'
```

`list-tools` 仅显示经过审核的 Linear 工具，并包含每个工具的 `hatch_permission`、`hatch_action` 和 `hatch_permission_label`。对于多态的 `save_*` 工具，权限字段会显示执行时使用的基于 `id` 的选择器。当 OAuth 授权为只读时，Linear 会将其写入工具从实时目录中隐藏；切勿将全为只读的“工具”列表描述为缺乏写入支持。在这种状态下，`list-tools` 会将源自清单的 `additional_access` 条目与提供者公布的工具分开返回。当用户请求其中一项功能时，请按照上述“Additional Linear access”链接的精确格式分享其 `scope_add_url`，等待同意后，再重新运行 `linear list-tools` 以获取提供者的实时写入工具模式。未知或新公布的提供者工具在审核完成前仍不可用。读取权限遵循用户的连接器设置；更改需要相应的细粒度审批。切勿自动重试失败或超时的写入操作，因为其副作用可能已经完成。
