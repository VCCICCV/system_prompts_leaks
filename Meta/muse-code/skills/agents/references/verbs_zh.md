# 代理合约动词（`scripts/agents.py`）

本技能自带的唯一辅助工具。它维护一个**项目文件夹**——包含目标、记忆、线程和收件箱——并将协调员的决策转化为具体操作：通过 `host-manager`（本机）或 `fleet-manager`（已保存的机器）开启线程，记录每个线程的归属，一次性归档事件，并计算状态回复所涉及的分组。它本身不作任何决策：哪些工作会成为线程、在线程运行在何处、何时完成以及需要记住什么，均由协调员调用决定（参见 `references/coordinator.md`）。

```
python3 scripts/agents.py <动词> …
```

协调员最常输入的六个动词如下（其余内容将列出所有标志）：
```
propose <slug> --threads-json -   # 从 stdin 接收 {"threads": [{"id","name","brief","worktree","owns":[…],"test_command","unattended",…}]}
tick <slug> --arm monitor --command "<要打印的命令>"
follow <slug> --pr <url> [--pr <url>…]
stop <slug> <id> [<id>…]
ack <slug> <id> [--progress NN --basis "<理由>"]
inbox drain <slug>
```

该辅助工具基于标准库 Python 编写，可通过技能目录中的路径直接运行。它会查找同级技能 `host-manager` 和 `fleet-manager`（分别为 `../host-manager/scripts/lane_runtime.py` 和 `../fleet-manager/scripts/fleet_manager.py`）；`MUSE_AGENTS_HOST_MANAGER` 和 `MUSE_AGENTS_FLEET_MANAGER` 则用于指定其他命令（测试时会在这些变量中注入模拟对象）。只有当 `fleet-manager` 的 `open` 动词返回 `--help` 时，才会将其计入有效计数（每次运行仅探测一次）：若仅存在重命名文件，则在 `doctor` 中会被判定为“找到但无 `open` 动词”，且不具备 `remote_threads` 能力；若辅助工具在该探测时崩溃或挂起，`doctor` 会报告故障（`fail`），并且在需要远程会话的动词执行时，将被视为工具故障（退出码 6，并输出最后一条 stderr 日志），绝不会被认定为“无动词”。项目数据存放在 `~/.muse/projects/` 目录下；`MUSE_PROJECTS_HOME` 可覆盖此默认目录。

## 统一的 JSON 格式

每个动词都会在 stdout 上输出**一个 JSON 对象**，无论成功或失败——其格式与 `host-manager` 和 `fleet-manager` 输出的完全一致（两种文本模式除外：`tick --wake-line` 只输出一行 WAKE 信息，`pick` 则先输出要发布的消息，再输出一行 `--- dialog ---` 以及对话规范）：

| 键 | 含义 |
| --- | --- |
| `outcome` | 发生了什么，单字描述 |
| `provider` | 线程所属的会话提供者（如 `herdr`、`tmux`），或 `null` |
| `ref` | 项目行对应 `<slug>`，线程行对应 `<slug>/<thread>`，或 `null` |
| `capabilities` | 当前安装具备的能力：`local_threads`、`remote_threads`（发现 fleet-manager）、`unattended_flag`（host-manager 的 `open` 宣称支持 `--unattended`——D16 版本的写法，在此情况下普通 `open` 即为引擎自身的提示）、`trusted_flag`（其 `open` 宣称支持 `--trusted`——引擎预先为 Claude Code 或 Codex 线程植入信任记录，第 6 次修订）、`hooks_flag`（其 `open` 宣称支持 `--hooks-approved`——协调员批准的钩子已预置在线程的配置文件中，第 9 次修订） |
| `progress` | 按顺序列出每一步的状态，同时实时输出到 stderr |
| `next` | 下一步要做的事情——带有当前生效标志的命令，或“结束本轮”；始终存在，重复读取也不会改变结果（详见“`next` 的含义”部分） |
| `error` | 仅在失败时出现：第一条错误信息 |
| `receipt` | 仅在写入类动词中出现：包括 `what`、`project`、`thread`（若有）、`who`、`when`；`go` 会返回收据，每开启一个线程即生成一份 |

stdout 传输的是 slug、ID、名称和路径，绝不包含环境变量。各动词特有的键值将在下方按动词分别列出。所有写入类动词均需指定 `--asked-by WHO`（默认为 `MUSE_AGENTS_ASKED_BY`，否则使用登录名）；该信息会记录在收据中，并在开启远程线程时置于 fleet-manager 动词之前。

## 环境变量| 变量 | 含义 |
| --- | --- |
| `MUSE_PROJECTS_HOME` | 项目目录（默认为 `~/.muse/projects`）；传递给每个本地线程 |
| `MUSE_AGENTS_HOST_MANAGER` / `MUSE_AGENTS_FLEET_MANAGER` | 同级辅助工具的命令（默认为同级技能的 `scripts/` 目录） |
| `MUSE_AGENTS_ASKED_BY` | `--asked-by` 的默认值 |
| `MUSE_AGENTS_TMUX` | 协调器自身会话所使用的 tmux 命令（`tmux -L <socket>`）；作为 host-manager 的全局 `--tmux` 参数在每次调用时传递，因此线程会在该服务器上打开、查找并停止；未设置时使用 host-manager 的默认值 |
| `MUSE_AGENTS_PROJECT`、`MUSE_AGENTS_THREAD`、`MUSE_AGENTS_ROLE` | 由 `go` 命令在某个线程上设置；`remember` 拒绝将 `MUSE_AGENTS_ROLE` 设置为 `thread`。`MUSE_AGENTS_ROLE=launcher` 由一个程序设置，该程序会为其随后开启的协调器会话运行 `init`（即启动器）：`init` 不记录任何协调器（`coordinator: null`），且该会话的首次 `resume` 会被绑定 |
| `MUSE_EXPERIMENTAL_AGENTS=on` | 由 `init --detach`、`go` 和 `follow` 传递到它们开启的每个本地会话中（此处可见该技能，因此新会话也必须能看到它——ADR 38715 D15）；fleet-manager 的 `open` 不接受 `--env` 参数，因此远程线程仅从其所在机器的环境变量中获取（进度行会注明这一点） |
| `AGENTS_TEST_LAUNCHER_ARGV` | 仅用于测试，在“seams”开关开启时使用：一个 JSON 列表，用于替代启动会话的命令行参数 |
| `AGENTS_TEST_COORDINATING_PID` | 仅用于测试，在“seams”开关开启时使用：一个用于替代协调进程的 PID（测试套件固定使用自己的 PID，因此任何测试的身份都不会反映是谁运行了 `run.sh`，以及该 shell 是否在首次测试后仍然存在——#42960） |
| `AGENTS_TOOL_TIMEOUT_S` | 对单次同级辅助工具调用的失败时限（默认为 120 秒）；绝不用于等待 |
| `AGENTS_NOW` | 用于测试的固定时钟 |
| `MUSE_AGENTS_TEST_SEAMS` | 值为 `1` 时启用仅用于测试的 seams；其他任何值则忽略它们（在进度信息中显示为 `ignored_env`） |
| `AGENTS_TEST_HOLD` | 仅在上述开关开启时用于测试：格式为 `<point>:<ready>:<go>`——在执行 `inbox-put` 或 `accept-judged` 时（两者均在锁内，后者在 `done`/`proposed` 校验之后），辅助工具会打开 go 管道，向 ready 管道写入 `reached`，并阻塞读取 go 管道，以便测试套件能够验证锁是否已被持有，以及在另一写入者操作期间会发生什么 |

`state.json`、收件箱及每个线程的记录都在同一把文件锁下写入（`<slug>/.lock`, `flock`）：线程在协调器执行 `tick`、`context` 和 `resume` 时运行 `report` 和 `inbox put`；每对读-修改-写操作都是原子的；`tick` 和实时的 `follow` 路径会在其（未加锁的）存活探测后重新在锁下读取记录，而 `follow` 的重新打开路径则基于加锁后的重新读取来获取最新记录，因此在此期间提交的报告会被保留，且 `accept` 会在锁下判断 `done`/`proposed`。

## 统一的退出码表

| 代码 | 含义 |
| --- | --- |
| `0` | 正常 |
| `2` | 使用错误——消息中会指明相关标志或参数 |
| `3` | 被守卫拒绝：不存在该项目或线程（当 slug 对应的文件夹处于归档状态时，记录中会包含 `"archive_path"` 和 `"archived_at"`，`next` 会读取这些信息），slug 已被占用，线程未命名或未被提议，协调器或线程仍在运行但未确认，线程尝试写入内存，接受已完成或从未运行的线程，手写 Monitor 记录在 arm 时被拒绝（`arm_not_persistent`；`next` 为准备就绪的记录）。无任何变更 |
| `4` | 当前不支持：此技能旁无 host-manager，或指定了某台机器但未使用 fleet-manager；`next` 会给出替代方案 |
| `5` | 因需人为介入的步骤而停止（host-manager 或 fleet-manager 停止时返回 5）；`next` 为该步骤 |
| `6` | 证据不可用：服务提供方无法响应，或无法打开指定线程。除非记录中注明 `created: true`，否则无任何变更 |
| `7` | 内部错误——请报告该记录；无任何变更 |

主机管理器或车队管理器的退出码会原样传递，并附加在 `underlying` 下的底层行上。
  
## `next` 的含义
  
`next` 指导的是转向，而非循环：它指明了动词执行后要做的唯一一件事，且在写入操作后绝不会响应 `context <slug>` 或 `overview <slug>`。判断标准仍遵循 `SKILL.md` 中的规定。
| 之后 | `next` |
| --- | --- |
| `doctor` | 第一回合的 `init` |
| `init` | 当给定了 `--done-means` 时，执行 `propose <slug> --threads-json -`；否则先写 `## Done means`，再写该提案行；
| `init` 针对以 `/agents` 或 `agents:` 开头的请求 | 同上，但加上前缀：在 /agents 下，除非需要同时开启多条线程或长时间等待（此时使用线程，或仅开启一条跟进线程），否则由您自行处理；计划行需注明选择及其理由（ADR 38715 修正案 10，#42241；它取代了所有者规则 30/31 中的默认多线程策略）；
| `context` | 写下 `## Done means`；若无线程存在，则执行 `propose …`；收件箱按顺序推进（针对 `PR:` 报告执行 `follow <slug> --pr <url>`，处理 `BLOCKED(HUMAN)` 问题，执行 `ack`，验证后对已合并 PR 的线程执行 `accept`——跟进线程在其最终报告时自动结束——对于已消失的线程则判断是否重新开启），随后结束本回合；若某回合首次查看时 `changed` 为空，则根据事实说明情况（`since_last_context_s`、`unchanged_for_s`、`context_call`）；“立即结束本回合——上下文会告知已静止多久以及何时再次唤醒；否则回复用户并结束本回合”；
| `propose` | 显示列表，结束本回合；用户后续发出 `go <slug> <ids>` 时再行动——或在 `start_threads: auto` 下仅执行该 `go` 行；每条线程均由单独的 `go` 命令开启，线程即为主管-管理器会话；项目工作绝不用 `subagent_spawn`（所有者规则 52：进行中的子代理路径既无报告也无唤醒）；
| `go` | 重复 `text`；若未记录唤醒，则执行 `tick <slug> --arm monitor\|scheduler --command "…"`（或在两者均不存在时执行 `--arm passive --monitor-failed "<line>"`——上述主管-管理器语句也适用于此 `next`），并结束本回合；否则在 TUI 中重复 `text`（各线程的附加命令）——仅在频道中执行 `channel_line`：附加命令绝不会到达频道（所有者规则 48）——随后“绝不睡眠任何时间（唤醒会带来报告）”，最后执行结束行，无论何种路径——首次 `go`、已在武装唤醒下的后续 `go`（在已唤醒的基础上再开启线程）、重新开启：“立即结束本回合；监控器会唤醒你。”——给协调员的一则备注，不含时间戳，从不发布（已粘贴至计划中）（调度器：“……调度器在另一个进程中计时，用户的下一条消息会将你唤回”；被动：“……没有任何东西会唤醒你（被动）——用户的下一条消息会将你唤回”）；
| `tick --arm`、`tick` | 与 `go` 在武装后执行相同的结束行（“立即结束本回合；监控器会唤醒你。”，或该层级自己的措辞）；普通 `tick` 结束本回合，并在下次唤醒时再次查看——若无唤醒记录，则执行武装行；当 `changes` 不为空时，先执行 `remember <slug> --text "<什么发生了变化；接下来是什么>"`（变化中的唤醒检查点）；
| `follow` | 向用户重复 `text`（跟进线程及其附加命令）；结束本回合；“绝不睡眠任何时间（唤醒会带来报告）”；跟进线程在此处汇报——同样的“绝不睡眠”措辞也适用于 `go`（首次 `go`、重新开启以及带有活跃线程的局部 `go`——无论是新开启还是已在运行——均相同；局部 `go` 时，先执行失败线程的修复，再执行附加命令列表和“绝不睡眠”的措辞，最后是武装及结束行）以及传递 `BLOCKED(HUMAN)` 回答的 `context` 流动（#41850：“每次之后休眠 240/280/90”）；收到 `queued` 跟进确认后，“切勿自行落地；绝不睡眠任何时间（唤醒会带来报告）”；
| `report`、`inbox put` | 存档以供协调员下次唤醒时查阅（`PR:` 报告会标明协调员在读取该报告的回合中运行的 `follow` 行）；
| 对已完成报告（`PR:` 行或已完成的 STATUS）的 `ack` | “立即核实；当满足 `done-means` 时，在本轮唤醒中执行 `accept <slug> <id>`（关闭其会话）；否则说明还缺什么”——每条已完成线程确认一次，最后“然后结束本回合”（#41851：已确认的完成报告一直未被接受，会话仍在延续，直到用户提出要求）；
| 对进展报告的 `ack`、`accept`、`remember`、`stop`、`inbox drain` | 继续处理手头事务；若无其他待处理事项，则结束本回合——`inbox drain` 首先会提示 `<n> 条报告未读——<姓名>`，当仍有报告既未被确认也未被接受时：清空收件箱，但不影响您的阅读进度；
| `relay --asked` | 等待用户的答复；收到后，执行 `relay <slug> <id> --fingerprint <fp> --answer "<他们的答复>"`；结束本回合；
| `relay --answer`（已送达） | 答复已随线程送达；提问在其下一次报告时关闭；结束本回合；“绝不睡眠任何时间（唤醒会带来报告）”；
| `relay --answer`（失败） | 未送达，并附原因：该转达仍为待办——一旦发送可以成功，再次执行相同的 `relay … --answer`；说明什么未送达，绝不说已被转发；
| `overview` | 根据 `text` 回复用户（“等待您”的行再次携带线程的附加命令），然后结束本回合；
| `pick` | 将 `--- dialog ---` 行之上的内容原样作为您的消息发布，随后以该行下方的 JSON 提问（其选项同样包含文本）；结束本回合；
| `resume` | 自行设置唤醒；
| 拒绝 | 清除该拒绝的命令；在辅助工具以及每个子命令上使用 `--help` 选项，会以每行一个标志的形式列出所有标志；
辅助工具的源代码并非供阅读之用。

## 项目文件夹结构

```
~/.muse/projects/<slug>/
  PROJECT.md              目标、请求（用户输入的消息）、“完成的含义”、范围、仓库、常规指令、设置
  MEMORY.md               共享记忆——仅通过 `remember` 命令写入
  TASKS.md                用户的待办清单（包含指针；问题和拉取请求才是最终依据）
  state.json              schema agents-project/v1：协调人、仓库列表（按初始化时记录；线程仅继承来自用户在 Muse 的 trust.json 中也信任的仓库的信任——init 指明哪些缺少记录）、唤醒臂、游标；仅在收件箱路径下存在（不存在时表示监控路径，FR-41038-5）：`wake_path: inbox`（在初始化时记录并保留至归档——ADR 41038 D5；监控唤醒在两条路径上均为最低优先级，直到 #41228）、`inbox_target` / `inbox_label` / `inbox_reason`（报告目标：本次会话的 Muse 会话 ID、其所属的工作区标签、无其他原因时的原因）
    tracking.json           schema agents-tracking/v1：进度账本——工作流、线程→工作流、每个活跃线程每轮观察一次、待处理的问题（见下文）
  threads/<id>/record.json   schema agents-thread/v1（见下文）
  threads/<id>/brief.md      线程开启时所收到的简报内容
  threads/<id>/report.md     线程自身的报告（全文重写；简报中会注明此路径）
  threads/<id>/leftover/     已落地检出后移出的报告文件归档
  threads/<id>/remember.md   报告中的 `## Remember` 部分，供协调人查阅
  library/                 线程为项目生成的文件；wake.sh，辅助工具自身的唤醒循环
  inbox/new/<seq>-<hash>.json   未处理的事件
  inbox/done/<seq>-<hash>.json  已处理的事件（为保证幂等性而保留）
