# 舰队管理器命令（`scripts/fleet_manager.py`）

命令的契约与主机管理器共享：每个命令对应一个 JSON 对象，一张退出码表，每次写操作对应一行收据。`fleet-manager` 为每个目标添加了一台机器，并增加了针对机器的命令。`<fleet> <verb> --help` 会打印某个命令的可用选项。

```text
<fleet> = python3 <skill-dir>/scripts/fleet_manager.py
<fleet> [--mode herdr|tmux|auto] [--asked-by <who>] <verb> …
```

## 封装结构

每个命令在标准输出上精确地输出一个对象，无论成功或失败：

| 键 | 含义 |
| --- | --- |
| `outcome` | 发生了什么，单个词：命令自身的成功词（如 `healthy`、`detected`、`context`、`listed`、`machines`、`status`、`read`、`dialog`、`attach_command`、`resources`、`waited`、`ready`、`sent`、`notified`、`not_shown`、`keys_sent`、`answered`、`opened`、`interrupted`、`closed`、`adopted`、`forgotten`、`connected`、`fetched`）；或来自退出码表的失败词 |
| `provider` | 提供服务的后端：`herdr`、`tmux`，或未选择时为 `null` |
| `ref` | 命令作用的目标（句柄、地址、机器标签）；全舰队读取时为 `null` |
| `capabilities` | tmux：`liveness`、`scrollback`、`guarded_input`、`attach_by_name`；Herdr：前四项加上 `agent_status`、`dialogs`、`prompt_readiness`、`wait`、`wait_for_output`、`workspace`、`tab`、`worktree`、`notify` |
| `progress` | 每已完成一步输出一行（同时回显到标准错误） |
| `next` | 推动流程前进的下一步命令；无后续步骤时为空（成功状态不使用 `ok`；所有失败状态均携带 `next`，若无更具体的说明则为 `doctor`） |
| `receipt` | 仅适用于写操作：`what`、`session`（身份元组）、`who`、`when`（ISO-8601 UTC），以及与 `line` 相同的内容（含 `HH:MM UTC` 时间） |
| `error` | 仅在失败时出现：第一个出错的原因 |
| `text` | 用于渲染某些内容的读取结果（看板、读取内容、对话、上下文摘要） |

退出码：

- `0`：成功。
- `2`：`usage`——消息中会指明相关选项或地址。
- `3`：`refused` / `no_such_session` / `identity_mismatch` / `session_live` /
  `name_taken` / `composer_not_empty`——被守卫拒绝，未发生任何变更
  （`no_such_session`：该句柄指向的并非活跃会话，而是已结束的会话，而非未知会话）。
- `4`：`unsupported_by_provider` / `no_provider`——`next` 中会给出替代方案。
- `5`：`needs_user_action`——需要用户执行的操作，如安装命令、登录或二次认证、确认等；`next` 即为该步骤。
- `6`：`provider_unreachable` / `failed` / `tmux_unavailable` /
  `herdr_unavailable` / `herdr_add_failed` / `composer_unreadable`——守卫无法查看，因此未执行任何操作；除非日志中注明 `created: true`，否则无任何变更。
- `7`：`internal`。

有一个命令采用流式输出：`events`（每项变更输出一行，面向单个 Monitor；`events --once` 则输出一个对象）。

## 地址格式

`s3`（由看板生成的句柄）或 `machine[:server]/<ref>`：其中 `machine` 可为 `local` 或机器标签；`server` 为 Herdr 会话名（默认为配置文件中的会话名）；`ref` 可为窗格 ID（如 `w1:p2`）、唯一的活跃代理名称、人类可识别的标签（窗格标签、终端标题、标签页或工作区的标签——优先精确匹配，其次进行不区分大小写的唯一匹配；若有两个匹配，则列出候选对象），或 tmux 会话名。窗格 ID、名称和标签均为服务器本地标识，切勿省略 `machine` 部分。

