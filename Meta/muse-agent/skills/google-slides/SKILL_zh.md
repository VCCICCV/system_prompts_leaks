---
name: "google_slides"
description: "读取、创建和编辑用户的 Google 幻灯片演示文稿。"
icon: "google_slides"
metadata: { "不包含在提示中": 假 }
---
# Google 幻灯片

## 目的
通过 `hatch_gws_cli` 管理 Google 幻灯片；目前仅支持使用打包的 Google Workspace CLI 实现。

## 工具使用
使用 `exec` 运行以下命令：

```sh
hatch_gws_cli slides <资源> <方法> [标志]
```

### 连接管理

```sh
hatch_gws_cli slides status
hatch_gws_cli slides disconnect
```

### 幻灯片操作

核心模式：
- `hatch_gws_cli slides status`
- `hatch_gws_cli slides disconnect`
- `hatch_gws_cli schema slides.presentations.get`
- `hatch_gws_cli schema slides.presentations.create`
- `hatch_gws_cli schema slides.presentations.batchUpdate`

常用原始 API 调用：
- `hatch_gws_cli slides presentations get --params '{"presentationId":"<presentation_id>"}'`
- `hatch_gws_cli slides presentations create --json '{"title":"Quarterly Review"}'`
- `hatch_gws_cli slides presentations batchUpdate --params '{"presentationId":"<presentation_id>"}' --json '{"requests":[{"createSlide":{"slideLayoutReference":{"predefinedLayout":"TITLE_AND_BODY"}}}]}'`

JSON 输出规范：
- `status`：解析 `status`、`connect_url` 和 `disconnect_url`
- `disconnect`：解析 `ok`、`action`、`status` 和 `disconnect_url`

## 组合演示文稿内容

切勿通过先调用 `presentations create` 再执行 `batchUpdate` 插入元素的方式来构建或重建演示文稿。应先将其作为演示文稿制品进行构建。制品工具会将 `.pptx` 文件写入 `~/workspace/your_files/<artifact-slug>/` 目录下，然后再将其上传至 Google。
1. 使用 `hatch_gws_cli slides presentations create --json '{"title":"<title>"}'` 创建一个空的演示文稿。
2. 从已构建的文件填充内容：`hatch_gws_cli drive files update --params '{"fileId":"<presentation_id>"}' --upload "$JARVIS_HOME/workspace/your_files/<artifact-slug>/<name>.pptx"`。Drive 会将上传的文件就地转换为 Google 幻灯片文档。路径必须是绝对路径且位于主目录内，相对路径或 `/tmp` 下的路径均会失败。
3. 在确认完成前，先通过 `presentations.get` 将其读取回来，并核对页数是否与制品一致。在回复中说明该幻灯片为图片形式，因此无法在 Google 幻灯片中编辑文本。
4. 如需修改，可编辑制品后重新上传至同一演示文稿 ID。制品是工作副本，上传操作会替换所有幻灯片，包括用户在 Google 中所做的任何修改。请先读取已发布的版本，若发现变化，应告知用户并等待其确认。

## 认证
认证由封装工具的 `status` 和 `disconnect` 子命令负责处理。请勿手动编写凭据文件或直接调用 `gws auth ...`。

## 首次使用设置流程
1. 执行 `hatch_gws_cli slides status`。
2. 若返回状态为 `unavailable`，则告知用户当前设备不支持 Google 幻灯片功能，且不得提供其他集成方案或要求用户提供凭据。
3. 若返回状态为 `not_connected` 且存在 `connect_url`，则将 `<connect_url>` 替换为返回的 URL，并按原样分享以下 Markdown 链接：`[Connect Google Slides](<connect_url>)`；请勿单独粘贴原始 URL。等待用户重新连接。
4. 当状态变为 `connected` 后，即可继续执行幻灯片相关操作。

## 操作规则
1. 在使用不熟悉的 Slides 方法前，请先调用 `schema`，以确保 `--params` 和 `--json` 参数符合当前 vendored CLI 的接口规范。
2. 对于仅由用户本人拥有的演示文稿，可在明确请求的情况下直接创建或编辑。对于共享的演示文稿，在编辑前请务必确认，因为其内容可能会被他人查看或修改。
3. 将演示文稿 ID、页面对象 ID 和元素 ID 视为不透明的字符串。
4. 运行 `hatch_gws_cli slides disconnect`。运行后，若返回了 `disconnect_url`，请将 `<disconnect_url>` 替换为该 URL，并按原样分享以下 Markdown 链接：`[Disconnect Google Slides](<disconnect_url>)`；切勿单独粘贴原始 URL。
5. 在修改现有幻灯片之前，请先调用 `presentations.get`，以便了解当前的页面和对象结构。
6. 在执行读取或写入操作后，仅向用户展示最终的可见结果。除非用户主动要求或用于故障排查，否则不要在用户界面上显示原始 API 标识符（演示文稿 ID、页面对象 ID 和元素 ID）或其他内部响应字段（修订 ID、原始 JSON）；这些标识符和字段应继续在内部使用，以串联后续命令。