```

`PROJECT.md` 是可供人工编辑的 Markdown 文件。`init` 会写入以下标题，
`context` 则负责读取它们：“## Goal”、“## Request”、“## Done means”、“## Scope”、“## Repositories”、“## Standing instructions”、“## Settings”。设置项以 `key: value` 形式列于 “## Settings” 下：

| 键            | 默认值 | 含义                                                                 |
|---------------|--------|----------------------------------------------------------------------|
| `max_parallel` | `4`    | `go` 命令将同时保持打开的线程数                                     |
| `start_threads`| `propose` | `propose`（等待 `go` 命令）或 `auto`（由用户指定）                     |
| `unattended`   | `inherit` | 新线程的无人值守状态：`inherit` 表示继承该协调人的审批状态（若审批关闭则设为无人值守）；也可直接设置为 `true` 或 `false` |
| `follow_every` | `5m`   | 跟踪线程的运行频率，写入其简报中                                     |
| `stuck_scans`  | 未设置 | 项目自身的连续运行轮数阈值——即多少个唤醒周期内必须有其他线程活动（非原始的 30 秒周期），否则会触发线程的 `flat` 标志；未设置时仅记录时间（`quiet_for_s`），不设置标志 |

`MEMORY.md` 由 `## <日期> <标题>` 条目组成。`TASKS.md` 则由 `- [ ]` 行构成。**线程记录**（`threads/<id>/record.json`，`schema: agents-thread/v1`）：
`id`、`name`、`kind`（`work` 或 `follow`）、`status`、`brief`（协调员的文本）、`cwd`、`repo`、`worktree`（来自提案的分支名，或为 null）、`worktree_path`（由 `go` 为其检出的路径，或为 null）、`machine`（`local` 或某个标签）、`provider`、`ref`（提供者的裸引用：地址由 `machine` 和 `ref` 组成，仅在首次设置时确定）、`server`、`engine`（在 `propose` 时设定——使用提案的引擎，否则使用协调员自身的引擎——并在开放收据的身份信息中确认）、`engine_args`（提案中的参数列表）、`identity`（提供者作为主机管理器或舰队管理器所报告的元组；远程线程的 `status` 回答会与其逐项比对，`identity_drift` 列出差异字段，格式为 `key: 'was' -> 'now'`，按照舰队管理器提供的 `status` 显示，当以该线程名义开启的会话并非为其实际开启的会话时）、`model`、`effort`、`unattended`、`posture_source`（决定请求姿态的规则——`thread`、`project`、`inherited`、`unreadable`；FR-38715-15）、`allow_list_path`（助手写入线程检出目录的引擎自有规则文件；若无则为 null）、`posture_applied`（开放收据所报告的内容：若助手指定了某姿态，则以其表述为准，如 `engine_default`；否则取自其姿态标志列表——若有标志生效则为 `unattended`，若无且未请求则为 `attended`，若请求了 `unattended` 但收据未携带任何标志则为 `unknown`，此时由主机管理器仅为 Muse 添加该标志；若助手未指定 `--unattended` 则为 `provider_default`）、`proposed_at`、`amended_at`（`propose --replace`）、`opened_at`、`receipt`（由主机管理器出具）、`open_note`（提供者在开放时留下的备注，若有）、`attempts`（失败的开放尝试次数）、`prs`（`[{url, last_event, head, at}]`）、`report`（`{digest, at, status_line, blocked_line, decisions_line, pr, remember, progress}`——`progress` 是线程的 `Progress: NN% — <basis>` 行，格式为 `{percent, basis}`，若无则为 null）、`acked_digest`、`calibrated`（由 `ack --progress` 提供的 `{value, basis, rung, at}`，在此之前为 null）、`workstream` 和 `workstream_ref`（提案的标题和引用，若无则为 null）、`evidence`（在 `accept` 之前为空数组）、`ended_at`、`stop_receipt` 和 `session_ended_at`（当 `stop` 或 `agents.py archive` 结束其会话时）、`last_agent_status`（`tick` 最后看到的屏幕输出或 Herdr 的裁决，因此一个处于“等待您”状态的线程会在每次状态转换时更新一次）。

**线程状态**是一种已记录的事实：`proposed` → `running` →（在提供者报告其已结束并经核实后变为 `exited`）→ `done`（仅在 `accept` 时发生）；`stopped`（由 `stop` 或 `agents.py archive` 引发）；`orphaned`（既无报告又失去身份，由 `tick` 或 `resume` 设置）。线程自身的“done”只是其报告中的一项声明，而非状态。状态是对记录的描述；会话可能会长于状态本身（一个已“done”的线程的引擎仍会继续运行，直到被 `stop` 或 `agents.py archive` 结束）。

**分组**（由 `context` 和 `overview` 按照以下顺序根据事实计算，以首个匹配为准）：

| 分组 | 事实 |
| --- | --- |
| `done` | `status = done`（已记录证据） |
| `orphaned` | `status = orphaned`，或实时检查结果显示已消失/不匹配，且不存在比 `opened_at` 更新的报告 |
| `waiting-on-you` | 报告的 `BLOCKED(HUMAN):` 行比 `acked_digest` 更新，或提供者报告该代理处于“阻塞”状态（Herdr 自己的状态，从 host-manager 的 `list` 行读取——其 `status` 动词仅表示存活），或（tmux，本机）可见屏幕显示对话框或权限提示（`evidence: screen`，`screen: prompt on screen`） |
| `ready-for-review` | 存在一条摘要不是 `acked_digest` 的报告 |
| `landing` | `prs[].last_event` 为 `enqueued`、`queued` 或 `merging` 中的任意一种 |
| `unreachable` | 提供者对某个运行中的线程回复了 `transport_unreachable`（ADR 41038 § 故障模式）：其传输通道已中断但仍在继续运行——绝不会成为 `orphaned`，也不会被列入 `unknowns`；该行带有 `unreachable` 标记（提供者的原话）及其附加信息行，且 `text` 列以附加命令的形式列出该线程；`tick` 对其不作任何记录 |
| `working` | 处于存活状态，且提供者报告状态为 `working` 或未报告任何状态（仅表示存活：Herdr 无法检测到的引擎状态），或（tmux）屏幕显示活动行（`evidence: screen`）；若 tmux 屏幕未显示任何可读内容，则仅以存活状态为准，并在该行标注 `evidence: liveness` |
| `idle` | 处于存活状态，且提供者报告其他任何状态（`idle`、`done`、`unknown`——这些分组由 host-manager 的 `list` 行给出），或（tmux）屏幕显示空闲的 composer 且无任何进程运行，或 `status = exited` 且其报告已被确认，或 `status = stopped` |
| `proposed` | `status = proposed`——由 `propose` 写入，尚未由 `go` 打开 |

host-manager 的 `read` 行上的 `group`（msp 提供者自身的状态：
`working|waiting-on-you|idle|unknown`）在读取屏幕之前就已确定；
`unknown` 会直接归入该分组（在被读取之前，每个 msp 线程仅凭存活状态都被视为 `working`）。tmux 或 msp 提供者的 `status` 只能反映存活状态，因此对于本地存活的线程，辅助程序只需读取一次可见屏幕（host-manager `read <ref> --tail`）并据此判断：识别出已知的对话框和权限提示行、引擎的活动行以及空闲的 composer。如果某一线程的提供者无法作出回应，则该线程没有分组：它会被列入 `unknowns`，并附上 `unknown_reason`（提供者的原话），同时在 `text` 中注明其所在机器及最后已知状态。

## 跟踪账本

`tracking.json`（schema `agents-tracking/v1`）是项目的进度账本——包含指针与检查点，但从不作为任务的权威依据（issues、PR 和工件仍具权威性）。它与 `state.json` 使用同一项目锁，通过原子替换方式写入，缺失时可从记录中重建；若无法读取则会发出警报：任何触及它的操作都会返回 `tracking_error`（包括文件路径及修复方法——将文件移至别处；下一次 tick 会根据记录重新生成新的账本，而之前的版本将丢失），此时状态表仅显示这一条错误信息，绝不会为空表。它保存的内容包括：

- `workstreams`: `{<id>: {id, title, source_ref}}` — `id` 为标题的 slug，`title` 和 `source_ref` 为提案的；`threads`: `{<线程 id>:
  <工作流 id> | null}`。两者在每次写入时均根据记录（`workstream`、`workstream_ref`）重新构建：一个工作流仅包含状态表，绝不会包含会话名称或标签片段。
- `observations`: `{<线程 id>: [点, …]}`，仅追加——每个运行中的线程每 `tick` 记录一个点（包括未变化的值：平缓的序列即为卡住的证据），该线程在 `ack` 和 `accept` 时各记录一个点，以及当线程状态自上一个点以来发生变化时记录一个转换点（停止、退出、孤立、完成——序列在此终止）。一个点包含：`at`、`group`、`status`、`live`、`self_reported`（`{value, basis, source: report}` 或 null）、`artifact_rung`（`{value, basis}`）、`calibrated`（`{value, basis, rung, at}` 或 null）、`progress`（显示的数值，见下文）、`current_action`（报告中的 `STATUS:` 行）、`pending_question`（未决条目的指纹或 null）、`blocker`（报告中的 `BLOCKED(HUMAN):` 行）、`evidence`（含最后事件的 PR 链接，以及 `accept` 的证据）、`flags`。
- `pending_questions`: `[{fingerprint, thread, text, surfaced_at, status,
  closed_at, relay_state, asked_at, relay}]` — 每个问题对应一项：报告中的 `BLOCKED(HUMAN):` 行（`report:<id>:<digest>`）或屏幕上的对话（`dialog:<id>:<stamp>`，每次状态转换时记录一次）。问题保持开放，直至线程继续推进——下一份报告覆盖该行、屏幕退出对话或线程完成——这便是从当前视角看来的答案送达；`ack` 并不关闭任何问题（用户仍保留该问题）。报告中的问题遵循其接力生命周期（#44029，§ `relay`）：开放时 `relay_state` 为 `blocked-unasked`，一旦提问（或回答）被记录则变为 `asked-relay-owed`，接力发送完成后变为 `relayed-awaiting-worker`；`asked_at` 为记录的提问时间，`relay` 为记录的接收信息（`{thread, fingerprint, answer, at, send}`），在此之前两者均为 null。对话条目中的三个接力字段均为 null：其答案直接在屏幕上给出，从未经过接力。

**Artifact rung**——仅依据记录中的事实得出的证据值，绝不基于主张：`proposed` 为 0；无 PR 时为 25（“暂无 PR”）；PR 已打开（除落地或 `merged` 外的任何事件）为 50（“PR 打开”）；处于 `enqueued`、`queued` 或 `merging` 状态为 90（“落地中”）；`merged` 为 95；`status: done` 为 100（“已接受”）。若有多个 PR，则取最低值，基准以已合并的 PR 数量为准。**显示值**（行与点中的 `progress`）：协调员的 `calibrated` 值，在其所参照的等级仍然有效期间使用——此后发生的变动（如合并、接受）将取代该值，直至下次 `ack --progress`；否则仍采用该等级本身。绝不超过该等级，且 `self_reported` 值亦不得将其提升（用户只看到一个值；两种原始数据仍保留在账本中供协调员参考）。

**序列相关事实**在每一行 `context`/`overview` 中呈现：`elapsed_s`（自开启以来的时间）、`quiet_for_s`（自序列上次变化以来的时间——即尾部不变序列的第一个点；若尚未有记录则为自开启以来的时间）、`pending_question`（未决条目或 null）以及 `flags`，这些是协调员判断的候选事实：“flat”（尾部不变序列至少跨越了项目设定的 `stuck_scans` 唤醒轮次——即其他线程的点发生改变的 tick，而非原始 tick；无设置，无标志）；“regressed”（线程自身的 `Progress:` 低于先前值，或 artifact rung 低于峰值——意味着丢失了某个 artifact；但协调员自身的向下 `ack --progress` 不计入，那只是其判断，而非线程自身回落）。系统未预设通用阈值，助手也不估算完成时间：事实就是序列、等级和已耗时间，解读权在于您。**状态表**（`overview --table`；`overview` 的 `table`；每次唤醒时 `context` 的 `text` 尾部——由辅助函数渲染，从不手动输入）：`<slug>：N 个线程 · L 已落地[ · A 已接受]`——`landed` 统计带有 `merged` PR 事件的线程数，`accepted` 统计未带该事件但已标记为 `done` 的线程数（仅在有此类线程时显示；当用户表示“不要合并”时，会反馈“3个中有2个已落地”）；在每个工作流标题下（其余归于 `other:`；若无线程归属则不设表头），每条线程占一行——首列显示 `<name> (<engine>) [<id>] · PR <n>`（会话名仅在附加命令中出现），接着是进度条 `██████░░░░ 60% <basis>` 及其对应的 ETA 和基准值，以及分组和 `quiet <time>`——随后最多两行页脚（`landed L of N[ · accepted A] · inbox K pending · wake <tier>`；若有标志则显示 `flags: …`），并为每个待处理的问题添加一行 `Needs you: <name> (<engine>) [<id>] — <question>`，以及对话及其附加命令。`overview` 还会返回 `needs_you`（即待处理的条目），与 `context` 的行为一致。

## 收件箱事件与幂等键

事件格式为 `{schema: "agents-event/v1", key, kind, at, thread, text, data}`。`inbox put` 每次按 `key` 存储一次事件；若使用已在 `inbox/new/` 或 `inbox/done/` 中的键再次调用 `put`，则视为 `duplicate`（退出码 0，`deduplicated: true`，不写入任何内容）。各来源对应的键结构如下：

| 来源 | 键 |
| --- | --- |
| 线程报告 | `report:<thread>:<报告正文的SHA256摘要，12位十六进制>` |
| PR 事件 | `pr:<head sha>:<event>`（`checks_failed`、`review`、`enqueued`、`merged`、`conflict` 等） |
| 线程状态变更（`tick`） | `thread:<thread>:<status>:<opened_at>` |
| 转为“等待您”的线程（`tick`） | `thread:<thread>:waiting-on-you:<stamp>`——文本中携带其附加命令；每次状态转换时记录一次 |
| 外部订阅 | `sub:<source>:<seq>` |
| 用户或协调员手动录入 | `user:<任意文本>` |

在收件箱路径上（ADR 41038 D1；`wake_path: inbox`），`report` 事件和 `pr …:merged` 事件还会作为 `agents-message/v1` 消息发送给协调员会话——这是监控器 WAKE 行之外的快速通道，后者仍以同一事件命名——而通过 `inbox put --message` 存档的副本则使用消息自身的 `key:`：即被送达两次的副本，或已被监控器命名过的副本，只存档一次。

## `doctor`

```
doctor [<slug>]
```

返回健康状态（0）或首个缺失项（`needs_user_action` 5，`no_host_manager` 4，`sandbox_blocked` 5）。检查项包括：`python`、`projects_home`（可写）、`host_manager`（找到且自身 `doctor` 结果正常）、`tmux`（仅当主机管理器的 `list` 在健康状态下失败时出现；若套接字返回 `Operation not permitted` 则判定为 `fail` 并标记 `sandboxed: true`——即沙盒环境下的 shell，无法打开、读取或停止会话；否则发出警告并说明原因）、`fleet_manager`（找到或缺失——缺失时发出警告，远程线程不可用）、`git`、`gh`（缺失时发出警告：后续线程需要它）、`wake`（列出 `monitor`、`systemd-run`、`launchctl`、`crontab` 等工具的存在情况——由协调员选择并启用）。`capabilities` 字段在主机管理器的 `open --help` 显示支持无人值守模式时标注 `unattended_flag`（仅探测一次）。`sandbox_blocked`（5）：`next` 指明升级后的 shell（`sandbox_permissions: require_escalated`）或不具备沙盒权限的协调员会话；`init`、`propose` 和 `context` 仍可在沙盒环境中运行。在 `sandbox_blocked` 下，应以升级后的 shell 执行 `init` 及其后所有命令（录制的协调员必须是执行 `go` 的那个 shell）：该 shell 无法访问 tmux 套接字。指定项目 slug 时，还将检查该项目文件夹、协调员记录及唤醒臂配置（仅在收件箱路径上 `wake_path: inbox` 时生效）。

## `init`

```
init - [--done-means TEXT] [--slug S] [--repo DIR …] [--max-parallel N]
       [--goal "<任务在线程中的表述>"]
       [--unattended|--attended] [--start-threads propose|auto] [--engine muse|claude|codex] [--detach] [--asked-by WHO] <<'EOF'
<用户的完整信息>
EOF
```

