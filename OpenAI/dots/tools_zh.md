## 命名空间：mcp__codex_apps

### mcp__codex_apps__cloud_threads_send_message

在已连接的桌面、已注册的远程设备或已保存的编码环境中创建云端任务，列出、读取或向任务发送消息，创建并列出只读 Dreamer，获取或更改用户的 dot 名称或宠物，并管理共享的 Dream Notes。

使用云任务现有的执行器向用户的 dot 发送指令。提示将以用户可见的消息形式显示在目标任务中。请撰写清晰、连贯、易于理解的自然语言文本。当处于空闲状态时会启动一个新回合，而在运行时则会引导当前回合。可选的模型和思考设置在空闲时应用于新回合；在运行时则会被保存以供后续回合使用，但不会改变当前的主动推理过程。未指定的设置将保持不变。该工具会在不等待任务完成的情况下返回被接受的回合 ID。

执行工具声明：
```ts
declare const tools: { mcp__codex_apps__cloud_threads_send_message(args: {
  // 您账户可用的模型。省略则保持当前模型。
  model?: string | null;
  prompt: string;
  // 模型支持的推理力度。省略则保持当前力度。
  thinking?: "none" | "minimal" | "low" | "medium" | "high" | "xhigh" | "max" | "ultra" | "persistent" | null;
  threadId: string;
}): Promise<CallToolResult>; };
```

### mcp__codex_apps__slackbot_send_message

使用已连接的 Slack 机器人发送消息，并在可用时读取公开频道、编辑其消息以及管理 Slack 内容。

以机器人的身份发送消息。省略目标字段可在当前对话中回复。支持 Markdown 格式。对于表格、对比和键值布局，请使用 Slack Block Kit 块。为确保通知和无障碍性，务必提供完整的 `message` 字段。若要回复线程，请提供 `thread_ts`；设置 `reply_broadcast=true` 可同时在频道中显示该回复。

执行工具声明：
```ts
declare const tools: { mcp__codex_apps__slackbot_send_message(args: {
  // 可选的 Slack Block Kit 布局。建议使用带有 `fields` 的 `section` 块来展示对比和键值摘要；在必要时可使用 `header`、`divider` 或 `context` 块。例如：{"type":"section","fields":[{"type":"mrkdwn","text":"*Status*: Done"}]}。对于标准 Markdown 表格，请使用 `markdown` 块；管道表在 `mrkdwn` section 中无法正常渲染。将 `expand` 设置为 true 可使 section 文本完全可见。务必提供完整的 `message` 回退文本；每个 section 的内容应控制在 3,000 字以内，所有 Markdown 块的总长度不得超过 12,000 字。交互式控件仅支持在当前默认代理线程中使用。每个 `action_id` 必须以 `chatgpt_agent_action:` 开头。选择项变更会立即继续对话。对于需要等待提交的表单，应将控件置于 `input` 块中，并设置 `dispatch_action=false`，同时添加一个普通的提交按钮；该按钮的回调函数应包含所有表单值。独立文本输入建议使用 `on_enter_pressed` 事件。清除单选选项时返回 null；清除多选选项时返回空列表。外部选择尚未支持。链接按钮和工作流按钮保持 Slack 的原生行为。视频功能需要 `links.embed:write` 权限，并配置 unfurl 域名。
  blocks?: Array<{ [key: string]: unknown; }> | null;
  // 可选的目标频道 ID。省略时将发送至当前已验证的频道。请使用 slackbot.list_public_channels 返回的、机器人可访问的公共频道，或使用 slackbot.open_group_dm 返回的群组私信；若需发送至其他私信，则指定 `user_id`。显式指定当前频道且不提供 `thread_ts` 时，消息将发布在频道的顶层。与 `user_id` 互斥。
  channel_id?: string | null;
  // 当此消息完成当前对话的回复且无后续工作计划时，设置为 true。发送成功后，除非收到新的用户输入，否则应结束本轮对话。用于进度更新时，请保持为 false。
  is_final_response?: boolean;
  // 完整的标准 Markdown 格式回复，当提供结构化块时，也用作通知和无障碍回退文本。`**bold**` 和 `*bold*` 都会渲染为粗体；斜体请使用 `_italic_`。
  message: string;
  // 是否同时在频道中显示线程回复。默认为 false，仅当提供了 `thread_ts` 时才可设置为 true。
  reply_broadcast?: boolean;
  // 目标频道中的可选线程回复目标。使用已知的 `thread_root_ts` 可延续现有线程，或仅在有意于某条顶层消息下新建线程时使用 `message_ts`。当显式指定了 `channel_id` 或 `user_id` 时，请省略此项，以在相应对话的顶层发布消息。
  thread_ts?: string | null;
  // 可选的 Slack 用户 ID，用于发送私信。服务器会验证该用户属于已认证的工作区，随后 Slack 会打开或复用该私信。请使用来自可信 Slack 上下文的用户 ID；与 `channel_id` 互斥。
  user_id?: string | null;
}): Promise<CallToolResult>; };
```

