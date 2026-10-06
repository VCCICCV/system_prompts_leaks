---
name: artifact-capabilities
description: |-
  已发布工件页面可被赋予的运行时能力——这是静态 HTML 本身无法提供的功能，例如：读取实时或联网数据、记住用户在页面上的操作（如投票、报名表、待办清单，或就地编辑的文档——它会自动保存新版本）、维护跨浏览者共享的状态、识别当前查看者、向 Claude 提出专属问题、存储用户上传的文件，或将文件提供给浏览者下载。该页面还负责提供当前用户的实时能力列表以及类型化的调用定义。在编写页面之前，只要任何此类运行时行为能够提升工件的实用性，就应加载它。
---
# Artifact 运行时能力

已发布的 Artifact 页面可以通过向 Artifact 工具传递 `capabilities: {name: config}` 来声明**运行时能力**——即 claude.ai 查看器在页面打开时赋予该页面的权限。控制平面是有效名称和配置形状的权威。声明相关的操作：在重新部署时**省略** `capabilities` 会原封不动地沿用之前存储的声明（并保留该 Artifact 的存储合约固定版本）；使用一个**空对象** `{}` 表示明确清空所有能力；而一个**非空对象**则表示完整声明（任何已存储但未再次声明的能力都会被撤销）。移动已重新发布 Artifact 的运行时版本是一种明确的操作——传入 `contract: 'latest'` 来升级，或指定某个具体版本来固定或回滚——绝不是编辑的副作用。

**可用能力**：`artifact`、`assets`、`comments`、`db`、`downloads`、`mcp`、`room`、`sample`、`self`、`user`——这是您可以声明的全部能力名称；以下能力内置于每个页面，无需声明即可调用（切勿在 `capabilities` 中传递）：`permissions`。未列出的能力对该用户不可用。

> 本会话中工具的拼写：`Artifact` 工具的 `action: "read_db"` / `"write_db"` 携带的 `db_op` 实际上属于 `ArtifactData` 工具，其 `action` 即为该 `db_op`（"get"、"list"、"query"、"set"、"update"、"str_replace"、"delete"、"batch"），其余字段保持不变——首次需要时请通过 ToolSearch 加载它。请按照上述替换方式阅读以下步骤。

运行时合约 0.2.52


能力命名空间位于 `claude.use(name)` 之后：`const db = await claude.use("db")` 解析该能力的命名空间，若当前视图无法运行该能力则返回 `null`（未提供服务、未被授予权限或加载失败——设计上三者无法区分）。请对 `null` 做分支处理，并以能力缺失为前提进行设计。`window.claude` 只包含 `use` 方法：绝不承诺存在 `window.claude.db`、`.room` 或 `.artifact` 等成员，因此切勿直接读取这些属性——在没有它们的情况下渲染页面，并在 Promise 解析后（稍后触发，绝不会在脚本首次执行时发生，且与 DOMContentLoaded 无先后顺序；若 10 秒内仍未有查看器响应，则返回 `null`）再启用相关功能。解析后的命名空间是冻结且由平台拥有的：调用其方法并保留引用，切勿对其赋值、使用 `defineProperty` 修改，或替换其中的成员（可为其封装自定义辅助函数）。权限检查发生在每次调用时：首次调用时可能会出现权限提示、速率限制或策略拒绝，但绝不会在 `use()` 调用时发生。再次等待 `use("db")` 是免费的（已缓存）；未知名称将解析为 `null`。


--- 能力：artifact ---