`init -` 从标准输入读取任务，方式与 `propose --threads-json -` 读取 JSON 相同：使用带引号的 here document 可以将用户的反引号和 `$(…)` 保留为文本；切勿将其置于双引号内：用户输入中的反引号或 `$(…)` 会在其克隆环境中作为命令执行；绝不会为了使其对 shell 安全而修改消息。用户输入即为完整消息——包括定义 PR 的每一个编号需求、引用字符串和句子，以及合并信息、测试命令或源地址，但不包括其标题：后续简要引用的内容仅来自 `## Goal`；若进行转述，则会丢失这些内容。`init "<message>"` 若用双引号括起，会让 shell 在用户克隆环境中执行其中的反引号命令 `python -m pytest -q`，并将输出作为目标；而另一位协调人则去除了全部 70 个反引号以避免这种情况。位置参数仍被接受；若其中包含反引号或 `$(…)`，则会返回警告：`[{kind: `task_on_the_command_line`, text, next}]`，且 `next` = `通过标准输入传递消息（init -）`，该行本身的 `next` 即以该警告开头——此警告会被记录为已接收，绝不会被拒绝。`init -` 若标准输入为空，则视为“用法错误”（2），不会创建任何内容。普通的 `init` 会创建文件夹，并将你和本次会话记录为协调者；其返回结果包含医生的检查。请将 `--slug` 保持简短——最多24个字符：该 slug 会作为每个会话名称的前缀（`<slug>-<id>`），而超过63个字符的名称会在80列的终端中换行，并挤占 Monitor 行的空间。它还会创建以下内容：`PROJECT.md` 文件，其中包含由 `--goal` 指定的 `## Goal` 部分——即线程们所理解的任务描述，需由你亲自撰写；`## Goal` 是每个线程都会读取的内容，一旦线程读到那些针对你的指令（“提出议题并等待我的指示”、“你负责协调”、“不要自己动手”），就会自动成为协调者，因此请勿在目标中加入这些内容。若未指定该标志，则原样保留输入的消息作为目标。无论是否使用该标志，输入的消息都会完整保留在 `## Request` 部分；`--done-means` 指定的 `## Done means` 部分（用于标识项目结束的证据；若未指定则为空——此时由协调者在任何线程之前补充）；`--repo` 指定的 `## Repositories` 部分（默认为从当前目录向上查找的仓库根目录）；以及由各标志设置的 `## Settings` 部分。此外，`MEMORY.md`、`TASKS.md`、`threads/`、`library/` 和 `inbox/` 目录均为空。不会创建其他标题。第二次克隆会被视为第二个 `--repo`：助手仅在两个检出共享同一个 `.git` 共用目录时才将其计为一个仓库（工作树属于这种情况；而另一个克隆则不属于）。本次会话会被记录为协调者（`state.json.coordinator`：若存在 `MUSE_LANE_BACKEND` 或 `MUSE_LANE_REF`，则为 `kind: session`；否则为 `kind: process`，包含 `user`、`host`、`pid`——即协调进程，也是助手最近的祖先进程，且既非 shell 也非 Python 解释器——例如，作为启动器 fork 真正解释器的 `python3` 是助手每次调用时的父进程，但绝不是协调者——还包括 `command` 和 `started` 时间戳，这些信息可通过 Linux 上的 `/proc` 或其他平台上的 `ps` 命令，在同一主机上进行 `resume` 和 `context` 的核验）。若调用者为后续开启的协调者会话运行 `init`，则需通过 `MUSE_AGENTS_ROLE=launcher` 明确说明：在其自身进程树中，该会话并不具有任何身份——遍历至启动器自身的 TUI 后，发现已开启会话的 `resume` 对应着一位活跃的协调者——因此不会进行任何记录（`coordinator: null`，一行 `progress` 提示如此，`next` = 来自该会话的 `resume <slug>`），首次 `resume` 即完成绑定。`--slug` 默认采用任务开头的几个词作为 slug；若该 slug 已被占用，则返回 `slug_taken`（3），且 `next` = `resume <slug>`。若任务未指定 `--slug`，且以 `resume`、`continue`、`reopen` 或 `pick` 开头（或仅为单个词），同时指定了一个已存在的项目，则返回 `resume_instead`（3，不创建任何内容，`next` = `resume <那个 slug>`）：要求接手某个项目的全新会话绝不会无意中创建一个孤立的项目。协调者会被记录为开启会话的宿主管理器（`MUSE_LANE_BACKEND`/`MUSE_LANE_REF`）；否则，在 tmux 内部，则记录为 TUI 自身的窗格（`$TMUX`、`$TMUX_PANE`）——无论是来自沙盒化的工具 shell 还是提权后的 shell，两者看到的都是不同的进程，却共享同一个窗格（默认模式下的协调者，其自动批准的 `init` 记录了窗格，随后批准的 `go` 则以进程形式出现，但首次 `go` 被拒绝）——可通过该 tmux 服务器上的宿主管理器 `list` 命令进行核查（活的记录会列出该窗格）；在沙盒之外，进程（pid、启动时间戳）会依附于窗格记录，而曾记录过某进程的项目，也会接受从其窗格视角看到的同一进程；否则，则按进程（用户、主机、pid、启动时间戳）在 Linux 上通过 `/proc`、在其他平台上通过 `ps` 进行核查；而在沙盒化的工具 shell 中（PID 命名空间下，协调进程为 `pid 2 sh`，任何主机上的 `ps` 都无法识别），由于没有窗格，该 pid 绝不会被记录：协调者处于“不透明”状态——无人能证明自己就是协调者，因此从任何会话发起的 `resume` 都会返回 `coordinator_unknown`，直到人工确认接管为止；具体状况由一行 `progress` 提示。`--detach` 会通过宿主管理器的 `open` 命令开启一个协调者会话，启动器会加载本技能并执行 `resume <slug>`；回执中会携带该会话的身份。`--detach` 必须配合 `--unattended` 使用（`usage`，2，不创建任何内容；若未指定，则即使本会话本身处于无审批状态——无人会坐在一个分离的协调者位置上，因此该措辞仍归用户所有，不会被继承）：无人会响应分离会话的提示，因此必须以 `--unattended` 方式开启（`coordinator.posture_applied: unattended`；带有 `progress` 提示的一行，默认值来自早于 D16 的宿主管理器）。分离的协调者会继承开启它的会话所设定的配置（2026年9月20日生效的所有者规则；ADR 38715 D15/D16）：`--env MUSE_EXPERIMENTAL_AGENTS=on`，以及启动会话本身的 `--model`、`--reasoning-effort`、`--provider`、`--preset`、`--base-url` 和工具调用开关，以 `--engine-arg=--flag=value` 形式的参数传递，从该会话的命令行读取（即协调进程的 argv；但绝不包括其姿态标志、工作空间、工作树、恢复或提示相关的参数——那里的无审批标志仅用于线程继承的姿态调整，见修正案6）。若无法读取命令行（如沙盒化工具 shell 或并非 Muse 会话的启动器），则一律不予猜测。`initialized`（0）：包含 `slug`、`path`、`coordinator`、`settings`、`created: true`，若使用了 `--detach`，还会包含 `launch_settings`（`env`、`engine_args`、`engine_args_source`：`launcher argv` 或未能读取的原因；`binary`、`binary_source`，以及当 PATH 中存在 `muse` 时的 `binary_fallback`：分离的协调者会运行本会话自身的二进制程序——`MUSE_BIN`，否则则解析本会话的命令；绝不会在未声明的情况下使用 PATH 中陈旧的 `muse`）。该行还记录了首轮操作在接下来四次调用中获取的信息（`doctor`、`context`，以及围绕每次 `init` 进行的 `PROJECT.md` 读写）：医生的 `checks` 和 `health`（`{"outcome": "healthy"}`，或医生拒绝时的 `outcome`、`error`、`next`——如 `no_host_manager`、`sandbox_blocked`，均由宿主管理器直接说明——仅报告，不主动抛出异常：无论如何都会创建文件夹），以及 `context` 为新项目返回的画面（`project`、`memory`、`tasks`、`threads`、`inbox`、`coordinator`、`wake`、`host_manager`、`text`；`changed` 和 `since_last_context_s` 均为 `null`——`init` 不移动上下文游标，因此 go 轮的 `context` 仍是首次调用的结果）。`next` 为提案行（`propose <slug> --threads-json -`），前提是已提供 `--done-means`；否则为“先在 PROJECT.md 中写下 `## Done means`……再提出提案……”；若使用了 `--detach` 或由启动器发起，则 `next` 仍为之前的恢复行。目标轮的操作顺序为：`init --done-means "…"`，然后 `propose`：无需单独的 `doctor` 或 `context`。**唤醒路径（ADR 41038 D5）。** `init` 只读取一次 `TBH_AGENTS_SESSION_PROTOCOL` 标志（1/开启/真/是，不区分大小写）；后续的任何操作都不会再读取该标志，因此项目会一直沿用该路径直至归档。若为关闭或未设置：则行为与当前完全一致——不会写入任何键值，且缺失 `wake_path` 时将默认使用监控路径。若为开启：辅助程序会在本地 Muse 会话列表中解析本次会话（通过 `muse session-message list --json` 命令；`MUSE_AGENTS_SESSION_LIST` 可覆盖该命令）：在当前进程的工作区标签（或车道自身的 `session_name`）下，必须且仅有一条记录作为报告目标，并将其记为 `inbox_target`（同时记录 `inbox_label`），此时 `wake_path` 被设为 `inbox`，并在 `progress` 行中予以注明；唤醒循环会被写入，且唤醒臂仍保持当前状态——在 #41228 之前，两条路径均以监控唤醒为最低保障（第 21 轮 B1：工作树线程的消息会停在协调员的准入卡之后，并随线程一同终止）。若不存在这样的记录——即列表已关闭（`external_agent_ingress_closed`），或者在该标签下没有记录、或多于一条记录——则会在监控路径上打开该项目，并抛出警告 `inbox_wake_unavailable`（记录在 `warnings[]` 中，其 `text` 字段说明原因，`next` 指向监控臂）。在收件箱路径上，`init`、`context` 和 `resume` 行都会携带 `wake_path: inbox`（`resume` 还会包含 `inbox_target` 和 `resubscribed`）；而在监控路径上，则沿用当前的行内容及当前的 `state.json`，逐键比对（FR-41038-5）。

## `context`

```
context <slug>
```

一次调用即可获取完整视图，适用于单次协调员轮次。它仅改变自身在 `state.json` 中的 `changed` 游标，且与 `tick` 类似，在该线程的编排器空闲后，会在跟踪记录上生成一条排队的 PR（`follow_delivered`，记录本次调用所生成的 URL；参见 `follow`）。当调用者为已记录的协调员时，会更新 `last_context`；若该协调员无稳定身份（`opaque`，无法识别其身份），则无人能据此辨认。非该协调员的会话会在 `context_cursors.<kind|identity keys>` 下使用各自的游标读取同一视图——通过 `resume`/`not_coordinator` 键进行比较；对于无身份标识的调用者（`opaque`），则会为每位用户和主机各保留一行——因此，第二个 TUI 的 `resume` 或 Shell 的 `context` 不会消耗协调员的增量变更：每个调用者仅在变更发生时看到一次；最多保留八条“陌生人”记录，最旧的优先被移除，而被驱逐的调用者下次调用时将如同首次调用一样。- `project`：`slug`、`path`、`goal`、`done_means`（当协调者尚未填写时为空字符串——请先填写）、`scope`、`repositories`、`standing_instructions`、`settings`；
- `memory`：`MEMORY.md`中各标题及其日期（仅索引，不含正文）；
- `tasks`：`TASKS.md`中每行任务的状态，标记为`done: true|false`；
- `threads`：每条线程一行，包含记录字段（含`attach`），以及`live`（由主机管理器或舰队管理器提供的状态，无法响应时为`null`）、`agent_status`（Herdr提供的状态，或tmux会话的判定结果）、`group`、`evidence`（对于tmux线程，取值为`screen`或`liveness`）、`screen`（屏幕显示内容），以及账本中的各项信息——`artifact_rung`、`self_reported`、`calibrated`、`progress`（显示的数值）、`elapsed_s`、`quiet_for_s`、`eta`、`flags`、`pending_question`（参见“跟踪账本”）；无法响应的提供方会降低`coverage`，将该线程归入`unknowns`而非分组，并将其反馈记录在`unknown_reason`中；
- `inbox`：`pending`（`inbox/new/`目录下的事件，按时间先后排序）及`pending_count`；
- `worker_notes`和`repainted`（FR-43932-6）：针对该项目的Herdr工作节点待处理备注——在线的协调者下属若处于非空闲或阻塞状态，则报告为`unknown`而非`idle`/`blocked`；每次被拦截的报告都会在其项目目录中留下一条备注（`pane`、`server`、`machine`、`slug`、`kind`、`at`、`seq`，通过复制通道读取）；将备注视同其他线程事件进行处理，并使用`relay`转发备注中的问题，切勿自行给出答案。当记录在案的协调者已失效时，超过45分钟的旧备注会被在此处重新标注（为其窗口赋予新的`idle`状态，`seq`比原备注加1），并列入`repainted`（`delivered: false`表示目标机器的套接字未响应——该备注仍处于待处理状态）；
- `coordinator`：记录在案的协调者身份，以及其是否为此进程；
- `wake`：记录的唤醒配置（`tier`、`command`、`armed_at`、`armed_by`、`means`——该层级在协调者会话中执行的操作，也在`text`中重复列出——、`env`和`tick_command`），或为`null`——`null`表示该项目不会被唤醒；协调者可通过`tick --arm`激活一个唤醒配置，或明确声明不启用任何唤醒；
- `changed`：自本次调用者上次`context`以来发生变化的内容（线程分组、新事件、记忆条目、任务状态）；首次调用无可供比较的数据，会予以说明；
- `plan_lines`和`channel_line`：需在唤醒行下方发布的勾选列表，按打印格式呈现（参见§`go`）；
- `since_last_context_s`：自本次调用者上次`context`调用以来的整秒数（首次调用时为`null`）；若期间无变化且不足一分钟，`text`末尾会注明“自<n>秒前无变化”——这是事实陈述，而非拒绝回应；
- `unchanged_for_s`：自本次调用者上一次画面与前一次不同以来的整秒数（首次调用时为`null`，报告有变化时为`0`）：表示持续无变化的时间长度（协调者在活跃时段内约有50%–75%处于休眠状态）；
- `host_manager`：本辅助工具用于自身调用的确切主机管理器命令前缀（当设置`MUSE_AGENTS_TMUX`时包含`--tmux`），以确保您的`send`/`read`/`resources`调用能到达同一服务器；若技能旁无主机管理器，则为`null`；
- `text`：以简洁几行呈现相同画面，可供用户安全阅读——每条线程一行，格式为`  - <name> [<id>] (<machine>): <状态行，或状态词>`，名称在前，ID置于括号内；未归属任何分组的线程将在`unknown`下单独成行，注明所属机器、提供方的理由及最后已知状态——宕机的机器应如实记录为“机器宕机”，而非缺失一行，随后列出状态表（见下文）；
- `needs_you`：按时间先后排列的未决`pending_question`条目（包括尚未被记入账本的`BLOCKED(HUMAN):`报告）；
- `next`：收件箱中的事件按顺序推进（绝非“清空收件箱”：本次调用已读取这些事件）；对于已读报告事件的`ready-for-review`或`waiting-on-you`线程，无论`changed`如何，均从其记录中获取相同的动作（跟进、问题、确认），切勿在仍有未读报告时宣称“无变化，结束本轮”。
- `tracking_error`：仅在无法读取`tracking.json`时出现——包含文件路径及修复方案；此时表格即为这一行。每一次调用都呈现全貌。一个未发生任何变化的注视会携带以下事实——`since_last_context_s`（自该调用者上次的`context`以来）、`unchanged_for_s`、`context_call`（自上次有变化以来该光标所作的注视次数）——并且，从第二行开始，显示文本`nothing moved since <n> s ago — look <N> since anything did; <what brings you back>`（监控器及其手臂时间、调度器或被动话语，或者在未记录唤醒时的手臂提示）。这是否为同一轮中的第二次注视，由你来判断：辅助程序不基于时钟进行同轮猜测（仅提供文本和提示，无守卫）。

`no_such_project`（3）指明缺失的文件夹；`next`为`init`。当 slug 被归档时显示`"archived"`（3）：“archive_path”、“archived_at”，`next`读取已归档的`PROJECT.md`（对于已归档的 slug，每个项目动词均以这种方式响应）。

一个无人安装的监控器（`tick --arm monitor`已记录，但未调用监控器工具，且两条报告已在十分钟前发出但未读）：当记录显示`tier: monitor`、`library/wake.lock/pid`未指向任何运行中的进程，且手臂状态已超过 15 秒（循环在其启动的第一秒内写入其 pid）时，`text`以`wake: monitor armed on record, but no loop is running`开头，`next`则以`call monitor(<the recorded command>) now`打开（若记录行为是`monitor(...)`，则显示该行；否则显示包裹它的就绪行）。这是一个事实，绝非解除武装：记录依然有效。

## `propose`

```
propose <slug> (--threads-json PATH|-) [--replace]
```

