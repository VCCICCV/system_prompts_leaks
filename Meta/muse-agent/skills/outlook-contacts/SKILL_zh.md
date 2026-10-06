---
name: "outlook_contacts"
description: "在用户的 Outlook 账户中列出、搜索、创建、更新和删除联系人。"
icon: "outlook_contacts"
metadata: { "不包含在提示中": false }
---
# Outlook 联系人

## 目的
使用本地辅助 CLI 管理 Outlook 联系人，以保持提示简洁。  
该连接器同时支持个人 Microsoft 帐户以及 Microsoft 365 工作或学校帐户。

## 工具
使用以下命令：

```sh
outlook-contacts <子命令> [选项]
```

核心子命令：
- `disconnect`
- `list [--page-size 10] [--page-token <token>]`
- `get "AAMkAD..."`
- `search "alice smith" --page-size 10`
- `create --given-name "Alice" --family-name "Smith" --email alice@example.com --phone "+15551234567"`
- `update "AAMkAD..." --given-name "Alice" --email newalice@example.com`
- `delete "AAMkAD..."`

常用选项：
- `--timeout-secs N`（默认 `30`）

使用 `--page-size` 指定返回结果的数量。`--top`、`--limit` 和 `--max-results` 仅为兼容性别名，新命令中请勿使用。

JSON 输出规范：
- `disconnect`：解析 `ok`、`action`、`status` 和 `disconnect_url`
- `list` / `search`：解析 `ok`、`count`、`next_page_token`（存在时作为 `--page-token` 传回）、`total_people`，以及 `contacts[].resource_name`、`contacts[].display_name`、`contacts[].emails`、`contacts[].phones`
- `get`：解析 `ok` 以及 `contact.resource_name`、`contact.display_name`、`contact.emails`、`contact.phones`、`contact.organization`、`contact.title`
- `create` / `update` / `delete`：解析 `ok`、`action` 和 `resource_name`

## 认证
此技能依赖于由 `outlook-contacts` 管理的 Outlook 连接器。

使用以下命令：

```sh
outlook-contacts --status
```

规则：
- 进行连接或重新连接时，将 `<connect_url>` 替换为返回的 URL，并仅分享此 Markdown 链接：`[Connect Outlook Contacts](<connect_url>)`；请勿单独粘贴原始 URL。
- 断开连接时，运行 `outlook-contacts disconnect`。当存在 `disconnect_url` 时，将 `<disconnect_url>` 替换为返回的 URL，并仅分享此 Markdown 链接：`[Disconnect Outlook Contacts](<disconnect_url>)`；请勿单独粘贴原始 URL。若不存在，则说明连接器已断开。
- 切勿使用共享的 Outlook 联系人连接器辅助 CLI。
- 不得手动编写令牌文件或猜测连接器状态，应依赖 `outlook-contacts --status`。

## 操作规则
1. 使用 `search` 根据姓名、电子邮件或电话号码查找联系人。
2. 使用 `list` 通过 `--page-token`（跳过值）分页浏览联系人。
3. 联系人 ID 是不透明的 Graph 字符串（例如 `AAMkAD...`），请使用 `list` 或 `search` 返回的确切值。
4. `create`、`update` 和 `delete` 可在用户请求明确且无歧义的情况下直接执行，无需额外确认。
5. 更新联系人时，仅指定的字段会更改，未指定的字段将保留原有值。
6. 如果 CLI 报告认证错误，请再次检查 `outlook-contacts --status`，并引导用户重新连接。
7. 切勿打印连接器密钥或输出原始联系人数据，除非用户明确要求。
8. 切勿向用户显示原始 Graph 标识符（如 `AAMkAD...` 等联系人 ID）或其他内部响应字段（变更键、页面/跳过令牌、原始 JSON）——包括摘要、列表或单项注释中。ID 仅可在内部用于串联后续命令（规则 3）。唯一例外是用户明确要求原始 ID，或必须展示 ID 以排查故障时。