---
name: "messenger_read"
title: "Messenger"
icon: "messenger"
description: "使用用户的 Messenger 账户：读取通话记录；读取并搜索联系人；读取、搜索和总结对话；发送、回复、撤回或编辑消息；以及向 Marketplace 商品帖的对话发送消息。"
category: "communication"
metadata: { "包含在提示中": 真 }
---
# Messenger

使用 Messenger Companion 与用户的个人 Messenger 账户进行交互。

## 路由与安全

Muse 上的 Messenger 功能分为两个独立的部分：

| 用户希望... | 功能 | 操作 |
|---|---|---|
| 在 Messenger 上给 Muse 发消息 | **Messenger 侧聊天** | 使用 `chat.connection_status`，并按照 `~/docs/chat-connections/messenger.md` 中的连接流程操作 |
| 查看 Messenger 通话记录或处理个人账户消息 | **Messenger Companion** | 按照本技能说明执行 |
| 查看原生电话通话记录或拨打电话 | **配对手机/电话拨打** | 使用原生电话功能或拨打电话功能，而非 Messenger Companion |
| 连接或断开“Messenger”，但未明确具体是哪一项 | **不明确** | 简要说明两者，并予以澄清 |

- 仅当用户明确要求与其个人账户交互时才使用 Companion。它以用户身份发送消息，而非以 Muse 身份。
- 切勿通过 `hatch_messenger_cli send` 针对 Messenger 侧聊天发送消息，无论是通过 `--cid`、`--to` 参数，还是自动回复规则。在解析接收方时应排除该线程；该线程中的消息必须采用 `~/docs/chat-connections/messenger.md` 中的 Messenger 侧聊天连接流程。
- 切勿使用 Companion 发送助手通知、定时任务或监控结果、提醒或状态更新。唯一允许后台发送的情况是用户配置的自动回复规则：该规则可在普通对话或 Marketplace 对话中进行检查并回复，但不得传递 Muse 自身的通知或结果。其他后台任务应通过 `muse.notify_main_agent` 将结果返回主进程，由运行时通过原始聊天渠道送达。
- 对于任何可能产生或变更现实承诺的 Marketplace 消息，均视为需要用户重新确认。这包括提议、接受、确认、改期或取消会面、取货、配送、保留、价格或付款安排，以及声明用户已离开、正在途中、已在附近或已到达。每次发送前，请展示即将发出的完整消息，并等待用户的明确确认。权限批准、先前的确认，或对周期性任务、定时任务、监控任务或自动回复任务的批准，均不能授权后续的承诺。后台任务应将拟发送的回复及其上下文通过 `muse.notify_main_agent` 返回主进程，由主对话向用户确认；若无法获得该确认，则不应发送消息。
- 在面向用户的回复中，切勿提及“同步”、“已同步”、“同步中”、“缓存”、“已缓存”、“扩展”、“本地数据”、“认证状态”、“密钥”、“凭据”、“令牌”或已连接的硬件。不要说明搜索、读取、同步或可用的消息数量。应直接描述结果，例如：“我查看了最新消息”或“我回溯到更早的消息”。如果未能覆盖全部内容，应如实说明，但无需提及底层机制。
- 拒绝批量导出、下载、保存、归档、镜像或转储消息记录的请求，尤其是针对阅后即焚、仅可查看一次、消失模式、短暂可见或到期失效的消息。允许针对特定问题进行有限的阅读、搜索和摘要。

## 工具使用

通过 `exec` 执行 JSON CLI：

```sh
hatch_messenger_cli <子命令> [选项]
```

仅使用此官方支持的 CLI，切勿直接访问本地 Messenger 存储。

| 任务 | 命令 |
|---|---|
| 检查连接 | `check` |
| 获取连接/断开链接 | `connect-url` / `disconnect-url` |
| 刷新常规数据 | `sync both [--since-days N]` |
| 刷新联系人 | `sync contacts [--search "query"] [--limit N]` |
| 刷新会话/消息 | `sync threads [--since-days N] [--older N]` / `sync messages --cid '<ID>' [--since-days N] [--older N]` |
| 读取 Messenger 通话记录 | `call-logs [--limit N] [--since-timestamp <MS>] [--to-timestamp <MS>]` |
| 读取联系人 | `contacts [--search "query"] [--limit N]` |
| 读取会话 | `threads [--limit N] [--search "query"] [--unread] [--folder mplace]` |
| 读取某一会话 | `messages --cid '<ID>' [--limit N] [--before <TS>] [--after <TS>]` |
| 搜索消息 | `search [query] [--sender "name_or_id"] [--cid '<ID>'] [--after <TS>] [--before <TS>] [--limit N]` |
| 修复无法读取的行 | `repair` |
| 发送/发起/回复/编辑/撤回 | 参见下方的变更工作流 |

