---
name: "google_drive"
description: "与用户的 Google 云端硬盘协作：文件、文件夹、上传、下载及共享。"
icon: "google_drive"
metadata: { "不包含在提示中": 假 }
---
# Google 云端硬盘

所有操作都通过 `hatch_gws_cli drive ...` 执行。原始 API 调用以空格分隔（`<资源> <方法>`），并使用 `--params` 接收查询和路径参数的 JSON 对象，创建或编辑时还需提供 `--json` 请求体。运行 `hatch_gws_cli drive <命令> --help` 可查看该命令的选项，运行 `hatch_gws_cli schema drive.files.list`（以及其他类似命令）可查看原始方法的 `--params` 和 `--json` 数据结构。在使用不熟悉的 API 方法前，请先确认其数据结构。

## 连接

在执行任何命令获取数据之前，云端硬盘需要进行一次性的连接。运行 `hatch_gws_cli drive status` 检查连接状态。如果返回未连接且包含 `connect_url`，请原样发布该 URL：“[连接 Google 云端硬盘](<connect_url>)”，然后等待用户点击。如果没有返回 URL，则报告连接不可用并停止；切勿自行生成链接。连接步骤由用户完成，因此切勿自动打开登录页面、驱动浏览器前往登录页、引导用户进入设置或 Google 账号页面，也不得要求用户提供凭据。如果状态显示不可用，请告知用户此设备上无法使用 Google 云端硬盘，并停止后续操作。

断开连接时，运行 `hatch_gws_cli drive disconnect`。仅当该命令返回断开连接的 URL 时，才发布 “[断开 Google 云端硬盘连接](<disconnect_url>)”。如果发布链接后状态仍显示已连接，请再次发布同一链接，让用户在浏览器中完成操作；切勿将用户引导至 Google 账号、安全设置或第三方应用权限页面，也切勿循环检查状态。如果没有返回链接，重新运行一次状态检查，并根据其结果报告当前状态，但不要凭空捏造链接。如果后续命令报出认证错误，再次运行状态检查：若显示未连接，则发布连接链接并等待；若显示已连接，则重试一次该命令；若无可用链接，则报告云端硬盘不可用。认证流程仅通过 `status` 和 `disconnect` 命令进行。切勿手动编写凭据文件或直接调用原始的 `gws auth` 命令。

大多数用户只有一个 Google 账号，这也是默认设置：无需指定账号标志即可直接运行命令。如果用户指定了多个已关联的云端硬盘之一（如“我的工作云端硬盘”），请先运行 `hatch_gws_cli drive accounts` 列出这些账号，确保用户输入与其中一个 `display_name` 完全匹配，然后将该账号的 `account_id` 作为 `--account <account_id>` 传递。如果没有完全匹配的账号，请询问用户具体是哪个账号，切勿猜测或默默认为默认账号。

## 常见操作流程

### 查找与读取

- 查看云端硬盘中的内容：`drive files list --params '{"pageSize":20}'`。要浏览某个文件夹内的内容，可通过其 ID 查询：`drive files list --params "{\"q\":\"'<folder_id>' in parents and trashed=false\",\"pageSize\":50}"`。
- 搜索文件：`drive files list --params "{\"q\":\"name contains 'budget' and trashed=false\",\"pageSize\":20}"`。云端硬盘的查询条件使用单引号；如需搜索文件内容，请使用 `fullText contains 'text'`。
- 获取单个文件的详细信息：`drive files get --params '{"fileId":"<id>","fields":"id,name,mimeType,parents,webViewLink,modifiedTime,owners"}'`。

### 上传与下载

- 上传用户提供的本地文件：`drive +upload ...`（运行 `drive +upload --help` 查看其选项）。请先确认本地路径存在，并通过传递目标文件夹的 ID 将文件放入该文件夹。`+upload` 始终会创建新文件。每次上传的路径必须是绝对路径且位于主目录内，相对路径或 `/tmp` 下的路径均会失败。
- 替换现有文件的内容，同时保留其 ID 和链接：`drive files update --params '{"fileId":"<id>"}' --upload <absolute_path>`。将 Office 文件上传到 Google 原生文档或幻灯片文件时，系统会就地转换格式。此类转换由 Docs 和 Slides 技能负责处理，切勿向 Google 表格文件上传。Google 会替换整个文件内容，因此上传操作会丢弃用户原有的其他标签页。表格文件的格式化应改用 Sheets API 来实现。
- 将存储的二进制文件下载到本地路径：`drive files get --params '{"fileId":"<id>","alt":"media"}' --output <path>`。输出路径由用户指定，因此请询问用户或复用其先前指定的路径。