使用 `artifact` 适用于那些需要记录用户行为的页面：投票、报名表、清单、跟踪器、看板等——页面本身就是记录；数据保存在服务器端，或由 Claude 种子化注入或读取回来，这些都属于 `db` 的范畴。声明 `capabilities: {artifact: {}}`；`const artifact = await claude.use("artifact")`，然后调用 `await artifact.publish(html)` 将 `html`（必须是完整的文档，包括 DOCTYPE）作为新版本保存，所有打开的视图（包括当前视图）都会重新加载到该版本。除非页面主动发布，否则查看者输入、勾选或拖动的内容都不会被保留。因此，请将共享状态嵌入您发布的 HTML 中，并以此为基础渲染页面；当交互完成时，更新状态、重新生成文档并发布——切勿序列化实时 DOM；将连续的快速编辑合并为一次发布；仅在查看者操作后才发布，绝不在页面加载时发布。`conflict` 是常见现象（每个视图都会重新加载到最新版本，放弃本次编辑）：无需重试。对于只读视图，发布操作会返回 `not_granted`/`not_writer` 错误——请渲染只读视图。


--- 能力：assets ---`assets` 用于存储该工件的已上传资源：`const assets = await claude.use("assets")`；`await assets.upload(blob)`（支持图片、SVG、视频、PDF、字体、CSS/JS，或 CSV/Markdown/JSON/文本数据；上限为 20 MiB，其中 CSS/JS 16 MiB，SVG 2 MiB，且在上传时会进行内容净化）返回 `{id, url, sizeBytes, contentType}`；`assets.list()` 返回 `{assets, usage}`（包含存储用量统计及孤立资源的清理功能）；`assets.delete(id)` 用于永久删除某个资源：仅在用户明确操作时执行，并更新数据库中记录该 ID 的行。声明 `capabilities: {assets: {}}`；声明此能力的页面应为组织内部页面（绝不可公开）。仅限写入：对于只读用户，调用 `use("assets")` 将返回 `null`；当返回值为 `null` 时，隐藏相关 UI 并妥善处理拒绝响应码。将 `id` 存储于数据库行中作为持久化指针和索引；直接使用返回的 `url` 作为 `<img>`、`<video>` 或 `<a>` 标签的源地址（SVG 仅支持作为 `<img>` 或 CSS 背景）；在任何视图中，存储的 ID 均可通过 `"/_blob/" + id` 访问。配额按工件级别计算（基于 `usage`）。类型定义对允许的类型及错误码具有权威性。


--- 能力：评论 ---

`comments` 将页面自身的评论 UI 与工件的共享评论存储连接起来：`await claude.use("comments")`（返回 `null` 表示不可用）。声明时需指定权限：`capabilities: {comments: {"composer_only": true}}` 仅授予 `openComposer({element}|{range})` 权限——以评论模式打开编辑器，无需用户同意，工件仍可公开分享；建议用于可被发现的入口点。完整形式 `{comments: {}}` 还会增加在用户同意后的写入操作；通过公开链接访问者及邮件邀请者将收到 `null`。在任一形式中添加 `"customAnchors": true` 可启用 `customAnchors()`（需在页面加载时注册），适用于由页面自行定位评论锚点的情况；对于画布、WebGL、视频等自定义锚点，必须使用完整形式。仅限写入：Shell 会渲染所有评论线程，切勿自行构建评论列表。其他操作仅应在用户明确触发时调用，切勿在页面加载时自动执行。使用前请仔细阅读类型定义，其中对方法、数据结构、边界条件及错误码均有明确规定。


--- 能力：数据库 ---

`db` 用于存储页面之外的数据：用户希望保存或预置的数据、Claude 后续读取的数据、超出单次视图展示范围的数据、针对不同用户的私有状态，以及允许多个实时编辑者协作的数据。如果页面本身可以作为记录，则应重新发布（工件）。这是一个 JSON 文档存储：`const db = await claude.use("db")`。在此处使用 `write_db` 和 `read_db` 进行数据预置或检查，切勿硬编码种子数据。声明 `capabilities:{db:{}}`：默认情况下，已登录用户可读取共享文档；只有具备交互或编辑权限的用户才能写入，仅查看或仅评论的用户以及外部链接访客均无写入权限。`rules` 可设置路径级别的最低权限等级：`view` < `interact` < `admin`（可编辑）< `owner`。每个用户的 `data/users/<id>/` 目录对其自身私密，甚至对所有者也受保护（需 `user` 权限）。`db.doc("tasks/t1")` / `db.collection("tasks")`：支持获取、设置、更新、删除操作，以及 where、orderBy 和 limit 等查询功能，并可订阅快照。每个查询仅订阅一次，切勿在渲染过程中重复订阅；每篇文档每次仅允许一次写入，且仅在数据发生变化时执行。采用最后写入者胜出机制，不支持事务；可通过 `acquire({holder})` 获取独占写入锁。切勿存储敏感信息；共享数据不可信。