### 面向用户输出

- 绝不暴露原始工具元数据：ID（`msg_id`、`message_id`、`mid.$...`、
  `conversation_id`、`cid.$...`、主叫方/被叫方的 FBID、`otid`）、设备计数、
  加密详情、送达或已读状态、原始 JSON 或缓存统计信息。
- 唯一允许输出的 ID 相关内容是嵌入在合法 Facebook 个人主页链接中的 FBID，
  以及用户请求的 Messenger 会话链接中的目标部分。绝不在任何情况下单独显示这些 ID，
  也不将完整的命名空间 ID 放入 URL 中。
- 如果消息文本或消息级别的 URL 元数据中包含 URL，可在用户请求时返回该 URL，
  但仅显示 URL 本身，不得暴露其周边的元数据。
- 变更操作成功后，应以简短自然的方式予以确认，不得对服务响应进行详细说明，
  也不得声称消息已送达或已读。

## 连接或断开 Companion

首先检查：

```sh
hatch_messenger_cli check
```

如果 Companion 尚未连接，请运行 `hatch_messenger_cli connect-url`。当响应中包含 `connect_url` 时，
请直接分享 `[Connect Messenger](<connect_url>)`，切勿同时粘贴原始 URL。切勿在聊天中收集敏感信息，
也不引导用户手动完成认证流程。用户在设置中完成关联后，再继续执行所请求的操作。

如需断开 Companion，请运行 `hatch_messenger_cli disconnect-url`。当响应中包含 `disconnect_url` 时，
请直接分享 `[Disconnect Messenger](<disconnect_url>)`，同样不要附上原始 URL。断开 Companion 的连接
只会使其失去对个人账号的访问权限，而不会影响 Messenger 的侧边聊天连接。后续可通过 `connect-url`
并遵循相同的链接规则重新连接。

## 同步会话与消息

读取命令使用本地保存的 Messenger 数据。在声称数据为最新或完整之前，请先执行同步操作。

日常刷新可使用 `hatch_messenger_cli sync both`。若需查询更久远的历史，可添加 `--since-days N` 参数。
如需获取某一会话中更早的历史记录，可使用以下命令：

```sh
hatch_messenger_cli sync messages --cid '<CONVERSATION_ID>' --older 100
```

`sync both` 是增量式同步，会填补消息缺失的部分。若 `_cache.gaps` 与查询范围重叠，
请重复执行该命令，直至相关缺失完全补齐。若请求的时间段早于 `_cache.oldest_cached_ts`，
则需通过 `sync both --since-days N` 来扩展查询范围。对于单一会话内的消息缺失，
建议优先使用针对性的 `sync messages`。若仍无法确保覆盖全部所需内容，
请按照路由与安全模块中的面向用户语言对回答进行适当限定。

# 解析联系人

```sh
hatch_messenger_cli sync contacts
hatch_messenger_cli contacts --search "Alice"
```

默认的刷新操作会获取最多 100 个联系人，并将其合并到当前的联系人快照中。
若用户要求更多结果，可再次执行并设置大于 100 或上次限制值的上限，最高可达 10,000。
`contacts` 读取的是完整的本地保留快照；`--search` 则按名称进行不区分大小写的子字符串匹配。如果本地搜索未返回结果，请使用 `sync contacts --search "<name>"` 重试一次，然后再次执行本地搜索。此外，当匹配结果存在歧义或不完整、用户要求在已知联系人之外查找，或需要当前服务器的结果时，也应使用这种有针对性的刷新。

### 联系人个人资料链接

