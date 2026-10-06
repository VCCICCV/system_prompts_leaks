---
name: "canva"
description: >-
  创建和编辑 Canva 设计，生成图片，移除背景，恢复
  可编辑图层，并与 Canva 库协同使用。适用于品牌化演示文稿。
  以及演示文稿、营销活动的美术设计、宣传单和横幅，并应用已保存的品牌套件。
  以及模板，并通过 Canva 对社交媒体帖子和动态的设计进行尺寸调整
  官方MCP服务器。
icon: "canva"
metadata: { "不包含在提示中": 假 }
---
# Canva

使用已安装的 `canva` 命令行工具。首先运行 `canva status`。如果显示 `not_connected`，请运行 `canva authorize-url`，并仅分享返回的 `connect_url`。如果 CLI 报告 OAuth 设置已过时，请让用户在设置中取消连接 Canva 后重新连接。切勿在聊天中请求令牌。

运行 `canva list-tools` 获取实时工具列表，并且仅使用其中列出的工具。严格遵循其输入 schema，并包含简明的 `user_intent`。`hatch_permission_overrides` 用于标识依赖于参数的权限；开启编辑事务属于写操作，删除页面则属于删除操作。部分工具需要 Canva Pro、Enterprise 订阅或可用的 AI 额度。

解析前请保存每条原始响应。先检查 `result.isError`，然后读取存在的 `result.structuredContent`，或解析 `result.content` 中的 JSON 文本块。这两种响应形式均有效。本地解析失败并不意味着变更未成功：继续之前应恢复其响应或检查已保存的状态。切勿仅为获取输出而重复执行复制、创建或提交操作。

```text
canva search-designs [--query <关键词>] [--continuation <token>]
canva get-design-content --design-id <id>
canva get-design-pages --design-id <id>
canva list-tools
canva call-tool --name <工具名> --arguments-json '<JSON对象>'
```

## 创建设计和图片

使用 `create-design` 创建一个新的可编辑版面：社交帖子、信息图、海报、传单、演示文稿、文档或表格。将所有必要的事实和所需文案放入 `brief` 中；该工具不会继承聊天或已连接来源的上下文。请先获取用户选定的来源。若需精确尺寸，请在简报中注明尺寸和方向。如提供 `format` 参数，请同时注明方向；模糊的格式可能会被解析为正方形。

仅当用户提供或批准了演示文稿大纲时，才传递 `outline` 参数。

使用 `generate-image` 创建明确要求的独立图片、照片、插图、艺术作品或信息图的图像。每张上传的源图片对应一个 `MEDIA` 引用。此操作生成的是图片而非可编辑的页面布局。再次编辑生成的图片时，请使用其返回的 `media_id`。

`create-design` 无法应用品牌套件或品牌模板。对于明确的品牌化需求，请使用现有的旧版流程：`generate-design`、`get-design-candidates` 和 `create-design-from-candidate`，向用户展示候选方案供其选择。当 `create-design` 可用时，这些工具已被弃用；`create-design` 失败并不意味着可以回退到这些旧工具。尽管 `generate-design` 的枚举中有演示文稿选项，但它实际上并不支持演示文稿。旧版的大纲审核组件和结构化演示文稿生成器未通过此 CLI 公开。当特定品牌的演示文稿工作流不可用时，切勿以非品牌化的生成方式替代。选定的品牌模板仍可通过 `create-design-from-brand-template` 复制或自动填充。

创建和图层分离会返回异步任务。当未显示任何 Canva 小部件（包括使用 CLI 时），请轮询相应的 `get-create-design-async-job`、`get-generate-image-job` 或 `get-separate-image-layers-job`。务必遵守返回的每次等待时间及更新的续传令牌。切勿因任务仍在处理中而启动新的写入操作。遇到终端错误时应停止；尊重配额限制和内容审核失败。完成的结果请使用下方的预览流程展示。对于生成的图片，请附上返回的 Canva 上传链接，并标注 **打开生成的图片**。

## 上传与转换图片

对于大小不超过 256 MiB 的附件、本地文件或生成的文件，请使用
`canva upload-file --file <path> --user-intent '<purpose>'`。在获得批准时，
系统会显示所选文件，并为工作区中的图片提供预览。CLI 会获取一个一次性上传 URL，
并在用户确认后发送原始字节；后续调用中请使用返回的资源 ID。切勿对结果未知的上传重复操作。
对于更大的文件，仍可使用低层级的 `create-upload-url` 流程：以单次 POST 请求发送原始字节，
`Content-Type` 设置为 `application/octet-stream`，无需分块、JSON 或 Base64 编码。绝不可重试已使用的上传 URL。

