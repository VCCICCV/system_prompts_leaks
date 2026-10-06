# 提供者说明：按版本验证的行为

本技能所依赖的内容，以及各项内容的验证方式。未标注验证方式的行表示为假设，并已明确注明。当提供者版本发生变化时，请重新验证相应行；辅助工具会输出其所检测到的版本信息（通过 `doctor` 和 `list` 命令）。

## Herdr

| 版本 | 行为 | 验证方式 |
| --- | --- | --- |
| 0.9.0 | Socket API 在每个连接上只处理一个请求并关闭该连接；`events.subscribe` 会保持连接打开，并持续推送 `{"event","data"}` 格式的事件流。 | 辅助工具的模拟服务器复现了这一行为；实时 `events` 测试运行（#36636 证据）|
| 0.9.0 | 关闭工作区或标签页时，只会发出一次 `workspace_closed` 或 `tab_closed` 事件，而不会针对每个窗格单独发出 `pane_closed` 事件；辅助工具会将容器级别的事件进行归并。 | 实际测量验证（#36636）；`test_fleet_lifecycle.py` 确认了这一点 |
| 0.9.0 | `pane.agent_status_changed` 事件是按窗格独立触发的：在订阅之后新出现的窗格需要重新订阅（辅助工具会重新打开事件流）。 | 实际测量验证（#36636）|
| 0.9.0 | 当尝试启动某种代理类型时，如果该类型不在管理范围内，`agent start` 会以退出码 2 和纯文本错误信息 `unsupported interactive agent kind: <kind>` 拒绝请求（不返回 JSON）；此时辅助工具会直接在相应窗格中以命令形式运行该代理，并等待其被检测到。 | 实际测量验证（#36636）|
| 0.9.0 | 对于处于阻塞状态的代理，`agent prompt` 在用户输入之前就会以 `agent_blocked` 事件拒绝；若从非正常状态使用 `--wait` 参数，则必须在 5000 毫秒内观察到代理进入正常或阻塞状态，否则会触发 `agent_prompt_stalled` 错误。 | 通过 `herdr agent prompt --help` 文档及实际运行验证 |
| 0.9.0 | 新启动的代理可以在未提交的情况下接收提示并将其放入编辑器（Muse 1.0.3、Claude Code）：辅助工具会先检查内容，若发现已有文本则拒绝（退出码 3），并自动按下回车键（`needed_enter: true`）。 | 实际测量验证（#31985）|
| 0.9.0 | 恢复的窗格并非原先的进程，且在服务器重启后窗格 ID 可能会被重新分配；ID 并不会可靠地从 `w1` 开始重新计数——在执行 `herdr session stop <name>` 并通过主节点重启后，已命名的会话继续递增编号（`w5` → `w6`）。之所以存在这样的标识元组（`server`、`ref`、`cwd`、`engine`），是因为资源复用而非编号机制。 | 参见 herdr-projects 第二/三阶段的发现（重启后从 `w1` 开始计数）；以及 #38715 第八轮 FM2 对已命名会话的测试（连续编号现象）|
| 0.9.0 | 已保存的机器若其 SSH 主节点处于运行状态，则可能拒绝第二次 SSH 连接（`Session open refused by peer`）：Herdr 桥接层仅保留一个连接槽位。只能通过主节点建立连接（`ssh -O forward`），而不能直接使用新的 `ssh <host>` 命令。 | 在舰队环境中测量验证（#31985，“SLOT_FACT”相关说明）|
| 0.9.0 | `pane wait-output <pane> --regex <re> --timeout <ms>` 会立即搜索最近的快照，随后开始轮询；也可指定 `--source recent-unwrapped` 来获取未包装的最新输出。 | 通过 `herdr pane wait-output --help` 文档验证 |
| 0.9.0 | `notification show <TITLE> --body <TEXT>` 用于显示全服务器范围的通知（无针对特定窗格的目标）。 | 通过 `herdr notification show --help` 文档验证 |
| 0.9.0 | `worktree open --path <PATH> --label <TEXT> --no-focus` 用于将现有的 Git 工作树作为工作区打开。 | 通过 `herdr worktree open --help` 文档验证 |
| 0.9.0 | `workspace create --label <TEXT> --cwd <PATH> --no-focus` 返回根窗格；`--env KEY=VALUE` 参数存在，但辅助工具并未传递环境变量（启动环境的继承仍被推迟）。 | 通过 `herdr workspace create --help` 文档验证；`test_fleet.py` 未使用 `--env` 参数 |
| 0.9.1 | 当用户正在输入部分文本时收到的提示会与现有文本合并并一并提交。辅助工具会优先读取编辑器内容，若发现已有文本则拒绝操作（退出码 3）。 | 根据 herdr-projects 的记录提出，并纳入身份与安全规范；此处未重新验证（本机运行 0.9.0）——在 0.9.1 版本的实际测试确认之前，暂按此假设处理 |
| 0.9.0 | 一个刚启动的 Muse 会话停留在其工作区信任对话框界面时，在一次测试中显示为“blocked”，而在接下来的两次测试中显示为“idle”（`agent wait --until blocked` 返回 idle）；无论哪种状态，“dialog”都会显示相应的提示。当 Herdr 未标记该状态时，使用 `approve --force` 即可响应“dialog”所显示的内容。 | 在本机进行三次黑盒循环测试 |
| 0.9.0 | `machine add <SSH_TARGET> --label <LABEL> [--remote-session <NAME>]` 用于准备远程服务器并保存配置文件；目标主机必须位于参数首位——若选项置于首位，则会报错 `usage: herdr machine add <ssh-target> --label <label> [--remote-session <name>]` 并退出（退出码 2），尽管 `--help` 显示的是类似 Clap 风格的 `[OPTIONS] --label <LABEL> <SSH_TARGET>`。其 SSH 连接通过专用的控制套接字进行（`-F /tmp/herdr-ssh-<pid>-0/config -S /tmp/herdr-ssh-<pid>-0/ctl`），并在返回前通过 `-O exit` 结束，因此登录过程中的任何信息都不会留存供后续 `connect` 使用。其提示和输出均发送至终端，故 `connect` 会将它们重定向至标准错误流，而标准输出保持不变。 | 分别测试两种参数顺序的真实二进制程序，运行于隔离的 HOME 目录下（`test_fleet.py` 中的模拟版本同样拒绝选项前置的写法）；SSH 参数通过 PATH 上的带日志功能的 `ssh` 命令捕获 |
| 任何版本 | 0.8.x 客户端没有 `machine` 命令（报错“unknown command”）：舰队仅限本地使用。 | 由 `list_machines` 处理；自 0.9.0 以来未重新验证 |## tmux