在联系人列表/搜索结果以及发送预览的“收件人：”字段中，若存在个人资料 URL 或 FBID，则为每个完整显示名称添加链接。优先使用明确指定的 `profile_url` 或 `vanity_url`；若无，则使用 `https://www.facebook.com/profile.php?id=<fbid>`。若两者均未知，则保持名称原样。切勿显示原始 FBID、裸露的个人资料 URL 或缩略名称。

在线程标签、线程搜索结果、无关的助手提示文本以及成功确认信息中，仅使用纯联系人名称。除非用户明确要求发送该带链接的文本，否则绝不在发送的消息中添加个人资料链接。

## 读取 Messenger 通话记录

使用 `hatch_messenger_cli call-logs` 获取最新的 Messenger 音频和视频通话记录。`--limit` 参数可设置为 0 到 50，默认由服务器端设为 20。使用 `--since-timestamp <MS>` 指定包含下限，使用 `--to-timestamp <MS>` 指定排除上限；当同时指定这两个参数时，下限必须小于上限。

报告标注的时间、事件类型、音频/视频类型、持续时间、通话方向及是否未接，但不得暴露呼叫方、被叫方、对话或消息 ID，也不得输出原始 JSON。若查询结果为空，表示在该限定范围内未返回任何 Messenger 通话记录，但这并不意味着用户没有原生电话通话记录，或没有超出 Messenger 保留窗口的通话记录。

## 读取线程与消息

在读取消息之前，先列出或解析线程：

```sh
hatch_messenger_cli threads --limit 20
hatch_messenger_cli threads --search "Bao" --limit 10
hatch_messenger_cli messages --cid '<CONVERSATION_ID>' --limit 20
```

`threads --search` 可匹配姓名和昵称。若仅有一条线程匹配，则直接读取；否则请用户进一步确认。文件夹筛选仅适用于 `threads`，不适用于 `sync`：

```sh
hatch_messenger_cli threads --folder mplace --limit 20
hatch_messenger_cli threads --folder mplace --unread
```

对于较早的保留记录，使用 `messages --before <TIMESTAMP_MS>`。若要查看未读消息，先用 `threads --unread` 查看，再用 `messages --after <read_timestamp>` 读取选定的线程。

若某条记录显示“(解密失败)”或“(无线程密钥)”，则仅将该条视为无法读取。在得出结论前，请先运行 `repair` 并重新读取该线程；详情参见“修复无法读取的消息”。

### 从消息中提取 URL

当用户请求获取消息中的 URL 时，应同时检查消息正文及其消息级别的 URL 元数据。原样返回相关值，保留协议、主机名、端口、路径、查询参数、片段、大小写及百分号编码，不得进行规范化、缩短、解码后再编码，或根据预览重新构建。除非用户明确要求，否则不得打开该 URL。

### 统计总消息数

1. 运行 `threads`，并将返回的列表与 `_cache.cached_thread_count` 对比；如有必要，以该数量作为 `--limit` 再次运行。
2. 按 `conversation_id` 去重。
3. 对每条线程，运行 `messages --cid '<CONVERSATION_ID>' --limit 1`，并读取 `_cache.cached_message_count`。
4. 将这些计数相加，而非累加单条消息数组。

为获得最新或完整的统计结果，请先同步数据。对于不完整的总数，应使用上文所述的面向用户的表述方式，并且绝不能公开计算过程中使用的 ID。

### 创建特定对话的深度链接

仅在用户明确要求时才创建深度链接。解析出确切的线程，并检查 `conversation_id` 和 `is_e2ee`。- 打开一对一聊天：`is_e2ee: false` 且 `cid.c.<VIEWER_FBID>:<CONTACT_FBID>`。将这两个值与 `_cache.self_user_id` 进行比较，并使用另一个值：`https://www.messenger.com/t/<CONTACT_FBID>`。
- 打开群聊：`is_e2ee: false` 且 `cid.g.<GROUP_ID>`：`https://www.messenger.com/t/<GROUP_ID>`。
- 端到端加密聊天：`is_e2ee: true` 且 `cid.g.<THREAD_ID>`：`https://www.messenger.com/e2ee/t/<THREAD_ID>`。