对于非由本技能生成的句柄形式的地址（如 `send s17 …`，当用户将会话命名为 `s17` 时），不会被视为未知句柄而被拒绝；其解析方式与其他名称相同——先在全舰队范围内查找，再遍历每个会话的名称及用户可见的标签，优先精确匹配，其次进行不区分大小写的唯一匹配；若有两个匹配，则列出候选对象，且不会返回 `no_such_session`。由本技能生成的句柄始终优先于与其同名的会话。一个句柄携带身份元组
`(provider, machine, server, ref, cwd, engine)` — 其中 `server` 是 Herdr
套接字路径，或是该动词实际使用的 tmux 套接字标签；这些信息在句柄打开、接管或列出时被存储，并与其他字段一样进行比较 — 而且每个写操作动词都会首先检查整个元组：恢复的窗格并非之前的进程，窗格 ID 或名称也可能被重新分配给新的窗格；若出现不匹配，则返回 `identity_mismatch` 错误（退出码 3），并提示 `adopt <address>`，即有意将当前会话接管到指定地址下。`list` 命令会将此类句柄显示为 `drift (…)`，并且绝不会将其重新指向陌生对象。

## 读操作动词（可列入白名单）| 动词 | 作用 |
| --- | --- |
| `doctor [--no-start] [--no-install]` | 如果状态为 `healthy`（或 `needs_user_action` / `no_provider` / `provider_unreachable`）：检查以下各项（`python`、`herdr`、`tmux`、`machines_directory`、`provider`），以及每台机器的可达性，并执行下一条命令；若 Herdr 服务已停止，则启动之；若安装 tmux 不需要密码，则自动安装。 |
| `detect [--no-start] [--no-install]` | 为本主机确定的提供商信息：包括决策原因（`herdr_reachable`、`herdr_started`、`herdr_down`、`tmux_available`、`tmux_installed`、`requested`、`no_provider`），以及各提供商的相关数据（`installed`、`reachable`、`server`、`version`、`capabilities`）。 |
| `context [--reset]` | 生成一份摘要，包含以下内容：会话信息（每条记录包含 `identity` 和 `group`：`waiting-on-you`、`ready-for-review`、`working`、`idle`；MSP 主机的会话来自您在线主机的 `muse sessions --host <host>` 命令，每次最多四条，且有每主机的时限限制（`FLEET_MANAGER_MSP_LIST_TIMEOUT_S`，8 秒）——待处理请求为 `waiting-on-you`，正在运行的回合为 `working`，其余为 `idle`——会话中带有 `provider: msp` 标识；无法响应或超时的主机标记为 `unreachable`，并附上传输层的消息或“未在 N 秒内响应（列出待处理会话；可再次询问）”，绝不会显示“0 个会话”；不会读取或展示陌生主机及其会话）、群组信息、机器的可达性状态（`connected`、`stale`、`unreachable`、`disabled`，或针对目录条目中因验证前连接中断的情况标记为 `unverified`——其 `next` 仍为该连接，而非故障）、覆盖范围/未知情况、资源信息（`cpu_count`、`load_1m`、`memory_available_mb`、`disk_free_mb`、`sampled_at`）、自上次调用以来的变化、事件列表（`outage`、`recovered`；更新频率参见下方的“故障更新频率”）、文本内容。 |
| `list [<machine>] [--dialogs] [--hint]` | 显示看板（`text`）、精简的清单，以及按服务器分组的行（每台服务器包含 `agents`、`state`、`next_step`）；优先展示被阻塞的机器；无论是否已连接，均列出所有机器。Herdr 条目中包含标签——供人类查看的名称（快照中的窗格名、标签页名、工作区名；空值及标签页或工作区自身的编号不予显示）——看板和上下文会将窗格名（若存在）以引号标注在会话旁，以便人类为其命名的窗格名称能出现在他们阅读的看板上；工作区标签（通常由 `open --label` 指定）则保留在条目中并可供引用。 |
| `machines` | Herdr 的已保存列表，加上 `~/.config/muse/machines.toml`，以及您的 MSP 主机（通过传输 CLI 执行 `muse hosts`，启用 `TBH_AGENTS_SESSION_PROTOCOL`；提供商为 `msp`，来源为 `msp`，主机的传输 ID 作为标签和目标）：本机自身的广告、您登录家族的主机（ID 中包含您的登录名作为连字符分隔的令牌，如 `msp-<user>-<host>` 等）、您通过 `connect <host id>` 保存的主机，或 SSH 记录中已有的标签所指代的主机、本技能打开或接管的任何会话所属的主机，以及本轮中您指定的任何主机——绝不包括共享目录中的其他主机，其规模由 `msp.directory`（`advertising`、`shown`、`not_shown`）和一条 `notes` 行表示；每台主机包含提供商、模式（其所支持的模式——若某 MSP 主机的 ID 已被 SSH 注册记录用作标签或 ID，则该记录会额外标注 `msp`——其会话通过 SSH 提供商读取，对其的 `open` 操作优先使用 MSP 协议）、可达性（MSP 主机：当 `muse hosts` 显示在线时为 `connected`，离线时为 `unreachable`——绝非登录状态）、备注、下一步操作；被“影子化”的目录条目也会被 Herdr 保存；开启该标志后，还会显示 `msp` 相关信息（`state`、`note`、`hosts`），并在来源不可用时附加一条说明为何（无 CLI、旧版本缺少 `muse` 动词、传输层未响应）。 |
| `status <addr>` | 返回身份元组、运行状态、代理状态，以及句柄的 `identity_ok` 或 `identity_drift` 标记；若会话已不存在，则返回 `no_such_session`（退出码 3）——表示“已消失”；而 `provider_unreachable`（退出码 6）则表示“未知”；若 Herdr 窗格无代理（如 shell 窗格或接管的原始窗格），则返回 `status: no_agent`，并附带 `liveness_only: true`；MSP 会话的状态由主机管理器的 `status --mode msp --ref` 返回（`status` 旁附有 `group`）。 |
| `read <addr> [--lines N] [--chars N] [--tail] [--source visible\|recent\|recent-unwrapped]` | 返回会话的近期输出（文本、行数，以及原生状态）；`--tail` 表示从屏幕可见部分截取最后 `--lines` 行（默认 60 行），按引擎绘制时的呈现方式（空白行和分隔线会被压缩），包括浏览器边框——阅读方式与直接附加时相同；MSP 会话无屏幕：`--tail` 和普通读取均返回相同的传输尾部，状态则为 `group`。 |
| `dialog <addr> [--lines N]` | 显示被阻塞会话的请求内容（Herdr）。 |
| `resources [<machine>]` | 返回本机及所有可达机器上的负载、CPU 数量、内存和 `$HOME` 下的磁盘空间（通过便携式 Shell 探针逐级获取）。 |
| `wait <addr> [--until s1,s2] [--duration S]` | 在 Herdr 上等待某个会话（绝非轮询）：记录是否已到达（`reached` 为真或假）及最后一次状态；默认状态为空闲、完成或被阻塞；默认持续时间为 600 秒，由辅助程序强制执行（Herdr 自身的 `--timeout` 设置比此值晚 5 秒作为兜底）；Herdr 端的错误将以失败包裹形式返回——对 Herdr 未知的目标返回 `no_such_session`（退出码 3），其他情况返回 6——绝不在 Herdr 的正常流程中直接抛出错误。 |
| `events [--interval S] [--once] [--kinds …] [--duration S] [--replay-baseline]` | 为单个 Monitor 提供的舰队事件流（Herdr 对每台可达服务器执行 `events.subscribe`；不可达机器会在每个 `--interval` 重新探测）。