--- 能力：下载 ---

`downloads` 能力允许已发布的页面向用户提供生成的文件：声明 `capabilities: {downloads: true}`，然后调用 `const downloads = await claude.use("downloads")`（返回 `null` 表示不可用——此时应隐藏相关界面），并执行 `await downloads.save({filename, data})`。用户会看到保存确认提示并可选择拒绝——保存操作绝非静默或强制，因此应在用户明确意图时提供，并妥善处理拒绝情况。类型定义对调用接口及错误码具有权威性。


--- 能力：多客户端协议 ---`mcp` 允许页面调用用户设备上的 claude.ai 连接器：`await claude.use("mcp")`（返回 `null` 表示不可用）；该功能使用用户的凭据，且绝不会暴露任何令牌。需声明 `capabilities: {mcp: {servers: [{server, tools}]}}`；其中 `server` 是连接器的显示名称，或对于用户设备上的本地 MCP 服务器可使用 `host:<name>`（仅限 Claude 应用；否则返回 `server_not_connected`）。请保持清单尽可能精简：仅授予经用户同意的权限，且禁止公开分享。该能力包含两个部分：数据展示方面，通过 `watchTool(server, tool, input, handler, opts?)` 注册监听器（会回放缓存数据，在过时时刷新，并仅按 `refetchInterval` 定期轮询）；操作方面，通过 `callTool` 执行一次调用，并读取 `result.payload`，或直接调用 `(await server(name)).<tool>(input)` 获取负载数据。工具调用失败时会抛出 `tool_error` 错误，监听器会收到相应的错误事件。可根据错误码进行界面分支处理，仅对可重试的错误进行重试，对授权被拒绝的情况则直接丢弃数据，并显示数据的新鲜度信息（`cache.storedAt`）。类型定义中无需指定参数名和编码方式：应在每个工具的实际调用中观察并记录，或在发布时明确说明；切勿臆测。


--- 能力：权限 ---

`permissions` 是内置能力——直接调用即可，无需在 `capabilities` 中显式声明。权限提示默认为惰性模式：已发布的页面会立即渲染，而需要用户授权的能力会在首次使用时才弹窗请求——绝不会因权限问题阻塞页面的首次渲染。`state` 可直接读取，无需任何提示（可通过名称读取单个能力的状态，或不传参数读取全部状态）；`request` 则最多以批量对话框的形式请求权限（可指定具体权限名称，或不传参数请求所有权限）——如果页面确实需要在启动时一次性获取多项授权，可在启动时调用一次 `request`。用户选择“否”并不视为错误：这些调用绝不会被拒绝，且一旦拒绝，该次加载期间将不再重复弹窗——后续再次调用 `request` 也不会再出现对话框，下次加载时则会重新开始。因此，请根据返回的各能力状态进行分支处理，针对不同能力分别降级显示（如隐藏或禁用相关功能），而非直接报错或提供重试按钮；切勿在循环中反复调用 `request`——重复请求会被外壳限流，并可能被视为骚扰。


--- 能力：房间 ---

