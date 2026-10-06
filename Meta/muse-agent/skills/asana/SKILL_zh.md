---
name: "asana"
description: >-
  在 Asana 中规划并跟踪团队项目和任务。可用于将工作分配给负责人，
  更新截止日期和状态，设置任务依赖关系，并查看项目进度。
  通过 Asana 的官方 MCP 服务器管理评论、团队和工作区。
icon: "asana"
metadata: { "不包含在提示中": 假 }
---
# Asana

使用已安装的 `asana` 命令行工具。首先运行 `asana status`。如果显示 `not_connected`，请运行 `asana authorize-url`，并将返回的 `connect_url` 仅提供给用户。

OAuth 使用固定的 Muse Asana MCP 应用、PKCE 以及通过 authd 的 Asana MCP 资源指示符。CAGI 提供用于令牌交换、刷新和撤销的客户端密钥；该密钥绝不能进入 Muse 系统，也不应在聊天中请求。请勿请求普通的 Asana API 范围，因为 MCP 应用会拒绝此类请求。

运行 `asana list-tools` 可查看当前可用的提供商目录及模式，然后通过以下命令调用已发布的工具：

```text
asana call-tool --name <tool> --arguments-json '<json-object>'
```

`list-tools` 仅列出已审核的 Asana 工具，并包含每个工具的 `hatch_permission`、`hatch_action` 和 `hatch_permission_label`。未知或新上线的提供商工具在未完成审核前仍不可用。读取权限遵循用户的连接器设置；任何变更均需获得相应的细粒度批准。切勿自动重试失败或超时的写操作，因为其副作用可能已经执行完毕。