### `fetch <machine> <path>`

根据内容哈希，从某台机器上获取一份报告或库文件的本地副本。

- 首先向该机器查询文件的 SHA256 值；仅当本地副本缺失或其 SHA256 不匹配时才进行复制（最后一次的哈希值也会保存在状态文件中）。成功时返回 `fetched`，并附带 `copied`、`unchanged`、`sha256`、`bytes`、`home`、`rung` 等信息。
- 复制后的文件将存放在 `<state dir>/home/<machine>/<path>` 目录下，且仅在此处；无需指定目标路径（代理的工作目录通常是某个仓库，文件由用户手动移动）。
- 若遇到符号链接，则拒绝复制；若路径为目录或不存在，则报“usage”错误；超出复制上限时，同样报“usage”并注明上限名称；若为“local”类型，则视为“usage”（直接就地读取）。
- 对于缺少 `sha256sum` 或 `shasum` 的盒子、无法解码的复制内容，或哈希不匹配的字节数据，均不写入任何内容（标记为 `provider_unreachable`）。
- 代码绝不会通过此方式传输；代码应通过 Pull Request 回到本地。

## 可按子命令白名单控制的操作动词

| 动词 | 作用 |
| --- | --- |
| `send <addr> <text> [--automated]` | 发送一条仅供观看面板的人类可见的通知（Herdr 的 `notification show`；tmux 的 `display-message`）；此类消息既不会被键入，代理也不会接收——因此始终显示为“未发送”：当 Herdr 报告已显示时记为 `notified`，若仅为 tmux 状态栏消息（仅送达已连接的客户端）或 Herdr 标记为 `shown: false`，则记为 `not_shown`；无论哪种情况，`message` 字段都会显示“未输入任何内容”，且 `next` 指向 `--type` 形式的命令。 |
| `approve <addr> [--key K] [--force]` / `deny <addr> …` | 回答可识别的 y/N 或编号式对话框（Herdr）；若会话未被阻塞或对话无法读取，则拒绝（`--key` 必须在读取后使用）。`approve` 会先读取对话框：按下 Enter 键确认高亮选项（带有 `❯`/`›` 的行，或仅在编号行上的裸 `>`——Codex 的 `> You are in <dir>` 标语并非选择项），若该选项并非肯定答复，则拒绝（`code: agent_blocked`, `highlighted`），且 `next` 是附加命令。若 Herdr 将会话标记为空闲，但屏幕上仍显示对话框（如 Codex 的目录信任提示，或某些驱动器中的新 Muse 信任提示），则拒绝操作，并携带 `code: agent_blocked`，且 `next` 是唯一能解决该问题的命令：若肯定选项已高亮，则执行 `approve <addr> --force`，否则执行附加命令。 |
| `send <addr> --keys <key…>` | 用于回答任何其他对话框的命名按键（Herdr），受编辑器保护；对话框自身的选择行不被视为文本，因此按键可以直接传递以作出回应——而用户输入的任意一行文本则会被保留，无论其格式如何。 |

