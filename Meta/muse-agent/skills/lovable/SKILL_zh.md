---
name: "lovable"
description: >-
  在 Lovable 中阅读、创建、编辑和部署 Web 应用及项目。用于将一个
  将产品概要转化为可运行的网站或原型，审查现有项目
  代码、更新页面和功能，并通过Lovable的官方渠道发布变更。
  MCP server.
icon: "lovable"
metadata: { "不包含在提示中": 假 }
---
# Lovable

使用已安装的 `lovable` 命令行工具。首先运行 `lovable status`。如果显示 `not_connected`，请运行 `lovable authorize-url`，并将返回的 `connect_url` 仅分享给用户。

OAuth 使用 Lovable 固定的 Muse 客户端，并通过 authd 实现 PKCE。CAGI 提供用于令牌交换和刷新的客户端密钥；该密钥绝不会进入 Muse，也绝不能在聊天中请求。

首次连接时请求的是只读权限。在执行写操作之前，或在因缺少权限而写操作失败后，请运行 `lovable status --for-command <tool-name>`。如果返回 `scope_status: not_granted`，请准确复制 `scope_add_url`，并将其单独放在一行，格式为 `[Additional Lovable access](<scope_add_url>)`，以便客户端渲染原生访问按钮，然后等待用户完成授权后再重试一次。OAuth 授权不能替代 Hatch 的审批。切勿在聊天中构造作用域 URL 或请求令牌。

运行 `lovable list-tools` 可查看当前可用的服务提供商目录及数据模型，然后通过以下命令调用已发布的工具：

```text
lovable call-tool --name <tool> --arguments-json '<json-object>'
```

`list-tools` 仅列出经过审核的 Lovable 工具，并包含每个工具的 `hatch_permission`、`hatch_action` 和 `hatch_permission_label`。未知或新接入的服务提供商工具在未审核通过前均不可用。读取权限遵循用户的连接器设置；编辑、发布及数据库操作则需单独的细粒度审批。对于失败或超时的写操作，请勿自动重试，因为其副作用可能已经完成。
