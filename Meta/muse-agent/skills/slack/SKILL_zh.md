---
name: "slack"
description: >-
  在用户的 Slack 工作区中阅读和搜索消息、频道及话题线程。
  用于了解团队的讨论与决策，发布频道消息，以及
  通过 Slack 的官方 MCP 服务器回复同事。
icon: "slack"
metadata: { "不包含在提示中": 假 }
---
# Slack

使用已安装的 `slack` 命令行工具。首先运行 `slack status`。如果显示 `not_connected`，请运行 `slack authorize-url`，并将返回的 `connect_url` 仅分享给用户。

Slack 需要固定的已注册应用身份，不支持动态客户端注册。测试应用仅限于其开发工作区；生产应用必须先通过 Slack Marketplace 的审核并获得分发许可，其他工作区的用户才能进行连接。

运行 `slack list-tools` 可查看已审核的实时提供商目录子集及其权限标注，然后通过以下命令调用已发布的工具：

```text
slack call-tool --name <tool> --arguments-json '<json-object>'
```

Slack 的工具目录既包含读操作也包含写操作，并且可能会随时间变化。命令行工具仅允许调用经过明确审核的工具：读操作默认允许，而写操作则需要在预览参数后获得批准。对于未知或新引入的提供商工具，将直接拒绝执行。直接文件读取、签名上传 URL 以及未经审核的列表类工具，在未配备专门的交付处理机制之前仍不可用。切勿在聊天中请求 OAuth 令牌或应用凭据。