记录协调员的提案——即用户尚未批准的线程。
JSON 格式为：`{"threads": [{id, name, brief, cwd?, repo?, worktree?, machine?,
engine?, engine_args?, model?, effort?, unattended?, kind?, owns?, test_command?, workstream?}]}`。其中，`id` 是 `[a-z0-9-]{1,32}` 的字符串，在项目中必须唯一（若 `id` 已被占用，则返回 `thread_exists`, 3）；`brief` 是该线程所对应的一页简述；`workstream`（可以是标题，或 `{title, source_ref}`）仅用于在状态表中对线程进行分组，绝不作为会话名或标签使用；`propose` 会将分配信息写入 `tracking.json` 文件（若填写内容非标题，则视为 `usage`, 2）；`machine` 默认为 `local`；`worktree` 是一个
**分支名称**：对于本地线程，`go` 会在仓库旁为其创建该分支的独立工作区（当分支尚不存在时，从 `HEAD` 检出），并将路径记录在 `worktree_path` 中（`null` 表示在 `cwd` 中工作；同时操作同一仓库的两个线程各自指定一个分支）；对于运行于远程机器上的线程，其 `worktree` 值会原样传递给 fleet-manager 的 `open --worktree` 命令（此时它会命名该机器上已存在的某个工作树，Herdr 专用）；若 `worktree` 不是有效非空字符串（如 JSON 中的 `true`、数字或列表），则视为 `usage` (2)，并指明字段及合法格式——此校验在 `propose` 阶段完成，因此 `go` 不会直接看到此类情况；同样，任何 Git 无法识别为分支名称的字符串（如 `branch: feat/x`、包含空格、`..` 或以 `-` 开头的字符串）也会被视为 `usage` (2)——这些字符串会被原样传递，而所有因违反规则导致 `go` 失败的情况，都会附带 Git 的提示信息；运行于远程机器上的线程，其 `worktree` 值未经校验即被传递给 fleet-manager；`unattended` 取值为 `true` 或 `false`，表示该线程自身的无人值守状态，优先于项目设置及协调员的默认配置（若协调员以 `--yolo` 模式运行，任何 `false` 修改均会被忽略）；若未指定，则视为 `null`；其他任何值均视为 `usage` (2)。该字段的读取发生在记录仍处于提案状态或曾为此打开时（`posture_source` 未设置或为 `thread`）——重新打开的无状态线程会重新推导其无人值守状态；而在本规则生效前已提出但从未打开的记录，则会沿用其存储的 `false` 值，视为有人值守；`engine` 取值为 `muse|claude|codex|shell`，即 host-manager 或 fleet-manager 的 `open --engine` 命令为该线程启动的引擎，默认为协调员的默认引擎（即本辅助程序所在的会话环境；若无法确定，则为 `muse`）。Claude Code 协调员的线程默认使用 Claude Code 会话，除非提案另有说明；其他任何值均视为 `usage` (2)，并明确列出允许的四种选项；`engine_args` 是一个字符串 JSON 列表，即引擎的参数，每个参数通过 `--engine-arg` 逐个传递（引擎自身会识别并使用该标志，协调员继承的同名标志则会被舍弃，确保引擎不会收到重复参数）；`model` 和 `effort` 仅为 Muse 引擎的标志——若线程的引擎非 Muse，则视为 `usage` (2)，并指向 `engine_args`；启动器的 Muse 标志（如 `--model`、`--reasoning-effort` 等）仅由 Muse 线程继承，而 `agents` 环境变量则会传递至所有引擎；`effort` 必须是引擎支持的级别之一（`none|minimal|low|medium|high|xhigh|max|ultra`；其他任何值均视为 `usage` (2)，并列出允许的选项——若传入未知级别，引擎会立即退出，所有打开尝试均失败）；`model` 必须是引擎当前可加载的 ID：要么来自其上次启动时缓存的目录列表（位于数据根目录下的 `model-catalog/`，涵盖各提供商与配置），要么来自启动会话的 `--model` 参数；其他任何值均视为 `usage` (2)，并列出允许的 ID——引擎本身不进行校验：若无法解析模型名称，线程首次调用时即会失败，随后该线程将以 `orphaned` 状态结束且无报告；若两种来源均不可读，则 `model` 字段亦视为 `usage`，并予以注明——若留空，线程将继承协调员的设置；`kind` 默认为 `work`——若某行 `kind: follow` 且无 PR，则会被记录，但 `go` 不会打开该线程；该行的 `text` 以及后续的 `progress` 行会注明这一点，并提示使用 `follow <slug> --pr <url>` 来打开该关注线程，后者会在遇到第一行 `PR:` 时被打开（从提案状态开始，自 t=0 起轮询，直至有可关注的内容）；`owns` 是一个路径 JSON 列表（单个字符串会被当作只含一项的列表），即该线程独自修改的文件；`go` 会将其写入项目的每份简述中（对该线程显示“您拥有：……”，对其他线程显示“兄弟线程拥有：<id>：……”），因此若有两条线程都指定了 `CHANGES.rst`，系统会在冲突发生前即予以发现；`test_command` 是一条命令字符串，用于运行仓库的测试，并以“测试命令：……”的形式写入简述（若任一字段类型错误，则视为 `usage` (2)，并在任何内容写入之前报错）。每条线程在写入时均标记为 `status: proposed`。`--replace` 用于就地重写仍处于“待定”状态的线程——例如请求方更改了拆分方案，简述范围缩小——同时保留其 `proposed_at`、`attempts`，以及对于本地线程而言，若重写后指定的仍是同一分支、同一仓库（通过 `repo_root` 对双方进行规范比较，因此即使记录中带有尾部斜杠或符号链接，只要匹配即可）和同一台机器，则还会保留其 `worktree_path`（若分支、仓库或机器不同，则会丢弃该工作区，且仅在批处理写入时才会打印一行 `progress`，注明遗留的工作区）——并添加 `amended_at` 时间戳。运行于其他机器上的线程，其 `repo` 保持原样，不会在此磁盘上解析，也不会在此磁盘上保留任何工作区。线程自身的工作区，无论以 `repo` 还是 `cwd` 形式指定，均指向同一仓库。
工作线程的简述在其推送及 `PR:` 行处终止；着陆相关文字及关注行归属于 `references/roles.md` 第“实施者”部分，收据会注明着陆归属。若 `--threads-json PATH` 指向项目文件夹之外的位置（例如共享的 `/tmp/threads.json` 在几分钟内即被另一位协调员覆盖），则会读取该文件，并额外生成一条 `warnings` 记录（`kind: threads_json_outside_project`，`next` = 将 JSON 通过标准输入传递给下一级处理程序 (--threads-json -)）。警告不会拒绝任何操作；若无任何问题，则 `warnings` 字段为空。
已运行过的线程，无论是否带有相应标志，均视为 `thread_exists` (3)。`proposed` (0) 状态下，若存在线程（包括 id、名称、机器及是否被替换等信息），则 `next` = 向用户展示列表；目标清晰（仅读取一次，且在首次报告前不会执行任何不可逆或超出仓库范围的操作）→ 当前回合执行 `go <slug> <ids…>`，并向用户告知计划；否则询问是否打开这些线程，结束本轮，并对任何肯定答复（“无人值守”是一种状态，而非同意）→ 执行 `go <slug> <ids…>`（`start_threads: auto` 下的裸 `go` 命令；4 个目标回合中有 2 个自行进入 propose → go 流程）。没有任何内容会被打开。本地线程若既未指定 `cwd` 也未指定 `repo`，则会将项目首次记录的仓库（`state.json.repos[0]`，前提是该目录仍然存在）同时作为 `cwd` 和 `repo`——绝不会采用进程自身的目录（例如，从 A 仓库运行的协调员为 B 项目的线程分配了 A 仓库的工作区）——并会在 `progress` 行中注明这一情况；若项目状态早于 `repos` 记录，则仍沿用进程目录作为默认值。

## `go`

```
go <slug> <thread-id…> [--asked-by WHO]
```

打开指定的线程，每个主机管理器或舰队管理器各执行一次 `open` 操作：远程线程在收到提示之前，会先通过复制通道将其工作门记录副本下载到本地机器上（FR-43932-5：`<projects-home>/<slug>/thread-copy-<id>.json`），以便该线程首次停靠时即可进行门控；若打开失败，则会将其再次移除。正是这一副本使得线程自身的 Herdr 钩子能够向协调器报告“未知”状态，而不会向用户显示等待中的小圆点。被打开的可以是“待定”线程，也可以是“已停止”、“孤立”或“已完成”的工作线程，这些线程将以原位重新打开——保持相同的 ID、记录和检出状态（其自有分支无论脏净均可复用：脏数据即为线程自身的工作；若另有分支已检出，则仍标记为“worktree_failed”），开启一个新的会话，保留其“尝试次数”、证据及 PR 记录行，同时清空“ended_at”、“session_ended_at”、“stop_receipt”、“identity_drift”字段，加盖“reopened_at”（记录字段），并在记录中计入“reopens”次数（以及每一项“context”/“overview”记录），并在其收据上标注“reopened: true”。处于“运行中”的线程若收到此命令则仅返回“already_running”，并仅在您自行判断后才会被停止。以原位方式重新打开的“已完成”工作线程亦同（其会话于“接受”时结束；所有者规则 55，#41777）：其已接受的“报告”及“acked_digest”将被清空，因此计划视图中会再次显示 ☐，下一份报告将成为新的报告；未在 `go` 命令中明确提及的已完成线程则维持完成状态不变。路由规则（所有者规则 58）：关于某线程所负责工作的后续处理，将通过此次重新打开回到该线程——无论是其 PR、分支还是发现；用于“照看”的 PR 则由跟进线程自行创建（`follow --pr`），且只有在没有线程负责该项工作时，协调器才会亲自处理。跟进线程绝不会通过 `go` 重新打开：`follow <slug> --pr <url>` 会使其在 `max_parallel` 限制之外自行重新打开，并刷新其记录，而拒绝回复中的“next”字段会指向该线程。对于那些无 PR 可跟进的“kind: follow”待定记录，将被跳过——记录中显示“threads.<id>: skipped”，并以一行文本注明：“next”将由“第一个 PR 处的跟进线程打开：通过 follow <slug> --pr <url>”引导，其余 ID 则正常打开；若仅指定了此类记录，则此次调用被视为“empty_follow”（3，无任何线程打开，同样显示“next”）。ID 为必填项（使用说明，2，缺少时报错）：用户的 `go` 命令必须明确指定线程，不能仅凭一个单词启动线程。在任何线程打开之前，需依次进行以下检查：处于其他状态（如“退出”）的 ID 将被视为“no_such_thread”（3）；若存在可打开的 ID，而其中又有处于“运行中”的 ID，则该运行中的 ID 将被跳过——记录中显示“threads.<id>: already_running”，收据中标注“already_running: true”，并附带一行文本“<name> [<id>]: 已在运行 — 请附加：…”；其余线程则继续打开（曾有一次重试的 `go` 命令指示协调器停止了一个状态正常的线程）。若所有指定的 ID 均已在运行，则此次调用被视为“already_running”（0）：同样显示上述收据与文本，“next”将变为每条线程对应的“host-manager send <ref> --text '<line>' --type --automated”指令（对于卡住的线程，将根据您的判断予以停止，而非依据此收据）。若试图同时打开的线程数超过 `settings.max_parallel` 的限制，则会产生一条“warnings”记录（“over_max_parallel”，列出当前在线的线程及可容纳的数量），随后线程仍将打开：主机管理器的“admit”即为容量门槛（D16），且您的 `go` 命令不受本技能自身设置的约束。若使用的并非“local”机器，且其所属的舰队管理器的 `open` 命令仅返回“--help”（每次运行仅探测一次），则视为“unsupported”（4）——仅支持重命名操作的舰队管理器虽有该文件，但不提供会话相关命令。按线程逐一进行。首先，对于带有 `worktree` 的本地线程，会执行一次检出操作：`git worktree add <repo>-threads/<slug>-<id> <branch>`（若分支尚不存在，则附加 `-b <branch>`；绝不用 `--force`）；该记录的 `cwd` 即变为该路径，同时 `worktree_path` 也会记录这一信息。只有当该线程此前已打开过且记录中指明了该路径、并且该目录是一个干净的分支检出时，才会复用该目录；若目录中存在其他内容——例如同一 slug 和 id 的旧项目残留，或处于脏状态的树——或者 Git 拒绝检出（因分支已在别处检出，或父目录不可写），则该线程会被标记为 `worktree_failed`——不会为其打开任何内容，仍保持 `proposed` 状态，并记录下尝试详情（包括 Git 输出的 `fatal:` 错误行，但不包括其后的 `hint:` 提示）以及线程的 `engine` 信息；`progress` 行会注明线程在打开时所使用的引擎，失败时还会附上一段说明性文字——如果是非 Muse 引擎，则会指出问题出在 muse 二进制文件上，并在其后标注该线程实际使用的引擎名称。随后，简要文件会被组装完成（`threads/<id>/brief.md`：包含项目目标与完成标准、常规说明，以及 `## The project around you` 部分——这是助手在打开时读取的基础区块：`Repository:`；`Checkout: <cwd> on branch <branch> from <short base sha>`，每次通过一次限定的 `git rev-parse` 获取；当检出有工作树时显示 `Worktree: <path>`；当检出有远程仓库时显示 `Remote: origin <url>`；根据提案中的 `owns` 字段显示 `You own:` 或 `Siblings own:`；根据 `test_command` 显示 `Test command:`；运行于本机的线程会直接使用存储库和分支的原始拼写，不再进行 Git 探测）。接着，若 `MEMORY.md` 和 `TASKS.md` 的正文均在 2 KB 以内，则原样粘贴；若正文为空，则标注为 `is empty`（绝不针对空文件发出“请阅读”的提示：所有打开的线程都会读取这些文件）；否则，会注明条目数和文件大小。然后列出当前时刻的同级线程信息（ID、名称、状态，以及简要文件的首句，最多 200 字——绝不会展示完整简要）。最后，协调者的简要文件、助手编写的线程规则（即简要中的 `## How to work` 部分；`threads/<id>/brief.md` 中载有原文）、工作目录，以及精确的 `report` 命令——`--file threads/<id>/report.md`，报告存放在项目根目录下，而非检出目录中。随后，在主机管理器上执行 `open --name <slug>-<id> --cwd <cwd> --purpose … --prompt-file <brief> --exact-name --env MUSE_AGENTS_PROJECT=<slug> --env MUSE_AGENTS_THREAD=<id> --env MUSE_AGENTS_ROLE=thread [--env MUSE_PROJECTS_HOME=…] --env MUSE_EXPERIMENTAL_AGENTS=on [--engine-arg=--model=<m>] [--engine-arg=--reasoning-effort=<e>] [--engine-arg=<inherited>…] [--unattended]`；而在舰队管理器上则执行 `open <machine> --name … --cwd … --purpose … --exact-name [--worktree <value>] [--engine-arg=--model=<m>] [--engine-arg=…] --engine-arg=<the brief text> [--unattended]`。由于 Tmux 机器无法向提示文件发送就绪信号（舰队管理器不会发送任何信号并明确告知），因此简要文件会作为引擎的最后一个参数传递，方式与本地启动器相同，适用于所有服务提供方。`shell` 线程无论在哪一侧都不会以 argv 形式接收简要文件（主机管理器会执行 `bash '<the brief text>'`）：其打开命令中既不带 `--prompt-file`，也不含简要参数；记录中标明 `brief_delivery: file`，并注明 `brief_path`（`threads/<id>/brief.md`），`progress` 行会提及此事，由协调者手动输入读取该行的指令。其余线程则分别记录为 `brief_delivery: prompt-file`（本地）或 `engine-arg`（远程）。姿态（ADR 38715 D16，经第 6 号修正案修订）：无论是本地还是远程，请求的姿态即为线程自身的 `unattended` 设置（`true` 或 `false`——显式指定的 `false` 即为“有人值守”），若未设置，则采用项目层面的用户设定（`true`/`false`），若仍未设定（即“继承”），则采用协调者的设定——从协调会话的命令行中读取。其中，Muse 使用 `--yolo`/`--disable-approval`，Claude Code 使用 `--dangerously-skip-permissions`（或 `--permission-mode bypassPermissions`），Codex 使用 `--dangerously-bypass-approvals-and-sandbox`/`--ask-for-approval never`，均表示关闭审批；任何其他设置均视为“有人值守”，而无法读取的设置（如沙盒工具的 Shell 或未知引擎的启动器）也视为“有人值守”，并在 `progress` 行中注明。记录中的 `posture_source` 会标明所依据的规则来源。当姿态为无人值守且助手的 `open` 命令明确声明了 `--unattended` 标志时，该标志会被传递下去（`unattended_flag`）；若为“有人值守”，则不传递任何相关参数（引擎自带的提示；线程等待用户响应的提示）。对于由助手创建的检出目录中的本地有人值守线程，会优先加载针对该检出范围的引擎专用配置文件——Claude Code 的 `.claude/settings.local.json`（内部可编辑，记录中保存 `test_command`，并基于其所在分支进行 Git 操作；通过存储库的 `info/exclude` 文件将其排除在 `git status` 之外）；Muse 和 Codex 则无需此类文件（Muse 在首次提示时已应用持久化的前缀规则，且作用域为工作区；Codex 不读取目录级设置，其 `workspace-write` 沙盒已限制写入行为；相关信息会在 `progress` 行中注明一次）。这些文件被称为 `allow_list_path`；助手未编写过的文件则原样保留并予以说明；若检出目录无法容纳此类文件，则直接使用引擎自带的提示，并在 `references/allow-list.md` § Thread allow-list 中加以说明。`go` 的文本会注明各线程所采用的姿态（`threads run with my permission posture: <posture> (<why>)`，随后列出所有姿态与其自身设定不符的线程）。记录中的 `posture_applied` 会标注为 `unattended` 或 `attended`，前提是助手明确声明了相应标志（D16 规定：单纯执行 `open` 即为引擎自带的提示）；否则，会标注为 `provider_default`，并在 `progress` 行中说明（针对早于 D16 版本的助手，其默认值在此处并不明确）。每个线程也会继承协调器自身的设置（由所有者在2026年9月20日统一规定）：本地开放的`agents`门对，以及启动会话的引擎参数作为`init --detach`所携带的内容；但记录自身设置的标志位（如`model`、`effort`）具有优先权。该行的`launch_settings`字段指定了门对和来源，并在`engine_args`下记录了每个线程实际获得的参数。本地的Muse线程运行协调器自身的可执行文件，该文件通过`--engine <path>`传递给宿主机管理器（若发起当前会话的会话设置了`MUSE_BIN`，则使用该路径；否则使用解析得到的协调器命令）。`launch_settings.binary`字段命名该可执行文件，而在`binary_fallback`和一条`progress`日志中明确说明了当无法找到时将回退至名为`muse`的程序——这一过程绝不会静默进行。远程线程则保留其所在机器上原生的`muse`程序。记录会保存提供者、服务器、身份标识以及助手返回的收据，还有**原始**引用（`identity.ref`；fleet-manager 的 `ref` 是地址 `<machine>/<ref>`，助手在首次为后续的 `status`/`close` 构造该引用时会自行生成），助手给出的备注（在 `progress` 和 `open_note` 中），以及状态 `status: running`。同一个 `go` 请求中重复出现的 ID 只会打开一次。每个打开操作的 `--purpose` 都以 `[<instance>]` 结尾，其中包含六位十六进制数字，用于命名该项目文件夹（`context.project.instance`；两个使用相同 slug 的项目——即两套工作环境、同一台主机上的两位协调人——彼此不同）。当 `open` 对完全相同的名称返回 `name_taken` 时，会向 host-manager 查询该会话属于谁：如果会话处于活动状态且位于当前线程的目录下，并带有本实例的标记，则说明该会话属于当前线程自己（即之前打开但记录写入失败的会话），此时记录会将其接管（状态设为 `running`，并设置 `adopted: true`；除非该行已明确标注某种姿态，否则 `posture_applied` 保持与最初打开时一致）；若该名称下存在其他内容——例如另一个使用相同 slug 的项目，或是一个已失效的旧名称——则不会被接管或触碰：相关尝试会被记录，并且线程将以 `<slug>-<id>-<instance>` 的形式重新打开（`progress` 行会注明这一点）；该名称仅属于本项目，因此对该名称再次调用 `open` 时若仍返回 `name_taken`，则视为当前线程此前的打开请求，同样按上述方式接管。对于重试的 `go` 请求，若其自身的检出工作树已不可复用（如已被修改、处于其他分支），则会接管当前线程在该工作树中的活动会话，而非使其孤立。
`started`（0）表示所有指定线程均已打开或被接管；`partial`（6）表示部分线程已完成——`threads` 字段会按 ID 显示每条线程的状态：`opened`、`adopted`、`worktree_failed`，或底层的具体结果；任何操作都不会重试。失败打开的 `progress` 行及其 `attempts[].detail` 会记录助手给出的原因（host-manager 的 `message`，例如“车道在启动后1秒内退出”；fleet-manager 的 `error`；否则为最后一行标准错误输出），绝不会重复记录底层结果，而 `attempts[].next` 则是助手提出的修复方案（若有）。`partial` 状态下的 `next` 为首个失败线程的修复方案（`<id>: <cure>`），随后仅在同一次调用中又有其他线程成功打开时才会显示唤醒提示。当 `init --detach`、`follow` 或 `agents.py archive` 将助手的失败原样传递时，`error` 字段也会填入相同原因。`receipts`：每条线程对应一份收据，每份收据都附带 `attach` 信息。`mode_line`（host-manager 的 `mode=<x> (<why>)`：线程为何进入 tmux、Herdr 或 msp）会随 `opened as` 进度行、记录及 `go` 收据一同携带，以便协调人知晓。当打开收据的 `progress` 行显示 `msp not chosen: <reason>`（会话协议已启用，但此处无法使用 msp）时，该原因会原封不动地融入进度行——例如“mode=tmux (msp not chosen: <reason>; tmux; 无 Herdr 服务器)”——这样开启相应标志的用户就能清楚为何线程会使用 tmux（#41827）。`plan_lines`（适用于 `go`、`context`、`tick`、`ack` 和 `accept`，每次查看均包含）即用户所见的计划，可直接打印发布：每条已打开的线程对应一个 ☐，先写名称，再列出其所负责的内容（若无则显示简报首行）；一旦其完成报告被确认（即报告中包含 `PR:` 行，或 STATUS 开头为 done/finished/complete(d)/merged/pushed/landed，且摘要为 `acked_digest`——已确认的进度报告仍保留 ☐ 标记），则变为 ✅；一旦被接受（合并决策会保留自己的标记：仅在 `accept` 时将 ☐ 转为 ✅✔，因为协调人在唤醒时确认，而在归档时才接受），则进一步变为 ✅✔。尚未打开但已提议的线程仅出现在 `propose` 的 `plan_lines` 中——即在线程打开前展示的计划——而不会出现在其他场合。与其并列的 `channel_line` 是唯一一条通过 `reply` 发布该列表的消息，采用对话协调人的专属格式——`reply --to <lane> --replace-last <<'MSG'` … `MSG`，输入流中的各行（简报行中的反引号不会执行），`--replace-last` 在编辑计划的同时将其作为车道的最新消息（项目运行期间不再发布其他内容，确保其始终为最新；绝不重复发布）；车道会填充 `<lane>`（频道行：按线程列出的计划从未到达频道，而勾选后的列表则作为新消息发出；由模型生成，✅ 的重复发布在七次中有三次生效）。
本地线程若其 `cwd` 为本助手从协调人某仓库创建的工作树（`worktree_path`），且该仓库已在 `init` 时记录（`state.json.repos`），其根目录已为用户所信任，无论是在 Muse 自己的 `trust.json` 中（只读；绝非运行 `go` 的进程的 cwd），还是在本次协调会话的命令行中明确指定（`--trust-workspace` 或 `--yolo`，后者信任的工作空间即 `--workspace <root>`，若未指定则默认为协调人 cwd 周围的仓库；`--yolo` 会话的信任不会持久化，ADR 38715 第6项修正案第4条；存储中若存在显式 `untrusted` 记录，则优先于命令行设定），且共享同一 `.git` 目录，则会继承该信任及相应的钩子接受权：Muse 线程在打开时会带上 `--engine-arg=--trust-workspace`（为本次运行加载项目技能、规则和钩子），Claude Code 或 Codex 线程则由 host-manager 以 `open --trusted` 打开（引擎自带的目录级信任记录已预置，无需跳过权限标志），前提是 `open --help` 已明确提及（`trusted_flag`），否则 `progress` 行会提示引擎的对话界面仍在运行。打开成功后，记录会添加 `trust_workspace: true` 和 `trust_reason`；若打开失败，则不记录任何信任字段；被接管的会话则记录为 `null`。`progress` 行会注明：“因用户已信任该仓库，无人响应线程的信任提示。”
已安装钩子的接受则单独处理，且适用于所有本地线程（ADR 38715 第9项修正案，第64条所有者规定）：若协调人自有 Muse 设置文件存在（`$XDG_CONFIG_HOME`/`~/.config`，名为 `muse` 或 `tbh` 的 `settings.json`），且 host-manager 的 `open --help` 提及了 `--hooks-approved`（`hooks_flag`），则线程的 `open` 会携带该文件，host-manager 会在两者有差异时用其为线程的设置文件进行初始化——用户曾更改或拒绝的钩子仍会弹出提示；若未设置该标志，`progress` 行会提示引擎自身的钩子审查仍在进行。对于其启动器为其下属所有会话背书的协调人（其环境变量中设置了 `MUSE_AGENTS_LAUNCHER_TRUST=inherit`，且只有此类启动器才能设置），则会以 host-manager 的 `open --trusted` 方式打开所有本地线程、任意引擎及任意目录——利用检出本身的信任记录，使此类会话中的任何线程都不会显示任何形式的启动对话框（第65条所有者规定）。记录中会注明：“trust_reason: 属于其启动器背书的会话中的线程”。位于仓库根目录的工作线程、来自已记录但未受信或已受信但未记录的仓库的工作树、其他克隆的工作树，以及任何其他目录，则继续沿用引擎自身的提示（`trust_workspace: false`；跟随线程的规则参见 `follow`）。所有者规定日期：2026年9月20日（规范 FR-38715-9）。
每个已打开或被接管的线程都会记录 `attach`：将人类置于其会话前端的确切命令，由 host-manager 或 fleet-manager 的 `attach` 动词具体呈现（`tmux -L <server> attach -t =<name>`，同时启用服务器标志；`herdr agent attach <ref>`，针对远程线程的机器端形式），绝非在此处构造；若助手没有此类动词，则记录为 `null`，并附上 `progress` 行说明。此外还有 `attach_inside_tmux`：即同样的 tmux 命令，但在 `attach` 时加上 `switch-client` 参数（保留服务器标志；若为其他提供者的命令则为 `null`）——`tmux attach` 不允许嵌套，而 tmux 用户通常已处于某个会话内部（死胡同）。每条 `text` 行都会同时注明两种形式：调用方所在位置对应的格式（若助手环境中设置了 `$TMUX`，则为“attach: <switch-client 形式>（您已在 tmux 内部；若从外部连接则为 <attach 形式>）”；否则为“attach: <attach 形式>（在 tmux 内部：<switch-client 形式>）”），且每条线程的收据都会同时包含这两个字段。`go` 返回 `attach`（按 ID）和 `text`：每条线程一行——`<name> [<id>]: <简报首行> — attach: …`，先写名称——供用户在 `go` 后发布消息之用（所有者规定，2026年9月20日：用户无法访问的会话会被认为已消失）；每次打开时，“将 `text` 重复告知用户”都是下一个步骤——单一线程，重新打开——不仅仅是第一次。它还会写入 `library/wake.sh`（辅助脚本的唤醒循环，包含本次会话的 tmux 环境和项目主目录），并返回已就绪的 Monitor 行 `monitor_line`——即 `monitor(command="sh <project>/library/wake.sh", persistent=true, wake_delay_ms=0, show_lines=true, description="agents <slug>")`；在未启用唤醒机制的情况下，`next` 会指示按原样安装该行并将其记录下来（协调人员花了四到五次沟通才敲定这一行，将其写入共享的 /tmp，并选定两分钟的周期）。收件箱路径（`wake_path: inbox`）对此并无影响——唤醒循环和就绪行本身就是唤醒的核心逻辑（#41228）；自 `resume` 以来一直缺失的 `inbox_target` 也会在此时被重新解析。