`upload-asset-from-url` 和 `import-design-from-url` 仅接受已公开的 HTTPS 资源。请勿将本地或私有文件发布后再使用这些工具。上传媒体并不会将其直接放入设计中。`create-design` 没有用于指定媒体的输入参数；若必须使用特定的上传媒体，请通过编辑流程插入或替换具有已验证 Canva ID 的媒体，然后检查最终效果。

使用 `remove-background` 时，需传入已上传的 `MEDIA` 引用，以生成带有透明 Alpha 通道的新图像。此操作不会替换场景或裁剪主体。使用 `separate-image-layers` 时，传入已上传图像的 `asset_id`，可将平面图形转换为新的可编辑设计；原图保持不变。请通过 `read-design` 验证可编辑元素，而不仅依赖外观判断。

## 读取、编辑与组织
使用 `read-design` 可获取元数据、文本、页面元数据、缩略图及演示者备注。旧的 MCP 名称 `get-design`、`get-design-content`、`get-presenter-notes` 和 `get-design-thumbnail` 已被移除。CLI 中的便捷命令 `get-design-content` 现已改调用 `read-design`。`get-design-pages` 仍可用于获取已保存的页面预览。

进行编辑时，请在调用 `read-design` 时设置 `open_transaction: true`，并在 `filter.fields` 中包含 `thumbnails` 以获取编辑前的预览。在后续调用 `edit-design` 时，请使用该事务 ID、元素定位器和页面标记，并设置 `finalize: keep_open`。读取该事务以检查未保存的更改。在执行 `edit-design` 并设置 `finalize: commit` 且不添加任何操作之前，请先展示预览并获得明确确认。如需放弃编辑，请使用 `finalize: cancel`。这些功能已取代 `start-editing-transaction`、`perform-editing-operations`、`commit-editing-transaction` 和 `cancel-editing-transaction`，即便旧版服务器文档中仍提及这些名称。请及时取消过期事务并开启新的事务。

`merge-designs` 可合并或重新排序整个页面。每次调用前请明确确认具体操作；删除页面是不可逆的。`copy-design` 和 `resize-design` 会创建新设计并保留源设计。在调用 `autofill-design` 前，请先检查 `get-design-dataset` 或 `get-brand-template-dataset`，确保字段名称和类型匹配。仅当用户明确要求覆盖现有设计时，才设置 `update_in_place`。

对于品牌模板的更新，请先调用 `create-brand-template-draft`，通过当前事务流程编辑并保存设计，然后仅在用户要求全组织范围发布时再调用 `publish-brand-template`。发布后，该模板将成为可复用的共享模板。

在使用设计之前，请先解析 Canva 短链接。在导出设计前，请确认支持的导出格式（`get-export-formats`）。有关当前 Canva 产品支持的问题，请使用 `help` 获取帮助，而非用于描述本 CLI 的功能。

## 展示结果
交付设计或图片时，请在聊天中同时展示视觉预览与返回的 Canva 链接。将返回的图片内容保存，或将返回的缩略图 URL 使用 `curl` 下载至 `workspace/` 目录下的文件中，用 `read` 检查后，单独一行以 `![Preview](sandbox://workspace/path/to/image.png)` 格式附加。对于少量设计，请逐一预览；对于较长的演示文稿，请展示代表性页面并标注页码。即使签名后的预览 URL 过期，本地附件仍可继续使用。请确认图片显示的内容是否符合预期；仅凭 HTTP 请求成功并不能排除缩略图为空或已过时的可能性。如有需要，可请求生成新的缩略图，或将已保存的设计导出为支持的图像格式。切勿仅为获取预览而提交未保存的编辑内容，也不得将已保存的导出文件当作未保存的草稿呈现。若无法提供可用的预览，请予以说明，并保留 Canva 链接；切勿为修复预览而重复创建或编辑。

当被要求列出设计时，请省略 `--query` 参数，并按需使用续传令牌。此举并不授权进行后台抓取或批量索引。评论和回复对协作者可见，仅在用户明确要求时才发布。