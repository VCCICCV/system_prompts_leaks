---
name: "google_forms"
description: "读取、创建和更新用户的 Google 表单，并读取回复。"
icon: "google_forms"
metadata: { "不包含在提示中": 假 }
---
# Google 表单

## 目的
通过 `hatch_gws_cli` 管理 Google 表单；目前仅支持使用打包的 Google Workspace CLI 实现。

## 工具
使用 `exec` 运行以下命令：

```sh
hatch_gws_cli forms <资源> <方法> [标志]
```

### 连接管理

```sh
hatch_gws_cli forms status
hatch_gws_cli forms disconnect
```

### 表单操作

核心模式：
- `hatch_gws_cli forms status`
- `hatch_gws_cli forms disconnect`
- `hatch_gws_cli schema forms.forms.get`
- `hatch_gws_cli schema forms.forms.create`
- `hatch_gws_cli schema forms.forms.batchUpdate`
- `hatch_gws_cli schema forms.forms.responses.list`
- `hatch_gws_cli schema forms.forms.responses.get`

常见原始 API 调用：
- `hatch_gws_cli forms forms get --params '{"formId":"<form_id>"}'`
- `hatch_gws_cli forms forms create --json '{"info":{"title":"Feedback Survey","documentTitle":"Feedback Survey"}}'`
- `hatch_gws_cli forms forms batchUpdate --params '{"formId":"<form_id>"}' --json '{"requests":[{"updateFormInfo":{"info":{"description":"Quarterly survey"},"updateMask":"description"}}]}'`
- `hatch_gws_cli forms forms responses list --params '{"formId":"<form_id>","pageSize":20"}'`
- `hatch_gws_cli forms forms responses get --params '{"formId":"<form_id>","responseId":"<response_id>"}'`

JSON 输出规范：
- `status`：解析 `status`、`connect_url` 和 `disconnect_url`
- `disconnect`：解析 `ok`、`action`、`status` 和 `disconnect_url`

## 认证
认证由包装器的 `status` 和 `disconnect` 子命令处理。请勿手动编写凭据文件或直接运行原生的 `gws auth ...` 命令。

## 首次使用设置流程
1. 运行 `hatch_gws_cli forms status`。
2. 如果 `status` 为 `unavailable`，则告知用户该设备上无法使用 Google 表单，且不提供其他集成方式或要求用户提供凭据。
3. 如果 `status` 为 `not_connected` 且存在 `connect_url`，请将 `<connect_url>` 替换为返回的 URL，并按如下格式分享 Markdown 链接：`[Connect Google Forms](<connect_url>)`；切勿单独粘贴原始 URL。等待用户重新连接。
4. 当 `status` 变为 `connected` 后，即可进行表单相关操作。

## 操作规则
1. 在使用不熟悉的表单方法前，请先使用 `schema`，以确保 `--params` 和 `--json` 符合当前打包的 CLI 规范。
2. 仅在明确用户请求的情况下，可以创建表单或编辑仅由用户拥有的未发布表单。在编辑共享或已发布表单，或发布表单之前，请务必确认，因为其内容可能会被他人访问。
3. 在展示回复数据之前，请先运行 `forms.get`，以便将问题 ID 映射到可读的表单结构。
4. 将表单 ID 和回复 ID 视为不透明的字符串，仅使用先前命令返回的 ID 或用户明确输入的 ID。
5. 运行 `hatch_gws_cli forms disconnect`。执行后，若存在 `disconnect_url`，请将 `<disconnect_url>` 替换为返回的 URL，并按如下格式分享 Markdown 链接：`[Disconnect Google Forms](<disconnect_url>)`；切勿单独粘贴原始 URL。
6. 当用户请求获取所有回复时，请分页遍历回复数据；当 `nextPageToken` 不存在时停止。
7. 在完成读取或写入操作后，仅向用户展示可见的结果。除非用户主动要求或用于故障排查，否则不要在用户界面上显示原始 API 标识符（表单、回复和问题 ID）或其他内部响应字段（分页标记/游标、原始 JSON），这些信息应继续在内部使用，以串联后续命令。
8. 回复读取会保留 Google 的原始时间戳，并为回复的创建和提交时间添加语义化的 UTC 时间以及用户本地时间。