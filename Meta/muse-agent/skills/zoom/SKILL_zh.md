---
name: "zoom"
description: >-
  查看 Zoom 会议、通话、录制文件和文字记录。用于
  查看未接来电、汇总决策，并提取待办事项及
  所有者；还可与团队聊天、Canvas、任务、白板、中心和收入功能协同工作。
  通过 Zoom 的官方 MCP 服务器加速。
icon: "zoom"
metadata: { "不包含在提示中": 假 }
---
# Zoom

使用已安装的 `zoom` 命令行工具。首先运行 `zoom status`。如果显示 `not_connected`，请运行 `zoom authorize-url`，并将返回的 `connect_url` 仅分享给用户。

OAuth 使用固定的 Muse Zoom OAuth 客户端，并通过 authd 实现 PKCE。CAGI 提供用于令牌交换和刷新的客户端密钥；该密钥绝不能进入 Muse 系统，也不应在聊天中请求。

运行 `zoom list-tools` 可以查看来自所有官方 Zoom MCP 服务器的实时目录和模式。若只想查看某一台服务器，请使用 `zoom list-tools --server <server>`，其中 `<server>` 可为 `zoom`、`meeting`、`chat`、`canvas`、`tasks`、`whiteboard` 或 `revenue-accelerator`。

在返回该工具的服务器上调用已发布的工具，命令如下：

```text
zoom call-tool --server <server> --name <tool> --arguments-json '<json-object>'
```

出于向后兼容性考虑，默认使用 `zoom` 服务器。目前，专用的 `meeting` 服务器与一体化的 `zoom` 服务器上的会议和录制工具功能有所重叠，而其他专用服务器则提供了更广泛的产品能力。

提供商目录可能同时包含读取和变更操作，并且会随时间变化。CLI 会根据所选服务器的实时目录对工具名称进行校验，并在每次调用工具前要求明确确认。切勿自动重试失败或超时的工具调用，因为其副作用可能已经执行完毕。

如果专用服务器在升级后报告缺少 OAuth 范围，请提示用户重新连接 Zoom，以便批准扩展后的授权。未经用户确认，切勿断开现有连接。

部分服务器需要单独许可的 Zoom 产品。当某个服务器出现错误时，应视该服务器暂时不可用；对于报告 `ok: true` 的目录，应继续使用，而不应因此判定整个 Zoom 连接已失效。
 
目前，`zoom` 和 `meeting` 服务器均提供 `meeting_create`、`meeting_update` 和 `meeting_delete` 工具，因此请使用这些实时工具进行会议的安排和管理。切勿仅凭 OAuth 范围推断工具是否可用；在调用之前，请先通过 `zoom list-tools --server meeting` 确认该工具及其当前模式。