| 版本 | 行为 | 验证方式 |
| --- | --- | --- |
| 3.x | 使用 `list-panes -a -F …` 并结合 `#{window_activity}`（以秒为单位的纪元时间）作为活跃信号：60 秒内有活动则视为“工作中”，否则为“空闲”。不存在代理状态。 | tmux 格式化文档；tmux 测试套件 |
| 3.x | `new-session -d -s <name>` 拒绝使用已存在的会话名称（“重复会话”错误）；辅助工具同样拒绝，并将下一条命令命名为“status”。`open` 在同一调用中设置 `remain-on-exit on`，因此一旦引擎退出，会留下一个死窗格，其最后一行显示退出原因；辅助工具会将其杀死并返回“失败”（退出码 6），而不会返回“已打开”。所有窗格均已退出的会话会显示“状态：已退出”、“活跃度：已死亡”。 | tmux 行为；tmux 测试套件 |
| 3.x | 身份引擎是 `#{pane_start_command}`（在 `open` 添加的 `env -u …` 包装器之后的第一个单词），若无则使用 `#{default-shell}`：在整个窗格生命周期内保持稳定。`#{pane_current_command}` 被报告为“程序”，并决定活跃状态（当会话中任何窗格正在运行非 Shell 程序时，`close` 命令会被拒绝）。`list-panes` 每次读取时都会使用新的随机字段分隔符，因此路径或标题不会导致字段错位。 | tmux 测试套件 |
| 3.x | `open` 在 `env -u HERDR_ENV -u HERDR_PANE_ID -u HERDR_SOCKET_PATH -u TMUX …` 的环境下启动引擎，因此该会话不属于任何用户；Muse 引擎会获得 `--workspace <cwd>` 参数，且仅在 `open --unattended` 时才会添加 `--yolo` 选项（默认情况下，由引擎自身处理权限提示；截至 2026 年 9 月 19 日，由所有者决定）。`--worktree` 和 `--label` 在 tmux 上被标记为“提供商不支持”。 | tmux 测试套件 |
| 3.x | `capture-pane -p -J -S -<n>` 用于读取回滚缓冲区。针对受保护输入的检查会读取光标所在行，对所有引擎均适用：空白行或仅包含提示符（以提示符字符结尾）被视为可用；提示符符号后紧跟着 Muse 的一条空闲提示（整行显示或软换行至下方若干行；即 TUI 在回合结束后于空编辑器中绘制的浅色文本；该列表来自 TUI 的 `prompt_hint.rs` 表，与主机管理器共享）也被视为可用；其他任何文本都被视为未完成的输入，且不允许在其上继续输入；无法执行检查时则视为忙碌。`send-keys -l -- <text>` 和 `display-message … -- <text>` 确保以连字符开头的文本仍被视为文本。 | tmux 测试套件 |
| 3.7b | 仅带 `-t =<session>` 的窗格级命令（如 `send-keys`、`capture-pane`、`display-message`）无法解析目标（“找不到窗格”）；而 `-t =<session>:`（精确指定会话及其当前窗口）则可正常解析。`kill-session -t =<session>` 可正确识别为目标会话。 | 在本机 tmux 3.7b 上构建测试套件时进行验证 |
| 3.x | `display-message -t =<session>: -d 0 <text>` 是 `send` 命令的通知形式。 | tmux 行为 |
| 3.x | 先执行 `send-keys -l <text>` 再按 `Enter` 键会输入一行受保护的内容；`send-keys C-c` 表示“停止”；`kill-session -t =<session>` 表示“关闭”（除非得到确认，否则当有非 Shell 程序运行时会被拒绝）。 | tmux 测试套件 |
| 3.x | `-L <socket>` 用于选择私有服务器：测试和黑盒运行绝不会触及用户的现有会话（`FLEET_MANAGER_TMUX_SOCKET`）。 | tmux 测试套件 |