`room` 能力用于与当前正在浏览该页面的用户进行即时通信：
声明方式为 `capabilities: {room: {}}`；调用 `await claude.use("room")`
（返回 `null` 表示无法连接）。`emit(topic, data)` 用于发送一条消息；`on(topic, fn)` 用于监听消息。`presence(patch)` 用于设置您的状态（光标、选区、颜色等），平台会将此状态传递给新加入的用户，并在您离开时清空；`onPeers(fn)` 会接收所有用户的当前状态——请一并渲染，并标注“您”。所有数据均不会持久化，消息也可能丢失：如果某个用户当前未在线但最终必须看到该信息，则不应使用 `room` 数据——应改用 `db(data)` 或 `artifact(new version)`。请发送绝对状态。您接收到的消息来自同一组织内的其他用户，以及您自己的发布会话（类型为 “agent”）；其他人无法接入，因此页面必须独立运行并保持轻量化。任何人都可以设置自己的状态，因此这并非权威信息；事件主题默认为管理员专用（可编辑），除非特别开放：
{room: {topics: {reaction: "interact"}}}。特殊消息（如彩带）应发布到管理员专用的主题；若某个晚到的用户需要了解某些状态信息（如当前幻灯片），则应将其存储为数据库文档。


--- 能力：样本 ---`sample` 向 Claude 发出请求（声明 `capabilities:{sample:{}}`）：`const sample = await claude.use("sample")`（`null`：隐藏）；`await sample(input, opts?)` → `{text, truncated}`；`sample.json(input, opts?)` → 解析后的 JSON。`input`：可以是字符串，也可以是形如 `[{role:"user"|"assistant", content}]` 的数组，且以用户消息结尾。无记忆功能：仅发送指令、页面数据和输出格式。选项 `opts` 包括：`onText({text, delta})`（`text` 为截至目前的完整回答，需赋值；在回调触发前显示“思考中”，耗时约 5–60 秒）、`signal`（每次调用创建一个新的 AbortController；中断时返回 `cancelled` 拒绝）、`tools: [{name, description, inputSchema?, execute(input)}]`（Claude 可调用的页面函数；可返回小型纯数据或抛出异常；每轮计费，无缓存机制）、`images`（若 `(await sample.limits()).images` 支持）、`modelTier`（快速|默认|复杂）、`cache`（5 分钟内可重复使用；聊天场景下设为 `false`）。错误会返回 `{code, message, text?}`（`text`：部分结果，保留）：`not_granted` 时隐藏错误提示，`rate_limited` 时退避，绝不循环重试。费用由查看者承担；首次调用需征得同意；应在点击或稳定加载时发起调用。


--- 能力：self ---

`self` 是 `artifact` 能力的旧名称（已更名）。为保持兼容性，它仍被保留：已发布的页面以及先前生成的代码中若声明了 `capabilities: {self: {}}` 或调用了 `claude.use("self")`，则无需修改即可继续运行——两种名称均指向同一能力（本协议不承诺提供 `window.claude.self` 成员用于特性检测；应通过 `use()` 方法进行检测）。请勿在新页面中使用此能力：应声明 `capabilities: {artifact: {}}`，并通过 `await claude.use("artifact")` 获取命名空间；具体用法请参阅 artifact 部分。


--- 能力：user ---