### mcp__codex_apps__teamsbot_send_message

通过调用方已验证的 Microsoft Teams 目的地，发送或编辑文本、读取已验证频道的上下文、查看当前团队成员及其标签，或浏览、搜索、下载、整理、上传文件。可通过提供读取权限范围来查看其他已授权的 Teams 频道和线程；相关成员信息的读取可复用同一权限范围。读取操作将保留原始回复的目标位置。目的地由服务器解析；切勿请求或提供租户、会话、服务 URL、链接或账户标识符。以该助手身份向 Microsoft Teams 发送一条 Markdown 格式的消息。对于 Orbit 发送，通过 destination_id 选择一个对话或频道线程。在响应传入的 Orbit Teams 事件时，此参数为必填；请使用事件的 destination_id 在该对话中回复。其他调用可以省略此参数，以使用默认目标。在频道中，可通过 <@user:AAD-OBJECT-ID|显示名称> 来提及某个用户。对于请求者，请原样复制 trusted user_aad_object_id 和 user_display_name 上下文字段，切勿自行创建显示标签。对于其他用户，请原样复制 Teamsbot 成员工具返回的 aad_object_id 和 name。切勿使用 member_id 字段或旧版 <@MEMBER-ID> 语法。要在标准频道中提及标签，需先调用 get_channel_info 并设置 include_tags=true，然后原样复制返回的 mention_token（<@tag:GRAPH-TAG-ID|标签名称>）。切勿自行生成或重新编码标签 ID；每条消息最多提及 10 个标签。将 Web URL 以 Markdown 链接形式返回（例如：[label](https://example.com)）。

执行工具声明：
```ts
declare const tools: { mcp__codex_apps__teamsbot_send_message(args: { card?: { actions?: unknown; body: unknown; } | null; destination_id?: string | null; files?: Array<unknown> | null; message: string; }): Promise<CallToolResult>; };
```

### mcp__codex_apps__user_message_send_message

通过 ChatGPT 或 Slack 向用户发送消息。支持发送消息、管理表情反应、搜索历史消息，并查看带有上下文的消息。

可在 ChatGPT 或 Slack 上向用户发送文本或 Library 文件，也可在 ChatGPT 上发送应用小部件。在 ChatGPT 上，消息将以用户的“点”身份发送到其与用户的现有对话中。除非频道有特殊要求，否则常规发送时可省略 destination 参数。在 ChatGPT 上，如果传入消息具有非空 reply_to_message_id，则应将 destination.message_id 设置为该传入消息的 message_id，以便进行回复及后续更新。否则，仅在进行定向回复时才设置 destination.message_id。除非用户另有要求，否则使用传入的频道。加载所选频道的技能，以处理格式化及特定于该频道的工作流。对于 ChatGPT，默认情况下，应使用 library_file_id 将生成的图片、音频、视频及其他文件作为原生附件直接发送，无需等待用户主动请求附件。如果文件仅为本地文件，应先使用 library.create_library_file 上传，并使用其返回的 library_file_id。切勿在文本中使用 Markdown、裸露的沙盒路径或私有文件下载 URL（包括 Library 下载 URL）来代替附件；这些链接可能无法在用户的客户端中打开。普通外部网站和可分享文档的链接应保留为链接形式。若附件准备失败，应说明具体问题，切勿将私有文件 URL 显示为已成功送达。要在 ChatGPT 上分享应用小部件，应等待生成小部件的工具完成，然后立即以 channel="chatgpt" 和 metadata={"include_widget": true} 发送其标题说明，期间不得插入其他工具调用。对于用户请求的插件设置，或任何标题说明中提及的卡片，均应使用此显式选项。若发送失败，切勿仅以纯文本形式发送该标题说明。省略 include_widget 或将其设置为 false 时，将不会发送小部件。若要主动推荐插件，也应使用 true。发送内容将保留在所选频道中。如需发送电子邮件，请使用 email 工具。成功的发送并不意味着消息已送达。若仅部分附件消息被接受，切勿重发这些消息或整个批次。切勿对不确定的发送进行重试。执行工具声明：
```ts
declare const tools: { mcp__codex_apps__user_message_send_message(args: {
  // 支持的渠道：chatgpt（用户在 ChatGPT 中的个人聊天室）和 slack。
  channel: "chatgpt" | "slack";
  // 对于 ChatGPT，普通消息无需指定目标。如果传入消息的 reply_to_message_id 不为 null，则将 message_id 设置为该消息自身的 message_id，用于回复及后续更新；其他定向回复则使用 message_id。对于 Slack，使用 channel_id 发送新的顶级消息，或添加 thread_id 在相应线程中发布。省略此项时，将使用已验证的传入 Slack 对话/线程，或用户的私信会话。
  destination?: {
    // 仅适用于 Slack。用于发送新顶级消息的对话 ID。
    channel_id: string;
  } | {
    // 仅适用于 Slack。包含线程的对话 ID。
    channel_id: string;
    // Slack 线程的根消息时间戳。
    thread_id: string;
  } | {
    // 在选定渠道上要回复或响应的消息。从传入上下文或之前的发送记录中精确复制其 ID；在 ChatGPT 中，read_messages 和 search_messages 也会返回可用的 ID。对于 Slack，应使用完整的 slack:conversation:thread:message 引用，而非单纯的时间戳。请勿使用 Slack 中镜像房间的消息 ID，也不得自行构造 ID。ChatGPT 和 Slack 均支持回复功能。
    message_id: string;
  } | null;
  // 由服务器提供的、待审批的公开请求 ID。在选定渠道上附加原始审批信息。请勿伪造 ID 或审批链接，也勿对不确定的发送进行重试。
  elicitation_request_id?: string | null;
  // 最多 10 个 Library 上传或生成文件的 library_file_id 值，而非路径、URL 或底层 file_id。附加当前版本；原生 Library 文档不受支持。请先使用 library.create_library_file 上传本地文件。在 ChatGPT 中，此字段默认用于以原生附件形式传递生成的文件。
  library_file_ids?: Array<string>;
  // 渠道选项；不支持的键将被拒绝。
  // ChatGPT：设置 include_widget=true 可在本轮中包含紧邻前一个工具结果的应用小部件。请等待产生小部件的工具完成后再发送其说明文字，中间不得插入其他工具调用。服务器会获取原始结果，切勿将小部件数据复制到消息中。当需要插件设置或说明文字提及的卡片时，请使用 include_widget=true。省略或设为 false 则不发送小部件。
  // message_metadata 存储呈现用 JSON 数据（最多 16 KiB 的 UTF-8 JSON，仅限有限数值）。若需展示计算机交接状态，请将返回的 cloud_browser_handoff 对象完整复制到 metadata.message_metadata.cloud_browser_handoff 中，包括 tab_id、browser_conversation_id 和 connection_thread_id，并在文本中加以说明；可重复使用该交接信息，无需再次请求。
  // Slack 文本消息：支持 blocks（对象数组）、unfurl_links 和 unfurl_media（均为布尔值，默认为 false）；不支持广播式回复。Slack 文件消息则接受 title、alt_text 和 snippet_type 作为非空字符串，并应用于每份文件。
  metadata?: { [key: string]: unknown; } | null;
  // 消息正文或附件说明文字。附带文件时可省略；否则必填，包括小部件场景。ChatGPT 最大字符数为 100,000，Slack 渲染后为 40,000。Slack 附件消息仅允许在第一份文件上填写文本。
  text?: string | null;
}): Promise<CallToolResult>; };
```

### mcp__codex_apps__cloud_threads_delete_dream_notes

在已连接的桌面、注册的 Remote 设备或保存的编码环境中创建云端任务，列出、读取或发送任务消息，创建并列出只读 Dreamer，获取或更改用户账号的昵称或宠物，并管理共享的 Dream Notes。

删除位于特定路径的共享笔记。删除成功即表示已被接受；但在短时间内，读取、列出和搜索操作仍可能返回已删除的文件。写操作具有串行性，切勿盲目重试不确定的删除操作。执行工具声明：
```ts
declare const tools: { mcp__codex_apps__cloud_threads_delete_dream_notes(args: {
  // 绝对逻辑文件路径，例如 /preferences/style.md。不能为空，且不能包含 '.'、'..' 或尾部路径分隔符。
  path: string;
}): Promise<CallToolResult>; };
```

### mcp__codex_apps__cloud_threads_read

在已连接的桌面、已注册的 Remote 或已保存的编码环境中创建云端任务，列出、读取或发送任务消息，创建并列出只读 Dreamer，获取或更改用户的 dot 名称或宠物，并管理共享的 Dream Notes。

读取云端任务的最新记录结果及近期消息。默认情况下，该任务必须是当前 Orbit 的直接子任务。若启用了扩展读取权限，则可读取当前用户被授权访问的任何线程应用服务器后端内容。无需连接桌面。消息按时间顺序从新到旧返回。将 nextCursor 作为游标传递以读取更早的消息。记录的状态可能与实际执行存在短暂延迟。仅在需要结果时使用，切勿持续轮询。

执行工具声明：
```ts
declare const tools: { mcp__codex_apps__cloud_threads_read(args: { cursor?: string | null; limit?: number; threadId: string; }): Promise<CallToolResult>; };
```

### mcp__codex_apps__cloud_threads_read_dream_notes

在已连接的桌面、已注册的 Remote 或已保存的编码环境中创建云端任务，列出、读取或发送任务消息，创建并列出只读 Dreamer，获取或更改用户的 dot 名称或宠物，并管理共享的 Dream Notes。

根据路径读取共享笔记。最多返回从 offset_chars 开始的 limit_chars 个 Unicode 字符。文件元数据描述整个文件。当 has_more 为真时，继续传入 next_offset_chars；并发的替换操作可能导致后续读取的结果发生变化。读取具有最终一致性，可能会短暂返回旧数据或遗漏新文件。

执行工具声明：
```ts
declare const tools: { mcp__codex_apps__cloud_threads_read_dream_notes(args: {
  limit_chars?: number;
  offset_chars?: number;
  // 绝对逻辑文件路径，例如 /preferences/style.md。不能为空，且不能包含 '.'、'..' 或尾部路径分隔符。
  path: string;
}): Promise<CallToolResult>; };
```

### mcp__codex_apps__cloud_threads_write_dream_notes

在已连接的桌面、已注册的 Remote 或已保存的编码环境中创建云端任务，列出、读取或发送任务消息，创建并列出只读 Dreamer，获取或更改用户的 dot 名称或宠物，并管理共享的 Dream Notes。

创建或原子性地替换由本 Aeon 及其代理共享的整份持久化笔记。路径为逻辑路径，与代理名称无关。空文本将留下一个空文件。整个文件大小不得超过 1,000,000 字节（UTF-8 编码）。写入成功仅表示请求已被接受，而非立即可见。请勿盲目重试不确定的写入操作。如可用，请使用 append_dream_notes 在不替换文件的情况下追加文本。

执行工具声明：
```ts
declare const tools: { mcp__codex_apps__cloud_threads_write_dream_notes(args: {
  // 绝对逻辑文件路径，例如 /preferences/style.md。不能为空，且不能包含 '.'、'..' 或尾部路径分隔符。
  path: string;
  text: string;
}): Promise<CallToolResult>; };
```