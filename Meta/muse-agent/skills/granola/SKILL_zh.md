---
name: "granola"
description: "通过Granola的OAuth认证MCP服务器搜索并阅读Granola会议记录与文字稿。"
icon: "granola"
metadata: { "不包含在提示中": 假 }
---
# Granola

## 用途
通过托管的 Granola MCP 服务器（`https://mcp.granola.ai/mcp`）搜索 Granola 的会议记录、摘要和文字稿。

## 工具使用
使用 `exec` 运行以下命令：

```sh
granola-cli <子命令> [选项]
```

### 连接管理

```sh
granola-cli status
granola-cli authorize-url
granola-cli exchange-code --code <code> [--redirect-uri <url>]
granola-cli refresh
granola-cli disconnect
```

### MCP 操作

```sh
granola-cli list-tools
granola-cli call-tool --name <tool> --arguments-json '<json-object>'
```

`--arguments-json` 必须为 JSON 对象；数组或标量将被拒绝。请先使用 `list-tools` 查看工具目录及每个工具的 `input_schema`。

## 认证
OAuth 由 authd 通过动态客户端注册 + PKCE（S256）机制处理。凭据始终保存在 authd 内部，请勿读取或编辑连接器的认证文件。

## 首次使用设置流程

1. 运行 `granola-cli status`。
2. 如果状态为 `not_connected`，则运行 `granola-cli authorize-url`。当出现 `connect_url` 时，请将 `<connect_url>` 替换为返回的 URL，并按原样分享此 Markdown 链接：`[Connect Granola](<connect_url>)`；切勿单独粘贴原始 URL。等待用户完成授权。
3. 用户授权后，浏览器会经由 Muse 中继返回本虚拟机，authd 将自动完成代码交换。令牌到达后，代理将恢复运行。
4. 再次运行 `granola-cli status`。当状态变为 `connected` 时，输出还将包含已发现的 MCP 工具目录。

对于未配置中继的手动环境，在收到授权码后，请调用 `granola-cli exchange-code --code <auth-code>`。

## 操作规范
1. 在进行任何 MCP 操作前，请先运行 `granola-cli status`。如果状态不是 `connected`，请先完成设置流程。
2. 在调用 `call-tool` 前，请务必先运行 `list-tools`，除非您已知工具名称及其参数结构。切勿猜测工具名称。
3. `arguments-json` 必须为 JSON 对象；请为每个参数正确添加 JSON 格式。
4. 对于自然语言问题，优先使用 `query_granola_meetings`；对于元数据，使用 `list_meetings`；对于已知会议 ID，使用 `get_meetings`；仅当用户需要逐字逐句的详细内容时才使用 `get_meeting_transcript`。使用 `get_account_info` 可以确认当前连接的是哪个账户/工作区；当查询结果为空时，可检查 `mcp_note_access.scopes`——如果某个工作区的 MCP 访问权限仅为“公开”，则不包含个人笔记。`list_meetings` 默认显示最近 30 天的会议；若返回结果为零次会议，请先尝试使用 `time_range: "custom"` 并指定 `custom_start` 和 `custom_end`，再确认该账户确实没有会议，并使用 `list_meeting_folders` 查看该工作区实际包含的内容。
5. 在面向用户的回答中，请保留 Granola 的引用链接。
6. 请勿使用 Granola 进行日程安排或未来活动规划。
7. 当出现 401 错误时，令牌会自动刷新。如果 `call-tool` 持续返回 `unauthorized` 错误，请显式运行 `granola-cli refresh`，或请用户重新授权。