## 受权限提示保护的操作动词

| 动词 | 作用 |
| --- | --- |
| `adopt <machine[:server]>/<ref> [--name N]` | 为此技能未打开的会话生成一个句柄，并记录其身份（`--name` 用于重命名 Herdr 代理）；若已有其他会话使用该名称，则报“name_taken”错误（退出码 3），并指出当前持有者，此时不会进行任何采纳或重命名操作。 |
| `attach <addr>` | 用户运行的命令，用于进入该会话的前台；不会执行任何操作。 |
| `stop <addr>` | 中断当前回合（相当于 Ctrl+C）；会话仍然存在；对于无代理的 Herdr 面板，会在其 shell 中发送 `pane send-keys c-c`；MSP 会话没有 Ctrl+C 功能——此时返回 `unsupported_by_provider`，并建议使用 `close`。 |
| `close <addr> [--confirm "<the human's words>"]` | 关闭面板或终止 tmux 会话；若会话处于活动状态（正在工作、被阻塞，或任一面板中运行着非 Shell 程序），则报“session_live”错误（退出码 3），除非提供 `--confirm` 参数。在 MSP 会话的主机管理端，`close --confirm` 会结束其工作（中断当前回合，停止所有任务），且收据会注明主机将该行会话保持为“空闲”状态，直至将其卸载——此时主机自身会返回 `not_found`（`no_such_session`）；再次使用相同参数则会再次关闭会话。 |
| `forget <label> [--confirm "<words>"]` | 删除 `machines.toml` 中的一行配置（标记为 `forgotten`）；若该行对应的会话仍在运行，则报“session_live”错误，除非提供确认。对于由 Herdr 保存的机器，删除操作归 Herdr 管理（`next` 是 `herdr machine remove <id>`——Herdr 0.9.0 会从 `machine list --json` 中获取 ID，而非标签）。

