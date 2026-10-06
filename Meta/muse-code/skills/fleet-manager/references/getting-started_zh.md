# 入门：从无到有——五步完成首次会话

`<fleet>` 是 `python3 <skill-dir>/scripts/fleet_manager.py`，其中的目录取自提供该技能的读取结果（存在 `skill-dir` 时使用该目录，否则使用其显示的 SKILL.md 路径所在的目录）。每个命令都会输出一个 JSON 对象；`text` 是人类可读的部分，`next` 是当某些内容缺失时应执行的下一个命令，而 `<fleet> <verb> --help` 则列出某个动词的参数选项。

## 1–4. 医生、打开、上下文、引导

```text
<fleet> doctor                                   # 提供者、机器、下一步命令
<fleet> open                                     # muse、本仓库根目录、自动名称、本主机
<fleet> open --engine claude --cwd ~/repo --name reviewer --prompt-file brief.md
<fleet> context                                  # 整体情况；阅读 `text`，根据 `groups` 执行操作
<fleet> read s1                                  # 它正在做什么或说了什么
<fleet> dialog s1                                # 它在询问什么（Herdr）
<fleet> approve s1                               # 回答 y/N 或编号式对话
<fleet> send s1 "提醒：我将在5分钟后关闭此会话"   # 发送通知，无需输入任何内容
<fleet> send s1 "运行测试" --type --wait    # 作为提示输入；若编辑器非空则会被拒绝
```

`doctor` 的回答可能是 `provider: herdr`（服务端已响应，若未运行则启动）或 `provider: tmux`（存活状态、回滚缓冲、受保护的输入、仅支持打开/停止/关闭）；`needs_user_action` 表示 tmux 安装需要密码：执行 `next` 后重新运行。`open` 的回答是 `opened`，并附带 `handle`（`s1`）、`addr`、`identity` 和一份 `receipt`。`send --type` 且 `submitted: false` 表示需先 `read` 再次发送。

## 5. 添加一台机器

```text
<fleet> connect me@buildbox --label buildbox
```

一条命令，每一步输出一行 `progress`；遇到问题时会在第一步停止，并通过 `next` 指明修复方法。若本地没有 Herdr，则会自行开启一个 SSH 主节点（一次登录，最多一次二次认证提示），检查其运行环境，启动已停止的 Herdr 服务并转发其套接字，或将该机器记录为仅支持 tmux 的模式。若本地已有 Herdr，则会先通过 Herdr 添加机器（`herdr machine add`，由 Herdr 自行登录，返回前即关闭），随后复用已存在的转发或 SSH 主节点，否则再开启一个 SSH 主节点：新增主机的成本为 Herdr 的登录费用加上本技能的一次费用；再次执行 `connect buildbox` 通常无需额外费用。之后即可执行 `open buildbox --engine claude --cwd ~/repo`，而 `context` 将覆盖两台机器。远程会话写下的报告可通过 `<fleet> fetch buildbox /home/me/repo/report.md` 收回；内容未变则不会复制任何数据。若某台机器支持 MSP 协议（在 `machines` 列表中，当 `TBH_AGENTS_SESSION_PROTOCOL` 开启时，该机器的 `modes` 会包含 `msp`），则无需执行 `connect`：直接调用 `<fleet> open <host> --cwd /home/me/repo --prompt-file brief.md` 即可打开会话；切勿直接通过 SSH 登录支持 MSP 的主机 ID。

## 故障排除| 你看到 | 它意味着 | 执行 |
| --- | --- | --- |
| `unsupported_by_provider` / `no_provider`（退出码4） | 此提供者无法执行该命令，或不存在提供者 | 执行命令 `next` 中列出的命令（tmux 没有对话框：先执行 `read`，再执行 `send --type`） |
| `needs_user_action`（退出码5） | 某个步骤需要用户操作：密码、二次认证、确认等 | 在 `next` 中执行该步骤，然后重新运行 |
| `provider_unreachable`（退出码6）且伴随 `connect …` | SSH 主控进程已停止，或 Herdr 套接字无响应 | 在 `next` 中执行 `connect` 行，并附加一个终端 |
| `herdr_add_failed`（退出码6） | Herdr 无法添加该机器（其自身输出会说明原因） | 执行 `next` 中的 `herdr machine add …` 行；对于仅支持 tmux 的机器：`connect <target> --mode tmux` |
| `no_such_session`（退出码3） | 句柄所指的会话已不存在 | 执行 `list` 查看；该会话已结束或被关闭 |
| `identity_mismatch`（退出码3）“不是为其签发的会话” | 窗格 ID 或名称现属于另一个进程 | 执行 `list`，使用新的句柄 |
| `composer_not_empty`（退出码3） | 有人正在该会话中输入内容 | 执行 `read`，等待或完成当前行，然后重试 |
| `session_live`（退出码3）“处于活动状态；拒绝关闭” | 该会话仍在运行 | 先执行 `stop`，或使用 `close … --confirm "<用户的原话>"` |
| 某台机器在 `context` 中显示 `stale` | 该机器在跳过窗口期内停止响应（参见 `verbs.md` 中的“故障频率”） | 目前无需处理；保留最后一组记录；后续会生成一条“故障”记录 |
| `herdr machine list` 失败 | Herdr 客户端目录损坏 | 手动执行 `herdr machine list --json`；`local/...` 地址仍可正常工作 |

状态信息存储于 `~/.local/share/muse/fleet-manager/state.json`（包括句柄、故障记录、最后的 `context`、每个已获取文件的最新哈希值），已获取的文件存放在 `~/.local/share/muse/fleet-manager/home/<machine>/`，机器配置位于 `~/.config/muse/machines.toml`，转发规则和 SSH 主控进程则存于 `/tmp/fleet-manager-<uid>/`。删除状态文件只会丢失句柄；下次执行 `list` 时会重新生成新的句柄。