## `follow`

```
follow <slug> --pr URL [--pr URL…] [--cwd DIR] [--unattended] [--asked-by WHO]
```每个项目对应一个关注线程（`kind: follow`——按种类查找，无论其 ID 是什么：某个提案下的 `kind: follow` 行即为该线程，并且 `follow --pr` 会在其自身 ID 下、使用其独立的工作区打开它，绝不会在提案的当前工作目录下，也不会在同一活跃线程旁再开第二个关注线程——已失效的线程会以其自身 ID 重新创建）；它负责按照 `settings.follow_every` 的频率监控并合并 PR。在本地源码库（非 GitHub）中，PR 即推送到源码库的分支，而“落地”则意味着合并到源码库的主分支；关注线程驱动这一过程，在项目文件夹下的独立克隆中工作，从不在用户的克隆中检出、合并或创建分支。协调者和工作线程本身从不执行“落地”。其合并闸门是测试套件全部通过，而非一份可接受失败的列表；它以事件为先（对首个失败的检查、评审线程、冲突以及队列作出反应），对每个提交头进行自评，仅入队一次，在目标分支上验证合并结果，并在其所关注的任务均未处于开放状态时输出报告后退出。当不存在活跃的关注线程时，`follow` 会通过主机管理器的 `open` 命令开启一个新线程，并附上关注简报——其 `## Your task` 部分以“本项目所说的 PR 及其合并（目标本身的表述）：……”开头，列出所有提及 PR 或合并的 `## Goal` 句子（即裸源码库上的分支、合并至主分支），若有此类句子（关注线程已花费 12–18 轮重新推导它们），并说明在无自动化合并的目标分支上，“落地”即由关注线程负责（逐一合并 PR，并在测试通过后通过显式引用规范推送目标分支——这是唯一一种线程会推送目标分支的情况——随后记录为“merged”，而非“enqueued”；曾有关注线程因“我可能不会推送主分支”而停滞 746 秒）——以及 PR 列表（`following`, 0, `created: true`）及其姿态：经第 6 次修正后的 ADR 38715 D16 规定，此动词默认为 `--unattended`，否则采用用户设定的项目设置，若无则沿用协调者的姿态（`inherit`；当无法读取其指令时为 attended），记录为 `posture_source`；绝不从工作线程的姿态推断（在受控项目下，每条线程都设为 `unattended: true`，却让关注线程停留在待合并提示上——因此应在决定关注线程行为时以用户声明为准；记录中的 `posture_applied` 指明实际采用的姿态，`go` 命令会提前告知：“follow_posture”，并在关注线程存在前添加一行文本“关注线程：以<姿态>开启（<原因>）”，关注线程存活后则显示“关注线程：已启动，以<posture_applied>开启”——对于 D16 之前的记录，则采用 `provider_default`；在确认其运行后，若提供方无法响应，则显示“state unknown …”；对已启动的关注线程执行 `--unattended` 不会产生任何变化，并会在进度行中注明）。此外，还指定 `--engine-arg=--trust-workspace`，为其分配独立的工作目录——即协调者首个被记录的仓库的分离检出（`state.json.repos`，无论哪个进程打开它；若该仓库已不存在，则回退至调用方的仓库，并在进度行中注明这一默认情况），该目录由辅助工具在工作线程的工作树旁创建，路径为 `<repo>-threads/<slug>-follow>`（使用 `git worktree add --detach` 创建；若该仓库已有检出，则复用现有工作树；记录中的 `cwd` 和 `worktree_path` 会命名该目录，因此 `agents.py archive` 在任务完成后会将其清理）。绝不会直接使用用户的克隆（关注线程曾在用户的克隆中合并并推送，并四次移动其 HEAD）。若 Git 无法创建任何检出（仓库尚无提交记录），则仍会在该仓库中开启，并在进度行中注明其 HEAD 可能会发生变动——除非指定了 `--cwd` 或前一条关注记录已携带了工作目录（重新开启时，只要目录仍存在，就沿用原目录；若原目录已不存在，则回退至上述默认值——首个被记录的仓库，或调用方的仓库——并在进度行中注明；相对路径会被解析为绝对路径保存，而由早期辅助工具保存的相对路径则视同已失效；状态早于 `repos` 记录的项目则沿用调用方的仓库作为默认，且不作通知）。若该目录位于用户在 Muse 的 `trust.json` 中信任的协调者仓库内（勾选或分离的协调者开启时无人可回应信任提示；开启成功后，记录中标记为 `trust_workspace: true` 并注明 `trust_reason`；若 `--cwd` 位于该仓库之外，则保留引擎的提示；若 `--cwd` 不是一个目录，则在任何记录生成前即被拒绝，错误代码为 2；工作线程仅在由本辅助工具从协调者仓库中创建的检出中才会获得此标志，详见 `go` 命令）。当线程处于活跃状态时，新的 URL 会以一条自动化的命令写入其中——由主机管理器执行 `send <ref> --type --automated`，绝不会使用对等路径，因为 Muse 只允许与已先行发送消息的会话建立连接，而本辅助工具开启的线程从未有过此类连接（`following`, 0, `created: false`, `delivered`, `delivery: typed`，收据就在这一行——通知绝非送达，参见 #38715 第 18 条裁定）。每一行 `following` 都包含 `attach`（关注线程开启时记录的附加命令）和 `text`，即专为用户准备的一行信息——ID、简报、附件——与 `go` 命令类似；`next` 指示重复此操作。无论何种情况，记录都会保留每个 URL，每行都带有 `delivered: true|false` 标记（由线程自身的 `inbox put --kind pr` 创建的行标记为 `delivered: true`——它知晓该 PR；在此字段出现前创建的行则视为已送达）。该记录即为关注线程的 PR 列表：其简报会注明记录路径，并要求在每轮开始时阅读，因此即使输入行未能送达，正在处理中的 PR 也能及时到达（在中途开启的线程中，输入的行在整个会话期间都滞留在编辑器中未被发送）。在输入之前，会先通过 `read --tail` 判断编辑器的状态：当引擎运行、对话框存在，或编辑器中已有文本时，均不得输入；否则，或在退出码非 0 时——包括并非干净的 `sent` 收据（`composer_not_empty`）、未提交的输入（`typed_unsubmitted` / `composer_not_cleared`，即按下回车后文本仍在编辑器中）——都将被标记为“已排队”（0），并附上已排队的 URL、原因、底层触发因素（本次发送的行，若有）以及未送达的行，同时指示“结束本轮”；关注线程每轮都会阅读其记录中的 PR 列表，包括唤醒循环的下一轮（`context` 或 `follow` 会在编辑器空闲时输入这些 URL，`follow_delivered` 标注已输入的 URL；若 `follow` 调用时提供了新 URL，则会一并输入已排队的 URL）；绝不能自行“落地”（单一动作；禁止手动发送、禁止亲自跟进线程，也禁止重复执行——协调者曾被提供三种方案，最终自行合并并推送主分支）。若编辑器判定结果显示某线程处于空闲状态——无活动行、无对话框——则反馈为“编辑器过期”（6）：该线程已完成工作，但其编辑器中仍有先前未提交的行，等待或重试都无法清除（三通 `follow` 调用在 14 分钟内均被告知“处于中途”）——如果该行在任何发送发生前就已存在于空闲线程的编辑器中，则情况相同（该行已在编辑器中停留 17 分钟且无法清除；此时的底层原因是本次调用自身产生的“编辑器非空”判定，即编辑器中已有文本）；`next` 会显示该行的前 60 个字符，指出主机管理器的 `send --type` 无法清除或覆盖该行，并建议先停止再重新关注（`stop <slug> follow`，然后再次 `follow`；重新开启的线程简报中会带上该 URL），同时再次强调绝不能自行“落地”。每次 `tick` 和 `context` 的重试都不会在这样的行之上添加任何内容，而是以“您的记录中已排队新 PR：<数量>；请阅读<record.json 路径>”来引导其文本和下一步行动——仅计数和指向记录，绝不包含 URL：Muse 编辑器不会提交带有 `file://` 标记的行（导致交付失败的根本原因），而记录本身就是那份列表。简报指示线程每轮都进行读取。如果屏幕显示为空，则由发送方自行作出回应。
对于未提交的回执，`delivered: true` 一律不予响应。一个处于活动跟踪状态的线程，
若其记录的 `cwd` 已不再是一个目录，则会被标记为 `follow_thread_lost`（6）：
该线程中不会输入任何内容，URL 的记录状态为 `delivered: false`，`next` 指向“停止后重新开启”；`tick` 以及 `overview`/`context` 会将此类线程显示为“已丢失——其记录的检出路径 <cwd> 已不存在”（包括 WAKE 行），而非“运行中”（因为该跟踪线程已自行移除其检出，且在行状态仍显示“运行中”期间，有两条 PR 被“交付”至其中）。在收到一条干净的 `sent` 回执后，辅助程序会再次读取一次编辑器（由主机管理器执行 `read --tail`）：粘贴的那行内容依然被标记为 `queued`，`underlying` 等于 `typed_unsubmitted`/`composer_not_cleared`（即在编辑器保留该行时已发出 `sent` 响应，导致停滞达 991 秒）。一个已消失的记录，若其自身的活动会话被重新开启所接管（`adopted: true`），则会以相同方式录入其简报中未包含的 URL——仅凭接管本身并不视为交付（当编辑器未空闲时，即使 `adopted: true`，也会保持 `queued` 状态）。跟踪线程通过 `inbox put --kind pr --key pr:<head>:<event>` 来归档 PR 事件。
不指定 `--pr` 属于“用法错误”（2）。一个已结束或已停止的跟踪线程会在原地重新开启——“结束”是针对 PR 列表而言，对跟踪线程本身并非终态（`thread_done` 只会让协调器继续构建属于自己的着陆线程）：同一记录、新会话，其 `evidence`、`attempts`、报告及 PR 行均予保留，新 URL 会被追加，`ended_at`/`session_ended_at`/`stop_receipt`/`identity_drift` 将被清空，并加盖 `reopened_at` 时间戳（状态为“正在跟踪”，0，“创建：否”，“重新开启：是”；简报仅列出尚未合并的 PR）。一个无法得到提供方响应的运行中跟踪线程会被标记为 `follow_unknown`（6）——既无重新开启也无记录，`next` 为 `tick <slug>`。一个已退出的跟踪线程（`exited` 或 `orphaned`）将以全新记录的形式重新开启（不含报告、证据及结束时间戳；`created: true`），其所承载的 PR 正好是旧记录尚未看到合并的那些，外加新的 URL。`--cwd` 指的是跟踪线程的工作目录（默认为协调器首次记录的仓库，即 `state.json.repos[0]`；重新开启时沿用前一记录的设置）。每个项目对应一条跟踪线程，且需遵守 `max_parallel` 的限制。

## `report`

```
report <slug> <thread-id> (--file PATH|-)
```

