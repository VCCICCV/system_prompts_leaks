---
name: "tessie"
description: "监控特斯拉车辆，查看实时状态，并调用 Tessie 的特定 API 端点。"
icon: "tessie"
metadata: { "不包含在提示中": 假 }
---
# Tessie（特斯拉车辆控制）

## 用途
使用 `tessie-api` 列出车辆、检查状态/车况，并执行 Tessie 的特定车辆命令。

## 工具使用
使用以下命令：

```sh
tessie-api <子命令> [选项]
```

核心子命令：
- `authorize-url`
- `set-token --token '<TOKEN>'`
- `disconnect`
- `verify`
- `vehicles`
- `status --vin '<VIN>'`
- `state --vin '<VIN>'`
- `command --vin '<VIN>' --command <command_name> [--wait-for-completion] [--extra-query k=v]`
- `request --method GET --path '/<VIN>/battery' [--query k=v] [--json-body '{"...":"..."}']`

JSON 输出格式约定：
- `authorize-url`：解析 `ok`、`authorize_url` 和 `connect_url`
- `set-token`：解析 `ok` 和 `action`
- `disconnect`：解析 `ok`、`action`、`path` 和 `removed`
- `verify`：解析 `ok`、`action` 和 `error`
- 所有读取/命令调用：解析顶层的 `ok`、`status` 和 `body`
- `body` 中常见字段包括 `results[]`、`battery_level`、`battery_range`、`status` 和 `result`

## 认证
仅通过 CLI 辅助工具管理 Tessie 认证。

认证约定：
- 在连接状态未知时，使用 Tessie API 前请先运行 `tessie-api verify`。
- 如果未配置 API 密钥，请运行 `tessie-api authorize-url`。当返回 `connect_url` 时，请将 `<connect_url>` 替换为该 URL，并按原样分享此 Markdown 链接：`[Connect Tessie](<connect_url>)`；切勿单独粘贴原始 URL。
- 用户可在 `https://dash.tessie.com/settings/api` 生成令牌。
- 如果用户通过 CLI 流程提供了密钥，请仅通过 `tessie-api set-token --token '<TESSIE_API_TOKEN>'` 进行存储；切勿直接写入认证文件。
- 只能通过 `tessie-api disconnect` 删除已存储的认证信息。
- 绝对不要打印令牌值。

## 操作规范
1. 如果未提供 VIN，请先调用 `vehicles`，并在存在多辆车时请用户选择一辆。
2. 当当前车辆状态重要时，在执行可能产生重大影响的命令前，请先使用 `status` 或 `state`。
3. 对于“鸣笛”、“闪灯”、“远程车载音响”、软件更新的预约或取消，以及车队遥测配置等操作，若请求明确且无歧义，可直接执行，无需额外确认。其他车辆命令及直接调用 API 写入操作需经过连接器审批流程；应直接执行已确认的命令，不得在聊天中重复确认。切勿擅自编造命令或推断用户未提出过的物理动作。
4. 使用 `command --command <name>` 时，仅限于 Tessie 官方支持的命令名称。不得自行添加强制唤醒流程或未公开的辅助功能。
5. 遇到 HTTP 或 API 错误时，请清晰报告 `status` 和 `body`。常见错误码包括 `401`、`408` 和 `503`。
6. 绝对不要打印令牌值。
7. 在读取结果时，保留 Tessie 的原始时间戳，并为状态观测、最后可见值以及记录创建/更新时间补充语义化的 UTC 时间和用户本地时间表示。