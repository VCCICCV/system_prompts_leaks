---
name: "notion"
description: "通过 Notion MCP 搜索、阅读、创建和更新 Notion 页面。"
icon: "notion"
metadata: { "不包含在提示中": 假 }
---
# Notion

## 用途
通过托管的 Notion MCP 服务器（`https://mcp.notion.com/mcp`）与 Notion 的页面、数据库及工作区内容进行交互。

## 工具使用
使用 `exec` 运行以下命令：

```sh
notion-cli <子命令> [选项]
```

### 连接管理

```sh
notion-cli status
notion-cli authorize-url
notion-cli exchange-code --code <code> [--redirect-uri <url>]
notion-cli refresh
notion-cli disconnect
```

### MCP 操作

```sh
notion-cli list-tools
notion-cli call-tool --name <tool> --arguments-json '<json-object>'
```

`--arguments-json` 必须为 JSON 对象；数组或标量将被拒绝。请先使用 `list-tools` 查看工具目录及其各自的 `input_schema`。

## 认证
OAuth 由 authd 通过动态客户端注册 + PKCE（S256）机制处理。访问令牌存储于 `$JARVIS_HOME/user/auth/notion.json` 中，请勿手动编辑该文件。

## 首次使用设置流程

1. 运行 `notion-cli status`。
2. 如果状态为 `not_connected`，则运行 `notion-cli authorize-url`。当显示 `connect_url` 时，请将 `<connect_url>` 替换为返回的 URL，并按原样分享此 Markdown 链接：`[Connect Notion](<connect_url>)`；切勿单独粘贴原始 URL。等待用户完成授权。
3. 用户授权后，浏览器会经由 Muse 中继返回本虚拟机，authd 将自动完成代码交换。令牌获取成功后，代理程序将继续执行。
4. 再次运行 `notion-cli status`。当状态变为 `connected` 时，输出中还将包含已发现的 MCP 工具目录。

对于未配置中继的环境，请在收到授权码后调用 `notion-cli exchange-code --code <auth-code>`。

## 操作规范
1. 在进行任何 MCP 操作前，请先运行 `notion-cli status`。如果状态不是 `connected`，请先完成设置流程。
2. 在调用 `call-tool` 前，请务必先运行 `list-tools`，除非您已知工具名称及其参数结构。切勿猜测工具名称。
3. `arguments-json` 必须为 JSON 对象；请为每个参数正确地加上 JSON 格式。
4. 在调用会修改 Notion 页面、数据库或块的工具前，请确认用户意图。
5. 当出现 401 错误时，令牌会自动刷新。如果 `call-tool` 仍持续返回 `unauthorized` 错误，请显式运行 `notion-cli refresh`，或提示用户重新授权。
6. 查询结果会保留 Notion 的原始日期字段，并为记录的创建/编辑时间以及带时间的日期属性添加语义化的 UTC 和用户本地时间格式。仅日期类型的属性将保持为日期格式。