## MSP（主机管理器的模式 C）| 版本 | 行为 | 验证方式 |
| --- | --- | --- |
| transport CLI 2026-09 | `muse hosts --json` 返回的每一行都包含主机 ID、`msp_ready`、`availability`（online 或 offline），且不含 `authorized` 键（缺失即视为已授权）；不包含 `msp_ready` 的行不属于机器。运营方的目录范围较广：一台开发服务器上可有 20 台广告主机，验证器则可达 65 台。 | 2026-09-27 MSP 接受测试中的运营方数据行，以及本 PR 的实时数据行 |
| transport CLI 2026-09 | `muse sessions --host <host> --json` 返回的 `result.sessions[]` 包含 `{target: "<host>/<session id>", session: {sessionId, title, workspaceRoot, status: idle\|running\|notLoaded, activeTurnId}}`；`muse show <target>` 还会附加 `pendingRequests[]`。每台在线主机每次摘要生成时执行一次 `sessions --host`，每个运行中的会话执行一次 `show`。`--all-hosts` 会进行目录扫描，但仅限于 64 台主机；超出此数量时返回退出码 4，状态为“partial”，无数据行，并为每台主机生成一条 `errors[]` 记录——这些记录绝非面板的来源。 | 2026-09-27 实时数据（第 24 行；验证器 V-43535 原始报文）；host-manager 模拟数据同时支持这两种格式 |
| host-manager | `open --host H --cwd P --name N [--prompt-file]` 相当于先执行 `muse start`，再以 `muse send --busy queue` 发送简报；`send --steer` 即 `muse steer --turn <running>`；`close --confirm` 先执行 `muse interrupt --turn`，再执行 `muse task stop-all`，且主机仍会将该会话标记为 idle（无针对单个会话的结束指令）；若在 host-manager 未记录的会话主机上执行 `stop`，则会返回 `no_such_session`，因此本技能将 `stop` 重命名为 `close`。 | host-manager 的 `test_msp_provider.py`；本技能的 `test_msp.py` |