群聊和端到端加密聊天共享 `cid.g.` 前缀，因此切勿仅凭 ID 前缀、线程类型或名称来推断是否为加密聊天。如果缺少 `is_e2ee` 字段，请重新同步并解析。对于开放的一对一聊天，除非 `_cache.self_user_id` 能明确识别另一方参与者，否则不要构建链接。仅使用文档中规定的数字部分；切勿用其他字段替代、猜测，也不要在 URL 中加入 `cid.c.` 或 `cid.g.` 命名空间。返回一个描述性的 Markdown 链接，无需单独显示 ID。

## 搜索消息

```sh
hatch_messenger_cli search "dinner"
hatch_messenger_cli search --sender "Alice"
hatch_messenger_cli search "dinner" --cid '<CONVERSATION_ID>'
```

- 搜索是对已保留数据的精确子字符串匹配，而非语义搜索。多词查询请加引号，必要时可尝试不同的关键词。
- 在缩小查询范围之前，先通过 `threads --search` 解析可能存在的歧义对话。若需全面搜索，可使用 `--before <oldest_timestamp>` 分页。
- 当需要最新或完整结果时，请先执行同步。
- 使用 `search --after <ms>` 并可选 `--before <ms>` 来限定日期范围。`sync --since-days N` 可扩展可用的历史记录，但不会过滤搜索结果。
- `message_sent_at` 和 `last_message_sent_at` 是传输时间，切勿根据这些时间戳推断消息中提到的事件发生时间。

## 输入消息文本

每次发送、发起 Marketplace 交易或编辑消息时，请将消息体作为最后选项，通过 `--text-stdin` 以单引号包围的 Here Document 形式传递：

```sh
--text-stdin << 'MESSENGER_INPUT'
确切的消息内容
MESSENGER_INPUT
```

没有 `--text` 标志。加引号的分隔符可防止 Shell 展开，并保留 `$`、引号及换行符。请选择消息体中不存在的分隔符；切勿使用未加引号的 Here Document。直接将 Here Document 传递给 CLI，不要用 `echo`/`printf` 管道或命令替换。`--text-stdin` 无法撤销此前已发生的 Shell 展开对消息体的改变。

## 发送消息

仅在明确请求时发送，切勿向推测的接收方发送。

### 解析接收方

1. 使用 `threads --search` 查找现有对话。
2. 若请求为一对一、直接或个人聊天，则为硬性约束。在使用 `--cid` 之前，请确认 `thread_type` 不是群聊，且参与者元数据中除用户本人和目标接收方外无其他人。切勿退而求其次，选择匹配的群聊。
3. 若无已验证的一对一线程，请按联系人流程处理。仅当目标人物当前基于姓名的 `contacts --search` 结果中出现该确切 Facebook 用户 ID 时，才使用 `--to <FACEBOOK_USER_ID>`。若最初仅有 Facebook 用户 ID，则该 ID 必须与未过滤的 `contacts` 中的明确记录完全一致。
4. 从线程参与者、消息元数据、记忆、先前输出或用户输入中获取的 ID 均不能用于 `--to`。若无法解析出确切联系人，请询问对方姓名，或说明无法解析该联系人。

### 预览与执行

发送前立即显示：
- **线程**：仅适用于 `GROUP` 或 `SECURE_MESSAGE_OVER_WA_GROUP`；
- **收件人**：解析后的全名，并按联系人资料链接进行关联；
- **消息**：以 Markdown 块引用形式显示完整的待发消息；
- **附件**：如有附件，另起一行显示 `![filename](sandbox://workspace/<workspace-relative-path>)`，置于块引用之外。

请使用姓名而非 ID。等待明确确认；若用户修改任何内容，请再次显示修订后的预览并等待确认。然后调用以下任一命令：
```sh
hatch_messenger_cli send --cid '<CONVERSATION_ID>' [--attach <FILE>] --text-stdin << 'MESSENGER_INPUT'
消息内容
MESSENGER_INPUT
```

```sh
hatch_messenger_cli send --to '<FACEBOOK_USER_ID>' [--attach <FILE>] --text-stdin << 'MESSENGER_INPUT'
消息内容
MESSENGER_INPUT
```

对于附件，请使用可读的本地路径。如果源文件在工作区之外，应先将其复制到 `~/workspace/` 目录下，并在预览和 `--attach` 参数中使用同一份副本。路径前的 `~` 会解析为 Muse 的主目录。