### 创建与整理

- 新建文件夹：`drive files create --params '{"ignoreDefaultVisibility":true}' --json '{"name":"Q3 Docs","mimeType":"application/vnd.google-apps.folder","parents":["<parent_folder_id>"]}'`。顶级文件夹的父级使用 `"root"`。此举会禁用域名范围内的默认可见性设置；该文件夹仍会继承其父级的共享设置。
- 重命名：`drive files update --params '{"fileId":"<id>"}' --json '{"name":"Q3 Budget"}'`。
- 移动：`drive files update --params '{"fileId":"<id>","addParents":"<dest_folder_id>","removeParents":"<current_folder_id>"}'`。请先通过 `files get` 获取当前父级，以确保移除正确的父级。移动操作需要共享权限审批，因为目标文件夹可能会授予访问权限。
- 复制到“我的云端硬盘”：`drive files copy --params '{"fileId":"<id>","ignoreDefaultVisibility":true}' --json '{"name":"Q3 Budget 的副本","parents":["root"]}'`。如果指定了目标文件夹，则使用指定的目标文件夹。若未指定目标文件夹，则会继承源文件的父级，并需经过共享创建权限审批。
- 共享：`drive permissions create --params '{"fileId":"<id>"}' --json '{"type":"user","role":"reader","emailAddress":"alex@example.com"}'`。仅在用户明确要求编辑权限时才使用 `writer` 角色。

### 删除

- 建议先放入回收站，用户可自行恢复：`drive files update --params '{"fileId":"<id>"}' --json '{"trashed":true}'`。可通过设置 `{"trashed":false}` 恢复。
- 永久删除不可恢复：`drive files delete --params '{"fileId":"<id>"}'`。仅当用户明确表示希望彻底删除时才使用此操作。应明确告知用户该操作不可逆；对于文件夹，还应说明永久删除会一并移除用户拥有的其中所有文件和子文件夹。
  
## 规则

- 遵循工具的权限审批流程。更改私有文件、放入回收站、恢复及永久删除等操作，可在用户提出明确且无歧义的请求后执行。移动文件和更改权限需经共享权限审批。在未验证为私有目标且未明确设置为私有时创建或复制文件，以及所有带有 `+upload` 的操作，均需经过共享创建权限审批。“我的云端硬盘”中的位置本身并不排除域名范围内的默认可见性。在共享文件夹中创建文件或修改已共享的项目前，请务必确认，因为其他用户可能看到这些变更。
- 优先选择可逆路径。除非用户明确要求，否则应先将文件放入回收站而非永久删除，并告知用户回收站中的文件可恢复。
- 仅使用先前命令返回的文件和文件夹 ID，切勿自行编造或修改 ID。
- 在从本地路径上传或下载至本地路径之前，请先验证该路径的有效性，切勿自行创建路径。
- 与用户交流时仅使用通俗语言，切勿展示原始命令或 JSON 数据。除非用户主动询问，否则不得显示文件或文件夹 ID、etag、分页令牌、连接或断开连接的 URL，以及诸如 `not_connected` 或 `unavailable` 等状态词。应通过文件名及 Drive 返回的 `webViewLink` 来确认操作结果；必要时获取相关字段，切勿自行拼接 Drive URL。仅在工作上下文中保留 ID，以便串联后续命令。
- 切勿输出任何令牌、密钥或凭据信息。若工具输出中出现此类内容，应予以遮盖或删除。
- 读取结果时，保留 Google 的原始元数据时间戳，并补充语义化的 UTC 时间及用户本地时间，用于记录创建、修改、查看、共享、放入回收站以及变更的时间。

## 限制

- 阅读、编辑或导出 Google 文档、表格或幻灯片文件的内容，属于其他技能的职责，而非本技能。切勿运行 `drive files export` 命令，也不应通过 Drive 获取任何原生 Google 文件的内容，即便在其他技能的连接失败时作为备用方案亦不可行；此时应说明打开此类文件内容需要使用文档、表格或幻灯片技能，并终止操作。Drive 仅负责管理文件本身：查找文件及其详细信息，以及执行移动、共享或删除等操作。上述二进制下载方法（`alt=media`）仅用于获取已存储的二进制文件，绝不会处理原生 Google 文件。
- 在本系统中，评论、共享云端硬盘和审批请求并非一级流程。如果某项任务需要用到这些功能，请先使用 `hatch_gws_cli schema drive.<resource>.<method>` 检查其接口定义；对于任何写入操作，均按共享权限处理，并在执行前予以确认，因为目前尚未为这些功能提供经过测试的私有写入模板。