线程的报告以整体形式写入（`threads/<id>/report.md`；简报中的 `Report:` 行为 `--file -`，即从标准输入读取报告——一次 shell 调用，无需文件，线程自行生成报告，辅助工具负责保存副本——每个项目需要四次“附加并批准”操作，这些操作来自编辑工具对该文件的写入）。辅助工具会读取报告中出现的第一个 `PR: <url>` 行（无论该行位于何处；若 `PR:` 行为空，或写着 `none`、`n/a`、`-`、`no` 或 `nothing`，则视为无 PR：既无对应行，也无相应层级，更无后续提示——`PR: none` 曾在第 50 层级被视作一条 PR 记录）、最后一行 `STATUS:`、最后一行 `BLOCKED(HUMAN):`（其值为 `none`、`n/a`、`-`、`no` 或 `nothing`，大小写不敏感，末尾可带句号，或完全为空，则视为无阻塞：`blocked_line` 为 `null`），以及最后一行 `DECISIONS:`——即线程所作出但简报未予确认的决定：版本号、公开名称、不在其清单中的文件（上述无决定的值或无此行时，`decisions_line` 为 `null`；若报告早于该行出现，则按原样处理——例如，某个移除版本和三个额外模块曾未经质疑地通过）——作为一条收件箱事件（`kind: report`，键为 `report:<thread>:<digest>`），并将工作线程的 `PR:` URL 作为一条开放记录添加到其记录的 `prs` 中（跟进线程的 `PR:` 行不会新增记录：其记录仅为它所关注的 URL），同时将存在的 `## Remember` 部分移至 `threads/<id>/remember.md`——绝不会移入 `MEMORY.md`。若报告与上一次完全相同，则标记为 `reported` 并设置 `deduplicated: true`。`reported` 的字段包括：`digest`、`status_line`、`blocked_line`、`decisions_line`、`pr`、`remember: true|false`，以及 `self_reported`（最后一条 `Progress: NN% — <basis>` 行，格式为 `{percent, basis}`；若无此行，或无法解析（如百分比超出范围），则视为 `null`，静默处理：简报要求提供该信息，但辅助工具从不凭空编造；仅在制品层级下记录并显示）。对于有待决策的报告，`context` 的 `next` 字段会在确认前显示 `relay <id>'s DECISIONS to the user: …`。若线程 ID 不存在，则返回 `no_such_thread` (3)。

在收件箱路径上，该命令由线程自身的 shell 执行，随后将报告作为一条 `agents-message/v1` 会话消息发送给协调器的会话（使用 `muse session-message send --json --target <its id>`，正文从标准输入读取；`MUSE_AGENTS_SESSION_SEND` 会覆盖该命令）：首行为 `WAKE <slug>: <name> reported: <text>`（唤醒行的内容），接着是模式行，以及 `project:`、`thread:`、`kind:`、`key:`、`text:`、`report:` 等字段，再加一个空行，然后是报告原文。该行会携带 `message`——包括 `delivered`、`target`、`body`、CLI 的 `receipt`，或 `error`；若目标会话因自身原因暂存消息，则显示 `held: true`（CLI 显示为 `pending`）；若 CLI 拒绝接收，则显示 `send_with_tool`（`{tool: send_session_message, target, body}`）——当前运行时允许会话自身模型工具发送会话消息，但不允许 shell 发送（`unverified_target_receipt`、`causal_metadata_invalid`；#41210），因此 `next` 会提示“请使用您的 send_session_message 工具向会话 <target> 发送 `message.body`”，而简报内容相同的线程也会自行发送；辅助工具的尝试时限为数秒，因此回执总能在线程的工具窗口内返回。任何其他未送达的发送都会使 `next` 显示“未送达……”。无论如何，文件和收件箱事件均保留，无需重新输入，也不会重试。重复的报告则不会发送任何内容。

## `ack`

```
ack <slug> <thread-id…> [--progress NN --basis TEXT]
```协调员已读取当前报告：`acked_digest` 成为报告的摘要，因此线程会离开 `ready-for-review` 或 `waiting-on-you` 状态，直到报告发生变化。`acked`（0）；当无可确认时为 `no_report`（3）。若干标识如 `stop`（`ack <slug> a b c d` 曾是 `usage`，随后变为 `--help`，再进入按标识循环）：每次写入前都会检查每个标识——若无报告，则标记为 `no_report` 并且不予确认——该行返回 `threads`（按标识列出摘要），`ref` = `<slug>/<id,id,…>`；单个标识保持其原有形态（顶部为摘要）。

该行承载协调员对报告的反馈与处理：`decisions_line`（报告中的内容，无报告时为 `null`），若有报告则显示 `text` = `Decisions: <line>`（多个标识时每条标识一行，格式为 `Decisions: <id> — <line>`），并附带 `next` 提示“本轮向用户传达的内容：……”；随后针对每一行存在未合并 PR 且无运行中的跟进线程的情况，输出 `warnings: [{kind: `pr_without_follow`, thread, text, next}]`，其中 `next` = `follow <slug> --pr <url> — 落地由跟进线程负责，绝非您本人；您在任何克隆中均不执行 git 操作`（对于跟进线程自身已持有或已合并的 PR 不予警告）；最后以本轮结束语收尾。只陈述事实，绝不拒绝：确认无论如何都会生效。

`ack <slug> <id> --progress NN --basis "<why>"`——不得高于该行的 `artifact_rung`；较低的自我报告可能将其拉低——同时记录协调员判定的数值——单一线程、整数百分比、依据（已验证的内容，或线程自身的较低报告）——作为 `calibrated` `{value, basis, rung, at}` 返回；若数值高于线程的 artifact rung，则为 `usage`（2），提示上限及其依据（例如“`--progress 80` 高于 artifact rung 50（PR 未合并）”），并且不予确认；多个标识同时使用 `--progress`，或仅用 `--progress` 而无 `--basis`，亦属 `usage`。该确认还会写入线程的账本点（参见“追踪账本”一节）。

在输入指令后几秒内，针对一次键入的引导（`host-manager send <ref> --type`），发出 ONE 条 `read <ref> --tail`：说明面板显示的内容（是否已接收/尚无反应），绝不能停留在“已键入”状态；结果将在下一次唤醒时呈现。

## `remember`

```
remember <slug> (--text TEXT | --from-thread ID… | --decision TEXT) [--heading H]
```

在 `MEMORY.md` 中追加一条记录（格式为 `## <date> <heading>` 及正文）。`--from-thread` 会读取 `threads/<id>/remember.md`（报告中的 `## Remember` 部分），并在追加后删除该文件；若指定多个标识，则在一次调用中生成多个小节，且在追加前逐一检查（若某标识无相应小节，则为 `nothing_to_remember`，3，不写入任何内容）——返回结果包含 `blocks`（按标识去重后的标题）、`entries` 和 `receipts`；单个标识或仅用 `--text` 时则保持单块结构（另含 `blocks`）。只有协调员可进行写入；若由线程发起调用（调用方环境变量设置为 `MUSE_AGENTS_ROLE=thread`），则视为 `not_coordinator`（3），不会写入任何内容；线程的表述将保留在其报告中供协调员提取。返回 `remembered`（0），并附带 `heading`、`entries`（新增数量）及 `deduplicated`：若某区块字节完全相同且已在同一标题下存在于 `MEMORY.md` 中，则不再重复添加（`deduplicated: true`，`entries` 不变；无论是否重复，线程的 `remember.md` 均会被删除）。任务状态——问题、PR、`TASKS.md`——均不属于记忆范畴：`MEMORY.md` 记录的是项目所学，而非当前待办事项。

`--decision "<text>"` 则用于记录一项已确定的答案：在 `PROJECT.md` 的“决策”部分以及 `library/DECISIONS.md`（各线程副本）中新增一行，编号由辅助工具自动生成。返回 `remembered`（0），并附带 `decision`（标签）和 `decisions`（所有行，按顺序排列；`context` 中的 `project.decisions` 同样记录这些信息）。该选项不可与 `--text` 或 `--from-thread` 同时使用。

若 `host-manager send` 失败，则并非中转：应执行其回执中的 `next`（`--type`）；若仍失败，则说明未送达的内容，绝不能声称已转发（此内容已从 SKILL.md 第 6.2 步移至此处）。

## `relay`

```
relay <slug> <thread-id> [--fingerprint FP] (--asked | --answer TEXT) [--asked-by WHO]
```

记录一次 `BLOCKED(HUMAN):` 回答的中继（#44029）。报告问题在账本中的生命周期是独立的（§ 跟踪账本）：`blocked-unasked` → `asked-relay-owed` → `relayed-awaiting-worker` → 关闭（在工作人员下次提交报告时关闭，与之前相同）。仅执行 `ack` 操作无法使问题跨越上述任何状态：它只是读取报告，既不提问、也不回答或中继。

- `--asked` 记录协调员已向用户提出该问题一次：状态为 `asked` (0)，并设置 `relay_state: asked-relay-owed` 和 `asked_at`；此时不会发送任何内容。若问题已中继，则标记为 `already_relayed` (3)。
- `--answer TEXT`（使用 `-` 从标准输入读取）首先在账本条目上记录用户的答案，随后执行发送操作——在本机由 host-manager 执行 `send <ref> --type --automated --text <answer>`，在远程机器上由 fleet-manager 执行 `send <addr> <text> --type --automated`——并记录接收情况：`relay` = `{thread, fingerprint, answer, at, send: {outcome, delivery, receipt}}`。仅当满足 `deliver_to_follow` 的条件时才会被视为已送达：退出码为 0 且存在收据，并确认提交成功——在本机表现为发送后再次读取，此时提交者已不再持有答案（按下的回车键不代表送达），而在远程机器上则由 fleet-manager 自行验证发送后的结果：其判定为 `submitted`（`submitted: false` 表示未送达）；此时状态变为 `relayed` (0)，`relay_state: relayed-awaiting-worker`；问题将保持开放，直到工作人员下次提交报告将其关闭。整个检查→记录→发送→记录的过程都在项目锁的保护下进行，因此两个并发的中继操作不会同时提交同一份答案。
- 发送失败时记为 `relay_failed` (6)：条目仍保持 `asked-relay-owed` 状态，记录下答案及失败的发送信息；`next` 将显示未送达的内容，并提示重新尝试 `relay … --answer`（答案以 shell 引号括起；无需再次询问用户，因为答案已记录在案），而 `context` 中的 `next` 以及 WAKE 行会持续标注该中继为待处理，直至送达为止。
- 若线程无未决报告问题，或指定了不存在的 `--fingerprint`，或为 `dialog:` 类型的指纹（屏幕对话直接在屏幕上回答，从不中继），则视为 `no_open_question` (3)，不会发送任何内容。若未指定 `--fingerprint`，则默认针对该线程的唯一未决报告问题。
- 只有经记录的协调员才能执行中继操作（否则为 `not_coordinator`, 3，即 `accept`）。

在中继待处理期间，`context` 中的 `next` 会始终列出该中继（`relay owed for <id> (<fingerprint>): …`），即使报告已被确认且无其他进展——绝不会出现“无进展；结束本轮”的情况——并且 `tick --wake-line` 也会为尚未被报告段落提及的线程列出 `<Name> relay owed: <question>`。`needs_you`（在 `context` 和 `overview` 中）以及状态表中的 `Needs you:` 行都会携带该条目的 `relay_state` 信息。

若会话是在此辅助工具之外开启的——例如通过原生 `agentcloudctl` 或 host-manager 会话且无线程记录——则不存在待处理的问题、唤醒通知及中继任务：`relay` 会拒绝此类请求（`no_such_thread` / `no_open_question`）。协调员不得将此类会话视作受管理的线程；应在需要提问和中继时通过 `propose`/`go` 来开启会话。

## `accept`

```
accept <slug> <thread-id…> --evidence TEXT… [--asked-by WHO]
```在协调者验证的证据上将线程标记为“已完成”——例如已合并的拉取请求URL、目标分支上的提交、制品路径或测试运行。对于多个ID，所有这些ID共用同一条证据记录，比如`stop`：在写入任何内容之前，会先对每个ID进行判断（未知ID视为`no_such_thread`，已完成或从未运行的ID则按其拒绝状态处理，不写入任何内容），响应中携带`threads`（每个ID对应的`evidence`和`decisions_line`）以及`receipts`，而`next`则传递每个线程的决策；单个ID保持如下结构（外加`threads`）。若未指定`--evidence`，则返回`usage`（2）：即仅声明“已完成”的报告。
`accepted`（0）包含`evidence`和`decisions_line`（若有证据，则为报告内容；若无，则为`null`）；收据记录了接受者及时间，而`decisions`则记录线程自行作出的决定——此时`text`为“Decisions: <line>”，且`next`以“本轮向用户告知：……”开头。若不存在正在运行的跟进线程来承载该线程的未合并PR，则会附加一条`pr_without_follow`警告及其`follow --pr`指令（与上述`ack`的格式相同）：无论如何，接受都会生效。
当所有线程均处于其他状态时，`next`为“所有线程已完成：归档<slug>，然后结束本轮”；否则，在结束本轮的语句之后，会列出仍处于开放状态的线程（`<id> (<status>)`，包括跟进线程）。
对于远程线程，响应还包含`copies`（FR-43932-5）：每个已接受线程的副本处置情况——`removed`（已移除）、`tombstone-left`（副本在移除前被重写为`status: accepted`，但移除失败；墓碑已解除面板的阻塞），或`tombstone-failed`（两次写入均失败；副本将继续阻塞，直到其戳记超过45分钟，此处会明确说明失败原因，绝不沉默）。
`accept`以与`stop`相同的方式结束已接受线程的会话（主机管理器执行`stop`，车队管理器执行`close`；记录中包含`stop_receipt`和`session_ended_at`；每行`threads`及收据中均有`session_ended`，并在`false`时注明`session_ended_why`——“无有效会话”、“陌生人的身份漂移”或提供方的拒绝）；克隆、分支、记录和证据均保留，而`go <slug> <id>`则原地重新打开该线程（参见`go`；所有者裁定55，#41777）。提供方的拒绝不会导致取消接受：`accepted`仍为0，`session_ended: false`，拒绝信息位于该行的`stop_error`字段下（`outcome`、`error`、`next`），且`next`以“stop <slug> <id…>”开头。
对已完成线程的第二次`accept`被视为`thread_done`（3），且不追加任何内容（重新打开的线程会被再次接受，证据则追加）；从未运行的线程（`proposed`）则被视为`not_started`（3）。每次接受操作都会写入该线程的最终账本记录——`accepted`，100，以及其旁的证据（参见“追踪账本”）。

## `inbox`

```
inbox put <slug> --kind KIND --key KEY [--thread ID] [--text TEXT] [--json JSON]
inbox put <slug> --message (PATH|-)
inbox drain <slug> [--ids ID…]
````put` 命令会在其键下记录一条事件（`filed`, 0, `created: true`），或在该键已被见过时记录一条“重复”事件（0, `deduplicated: true`），无论是在 `new/` 还是 `done/` 目录中。`--thread` 参数会在任何写入操作之前被解析：对于未知的 `--thread`，会记录 `no_such_thread`（3）而不会写入任何事件。带有 `--thread` 的 `pr` 事件会更新该记录的 `prs[]` 字段（`last_event`, `head`）。你自己的 `context` 已经会读取（清空）它所返回的事件，因此唤醒操作无需再调用 `drain`；而其他人的 `context` 则不会读取任何内容，因为其自身的游标已经保证了这一点。`drain` 用于手动清除你希望清理的事件，以及当调用者并非已记录的协调人时的情况。`drain` 会返回待处理的事件（全部或指定 ID），并将它们移动到 `inbox/done/`——即协调人的“已处理”目录（`drained`, 0, `events`）。空的收件箱会被标记为 `drained`，且 `events: []`。行首还会显示 `unacked_reports`（最新报告既未被确认也未被接受的线程 ID，按最新优先排列），如果有未确认的报告，则 `next` 会打开 `<n>` 条未读报告——`<Name> [<id>]，…：请阅读 threads/<id>/report.md，然后进行确认……`——一次 `drain` 操作只会清空收件箱，而不会影响你的阅读进度；未读报告仍会保留在 WAKE 行上（四条报告被一起清除了三条，第四条再未被提及：#38715）。

`--message` 命令会将通过会话传递到达的消息（ADR 41038 D1：此处无文件夹的线程）记录在其自身 `key:` 下：`filed`（0；对于 `report` 类型，标题之后的正文会被写入 `threads/<id>/report.md`，并按 `report` 的方式读取，同时在行首标注 `digest`），或在该键已被见过时记录为“重复”（0, `deduplicated: true`）——同一份副本被传递两次只记录一次。如果消息体不是 `agents-message/v1` 格式，或是针对其他项目的，则会被标记为 `usage`（2）；若消息对应的项目不存在，则为 `no_such_thread`（3）；无论哪种情况都不会写入任何内容。在收件箱路径上，`put --kind pr` 命令若其键以 `:merged` 结尾，还会将该事件发送至协调人的会话（行首标注 `message`）；其余的 PR 事件则属于跟进线程本身，不会触发任何唤醒。

## `tick`

```
tick <slug> [--arm monitor|scheduler|passive [--command CMD] [--one-shot] [--monitor-failed LINE]] [--disarm] [--asked-by WHO]
```

这是协调人的节奏机制，可安全地由定时器触发执行：- 刷新每个在线远程线程的 worker-gate 副本戳记
  （FR-43932-5：本轮以30秒为周期的 `coordinator_seen_at`，
  通过副本通道原地更新——活跃协调器按周期而非关注来控制其工作线程；已失效机器的副本则自然老化，且每次心跳不会在某个副本上停滞）；
- 通过其提供者刷新每个开放线程的存活状态；报告后消失的线程变为“退出”，未报告即消失或身份不匹配的线程变为“孤立”，每次状态变更都会记录一条事件
  （`thread:<id>:<status>:<opened_at>`），因此重复的心跳不会产生新记录。不再出现在远程会话 fleet-manager 中的会话（可达机器上的 `status` 显示为 `no_such_session`）视为已消失；其在线身份（`cwd`、`engine` 等）与记录不符的，则被视为该线程名下的陌生人——对该线程而言已消失（记录中标记为 `identity_drift`），从未停止；无法响应的机器会使该线程状态未知；
