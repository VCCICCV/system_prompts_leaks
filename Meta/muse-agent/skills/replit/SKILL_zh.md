---
name: "replit"
description: "通过 Replit 的官方 MCP 服务器，阅读、创建、更新并发布应用。"
icon: "replit"
metadata: { "不包含在提示中": 假 }
---
# Replit

使用已安装的 `replit` 命令行工具。首先运行 `replit status`。如果显示 `not_connected`，请运行 `replit authorize-url`，并将返回的 `connect_url` 仅分享给用户。

OAuth 使用动态客户端注册和 PKCE 协议，通过 authd 进行认证。没有任何共享的客户端密钥会进入 Muse 系统，且绝不能在聊天中请求用户的凭据。首次连接时仅申请只读权限。在执行写操作之前，或在因缺少相应权限而写操作失败后，请运行 `replit status --for-command <tool-name>`。如果返回 `scope_status: not_granted`，请原样复制 `scope_add_url`，并将其单独放在一行，格式为 `[Additional Replit access](<scope_add_url>)`，以便客户端渲染原生的授权按钮，然后等待用户完成授权后再重试一次。OAuth 授权并不能替代 Hatch 的审批流程。切勿在聊天中构造作用域 URL 或请求访问令牌。

运行 `replit list-tools` 可以查看当前可用的服务目录及其数据结构，然后通过以下命令调用已发布的工具：

```text
replit call-tool --name <tool> --arguments-json '<json-object>'
```

`list-tools` 仅列出经过审核的 Replit 工具，并包含每个工具的 `hatch_permission`、`hatch_action` 和 `hatch_permission_label`。未知或新接入的服务提供的工具在未完成审核前均不可用。读取权限遵循用户的连接器设置；创建、编辑和发布应用则需要单独的细粒度审批。对于失败或超时的写操作，请勿自动重试，因为其副作用可能已经执行完毕。