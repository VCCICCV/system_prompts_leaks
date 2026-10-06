---
name: "dropbox"
description: >-
  搜索、阅读、上传、整理并共享用户中的文件和文件夹
  Dropbox 云存储。可用于上传或下载文件，以及收集客户的上传内容。
  通过文件请求，查看或创建共享链接，并创建、复制、移动或
  通过 Dropbox 的官方 API 删除内容。
icon: "dropbox"
metadata: { "不包含在提示中": 假 }
---
# Dropbox

使用已安装的 `dropbox` 命令行工具。首先运行 `dropbox status`。如果显示 `not_connected`，请运行 `dropbox authorize-url`，并将返回的 `connect_url` 仅分享给用户。连接成功后，运行 `dropbox list-tools` 以获取经审核的 Dropbox MCP 目录中的实时输入 Schema。

首次连接时会请求只读 OAuth 访问权限。在执行写入操作之前，或在因缺少访问权限导致写入失败后，请运行 `dropbox status --for-command <tool-name>`。如果返回 `scope_status: not_granted`，请原样复制 `scope_add_url`，并将其单独放在一行，格式为 `[Additional Dropbox access](<scope_add_url>)`，以便客户端渲染原生的授权按钮，然后等待用户完成授权后再重试一次。OAuth 访问权限不能替代 Hatch 的审批。切勿在聊天中自行构造 Scope URL 或索取令牌。

```text
dropbox list-tools
dropbox call-tool --name <tool-name> --arguments-json '<JSON 对象>'
dropbox call-tool --name <tool-name> --arguments-json '<JSON 对象>' --output <路径>
dropbox create-file --path <Dropbox 路径> --input <本地路径>
```

经审核的目录支持列出、搜索、读取和下载文件；查看文件及账户元数据；查看和创建共享链接与文件请求；以及创建、复制、移动、删除或共享内容。请严格按照 `dropbox list-tools` 返回的 Schema 操作。切勿调用该列表中未出现的工具。

使用 `create-file` 可以从本地文件创建或替换 Dropbox 中的文件。它支持文本文件和二进制文件，最大大小为 150 MiB。

文件请求用于收集来自他人的上传内容，但不会从本虚拟机上传本地文件。命令行工具不提供修订历史或版本恢复功能。

当所选工具返回二进制内容时，请使用 `--output` 参数。切勿在聊天中请求或暴露临时下载链接、OAuth 令牌或 Dropbox 应用程序凭据。在删除、移动或覆盖内容之前，请务必确认操作意图无误。