- 按照 `context` 的方式读取每个本地活跃线程的判定结果（Herdr 自身的状态，或主机管理器通过 `read --tail` 得出的判断，以及活动标记），并在线程变为“等待您”时记录一条事件
  （`thread:<id>:waiting-on-you:<stamp>`，附带其附加命令文本），因此对于对话或权限提示，被监控的 `text` 也会相应变化一次；记录中的 `last_agent_status` 将此类变化限制为每次状态转换仅一次；此外，当一个活跃线程连续两轮均显示“空闲”（无任何运行任务；Herdr 或 MSP 提供者的“空闲”状态），而协调器仍未收到报告时，也需记录一次——若无新报告，或最新已确认但未完成的报告，则记录为 `thread:<id>:idle:<spell>`（记录中空闲时段计数），文本为 `<id> 在无报告情况下进入空闲——其屏幕上可能有待处理的问题，请及时查看”，并附上附加命令，每出现一次空闲时段即记录一次（记录中的 `idle_rounds`；一次空闲回合即指全新 TUI 的空白编辑器，不算新消息）。如果线程屏幕上输入了问题，或线程在未报告的情况下停止，则此前一直处于静默状态（AUDIT-AGENTS-WAKE，#38715：四分钟的 `--wake-line` 输出为空，而问题仍停留在屏幕上）；
- 写入追踪账本（参见“追踪账本”部分）：本轮已掌握的信息（记录、存活判定、报告、PR 事件）为每个运行中的线程记一点，状态发生变化的线程记过渡点，并同步工作流及待处理问题；唤醒循环的 `--wake-line` 心跳也算一轮。无法读取的账本会在行中显示为 `tracking_error`，并伴随一条 `tracking: …` 进度行；本次心跳本身的工作仍会被记录；
- 每轮，包括 `--wake-line` 在内，只要某线程的编辑器为空，就将该线程在 follow 记录中排队的 PR 打印一次（参见 `follow`；JSON 行中标记为 `follow_delivered`，唤醒行无内容：送达不算新闻）；
- `--wake-line`：唤醒循环的模式（`library/wake.sh` 每30秒执行一次）；
  每个待处理事件都以 `<name> reported: <text>` 的形式呈现（PR 事件为 `<name> pr <event>: <text>`），名称来自线程记录，键值保存在 JSON `inbox` 中——JSON 心跳的 `text` 行格式为 `  - <name> [<id>] reported: <text>  (<key>)`——不含 JSON；一行打印 `WAKE <slug>: <segment>[; <segment>…]`，事件片段按最新顺序排列，不分种类——`<name> reported: <text>`、`<name> pr <event>: <text>`、`<name> thread: <text>`（如线程变为“等待您”）——随后是 `unknown: <ids>` 和 `<name> lost: …`，以及每个线程的最新报告（同一线程的旧未读报告仍留在 JSON `inbox` 中，不在行中：首次摘要主导每期 WAKE，后续报告才是新闻，超过80列的省略号之后的内容除外；JSON `tick` 不做筛选），有新闻时如此，无新闻时则为空，新闻持续期间内容一致，因此循环的最后一行守护机制会在相同新闻再次出现时重复打印；记录本身也是新闻：每个最新报告既未被确认也未被接受（`ready-for-review`、`waiting-on-you`）的线程，在每次心跳时都会以 `<name> reported: <text>` 的形式出现，直到被确认/接受为止，无论其事件是否仍在收件箱中（接受会缩减行数，因此循环会再次唤醒您查看剩余内容——清空但未读的报告绝非静默）；每线程一段：其“reported”文字优先于“moved”监控事件（准备评审或阻塞状态的转换会重复该报告），否则仅显示最新的“moved”事件，PR 事件则保留；行中最多容纳 `WAKE_LINE_BYTES`（500字节）——最新片段优先，超出部分显示为 `+N more unread`，确保线程不会因超出折叠位置而被无声丢弃（JSON `tick` 保留所有事件完整）；如果循环完全无法运行（退出码126/127），则打印一行 WAKE 表明原因；对于已归档或未知的项目，不输出任何内容并以退出码0结束（在文件夹消失前安装的循环不应将拒绝视为新闻）；关注线程针对所关注的 PR 提交的 PR 事件，在未显示“已合并”之前不算新闻——检查、评审、冲突及队列均由关注线程自行处理（五分钟内三次唤醒针对未读数量），而 JSON `tick` 仍会列出该事件。与其配套的 `--arm`/`--disarm` 属于使用说明；
- 使用不运行该项目自有 `library/wake.sh` 的命令执行 `--arm monitor` 时，会记录该臂状态，并新增一条 `warnings` 条目
  （`not_the_ready_line`：它可能永远不会唤醒您，而“准备就绪线”是其“下一站”）。辅助工具不模拟该线路：准备就绪线由 `go`、`context`、`tick` 及脚本自身头部打印，手写过滤器的判断由您自行负责；
- `library/wake.sh` 代表每个项目一个循环：它读取 `library/wake.lock/pid`，同一项目的第二个循环（第二台 Monitor 安装）保持沉默，仅在该处的 pid 消失时接管；一旦项目文件夹消失（已归档），所有相关循环，无论发声与否，都会在一次心跳内以退出码0结束，从而让 Monitor 自动终止（所有者裁定26，#38715；取代了“常驻循环”：三台 Monitor 意味着每期 WAKE 被唤醒三次）。`sh wake.sh --once` 可手动执行一轮。当记录中的臂状态对应的是一个循环 pid 仍然存活的 Monitor 时，执行 `--arm monitor` 会返回 `already_armed`（退出码0，不重新记录），并显示 `wake`（当前臂）、`loop_pid` 和 `next`，提示无需干预第二台 Monitor——其循环本身处于静默状态，停止操作只会增加一条记录；臂在循环启动时记录 `loop_pid`；
- `--arm` 记录谁在唤醒该项目：`tier`、`command`（对于 `monitor`，指协调器安装的 Monitor 线路；对于 `scheduler`，指单元或 cron 线路）、`persistent`、`armed_by`（该协调器）、`armed_at`；`--disarm` 则清除这些信息。记录并非调度——协调器自行安装 Monitor 或 Scheduler 条目，并在此处记录，以便 `context` 和 `resume` 能够判断是否存在唤醒及其归属；未指定 `--command` 即执行 `--arm monitor` 或 `--arm scheduler` 属于使用错误（退出码2，不记录）——没有对应条目的臂不会唤醒任何人；未附带 `--monitor-failed "<one line copied from the failed monitor( result>"` 即执行 `--arm passive` 亦属使用错误（退出码2，不记录；`error` 标明 Monitor 工具及准备就绪线，“next”提示应致电联系，随后再记录 `--arm monitor`）：被动绝非首次尝试——Monitor 才是真正唤醒您的工具，而未获呼叫的被动臂会导致所有报告无人处理（所有者自行运行；裁定52，#38715）。
  掌握该线路后，可在层级旁记录 `wake.monitor_failed`。Monitor 默认为持久型：若 `--command` 行注明 `persistent=false`，则属于 `arm_not_persistent`（退出码3，不记录），除非 `--one-shot` 明确表示只需一次唤醒——定时 Monitor 在窗口期结束后即告终止，再次启用时会重播源端自始至终保留的所有内容；重播内容携带首次出现时的键值，因此收件箱仅记录一次（dup`licate`)，且不会重复读取任何 `go`；持久化规则仅读取该行自身的 `persistent=`，从不读取带引号的内部文本；
- `ticked`（0）包含 `changes`（按线程：from、to）、`inbox_pending`、`inbox`（每个未读事件：key、kind、thread、text）、`text`（先列出未读事件，再列出变更，最后是唤醒行——即 Monitor 监视的那行：它仅在收件箱或某个线程发生变化时才会改变，无论上一次 tick 是由谁执行的，因此线程的报告在其首次被看到时即被视为一次变更）以及 `wake`；`next` 在有未读内容时为 `inbox drain <slug>`；当 arm 状态发生改变时，会生成一条 `receipt`（内容为：arm|disarm）；
- arm 还记录 `means`（该层级为协调者会话所执行的操作：Monitor 会唤醒它；调度器会在另一个进程中运行 `tick`——文件夹会刷新，会话不会被唤醒，而会在用户发送下一条消息时继续处理；被动模式则不执行任何操作）、`env`（arm 执行时所处的环境变量：包括 `MUSE_AGENTS_TMUX`、`TMUX_TMPDIR` 和 `MUSE_PROJECTS_HOME`）以及 `tick_command`（调度器应执行的命令行：即在上述环境中运行 `python3 agents.py tick <slug>`），并在该行中返回 `tick_command`。后续调用者若未设置 `MUSE_AGENTS_TMUX` 或 `TMUX_TMPDIR`，则会通过一条 `progress` 行从 arm 中获取这些值，因此在纯调度器环境下执行的 tick 将访问同一 tmux 服务器，而非空目录；若调用者指定了自己的 tmux，则会予以尊重。无法响应的 tmux 会使线程状态保持未知——绝不会消失：同样，如果 `status` 回答显示“不可达”、携带 tmux 连接错误、缺少布尔型字段 `live`、并非有效的 `status` 行，或在结果之外还附带了 `error`（例如辅助工具无法读取的列表，或非 JSON 格式的响应；只有来自可达提供者的有效且可解析的非 live 结果才会更新线程状态），或者无论是否 live，其回答均来自与记录中指定不同的 tmux 服务器（在没有 `MUSE_AGENTS_TMUX` 且无 arm 可填充的情况下，纯环境会尝试连接默认服务器；无论该服务器中存在什么内容，包括同名的陌生服务器，线程原本所属的服务器从未被查询过，也不会读取任何来自陌生服务器的信息）。此类线程会被列于 `unknowns` 下，并附带 `unknown_reasons`，既不记录也不归档，且 `text` 中会包含一行 `unknown:`，标明用于访问其服务器的 tick 命令（即 arm 的 `tick_command`，若本环境的 `MUSE_AGENTS_TMUX` 不存在或指向其他服务器，则使用基于已记录服务器信息合成的命令——绝不会重复使用刚刚失败的探测命令）；
- `--arm` 和 `--disarm` 仅限由记录中的协调者执行（否则提示 `not_coordinator`，并返回 3，且不作任何记录）；而普通的 `tick` 则对任何人开放。同样的缺失循环事实适用于每一个不是循环自身 `--wake-line` 的 `tick`：`text` 以 `wake: 记录时监控已就绪，但没有循环在运行` 开头，而 `next` 是 `立即调用 monitor(<所记录的命令>)，然后结束回合`（在宽限窗口内有一条由该臂发出的调用）。先前在一条被动臂上出现的提示 `先尝试 monitor(<就绪行>)；调用失败后才被动` 已被弃用：该臂改为直接拒绝（见上文），手中保留着那次失败调用的行。

## `overview`

“状态？”→ `overview <slug> --table`，原样显示，从不以项目符号形式呈现；一行 `flat` → ONE `host-manager read <name> --tail`，在该线程的行中说明，不在其他唤醒时扫描（从 SKILL.md 第6步移至此处，包括 `ack` 限制和 `init` 消息规则）。
仅对触发器进行深度读取——忽略过去的 `stuck_scans` 和存疑的自我报告，用户在 `accept` 之前会询问：“read <ref> --lines 200”一次，而不是每次唤醒都读取（从 coordinator.md § 关于“进入空闲”条目的字节空间一节移至此处）。

```
overview <slug> [--table]
```

各组以文本形式呈现，每个线程在其所属组标题下占一行，顺序为：`waiting-on-you`、`ready-for-review`、`working`、`landing`、`idle`、`orphaned`、`done`、`proposed`，最后是 `unknown`（其提供者未回复的线程：机器、原因、最后已知状态），以及收件箱计数和唤醒臂。`overview`（0）包含 `groups`、`unknowns`、`text`、`table`（状态表，参见“追踪账本”）和 `needs_you`；`overview <slug> --table` 将 `text` 转换为表格——即对“状态？”问题的原样回答。不存在 HTML 视图。

## `pick`

```
pick <slug> --question "<q>" --option "<label>=<path>" [--option "<label>=<path>" …]
```在两篇文本之间做出选择（两份倡导说明、候选人名单，以及一行 `DECISIONS:`），打印后即可发布：消息部分为一句导语，列出标签名称——“在 A、B 两篇文本中二选一，每篇在其选项预览中完整呈现；请在对话框中作出选择。”——并附上文件路径（当预览被截断时）。对话框是承载主体（在三分之二的运行中，消息仅保留这句导语，而预览则完整呈现两篇文本；在其余三分之一的运行中，消息与预览均完整呈现）。随后是一行 `--- dialog ---` 和一个 JSON 对象：`ask`（即 `request_user_input` 的参数对象，包含一个问题，字数限制为 500 字，为该工具的上限，并附有选项）以及 `next`，两者均不重复。每个选项都带有其 `label`（限制为 60 字），且除非文件为空，否则其文本会在该工具的各选项字段中重复出现两次：`description`（文本以单行形式呈现，限制为 240 字）和 `preview`（格式为 markdown，内容限制为 2000 字）。截断的描述以 `…（完整文本见预览）` 结尾，截断的预览以 `…（已按对话上限截断）` 结尾——绝不会标注“见上方消息”：模型在三轮中均跳过了此类提示。
先发布导语，然后以分隔线下方的 `ask` 对象作为参数调用 `request_user_input`，完全按照打印内容——`{"questions": [{"id": "pick", "header": "Pick", "question", "options"}]}`，其中 `questions` 是一个数组，绝非 JSON 字符串（模型曾重建调用，将 `questions` 作为字符串传递，导致重试时预览被丢弃）。描述和预览绝不改写：用户在对话框中阅读每篇文本，从不只看标签，也从不在选择后才读文本（#41037：选择是在对文本一无所知的情况下进行的，文本当时并未出现在对话框中；#41227：模型执行了该指令并发布了一条导语，下方却无任何内容，因此文本从此被纳入对话框）。相对路径位于项目文件夹下（如 `library/a.md`）；绝对路径则按原样使用。每次提问最多可设置两个或三个选项（#41234：这是该工具的上限）；第四个选项为 `usage`（2），明确指出这一限制：仅在两轮中提问。除此之外，不再拒绝任何操作：无法读取的文件将在其选项中打印 `(missing: <path>)`，并在标准错误输出中添加一行 `warning:`，无论哪种情况，程序均以退出码 0 结束。JSON 封装本身不受影响：标准输出包含消息、分隔线及 JSON 数据；即使没有项目文件夹，只要提供绝对路径，仍可正常运行。
  
## `set`
```
set <slug> mode herdr|tmux|msp|auto [--asked-by WHO]
set <slug> engine muse|claude|codex|auto
set <slug> every 2m|90s|quiet|auto
set <slug> heartbeat 5|1h|off
set <slug> sink "<command>"|none
```
`mode`：项目的模式固定值（ADR 41038 D3 规则 1）：后续每次 `go` 调用都会将其作为 `open --mode <x>` 传递给主机管理器，由其使用或给出拒绝理由（`mode_unavailable`、`mode_unreachable`），且绝不代为替代；`auto` 则移除该固定值，由系统自动决定（Herdr 在其服务器运行时启用，否则使用 tmux；`msp`——需配合 `TBH_AGENTS_SESSION_PROTOCOL`——仅在显式固定或由主机指定时启用，或在本机支持 MSP 时启用；固定值会覆盖上述两种情况，且取消固定标志的行为会被拒绝，并注明原因）。该值应以 `mode: <x>` 的形式写入 PROJECT.md 文件的 `## Settings` 部分，与其他设置并列，仅需设置一次。正在运行的线程会保持其启动时的模式（`running_keep_mode`）；项目在运行过程中不得切换模式。若用户主动要求，请在一行内说明该固定值；固定值代表用户的意愿，而非您的判断。
`engine`（#43739，规范 38715-agents FR-43739-4）：项目的默认工作引擎——当用户指定工作引擎时，`init --engine codex` 会写入该设置（“让 Codex 工作器来执行”）；之后可通过 `set <slug> engine claude` 来固定或更改；使用 `auto` 可移除该行。在 `## Settings` 下以 `engine: <x>` 的形式书写。若提案帖未指定 `engine`，则沿用默认值；但该帖自身的 `engine` 设置仍具优先权；若无此行，则该帖将运行协调器的默认引擎，否则为 Muse 引擎（QA r10 ENGINES D1）。已提交或正在运行的帖子则保留其原有设置。host-manager 会像对待任何帖子一样应用该引擎自身的配置与信任等级；Muse 标记（`model`、`effort`）仅限 Muse 使用。

`every`、`heartbeat`、`sink`（#41802）：观察者的各项设置，位于同一节中（`every: 120s` | `every: quiet`，`heartbeat: 5`，`sink: <command>`），已在运行的观察者会在每次休眠前重新读取这些设置——无需重启。“every”决定了“仍在工作”状态的更新频率（“每 2 分钟更新一次”→ `2m`；“频率更低”→更大的数值；“静默直至完成”→ `quiet`（`off` 与此同义）；`auto` 会移除该行，并由节奏表决定）；低于 10 秒被视为“usage”。“heartbeat”不仅按设定的节奏触发，还会每隔 N 分钟向 `watch` 收件箱发送一条事件，方便希望定期查看状态的 TUI 用户；默认关闭，`off` 可将其禁用。“sink”是每次渲染后的列表所输出到的命令（参见 `watch` 部分）；`none` 则表示不进行输出。相关操作通过 `set` 命令执行，需提供 `setting` 和 `value`（`auto`/`off`/`none` 后设为 `null`），并返回确认信息；也可用于设置其他参数或值（`usage`（2））。

## `watch`

```
watch <slug> [--sink "<command>"]                                  # 该项目的所有帖子（即 `go` 启动的内容）
watch <slug> --source cmd --step "<label>" [--sink "<command>"] -- <command…>   # 自定义的一个长步骤
watch <slug> --stop
```