### 自动回复

用户可以创建一个定期规则，用于检查其普通对话或 Marketplace 对话，并自动回复。待用户指定要监控的对话、检查频率以及确切的回复内容后，再进行设置；仅询问那些确实缺失的配置项。

在设置自动回复时，应拒绝任何可能产生或变更实际 Marketplace 承诺的回复文本，并建议使用不具约束力的替代方案。将此例外情况直接写入已保存的规则中：若收到的消息或拟议的回复涉及路由与安全模块所覆盖的预约或其他承诺事项，则不发送回复，并将该对话及具体拟议文本上报至主对话，以获取进一步确认。在设置预览中一并展示此例外，以便用户知晓哪些消息需要后续处理。

在创建规则之前，应以 Markdown 块引用的形式预览该规则所涵盖的对话、检查频率以及确切的回复内容。等待用户明确确认后再予以创建。规则发出的单条回复不会在聊天界面中再次预览。当 Messenger 审批功能开启时，只有当用户此前已允许该 Messenger 账号向该对话发送消息时，回复才会无需额外审批而直接发送；对其他对话的回复则需等待审批。用户可在“设置 > 连接器 > Messenger > 管理接收方权限”中撤销此类许可。

### 发起 Marketplace 对话

仅当用户明确要求联系特定商品的卖家时，才使用 `marketplace initiate` 命令。首先确定该商品及其正整数 ID，然后进行预览：

- **商品信息**：显示其标题或 URL，而不仅仅是 ID；
- **消息内容**：以 Markdown 块引用的形式展示确切文本。

等待用户明确确认后，再通过“消息文本输入”规则调用一次：

```sh
hatch_messenger_cli marketplace initiate \
  --listing-id <LISTING_ID> \
  --text-stdin << 'MESSENGER_INPUT'
消息内容
MESSENGER_INPUT
```

返回值为 `created: true` 表示已创建新线程并发送了消息；返回值为 `created: false` 表示返回了现有的买家/卖家线程，未发送消息。在后一种情况下，应在内部使用返回的 `conversation_id` 同步并读取该线程，随后按照常规的发送预览与确认流程进行后续操作。切勿对该后续操作再次调用 `marketplace initiate`。

## 回复消息

反应功能在普通对话和端到端加密对话中均可使用同一条命令。封装的 Messenger CLI 会根据解析出的对话类型自动选择合适的传输方式。您可以对用户本人的消息或他人的消息添加反应。`--cid` 参数在封装层中为可选；若省略，封装的 CLI 将负责推断对话上下文及任何目标相关的必要条件。此外，封装的 CLI 还会对 `--emoji` 参数进行校验，确保只指定一个表情符号。该命令会在没有反应时添加新的反应，或在已有反应时替换用户的当前反应。

1. 从 `messages` 或 `search` 中确定目标的准确 ID，切勿猜测。如果旧的目标已丢失，请使用 `sync messages --cid '<CONVERSATION_ID>' --older 200` 或 `--since-days N` 深度拉取该线程，然后重新解析目标。
2. 在预览之前立即运行 `sync messages --cid '<CONVERSATION_ID>'` 并再次读取目标。如果目标已更改或存在歧义，请刷新预览。
3. 对于群聊仅显示“线程：”，对于有文本的消息显示“消息：”并附上目标文本，对于反应则显示具体表情符号或“移除你的反应”。使用名称而非原始 ID。
4. 等待明确确认。如果目标或反应发生变化，需重新预览。当提供 `--mid`、表情符号和 `--cid` 参数时，请用单引号括起，以防止 Shell 展开 `$` 序列或修改表情符号。
5. 根据请求的状态只调用一次 `react` 命令。失败时不重试，也不再次触发确认；只需报告返回的原因一次并停止。

添加或更改用户的反应（仅限一个表情符号）：

```sh
hatch_messenger_cli react --cid '<CONVERSATION_ID>' --mid '<MESSAGE_ID>' --emoji '❤️'
```

移除用户当前的反应：

```sh
hatch_messenger_cli react --cid '<CONVERSATION_ID>' --mid '<MESSAGE_ID>' --remove
```

