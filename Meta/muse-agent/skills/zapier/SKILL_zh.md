---
name: "zapier"
description: "通过 Zapier 的官方 MCP 服务器，将 Muse 与各应用中的操作相连接。"
icon: "zapier"
metadata: { "不包含在提示中": 假 }
---
# Zapier

使用已安装的 `zapier` 命令行工具。首先运行 `zapier status`。如果显示
`not_connected`，请运行 `zapier authorize-url`，并将仅返回的
`connect_url` 提供给用户。

OAuth 通过 authd 使用动态客户端注册和 PKCE。Muse 中不会出现共享的客户端密钥，
且在聊天中绝不能请求凭据。Zapier 的 MCP 授权暴露的是身份范围，而非单独的读写范围，
因此默认的只读行为由连接器权限来强制执行。

运行 `zapier list-tools` 可以查看当前可用的提供商目录及其模式定义，
然后通过以下命令调用已发布的工具：

```text
zapier call-tool --name <tool> --arguments-json '<json-object>'
```

`list-tools` 仅列出已审核的 Zapier 自动化模式工具，并包含每个工具的
`hatch_permission`、`hatch_action` 和 `hatch_permission_label`。
未知或动态的托管模式工具在审核完成之前仍不可用。读取权限遵循用户的连接器设置；
执行写操作和更改配置则需要细粒度的授权。对于失败或超时的写操作，请勿自动重试，
因为其副作用可能已经完成。