`user` 提供当前页面的查看者身份信息，以及您的共享状态所涉及的人员范围：作者所在组织内的成员；其他用户被视为缺席。`const user = await claude.use("user")`；`null` 表示缺席（`user?.isOwner() ?? false`）。`isOwner()`/`canEdit()`/`can(name)` 无需额外配置（`canEdit` 等同于管理员权限；`can("data.write")` 表示可写入共享数据库文档，`null` 表示未被授权：输入内容将被保留；拒绝写入时自行决定）。若需使用 `id()`/`me()`（各组织内唯一的不透明 ID；`me()` 始终非空）以及 `profiles(ids)`，则需声明 `capabilities:{user:{}}`；添加 `scopes:["profile"]` 可获取姓名，并支持 `search(q)`；添加 `["profile","email"]` 还可获取邮箱地址。读取操作绝不会失败。仅存储 ID（`id()` 或 `hit.id`），切勿存储姓名、头像或个人资料：不同查看者看到的姓名可能不同，且一旦写入即冻结。应在每次渲染时解析：`const ps = await user.profiles(idsOnScreen)`，然后使用 `ps[id].name || '某人'`（已缓存：再次调用亦正确；提前声明会导致过时）。若无法解析，则 `name` 为空字符串：请使用 `||` 而非 `??`。聚焦时调用 `search('')`，并通过 textContent 设置姓名。**本会话中的连接器。** 连接器工具会在您的工具列表中显示为 `mcp__<connector>__<toolName>` 格式。请将 `server` 设置为 `<connector>` 部分——即 `mcp__` 与下一个 `__` 之间的内容（例如，对于 `mcp__claude_ai_Slack_beta__search`，`server` 就是 `claude_ai_Slack_beta`）。请原样复制该部分，包括大小写；发布时，系统会自动将其解析为连接器的显示名称。在页面自身的 `callTool`/`watchTool` 调用中，请传入连接器的显示名称（即在 claude.ai 上显示的名称），而不是该段字符串——查看者仅通过名称来识别连接器。发布结果会列出每个已解析段落对应的准确显示名称；如果页面调用与此不符，请修正后再重新发布。只有 claude.ai 的连接器才是有效的——本地配置的 MCP 服务器无效。清单中的 `tools` 数组应填写连接器上游工具的名称（由 `listTools()` 或 `/v1/mcp_servers` 返回），当上游工具名称包含 `.` 或空格时，这些名称可能与规范化的 `<toolName>` 段不一致。每个 `servers[]` 条目都必须包含一个非空的 `tools` 数组，用于列出页面所调用的工具——空的或省略的 `tools` 列表将被拒绝，且绝不表示“所有工具”；若要发布时不使用连接器，请从 `capabilities` 中移除 `mcp`（向 `capabilities` 传递 `{}` 以清空已存储的声明），而不要声明一个空的 `servers` 列表。在隔离或 CI 环境中，如果未加载连接器但设置了 `$CLAUDE_CODE_OAUTH_TOKEN`，可通过以下 Bash 命令获取连接器列表：`curl -H 'anthropic-version: 2023-06-01' -H 'anthropic-beta: mcp-servers-2025-12-04' -H "Authorization: Bearer $CLAUDE_CODE_OAUTH_TOKEN" https://api.anthropic.com/v1/mcp_servers?limit=1000`；在这种情况下，请将每个条目的 `display_name` 作为 `server` 的值（精确的显示名称始终可与工具前缀段一起使用）。

**调用契约**（运行时契约版本 0.2.52）。本契约的平台提供的 `window.claude` 类型定义位于“技能目录”下：`0.2.52/artifact.d.ts`、`0.2.52/assets.d.ts`、`0.2.52/claude.d.ts`、`0.2.52/comments.d.ts`、`0.2.52/db.d.ts`、`0.2.52/downloads.d.ts`、`0.2.52/mcp.d.ts`、`0.2.52/permissions.d.ts`、`0.2.52/room.d.ts`、`0.2.52/sample.d.ts`、`0.2.52/self.d.ts`、`0.2.52/user.d.ts`。在编写任何调用 `mcp` 能力的代码之前，请务必阅读 `0.2.52/claude.d.ts`（说明页面如何访问本契约上的各项能力）和 `0.2.52/mcp.d.ts`——它们是该契约版本的权威参考，优先于任何记忆中的 API 形状。请使用“读取”工具打开这些文件，而非直接使用 `cat` 命令：超过 Bash 工具内联输出限制的文件无法完整返回。这些类型定义仅涵盖调用的封装结构，而不涉及连接器工具的参数名或返回结果的形状。参数名应以本会话中加载的该连接器工具的定义为准。结果的形状应通过一次安全可执行的实际调用来了解——切勿仅为获取结果而进行写操作。已发布的页面在查看时也可通过 `describeTool(server, tool)` 自行读取工具的模式，前提是查看者已为该页面授权了相应连接器；但在发布前，本会话无法读取此响应，因此不能替代在此处对模式的预先读取。如果本会话中没有某个工具的模式且无法安全调用，请在发布时告知用户——在回复中说明，而非作为注释嵌入到已发布的页面中——切勿凭猜测填充结果形状。观察到的响应载荷属于用户的真实数据：应据此了解其形状，但绝不可将观测到的值作为示例或占位数据嵌入到已发布的页面中。
