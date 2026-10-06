---
name: "google_docs"
description: "读取、创建和编辑用户的 Google 文档。"
icon: "google_docs"
metadata: { "不包含在提示中": 假 }
---
# Google 文档

## 用途
通过 `hatch_gws_cli` 管理 Google 文档；目前仅支持使用内嵌的 Google Workspace CLI 实现。

## 工具使用
使用 `exec` 运行以下命令：

```sh
hatch_gws_cli docs <资源> <方法> [标志]
```

### 连接管理

```sh
hatch_gws_cli docs status
hatch_gws_cli docs disconnect
```

### 文档操作

核心模式：
- `hatch_gws_cli docs status`
- `hatch_gws_cli docs disconnect`
- `hatch_gws_cli schema docs.documents.get`
- `hatch_gws_cli schema docs.documents.create`
- `hatch_gws_cli schema docs.documents.batchUpdate`

常用原始 API 调用：
- `hatch_gws_cli docs documents get --params '{"documentId":"<document_id>"}'`
- `hatch_gws_cli docs documents create --json '{"title":"Project Brief"}'`
- `hatch_gws_cli docs documents batchUpdate --params '{"documentId":"<document_id>"}' --json '{"requests":[{"insertText":{"location":{"index":1},"text":"Hello, world!"}}]}'`

还提供了一些内嵌的辅助命令：
- `hatch_gws_cli docs +write --document <document_id> --text 'Hello, world!'`

JSON 输出规范：
- `status`：解析 `status`、`connect_url` 和 `disconnect_url`
- `disconnect`：解析 `ok`、`action`、`status` 和 `disconnect_url`

## 构建文档内容

切勿通过 `batchUpdate` 插入或 `docs +write` 来构建文档正文，无论是新建、改写还是重构。插入的纯文本不会保留格式。应先将内容作为文档工件生成。工件工具会在 `~/workspace/your_files/<artifact-slug>/` 下渲染一个带样式的 `.docx` 文件，然后再将其导入 Google：
1. 使用 `hatch_gws_cli docs documents create --json '{"title":"<title>"}'` 创建一个空文档。
2. 从已生成的文件填充内容：`hatch_gws_cli drive files update --params '{"fileId":"<document_id>"}' --upload "$JARVIS_HOME/workspace/your_files/<artifact-slug>/<name>.docx"`。Drive 会原地将上传的文件转换为 Google 文档，并保留原有格式。路径必须是绝对路径且位于主目录内，相对路径或 `/tmp` 路径均会失败。
3. 使用 `documents.get` 将文档读回，确认样式已正确应用后再视为完成。上传成功仅证明 Drive 已完成转换，但不能保证页面显示无误。
4. 如需修改，编辑工件后重新上传至同一文档 ID。工件是工作副本，上传会替换整个文档内容，因此会丢弃自上次上传以来在 Google 中所做的任何修改（包括用户手动修改的内容）。请先读取已发布的版本，若发现变化，应告知用户并等待其确认。

## 认证
认证由封装工具的 `status` 和 `disconnect` 子命令自动处理。请勿手动编写凭据文件或直接调用原始的 `gws auth ...` 命令。

## 首次使用流程
1. 运行 `hatch_gws_cli docs status`。
2. 若返回状态为 `unavailable`，则告知用户该设备上无法使用 Google 文档，且不提供其他集成方案或要求用户提供凭据。
3. 若返回状态为 `not_connected` 且存在 `connect_url`，则将 `<connect_url>` 替换为实际 URL，并按如下 Markdown 格式分享链接：`[Connect Google Docs](<connect_url>)`；切勿单独粘贴原始 URL。等待用户重新连接。
4. 当状态变为 `connected` 后，即可开始执行文档相关操作。

## 操作规则
1. 在使用不熟悉的 Docs 方法之前，请先调用 `schema`，以确保 `--params` 和 `--json` 参数符合当前的依赖 CLI 协议。
2. 对于仅由用户拥有的文档，创建和编辑操作可直接执行。编辑共享文档前请务必确认，因为其内容可能会被他人查看或修改。
3. 将文档 ID 视为不透明的字符串，仅使用先前命令返回的 ID 或用户明确输入的 ID。
4. 在更新文档前，优先使用 `documents.get` 以了解当前的文档结构。
5. 运行 `hatch_gws_cli docs disconnect`。运行后，若存在 `disconnect_url`，请将 `<disconnect_url>` 替换为返回的 URL，并按原样分享以下 Markdown 链接：`[Disconnect Google Docs](<disconnect_url>)`；切勿单独粘贴原始 URL。
6. 使用 `documents.batchUpdate` 修改现有文档的部分内容，而仅使用 `docs +write` 追加纯文本。如需构建文档正文，请参阅上文“文档内容的构建”部分。
7. 在执行读取或写入操作后，仅向用户展示最终可见的结果。除非用户主动要求或用于故障排查，否则不要在用户界面上显示原始 API 标识符（如文档 ID）或其他内部响应字段（如修订 ID、原始 JSON）；这些信息可在内部继续使用，以串联后续命令。