### `send <addr> <text> --type [--wait] [--until …] [--timeout ms] [--automated] [--no-verify] [--verify-seconds S] [--steer]`

在 MSP 会话中，`--type` 是代理下一轮将接收到的消息（`delivery: message`），而 `--steer` 则用于引导当前正在进行的回合（`delivery: steer`），两者均通过 host-manager 的 `send` 命令发送；`--wait`、`--until`、`--timeout` 以及验证相关的标志位则归类于 `not_applied`。若仅使用 `send` 或 `--keys`，则会被标记为 `unsupported_by_provider`：既无窗格，也无编辑器。

该命令会将文本作为提示输入到会话中：无论是指令、引导、问题还是提醒，都以指定的类型（`--type`）呈现；若传递的是中继文本，则会附加 `--automated` 标志。每次调用时都会弹出权限提示：预置的白名单模板并未包含针对 `send` 文本的通配符规则，因为通配符无法区分通知形式与 `--type` 类型，且一旦匹配即视为允许（通知形式本身仍可通过子命令进行白名单管理）。

- 若编辑器中已有内容（`composer_not_empty`，退出码 3），则拒绝执行——编辑器非空时，应等待或告知用户，绝不能声称已送达；同时，被锁定的会话也会被拒绝（需先响应对话框）。在 Herdr 中，当对话框的选项行被视为“已持有文本”且 Herdr 将会话判定为闲置时，实际表现为对话框界面：返回 `refused` 状态，并附带 `code: agent_blocked` 错误码及对话框选项；此时唯一可执行的命令是 `approve <addr> --force`，或在高亮选项非确认项时使用附加命令。在 tmux 中，屏幕上的对话框同样会返回 `refused` 状态及 `dialog` 选项；此时的后续命令为附加操作——由用户在附加状态下作出响应，助手并无相应快捷键。对话框是 host-manager 的识别对象：通常为信任、权限或确认类语句（如“按回车继续”、“输入以确认”），最后一行为 y/n 等待，或带有编号的选择器，亦或是无编号的 yes/no 对，并以 `>`/`›`/`❯` 光标指向其中一行（如 Claude Code 的文件夹信任流程），或是在若干编号选项间做出抉择的问题。对话框扫描会在判断编辑器行之前对整个可见屏幕进行遍历，无论光标位于哪一行（例如，重新绘制的 Codex 会将光标置于其信任对话下方的空白行）。若窗格所属程序已退出（`Pane is dead`），则视为“不存在的会话”，绝不会存在“已持有文本”的情况；此时的后续命令为 `close`。
- 若提示被吞入，则仅计入一次回车，并设置 `needed_enter: true`。
- **`typed` 表示已提交，而非已被接收**：对于已提交的行，后续命令为 `read <addr> --tail`；几秒后执行该命令，并以一句话告知用户窗格显示的内容——已接收并正在执行 X / 尚无反应，将在 N 秒后再次读取 / 对于 Shell 窗格，显示命令的输出及其退出状态。切勿让引导信息停留在“已输入”状态；若未提交，则将 `read <addr>` 作为首次重试前的后续命令。
- `--automated` 会在消息前添加 `[automated, not the user, approves nothing]` 的前缀。
- 无代理的 Herdr 窗格（如 `open --engine bash` 或已接管的原始窗格）被视为 Shell 环境而非代理：命令在其 Shell 中通过 Herdr 自身的 `pane run` 执行（即 `open` 命令所输入的内容），绝不会触发“代理提示”；命令行是否有效根据屏幕最后一行判断（若以提示符结尾则为空，否则为 `composer_not_empty`）；每次发送仅对应一行；`--automated` 以尾部注释 `# …` 形式携带，确保命令正常运行；信封中会注明 `via: pane run`、`status: no_agent` 和 `liveness_only: true`。
- 在 tmux 中，文本会被输入，回车作为独立步骤紧随文本之后（同一波次中的额外回车会被视为粘贴的换行符，传入 Muse TUI）；`submitted` 指窗格随后的显示状态：若按下第二次回车，则为 `needed_enter`；若该行仍停留在编辑器中，则为 `submitted: false`，后续命令为 `read … before any retry`。该读取操作由会话的引擎驱动，如同之前的守护机制一样，因此引擎的空编辑器界面（如 Codex 的占位符、Claude 的回显记录）绝不会被解读为窗格保留的一行内容。