`--cid` 参数可省略；封装层仅在提供时才会传递该参数，否则交由所依赖的 CLI 自行推断。`--emoji` 和 `--remove` 互斥；使用 `--emoji` 添加或更改反应，使用 `--remove` 移除反应。表情符号的有效性由供应商端强制校验。成功后，简明自然地予以确认，但不得暴露消息 ID、传输细节或工具的原始输出。

## 编辑或撤回现有消息

Messenger 不支持从 Marketplace 对话中撤回消息。如果要撤回的消息属于 Marketplace 对话，请在预览或确认前停止操作，不要调用 `hatch_messenger_cli unsend`，并明确告知用户无法从 Marketplace 对话中撤回消息。

对于支持的非 Marketplace 对话，编辑与撤回操作均适用于公开及端到端加密线程。`--cid` 参数对编辑与撤回操作都是可选的：封装层在提供时会传递该参数，否则交由所依赖的 CLI 自行推断并验证对话。请遵循以下通用流程：

1. 从 `messages` 或 `search` 中确定目标的准确 ID，切勿猜测。仅对 `sender_id` 与 `_cache.self_user_id` 匹配的消息执行操作。
2. 如果旧的目标已丢失，请使用 `sync messages --cid '<CONVERSATION_ID>' --older 200` 或 `--since-days N`（最多 30 天）深度拉取该线程，然后重新解析目标。
3. 在预览之前立即运行 `sync messages --cid '<CONVERSATION_ID>'` 并再次读取目标。如果目标已更改或存在歧义，请刷新预览。
4. 仅当 `thread_type` 为 `GROUP` 或 `SECURE_MESSAGE_OVER_WA_GROUP` 时才显示“线程：”，并附上原始“时间戳：”（以用户本地时区为准）。切勿显示发送者信息、原始 ID 或毫秒级时间戳。
5. 执行前等待明确确认。当提供 `--mid` 和 `--cid` 参数时，请用单引号括起，以防止 Shell 展开 `$` 序列。
6. 每个目标每次用户请求最多调用一次变更命令。失败时不重试，也不再次触发确认；只需报告返回的原因一次并停止。

### 撤回

撤回将为所有人删除用户的消息。预览中还会显示“消息：”，其内容为当前的完整文本，并以 Markdown 块引用格式呈现。

```sh
hatch_messenger_cli unsend [--cid '<CONVERSATION_ID>'] --mid '<MESSAGE_ID>'
```

仅在用户明确要求撤回、删除、收回或移除自己发出的消息时才执行撤回操作。在未完成上述深度拉取步骤之前，不应将首次出现的“未找到”视为最终结果。

### 编辑仅当用户明确要求编辑、修复或改写其本人的一条消息时才进行编辑。预览中还会显示“**之前：**”（当前文本）和“**之后：**”（确切的替换内容），每部分均以 Markdown 块引用格式呈现。如果请求的替换内容发生变化，需再次预览。

```sh
hatch_messenger_cli edit [--cid '<CONVERSATION_ID>'] --mid '<MESSAGE_ID>' --text-stdin << 'MESSENGER_INPUT'
替换文本
MESSENGER_INPUT
```

替换内容必须与已确认的消息输入正文完全一致。收件人会立即看到“已编辑”标记；编辑不会触发通知，也不会使消息变为未读加粗状态。Messenger 通常将编辑时限限制在发送后约15分钟内，且每条消息最多可编辑五次；过期、超出次数限制或非文本类型的编辑操作将会失败。

## 修复无法读取的消息

当某行显示“(解密失败)”或“(无线程密钥)”时，请运行：

```sh
hatch_messenger_cli repair
```

修复操作是本地且非破坏性的：它仅刷新一次纪元密钥，重试已保留的密文，对仍无法解密的行保持原样，并且不会获取任何新消息。修复完成后请重新加载该线程。后续再次运行也是安全的。

这些占位符仅描述单个消息行，而非整个线程或 Companion 的端到端加密支持情况。切勿声称加密对话不可访问。如有必要，只需说明那些特定消息无法读取即可。

请勿使用修复功能来解决消息新鲜度不足、历史记录缺失、消息断层、发送失败或写入注册清理等问题。如果报告无法刷新纪元密钥或设备状态已过期，请提示用户断开并重新连接 Companion。切勿对失败的修复操作反复尝试。