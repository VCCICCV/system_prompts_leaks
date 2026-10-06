---
name: "google_contacts"
description: "搜索、查看、创建、更新和删除用户的 Google 联系人。"
icon: "google_contacts"
metadata: { "不包含在提示中": 假 }
---
# Google 联系人

所有操作都通过 `hatch_gws_cli people <resource> <method>` 执行——资源和方法作为独立参数，参数以 `--params` JSON 对象传递，创建和编辑操作则需在 `--json` 中提供请求体。以下流程给出了每项任务的完整命令，请以此为准；如需查看某命令的选项，请运行 `hatch_gws_cli people <resource> <method> --help`；如需了解某方法的 `--params` 或 `--json` 结构，请运行 `hatch_gws_cli schema people.people.searchContacts`（依此类推）。

联系人的 ID 是一个不透明的 `resourceName`，例如 `people/c1234567890`；`people/me` 表示当前已连接的账号。读取操作必须明确指定所需字段：搜索时需提供 `readMask`，而 `get` 和 `connections list` 则需提供 `personFields` 掩码（均为逗号分隔的字段列表，如 `names,emailAddresses,phoneNumbers`）。切勿自行构造 `resourceName`——应直接使用读取操作返回的 `resourceName` 进行后续的编辑或删除操作。

## 连接
在执行任何命令并获取数据之前，Google 联系人服务需要进行一次性的连接。请运行 `hatch_gws_cli people status`。如果未连接，将该命令返回的完整 `connect_url` 以 `[Connect Google Contacts](<connect_url>)` 的形式发送给用户，并等待用户点击。切勿自行生成 URL、引导用户前往设置页面或要求用户提供凭据。

如需断开连接，请运行 `hatch_gws_cli people disconnect`，并将返回的 `disconnect_url` 以 `[Disconnect Google Contacts](<disconnect_url>)` 的形式发送给用户。

如果某个命令报告认证失败或未连接，请重新运行 `status` 并按照其返回的链接完成连接流程。如果 `status` 也无法访问，则告知用户此设备暂不支持 Google 联系人服务，并停止后续操作。认证流程仅通过 `status` 和 `disconnect` 完成，无需手动编写凭据文件或直接调用 `gws auth`。

## 常见操作流程

### 查找联系人
- 按姓名或邮箱查找：`people people searchContacts --params '{"query":"alice","readMask":"names,emailAddresses,phoneNumbers","pageSize":10"}'`。
- 浏览整个通讯录：`people people connections list --params '{"resourceName":"people/me","personFields":"names,emailAddresses,phoneNumbers","pageSize":50"}'`。可用于“我的通讯录里都有谁”或逐页浏览所有联系人。
- 已知 `resourceName` 后获取单个联系人的完整信息：`people people get --params '{"resourceName":"people/<id>","personFields":"names,emailAddresses,phoneNumbers"}'`。

### 添加联系人
`people people createContact --json '{"names":[{"givenName":"Alice","familyName":"Smith"}],"emailAddresses":[{"value":"alice@example.com"}],"phoneNumbers":[{"value":"+15551234567"}]}'`。只需提供姓名即可；待用户输入后，再添加 `emailAddresses` 和 `phoneNumbers`。

### 编辑联系人
`people people updateContact --params '{"resourceName":"people/<id>","updatePersonFields":"emailAddresses"}' --json '{"etag":"<etag>","emailAddresses":[<包含您修改后的完整列表>]}'`。

### 删除联系人
`people people deleteContact --params '{"resourceName":"people/<id>"}'`。

## 规则
- 在用户提出明确且无歧义的请求时，可直接执行联系人的创建、编辑或删除操作，无需额外确认。在编辑或删除前，请先确定目标联系人。
- 先读取再编辑或删除：首先通过搜索或浏览定位目标联系人，使用读取操作返回的准确 `resourceName`（及 `etag`），切勿对未找到的联系人执行任何操作。编辑时，请一并列出需保留的字段，以免更新时丢失这些信息。
- 与用户沟通时仅使用通俗语言。相关命令及其 JSON 输出仅供内部使用，不应出现在回复中——不得提及任何命令或选项（如 `hatch_gws_cli`、`--params`）、状态词（如 `not_connected`、`unavailable`）、`resourceName`，以及 API 字段（如 `etag`、分页标记）或原始 JSON。联系人的姓名、邮箱和电话号码是用户所查询的内容，应在回复中予以保留。
- 绝对不得输出任何令牌、密钥或凭据信息。若工具输出中出现此类内容，请立即遮盖或删除。
- 当 Google 提供来源更新时间时，读取结果会包含 `contact_source_updated_at` 字段，分别以 UTC 格式和用户本地格式显示。生日仍为日历日期。

## 限制
- 联系人组和标签、“其他联系人”（从邮件中自动保存的地址，而非完整的联系人）以及通讯录或域中的人员在此处不属于一等公民。如果某项任务需要使用这些资源，请先通过 `hatch_gws_cli schema people.<resource>.<method>` 检查其数据结构，并在进行任何更改前予以确认，但目前尚无经过测试的默认配置或最佳实践。