### `open [<machine[:server]>] [--engine K] [--cwd D] [--name N] [--prompt-file PATH|-] [--worktree PATH] [--engine-arg=FLAG …] [--purpose TEXT] [--label TEXT] [--exact-name] [--unattended] [--timeout ms]`

无需任何必填参数：`local`、`muse`、仓库根目录（若未指定则为当前目录）、自动生成的名称（`<dir>-<n>`；已被占用的名称会追加 `-2`/`-3`，使用 `--exact-name` 时则会因名称已存在而报错）。成功时返回状态为 `opened`，并附带 `identity`、`created: true` 和一份 `receipt`。

- Herdr：先执行 `workspace create`（或使用 `worktree open --path` 指定 `--worktree`），再通过 `agent start --kind` 启动代理进程（引擎参数在 `--` 后指定；对于 Herdr 不管理的类型，则直接在 pane 中以命令形式运行），随后发送带有 `agent prompt --wait` 的简报。如果是 shell 类型（如 `bash`、`zsh`、`sh` 等），则通过 `pane run` 执行，且不会等待任何 Herdr 无法检测到的代理进程；此时返回的状态为 `opened`，仅提供一个句柄，`status: no_agent`，`liveness_only: true`，`via: pane run`；简报不会被发送（`send --type` 只会执行一行命令）。`list` 或 `context` 会将该 pane 以仅存活状态的行显示出来（即 tmux 的形态），直到该技能仍持有其句柄为止；任何其他从未成为代理的命令都会被视为 `failed`（退出码 6），且 pane 仍可供读取与关闭。
- tmux：在 `env -u HERDR_* -u TMUX` 的环境下执行 `new-session -d`（Muse 引擎会附加 `--workspace <cwd>`）；若引擎立即退出，则返回 `failed`（退出码 6），并输出其最后一行日志，之后不留任何痕迹；`--worktree` 和 `--label` 属于“服务提供商不支持”的选项；简报也不会被发送（无就绪信号）。
- `--unattended` 对 Muse 引擎会添加 `--yolo` 参数，对其他引擎则不做任何修改（可通过 `--engine-arg` 传递该引擎自身的标志）；默认情况下仍保留引擎正常的权限提示；封装中会携带 `unattended` 和 `posture`（实际添加的标志）。
- MSP 主机（`modes` 包含 `msp`）：由主机管理器通过传输通道执行 `open --host <host> --cwd <dir>`，并将简报作为首次交互；`--cwd` 为必填项（用法中明确指出：必须是目标机器的目录，而非本地目录），`--engine` 若非 `muse`，以及 `--worktree`、`--label` 和 `--engine-arg` 均属于“服务提供商不支持”。成功时会返回 `mode: msp`、`mode_line`、`attach`（传输通道的尾部命令）、句柄、`session_receipt`（启动信息及简报），以及 `via: host-manager open --mode msp`。若在列出可用资源与发起打开之间，主机停止了服务或无法响应，则返回 `provider_unreachable`（退出码 6），并标记为 `created: false`；其余机器则按原有方式响应。所有 `open` 的结果记录中，无论在哪个服务提供商处，均会包含 `mode`（`herdr` | `tmux` | `msp`）。
- 超时后，在尝试第二次 `open` 之前请先执行 `list`：通常会找到同名的会话；再次执行 `open` 则会创建一个新的会话。
- 不支持 `--dry-run`：`progress` 会在每一步执行时输出步骤信息，而 `close` 则会撤销相应操作。

