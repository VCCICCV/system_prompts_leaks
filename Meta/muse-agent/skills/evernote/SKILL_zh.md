---
name: "evernote"
description: "通过Evernote的官方MCP服务器阅读和创建笔记。"
icon: "evernote"
metadata: { "不包含在提示中": 假 }
---
# Evernote

使用已安装的 `evernote` 命令行工具。首先运行 `evernote status`。如果显示
`not_connected`，请运行 `evernote authorize-url`，并将返回的
`connect_url` 仅提供给用户。

OAuth 通过 authd 使用动态客户端注册和 PKCE。Muse 中不会出现共享的客户端密钥，
且绝不能在聊天中请求用户的凭据。

运行 `evernote list-tools` 以查看当前可用的服务目录和数据模型，然后通过以下命令调用已发布的工具：

```text
evernote call-tool --name <tool> --arguments-json '<json-object>'
```

`list-tools` 仅列出已审核的 Evernote 工具，并包含每个工具的
`hatch_permission`、`hatch_action` 和 `hatch_permission_label`。未知或新接入的服务工具在未完成审核前仍不可用。读取权限遵循用户的连接器设置；创建笔记则需要相应的细粒度授权。对于失败或超时的写入操作，请勿自动重试，因为其副作用可能已经执行完毕。
