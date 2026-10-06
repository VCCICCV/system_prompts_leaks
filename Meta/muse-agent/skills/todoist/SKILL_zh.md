---
name: "todoist"
description: "通过Todoist官方MCP服务器，阅读并管理Todoist中的任务、项目、评论、标签、过滤器和提醒。"
icon: "todoist"
metadata: { "不包含在提示中": 假 }
---
# Todoist

使用已安装的 `todoist` 命令行工具。首先运行 `todoist status`。如果显示 `not_connected`，请运行 `todoist authorize-url`，并将返回的 `connect_url` 仅提供给用户。

OAuth 通过 authd 使用动态客户端注册和 PKCE。Muse 中不会存储任何共享的客户端密钥，且绝不能在聊天中请求用户的凭据。

首次连接时仅申请 Todoist 的只读权限。如果因未授予 Todoist 的写入或删除权限而导致请求的更改失败，请引导用户前往设置中进行授权；该操作的 Hatch 审批仍需单独进行。

运行 `todoist list-tools` 可查看当前可用的服务目录及数据模型，然后按如下方式调用已发布的工具：

```text
todoist call-tool --name <tool> --arguments-json '<json-object>'
```

`list-tools` 仅列出已审核的 Todoist 工具，并包含每个工具的 `hatch_permission`、`hatch_action` 和 `hatch_permission_label`。`delete-object` 工具的授权范围为：项目对应的 `projects.delete` 权限，以及其他所有对象类型的 `items.delete` 权限。未知或新接入的服务提供的工具在完成审核之前均不可用。读取权限遵循用户的连接器设置；相关变更需获得相应的细粒度审批。对于失败或超时的写入操作，切勿自动重试，因为其副作用可能已经执行完毕。
