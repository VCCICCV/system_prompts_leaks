---
name: "google_sheets"
description: "读取、写入并管理用户的 Google 表格。"
icon: "google_sheets"
metadata: { "不包含在提示中": 假 }
---
# Google 表格

## 用途
通过 `hatch_gws_cli` 管理 Google 表格；目前仅支持使用内嵌的 Google Workspace CLI 实现。

## 工具使用
使用 `exec` 运行以下命令：

```sh
hatch_gws_cli sheets <资源> <方法> [标志]
```

### 连接管理

```sh
hatch_gws_cli sheets status
hatch_gws_cli sheets disconnect
```

### 表格操作

核心模式：
- `hatch_gws_cli sheets status`
- `hatch_gws_cli sheets disconnect`
- `hatch_gws_cli schema sheets.spreadsheets.get`
- `hatch_gws_cli schema sheets.spreadsheets.create`
- `hatch_gws_cli schema sheets.spreadsheets.values.get`
- `hatch_gws_cli schema sheets.spreadsheets.values.append`
- `hatch_gws_cli schema sheets.spreadsheets.batchUpdate`

常用原始 API 调用：
- `hatch_gws_cli sheets spreadsheets get --params '{"spreadsheetId":"<spreadsheet_id>"}'`
- `hatch_gws_cli sheets spreadsheets create --json '{"properties":{"title":"My Spreadsheet"}}'`
- `hatch_gws_cli sheets spreadsheets values get --params '{"spreadsheetId":"<spreadsheet_id>","range":"Sheet1!A1:D10"}'`
- `hatch_gws_cli sheets spreadsheets values append --params '{"spreadsheetId":"<spreadsheet_id>","range":"Sheet1!A:D","valueInputOption":"USER_ENTERED"}' --json '{"values":[["a","b"],["c","d"]]}'`
- `hatch_gws_cli sheets spreadsheets batchUpdate --params '{"spreadsheetId":"<spreadsheet_id>"}' --json '{"requests":[{"addSheet":{"properties":{"title":"Q2"}}}]}'`

还提供了一些内嵌的辅助命令：
- `hatch_gws_cli sheets +read ...`
- `hatch_gws_cli sheets +append ...`

JSON 输出规范：
- `status`：解析 `status`、`connect_url` 和 `disconnect_url`
- `disconnect`：解析 `ok`、`action`、`status` 和 `disconnect_url`

## 格式化表格内容
通过 Sheets API 编写并格式化表格内容，切勿上传文件覆盖现有表格。

1. 使用 `spreadsheets.create`、`values.update` 或 `values.append` 写入数据，并将 `valueInputOption` 设置为 `USER_ENTERED`，以便 Sheets 解析日期、货币和公式。
2. 在一次 `spreadsheets.batchUpdate` 调用中设置样式。使用 `repeatCell` 配合 `userEnteredFormat` 设置表头样式和数字格式；使用 `updateSheetProperties` 锁定表头行；使用 `updateDimensionProperties` 设置列宽。对于新创建的表格，这些格式可以直接在 `spreadsheets.create` 的请求体中指定。
3. 在认为完成之前，先读取结果进行验证。成功的写入仅表示 API 接受了请求，但不能保证表格显示正确。`spreadsheets.get` 默认不返回单元格数据，因此需指定字段掩码：`hatch_gws_cli sheets spreadsheets get --params '{"spreadsheetId":"<spreadsheet_id>","ranges":"<tab>!A1:F10","fields":"sheets(properties(title,gridProperties(frozenRowCount)),data(rowData(values(formattedValue,effectiveFormat(backgroundColor,numberFormat,textFormat)))))"}'`
4. 切勿使用 `drive files update` 将 `.xlsx` 文件发布到表格上。Google 会替换整个文件的内容，导致其他工作表及用户的所有编辑被丢弃。若要修改表格的某一部分，请使用 `values.update` 或 `spreadsheets.batchUpdate`。若要让用户下载一个完整的电子表格，请构建一个包含所有内容的表格文件并附加该文件。

## 认证
认证由包装器的 `status` 和 `disconnect` 子命令处理。请勿手动编写凭据文件或直接运行原生的 `gws auth ...` 命令。

## 首次使用流程
1. 运行 `hatch_gws_cli sheets status`。
2. 如果 `status` 为 `unavailable`，则告知用户当前设备无法使用 Google 表格，且无需提供其他集成方案或要求用户提供凭据。
3. 如果 `status` 为 `not_connected` 且存在 `connect_url`，请将 `<connect_url>` 替换为返回的 URL，并按原样分享以下 Markdown 链接：`[Connect Google Sheets](<connect_url>)`，切勿单独粘贴原始 URL。等待用户重新连接。
4. 当 `status` 变为 `connected` 后，即可继续执行表格相关操作。

## 操作规则
1. 在使用不熟悉的 Sheets 方法之前，请先调用 `schema`，以确保 `--params` 和 `--json` 参数符合当前的 CLI 合约。
2. 除非用户明确要求使用其他 API 路径，否则请使用 A1 格式的单元格引用。
3. 对于仅由用户拥有的电子表格，创建、清空单元格和编辑操作可直接执行。对于共享的电子表格，在编辑前请务必确认，因为其内容可能会被他人查看或修改。
4. 将电子表格 ID 和工作表 ID 视为不透明的字符串，仅使用先前命令返回的 ID 或用户明确输入的 ID。
5. 运行 `hatch_gws_cli sheets disconnect`。运行后，如果存在 `disconnect_url`，请将 `<disconnect_url>` 替换为返回的 URL，并按原样分享以下 Markdown 链接：`[Disconnect Google Sheets](<disconnect_url>)`；请勿单独粘贴原始 URL。
6. 在需要检查当前工作表结构或单元格内容时，优先使用 `spreadsheets.get` 或 `sheets +read`，然后再进行写入操作。如需对工作表进行写入或样式设置，请参考上文“格式化电子表格内容”部分。
7. 在完成读取或写入操作后，仅向用户展示最终可见的结果。除非用户主动请求或您需要用于故障排查，否则不要在用户界面上显示原始的 API 标识符（电子表格 ID 和工作表 ID）或其他内部响应字段（分页标记/游标、原始 JSON）；这些信息应继续在内部使用，以便串联后续命令。