### `connect <ssh-target|label> [--label N] [--mode herdr|tmux] [--session S] [--no-login]`

一条命令，每一步对应一行 `progress` 输出；一旦遇到问题，便会立即停止，并显示 `next` 提示。`via` 字段会标明 `herdr` 或 `ssh-master`；`remote` 字段记录路径所观察到的信息。

- 在本主机上使用 Herdr 时：`herdr machine add <target> --label N [--remote-session S]`（目标在前：Herdr 0.9.0 的解析器不接受选项在前的参数）会以 Herdr 自己的登录凭据将该机器记录下来（Herdr 的机器列表为权威，无目录行），并在添加操作返回前关闭该登录。随后，若已有响应式端口转发则复用之且无需 SSH；否则，若已有可响应的主连接（无论是本技能的，还是用户 SSH 配置中保存的），则携带该转发而无需再次登录；若均无，则本技能打开其唯一的主连接，并启动或转发远程服务器。登录计费：首次连接需支付 Herdr 的登录费用及本技能最多一次的费用；重连时最多一次，通常无需额外费用。
- 添加失败时返回 `herdr_add_failed`（退出码 6），并将 Herdr 命令作为 `next` 指令，绝不会静默回退至 tmux。若添加成功（退出码 0），但后续执行 `herdr machine list` 读取失败，则视为 `provider_unreachable`，并以 `connect <label>` 作为 `next` 指令（绝不再次尝试添加）。
- 若无 Herdr，或使用 `--mode tmux`：保存至 `machines.toml`，打开 SSH 主连接（即唯一交互式登录、第二因子），验证远程提供者，启动已停止的远程 Herdr 服务器，转发其套接字，并完成记录。对于未安装 tmux 的机器，若安装无需密码，则直接安装；否则，将安装命令作为 `next` 指令（退出码 5）。
- 已保存的标签：通过活动主连接重新进行端口转发，先在其上启动已停止的远程 Herdr 服务器（仅显示一行进度，绝不询问）。
- MSP 主机：无需登录（传输 ID 不是 SSH 目标）；当 `muse hosts` 列表显示其在线时状态为 `connected`，离线时为 `provider_unreachable`，`next` 指令为 `open <host> --cwd <dir>`。非自有主机将以提供者 `msp` 的身份保存至 `machines.toml`，此后摘要视图会与您的主机一同显示；执行 `forget <host id>` 将删除该行（自有主机不单独成行：一旦停止通告即被移除）。
- 对于目录中已以其他标签验证仅可通过 tmux 访问的目标，指定 `--label <other>` 属于 `usage` 错误，并提示已保存的标签。`--no-login` 绝不尝试建立登录（退出码 5 或 6，并附带相关命令）。

## 提供者规则与远程访问阶梯