项目的观察者：每个项目仅有一个长期运行的子进程，由 `go` 启动（其 PID 记录在 `state.json` 的 `watch` 字段中，并通过启动时间戳（Linux 下 `/proc` 中的 `lstart` 字段）与进程绑定——若 PID 被复用，则视为其他进程，不会被信号唤醒，直接视为已终止；第二次 `go` 会继续维护它；若在帖子运行期间该进程意外退出，下次 `go`、`context` 或 `tick` 时都会将其重启——包括唤醒循环的一轮；该行会记录 `watch_restarted`），最终由 `agents.py archive`、`--stop` 或自身在所有行都变为终端状态（已完成、已接受、失败或已停止）时结束。它本身不会主动唤醒您。每隔 **20 秒**（`WATCH_POLL_S`），它会读取项目自身的记录以及每个正在运行的帖子的会话信息——与 `context`/`overview` 所计算的数据相同——并为每个已打开的帖子渲染一行：

```
☐ 运行中 · ⛔ 阻塞中（事实即问题） · ✅ 已完成（报告已确认） · ✅✔ 已接受 · ✖ 失败（会话及检查点均已丢失） · ⛔ 已停止
<标记> <名称> — <其所拥有的内容> [· <已耗时>] [· <一条事实：报告的 STATUS 行，若无则为最新提交]]
```

（这些标记与 `plan_lines` 相同，因此您的回帖与观察者编辑的内容会合并为同一列表；尚未提交的提案帖则不在其中，如同 `plan_lines` 一样。）

两个时钟。两次采样之间的**状态转换**（运行 → 阻塞、→ 完成、→ 失败，……）会一次性重写列表并重启节拍间隔；进入阻塞/完成/已接受/失败/已停止状态，或从阻塞状态退出，还会记录一条 `watch` 收件箱事件（`watch:<id>:<state>:<n>`，文本为 `<from> → <to> — <fact>`），唤醒循环会将其打印为 `<Name> moved: …`——监控器会在收件箱路径上同时发送该消息，从而唤醒你。运行中某行的最新报告若尚未被协调者确认，则会触发一次“待审核”状态的转换：每条摘要对应一条事件（`watch:<id>:ready-for-review:<digest>`，文本为 `running → ready-for-review — <STATUS>`），本身不作编辑（下一次节拍重写会携带该事实）；若某行在同一次采样中状态发生变化，则仅记录该次转换事件。启动批次不记录任何内容，对接收端而言也无新意：你刚刚打开了它们（无法折叠编辑的通道对此不发布任何内容）。否则，按照节拍表（`WATCH_CADENCE`）进行一次“仍在工作”的重写，以自观察开始以来的时间为键，“every”可覆盖该表：

| 经过时间 | 每隔多久编辑 |
| --- | --- |
| 少于10分钟 | 1分钟 |
| 10–30分钟 | 2分钟 |
| 30–60分钟 | 5分钟 |
| 1–2小时 | 10分钟 |
| 超过2小时 | 15分钟 |

列表的存放位置：始终为 `library/status.md`（`# <slug> — updated <t>`，随后是各行；`context`/`overview` 返回 `watch` = `{pid, status_file, updated_at, every, heartbeat, sink}`）；若使用 `--sink`（否则采用 `sink` 设置），则还会将数据送至该命令的标准输入，每次重写调用一次——通道的计划消息按 ID 原地编辑（由创建该项目的程序设置的通道自身接收命令）。接收端仅编辑协调者的计划帖：在该消息不存在时，返回带有空 ID 的 `skipped: no_plan`，观察者只记录一次等待，并由状态文件单独维护列表——观察者从不创建计划消息，也从未编辑过任何确认信息（#41959）。接收端契约：标准输入为渲染后的各行；argv 接收 `--message-id <id>`（来自其自身的上一条回复）、`--news` 表示状态转换批次、`--row` 用于 `--source cmd` 步骤（仅重写一行，而非整个区块）、`--stamp` 用于不作编辑的回复；标准输出为单行 JSON `{"message_id", "edited_at_ms"}`；非零退出码仅记录一次，且绝不停止观察；若返回 `{"outcome": "refused", "reason": …}` 并退出码为 3（不适用：目标为卡片），则仅学习一次——不再进行后续接收调用，状态文件继续维护列表。**退避机制**：在节拍重写之前，观察者会向接收端请求戳记；若收到比自身上次编辑更新的戳记（你已勾选或改写了列表），则跳过本次勾选并重启间隔；若无戳记，则不退避。`--source cmd` 则改为观察你自己的一个命令（其行为 `☐ <label> · <elapsed> · <last log line>`，退出时显示 `✅ <label>` 或 `✖ <label> · exit <n>`，`--stop` 时显示 `⛔ · stopped`；输出被分流；退出码即该命令的退出码；无收件箱事件——步骤本身的完成会唤醒通道）。

结果：`watch_ended`（0；`ended`：完成 | 失败 | 已停止，包含 `edits`、`transitions`、`events`、`elapsed`）——JSON 行为观察者最后一次标准输出，位于 `library/watch.log` 中，记录的是 `go` 启动时的情况；对于已在运行的观察发起第二次 `watch` 时，返回 `already_watching`（0，包含 `pid`）；对于 `--stop`，返回 `watch_stopped`（0，包含 `pid`）或 `no_watch`（0）；对于未指定命令的 `--source cmd`，返回 `usage`（2）。观察者不撰写任何文字描述：仅记录标记、名称、所负责的内容、经过时间以及来自源的一条事实；当你清醒时，你来解释其含义。

## `resume`

```
resume <slug> [--takeover --confirm "<the human's words>"]
```身份（`MUSE_LANE_BACKEND` 或 `MUSE_LANE_REF`）属于项目自身线程记录之一的会话，即为“协调者是线程”（3，已命名线程，未记录任何内容，无论是否使用 `--takeover`）：协调者绝不会是其下属线程。一个新的协调者会话会从文件夹接管该项目。它首先会询问前一个协调者是否仍在运行——通过其提供者确认，在 Linux 上通过 `/proc` 获取进程身份，其他系统则用 `ps` 命令（获取其 `pid` 和 `started` 时间戳，因此复用的 `pid` 不视为活跃的协调者）：若在运行，则为“协调者活跃”（3）——两个协调者同时存在会产生竞争；而其提供者无法回应、作为 tmux 窗格且服务器无法判断，或为“不透明”的沙盒化协调者，则为“协调者未知”（6）——它仍可能处于活跃状态——除非使用了 `--takeover --confirm` 并得到人工确认；对于已消失的进程身份，会输出一条“进度”信息；而对于位于其他主机上的进程（或由较旧的辅助工具写入且无 `pid` 的情况），则无法验证，并同样以“进度”信息告知。因此，同一主机上的第二个会话绝不会意外成为同一个协调者。随后，它会按身份核对所有打开的线程（未报告即消失或不匹配 → “孤立”；已报告后消失 → “退出”；仍在运行 → 保持不变；提供者无法响应则保留原状并列入“未知”列表），清空唤醒臂（与前一会话绑定的监控器不属于当前会话——需用 `tick --arm` 重新设置），将本会话记录为协调者，并返回与 `context` 相同的状态视图：“已恢复”（0），包含 `threads`、`unknowns`、`previous_coordinator`、`wake: null`、`previous_stopped`。若要接管一个由 `init --detach` 启动且处于活跃状态或无法验证的协调者，则会在记录本会话之前先停止该会话（`previous_stopped: true`；若停止失败，则直接跳过且不记录，以便重试时仍能找到；若在无法响应的探测后返回 `no_such_session`，则显示“在此服务器上未找到”——主机管理的 `stop` 是在调用方的服务器上查找——而在曾报告为活跃的探测后返回 `no_such_session` 则属矛盾：此时为 `no_such_session`（3，未记录任何内容），并由辅助工具的 `next` 处理——应在其自身的 tmux 服务器上停止该协调者，然后再次执行 `resume --takeover --confirm "<人类的话>"`）：无人占用该会话，否则会与新协调者并行运行，绝不能出现两个活跃的协调者；任何人自己的活跃会话都不会因接管而被停止。由 `init --detach` 记录的会话若自行恢复（其启动者执行 `resume`），则会继续保持 `detached` 状态及其记录中的姿态，因此 `context` 仍会如此显示，且来自其他会话的 `agents.py archive` 也能找到并停止它。

由启动器创建的会话绑定项目时（`MUSE_AGENTS_ROLE=launcher` 下的 `init` 未记录任何人；其开启的车道运行首次 `resume`），会被标记为 `detached: true`，如同 `init --detach` 的协调者：无人占用，因此任何会话执行 `agents.py archive --confirm` 都能将其停止并命名；而 `resume --takeover --confirm` 会在记录接管者之前先将其停止（shell 的归档曾为 `not_coordinator`，一次接管导致两条车道会话与已归档项目并存）。个人自己执行 `init` 的会话绝不会被如此标记。

在收件箱路径下，无论当前 `TBH_AGENTS_SESSION_PROTOCOL` 如何设定，`resume` 都会保持 `wake_path: inbox`（一条“进度”信息会注明），并重新订阅——本会话被确定为报告目标，并在记录中标识为 `inbox_target`（也在该行中，连同 `wake_path`）；“已重新订阅”列出那些下次报告将通过文件夹送达的运行线程；由协调者本人提交的远程线程报告则不会发送任何内容。唤醒臂被清空，循环逻辑按监控路径重写，而 `next` 仍是相同的臂条目。带有该标志的监控项目恢复后仍将维持监控路径，并予以说明。

## `stop`

```
stop <slug> <thread-id…> [--asked-by WHO]
```通过主机管理器的 `stop` 或车队管理器的 `close`，终止一个或多个线程的会话（多个 ID，如 `go`；在结束任何会话前都会检查每个 ID——未知的 ID 会被视为 `no_such_thread`，3，并且不会停止任何操作），无论记录的状态如何：处于 `running` 状态的记录会变为 `stopped`；而状态为 `done`、`exited`、`orphaned` 或 `stopped` 且其引擎仍在运行的记录，仅会失去会话（记录中标记为 `ended: true`，并附上 `stop_receipt`），其状态保持不变。`ended` 是本辅助函数使用的术语——表示一个正在运行的会话已被终止；`lingered` 是提供者使用的术语，原样传递（`true` 表示引擎忽略了退出指令而被强制关闭；`null` 表示未请求该会话退出）。已经消失的线程会被标记为 `stopped`，且 `ended: false`（车队管理器不再列出的远程会话同样被视为已消失）；以该线程名义存在的会话，若其身份与记录不符，则被视为“外来会话”——记录会被标记为 `orphaned`（若有报告则为 `exited`），并附带 `identity_drift`，此时既不发送任何指令也不关闭（`ended: false`）；处于 `proposed` 状态的线程从未建立过会话。调用者会在一切操作之前被检查：对于并非记录中协调者的会话，每次 `stop` 都会被判定为 `not_coordinator`（3），包括处于 `proposed` 状态的线程。若某个提供者对刚刚报告为“活跃”的会话返回 `no_such_session`（地址错误或提供者自身出现问题），则该拒绝会被原样传递（3）：只要会话仍可能在运行，就不会将其标记为已停止；后续流程为 `tick`，然后再尝试 `stop`。单个 ID 的响应格式为：顶部包含 `ended`、`lingered`、`status` 和一条 `receipt`（以及 `threads`，即该 ID 下的相同字段）。多个 ID 的响应则包含：`threads`（每行对应一个 ID，显示 `ended`、`lingered`、`status`，必要时还包括 `identity_drift`）、`stopped`（指那些会已被终止的 ID）以及 `receipts`，其中 `ref` = `<slug>/<id,id,…>`；若其中一个 ID 的提供者拒绝处理，则该拒绝信息会附在其对应的行下（包含 `outcome`、`error`、`next`），而其余 ID 对应的会话仍将被终止，该行会被标记为 `partial`（6），并注明其原因，后续流程为 `stop <slug> a b c`，此用法曾在 3 个项目中出现过。

## `agents.py archive`

```
archive <slug> [--confirm "<人类的话语>"] [--asked-by WHO]
```

按以下顺序进行收尾清理，每一步都以一行“progress”记录。第一行“progress”及其文本为：“Monitor: 不要执行 work_stop — 它会在一个 tick 内自行结束；其后的空“Monitor event”不属于输入（最多一行：“Monitor 已结束；项目已归档。”，不再重复关闭操作）”（三名协调员在读取收据后已将其停止；第一行会保留在“head”输出中）。收据中的“monitor_ended_line”即为该行内容：“Monitor 已结束；项目已归档。”，原样保留——这是已结束的 Monitor 轮次的全部文本，且“next”也明确指出这一点（实际口头传达的是“什么都不说”，并两次重复了关闭操作）。待下方的实时会话守护程序通过后，先解除唤醒状态——在停止任何线程之前，已清除唤醒标志并移除“library/wake.sh”，因此这些停止动作对任何人来说都不是新消息（一次 Monitor WAKE 在归档后被触发，并耗费了一个轮次；“disarmed”标识该层级，若未启用则为“null”，同时记录在行和收据中）。Monitor 仍在运行的循环则不予动用——一旦文件夹消失，它会在一个 tick 内以 0 退出，因此 Monitor 会自行结束（所有者规则第 26 条，#38715：该项目一直处于“watching”状态；此规则取代了保持存活的循环），随后运行时交付的空“Monitor event: agents <slug>”（一个“stream_ended”条目；TUI 显示“Monitor 'agents <slug>' — 已结束”）即为 Monitor 结束的标志，而非新的输入：什么都不说，并结束该轮次（一次“work_stop”与前一轮次的同一提示相同）；“next”对此予以确认。随后停止所有仍处于活动状态的会话——包括所有线程的会话，无论其状态如何（也包括已完成线程的引擎），以及由协调员在非当前执行该命令的会话中以“init --detach”方式开启的会话。无法响应的提供方所对应的会话亦视为活动。本辅助程序已结束的会话（“session_ended_at”）不再重复询问；已完成线程的活动会话（曾有一例“accept”未能结束，#41777）将不加说明地结束——其工作已凭证据被接受，而其空闲的引擎正是“threads_live”拒绝处理所有仍有线程处于活动状态的项目的依据（“threads_live”，3，用于标识尚未完成的线程的活动会话，其中包含协调员会话，当有任何线程处于活动且未使用“--confirm”时生效；“next”指明“accept”、 “--confirm”行及“stop <slug> <ids>”行）。对于每个 Git 报告为“干净”且其 HEAD 已合并至仓库默认分支的线程工作树——若仓库有远程源，则为其在“origin”上的副本，经一次有限的“git fetch”后获取；否则即为本地分支。若 fetch 失败，则以一行“progress”记录该副本的状态——“git worktree remove <path>”——**绝不使用“--force”**。若签出后仅存在线程的报告文件（根目录下的“report*.md”、“report*.txt”或“AGENTS-REPORT.md”），则视为“干净”：这些文件将首先移至“threads/<id>/leftover/”（每份报告各记一行“progress”），其余文件均不删除。若有污染或未合并的工作树，则保留在原处，并在“kept”中注明（“未合并至 origin/main”）。最后将该文件夹移至“~/.muse/projects/.archive/<slug>-<stamp>/”（记录仍可读取；无文件被删除）。分支与 PR 永不触碰。“archived”（0）伴随“stopped”（已结束会话的线程 ID）、“coordinator”（记录中分离的会话名称——“stopped (<provider>:<ref>)”；若会话已不存在或未找到对应会话，则为“not live (<provider>:<ref>)”；若项目无分离的协调员且由其他会话负责归档，则为“none”；若记录中的协调员本身具有会话身份并执行该命令，则为“this session (<provider>:<ref>)”——该命令从不终止自身所在的会话，因此负责归档自己项目的分离协调员会话将持续开放，直至用户手动关闭）。若记录中的协调员执行该命令时并无会话身份（仅为进程或窗口），则显示“this TUI session”。此外还包括“removed_worktrees”、“kept”、“archive_path”及一份收据。若读者提前关闭管道（“agents.py archive <slug> 2>&1 | head -5”），该命令也不会终止：被截断的行仍将计入“progress”，文件夹会被移动，收据会被写入，退出码仍为该命令自身的值。Slug 是每个命令中由字母、数字、`.`、`_` 和 `-` 组成的单个路径组件（参见“usage”第 2 条，否则适用其他规则）。

## 允许列表

读取并记录的动词——`doctor`、`context`、`overview`、`propose`、`report`、
`ack`、`inbox put`、`inbox drain`、`tick`（不带 `--arm` 选项）——可以按子命令被列入代理的允许列表，但不能作为独立的辅助工具被直接列入。`init --detach`、`go`、`follow`、`stop`、`agents.py archive`、`resume --takeover`、`remember`、`accept` 以及 `tick --arm` 会启动、结束、接管或记录权限，并始终停留在权限提示界面。`go`、`follow`、`accept`、`stop`、`remember`、`tick --arm`、`tick --disarm`、`agents.py archive` 和 `propose --replace` 还会对调用者进行检查：如果会话不是已记录的协调者（即 `state.json.coordinator`），则被视为“非协调者”（3，无变化；当该协调者已不存在，或为无法验证的进程身份时，`next` 为 `resume <slug>`——与普通 `resume` 能够继续执行的情形相同——而当协调者仍然在线，或其提供方无法响应时，则需使用 `resume <slug> --takeover --confirm`）。读取和记录类的动词则无需进行此类检查。对于已分离的项目，`stop` 和 `agents.py archive` 对任何会话均保持开放：因为已分离状态下没有协调者，且这两条命令仅用于结束操作，其中 `agents.py archive` 更是在用户确认后直接终止该分离会话。对于一个被记录为“opaque”的协调者（即一个无 tmux 窗格的沙盒化工具外壳），无论是否为其自身，都能被识别出来，因此这些动词在执行时会输出一行进度信息予以说明；而通过用户确认的 `resume --takeover --confirm` 则可最终确定其状态。Claude Code 的设置模板以及前缀规则无法表达的内容（如 `tick --arm`、`init --detach`）：
`references/allow-list.md`。

## 合约套件

该辅助工具的合约套件随其源代码树一同发布，位于技能目录中：包含标准库单元测试、离线模式、一个假的主机管理器和一个假的舰队管理器（通过环境变量 `MUSE_AGENTS_HOST_MANAGER` / `MUSE_AGENTS_FLEET_MANAGER` 注入）、每个测试独立的 `MUSE_PROJECTS_HOME` 目录、用于工作树分支的真实 Git 工具，且不包含任何休眠操作。