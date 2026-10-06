---
name: "box"
icon: "box"
description: "搜索、阅读、上传、下载、移动、重命名、删除、恢复以及共享 Box 中的内容；管理评论和元数据。"
metadata: { "不包含在提示中": 假 }
---
# Box

使用 `/opt/hatch/bin/box-cli` 访问 Box。

## 连接

运行 `/opt/hatch/bin/box-cli status`。如果未连接，运行 `/opt/hatch/bin/box-cli authorize-url`，并将返回的 `connect_url` 以“连接 Box”的形式分享。首次连接时请求的是只读权限。等待用户在浏览器中确认授权后，再次检查状态。

对于可能需要更广泛 Box 访问权限的操作，请先运行 `/opt/hatch/bin/box-cli status --for-command <command-or-method>`。如果是托管的 MCP 操作，应传入其确切的工具名称，而非 `call-tool`。当 `scope_status` 为 `not_granted` 时，原样复制返回的 `scope_add_url`，并单独成行发布为  
`[Additional Box access](<scope_add_url>)`，以便客户端渲染原生的授权按钮。等待用户确认后，再重试原始操作。切勿自行构造授权 URL 或直接使用原始的 Box 范围值。如果所需范围缺失，该操作也会通过与方法绑定的链接以失败状态返回。OAuth 授权不能替代 Muse 的审批。

## 使用

运行 `list-tools` 可查看可用的 MCP 工具、其参数模式以及所需的 Muse 权限。调用 `call-tool` 时，请使用这些模式，并提供由 Box 返回的 ID。`get_file_content` 用于读取文档文本。下载和修改操作默认需要 Muse 审批。

```sh
/opt/hatch/bin/box-cli list-tools
/opt/hatch/bin/box-cli call-tool --name who_am_i --arguments-json '{}'

# 浏览根目录。
/opt/hatch/bin/box-cli call-tool --name list_folder_content_by_folder_id \
  --arguments-json '{"folder_id":"0","limit":20}'

# 搜索 PDF 文件。
/opt/hatch/bin/box-cli call-tool --name search_files_keyword \
  --arguments-json '{"query":"project plan","file_extensions":["pdf"],"limit":10}'

# 查看文件元数据。
/opt/hatch/bin/box-cli call-tool --name get_file_details \
  --arguments-json '{"file_id":"<file_id>","fields":["name","size","permissions"]}'

# 读取文档内容。
/opt/hatch/bin/box-cli call-tool --name get_file_content \
  --arguments-json '{"file_id":"<file_id>"}'

# 创建文件夹。
/opt/hatch/bin/box-cli call-tool --name create_folder \
  --arguments-json '{"name":"Project notes","parent_folder_id":"<folder_id>"}'

# 上传文本文件。
/opt/hatch/bin/box-cli call-tool --name upload_file \
  --arguments-json '{"file_name":"notes.txt","file_content":"Meeting notes","parent_folder_id":"<folder_id>"}'

# 发布评论。
/opt/hatch/bin/box-cli call-tool --name create_file_comment \
  --arguments-json '{"file_id":"<file_id>","message":"Ready for review."}'
```

以下文件相关命令可用于本地文件的上传、完整下载、移动、重命名及删除。`update-folder` 和 `delete-folder` 需指定 `--folder-id`。使用 `upload-version` 可替换现有文件的内容；其必填的 `--name` 同时也会重命名文件。运行 `<command> --help` 可查看各命令的选项。

```sh
/opt/hatch/bin/box-cli download --file-id <file_id> --output workspace/report.pdf
/opt/hatch/bin/box-cli upload --input workspace/report.pdf --name report.pdf --parent-folder-id <folder_id>
/opt/hatch/bin/box-cli update-file --file-id <file_id> --name renamed.pdf --parent-folder-id <folder_id>
/opt/hatch/bin/box-cli delete-file --file-id <file_id>
```

下载操作会覆盖输出文件。在 Box 回收站被禁用的情况下，删除是不可逆的；删除非空文件夹时需使用 `--recursive` 选项。`list-trash` 可列出已保留的项目；将 `next_marker` 作为 `--marker` 传入可进行分页浏览。使用 `restore-file` 或 `restore-folder` 可恢复这些项目。其 `--fallback-parent-folder-id` 仅在原父文件夹已不存在时生效。

下载操作会报告保存路径和字节数；其他结果则显示在 `result` 字段下。`ok: false` 表示操作失败。REST 命令仅尝试一次：若 Box 返回 HTTP 202（文件未就绪），请稍后再试；若返回 HTTP 401，请先执行 `refresh` 再重试。在重试失败的更改之前，请先确认该更改是否已生效。
切勿通过其他命令重试因权限不足而失败的操作。`/opt/hatch/bin/box-cli refresh` 用于刷新连接。
`/opt/hatch/bin/box-cli disconnect` 用于断开 Box 与 Muse 的连接。