安装并运行 Herdr 时，其服务器应处于响应状态（若未运行则启动，绝不安装）；否则使用 tmux（若安装无需密码则直接安装，否则将确切命令作为 `next` 指令）；`--mode` 可覆盖默认行为；Windows 系统则为 `no_provider`。任何通告 MSP 的机器都被视为提供者 `msp`，无论 PIN 如何：其会话属于主机管理者的模式 C，需通过主机管理者的辅助工具（`FLEET_MANAGER_HOST_MANAGER`，若不存在则调用与此技能相邻的 `host-manager` 技能）来访问——每个动词仅调用一次该工具，两技能共同记录该会话。绝无 SSH 直接连接到 MSP 主机 ID 的情况，此处也不生成任何会话或命令 ID：均由主机管理者的提供者负责。`resources` 不读取 MSP 主机（无 Shell；标记为 `reachability: unsupported`），`events` 也不监控此类主机（由 `context` 处理）。

机器的访问遵循如下阶梯：转发的 Herdr 套接字（该服务器上的 `fleet-manager@<home host>` 窗格执行命令，输出经该窗格返回）→ 现有的 SSH ControlMaster（`BatchMode=yes`，`ControlMaster=no`，绝不新建登录）→ 若无法到达，则以 `connect` 作为下一条指令。输出经窗格或主连接返回，并在任一级别达到 `FLEET_MANAGER_COPY_CAP_BYTES`（4 MiB）上限时停止，同时标注该上限值；记录由本主机维护。
故障周期：停止响应的机器会被跳过两分钟（成功执行 `connect <label>` 可提前结束此窗口），并在 `context` 中将其最后状态保留为 `stale`；十分钟后记为一次 `outage`，恢复后记为一次 `recovered`。

文件拷贝采用惰性策略。任何动词都不会每轮复制单个文件：`context` 和 `list` 仅拉取状态信息（每台机器一份快照）。只有 `fetch` 才按内容哈希进行拷贝，重复 `fetch` 未更改的文件仅产生一次往返，不传输任何数据。符号链接从不跟随；代码通过 PR 返回。

## 环境`TBH_AGENTS_SESSION_PROTOCOL`（host-manager 的标志：开启时列出 MSP 主机并在其上打开连接；关闭时按字节逐个处理），传输 CLI 的覆盖配置，host-manager 会遵从该配置（详见其 `references/mode-msp.md` 文件）；`FLEET_MANAGER_HOST_MANAGER`（host-manager 的技能目录；默认值为同级的 `host-manager`）；`FLEET_MANAGER_MSP_HELPER_TIMEOUT_S`（300 秒：一次 host-manager 调用的超时时间）；`FLEET_MANAGER_MSP_LIST_TIMEOUT_S`（8 秒：在摘要中列出单个主机会话的超时时间；超过此时间则视为“尚未响应”）；`HERDR_BIN_PATH`、`HERDR_SOCKET_PATH`（Herdr 自身的配置）；`FLEET_MANAGER_SSH`、`_DIR`（用于正向代理和主控节点，路径为 `/tmp/fleet-manager-<uid>`）；`_STATE`（路径为 `~/.local/share/muse/fleet-manager/state.json`）；`_MACHINES`（路径为 `~/.config/muse/machines.toml`）；`_SESSION`、`_REMOTE_SOCKET`、`_TMUX_SOCKET`（一个私有 tmux 服务器）；`_ASKED_BY`、`_CONNECT_TIMEOUT_S`（25 秒：登录步骤的超时时间至少为 5 秒与该值中的较小者，无论剩余预算多少）；`_SERVER_WAIT_S`（12 秒：刚启动的远程服务器通过正向代理响应所需的最长时间）；`_COPY_CAP_BYTES`、`_CALL_TIMEOUT_S`/`_SSH_TIMEOUT_S`/`_API_TIMEOUT_S`；`_NOW`（用于测试的注入时钟）。