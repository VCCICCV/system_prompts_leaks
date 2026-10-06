---
name: fewer-permission-prompts
description: 扫描您的代码库，查找常见的只读 Bash 和 MCP 工具调用，然后在项目的 .claude/settings.json 文件中添加一个优先级排序的白名单，以减少权限提示。
---
# 减少权限提示

请查看我的会话记录中的 MCP 和 Bash 工具调用，并据此列出一个优先级排序的模式列表，以便将这些模式添加到我的权限白名单中，从而减少权限提示。重点关注只读命令。

权限的格式为：`Bash(foo*)`、`Bash(foo)`、`Bash(foo bar *)`、`mcp__slack__slack_read_thread` 等。

然后，将这些条目添加到项目的 `.claude/settings.json` 文件中的 `permissions.allow` 部分。

## 操作步骤

1. **定位会话记录。** 会话记录位于 `~/.claude/projects/<sanitized-cwd>/*.jsonl` 目录下，每行是一个 JSON 对象。工具调用以 `assistant` 类型的消息形式出现，其 `message.content[]` 中包含 `type: "tool_use"` 的条目。`name` 字段标识工具名称（如 `"Bash"`、`"mcp__slack__slack_read_thread"`）；对于 Bash 调用，`input.command` 是对应的 Shell 命令字符串。

   请扫描用户项目目录下的近期会话记录——而不仅是当前项目——以确保白名单反映其真实使用情况。为保证效率，可将扫描范围限定在一定数量的最近修改文件内（例如最近修改的 50 个 JSONL 文件）。

2. **提取工具调用频率。**
   - 对于 `Bash` 调用：解析 `input.command`，提取首词（处理 `sudo`、`timeout`、管道、`&&`、环境变量前缀等情况）。记录命令及其首个子命令的组合（如 `git status`、`gh pr view`、`ls`、`cat`）。
   - 对于 MCP 调用：记录完整的工具名称（如 `mcp__slack__slack_read_thread`）。
   - 统计这些调用在已扫描会话记录中的出现次数。

3. **筛选只读命令。** 仅保留不会改变系统状态的命令。以下是一些只读命令的示例：`ls`、`cat`、`pwd`、`git status`、`git log`、`git diff`、`git show`、`git branch`、`rg`、`grep`、`find`、`head`、`tail`、`wc`、`file`、`which`、`echo`、`date`、`gh pr view`、`gh pr list`、`gh pr diff`、`gh issue view`、`gh issue list`、`gh run list`、`gh run view`、`gh api`（GET）、`bun run typecheck`、`bun run lint`、`bun run test`（仅限不产生副作用的测试）、`docker ps`、`docker logs`、`kubectl get`、`kubectl describe`、`ps`、`top`、`df`、`du`、`env`、`printenv`，以及任何名称中带有 `read`/`get`/`list`/`search`/`view` 的 MCP 工具。

   删去所有涉及写入、删除、重命名、推送、合并、安装，或执行具有副作用的构建/测试的命令。如有疑问，请一律排除。

   **切勿将可能赋予任意代码执行能力的模式加入白名单。** 任何此类通配符规则（如 `Bash(python3:*)`）都等同于允许任意代码执行。此列表并不详尽——对同一类别的其他项也应适用相同原则：
   - 解释器：`python`/`python3`、`node`、`bun`、`deno`、`ruby`、`perl`、`php`、`lua` 等。
   - Shell：`bash`、`sh`、`zsh`、`fish`、`eval`、`exec`、`ssh` 等。
   - 包管理器相关命令：`npx`、`bunx`、`uvx`、`uv run` 等。
   - 任务运行器通配符：`npm run *`、`yarn run *`、`pnpm run *`、`bun run *`、`make *`、`just *`、`cargo run *`、`go run *` 等——明确的 `Bash(bun run typecheck)` 可以接受，但 `Bash(bun run *)` 不行。
   - `gh api *`、`docker run`/`exec`、`kubectl exec`、`sudo` 等类似命令。

4. **剔除 Claude Code 已自动放行的命令。** 这些命令无需添加到白名单，因为它们不会触发权限提示。如果在会话记录中发现这些命令，请直接跳过，不要向用户推荐。

- **始终自动允许（任何参数）：** `cal`、`uptime`、`cat`、`head`、`tail`、`wc`、`stat`、`strings`、`hexdump`、`od`、`nl`、`id`、`uname`、`free`、`df`、`du`、`locale`、`groups`、`nproc`、`basename`、`dirname`、`realpath`、`cut`、`paste`、`tr`、`column`、`tac`、`rev`、`fold`、`expand`、`unexpand`、`fmt`、`comm`、`cmp`、`numfmt`、`readlink`、`diff`、`true`、`false`、`sleep`、`which`、`type`、`expr`、`seq`、`tsort`、`pr`、`echo`、`ls`、`cd`。
- **仅在无参数时自动允许：** `pwd`、`whoami`、`alias`。
- **仅允许精确形式：** `claude -h`、`claude --help`、`node -v`、`node --version`、`python --version`、`python3 --version`、`ip addr`。
- **仅允许安全标志（已验证）：** `xargs`、`file`、`sed`（只读表达式）、`sort`、`man`、`help`、`netstat`、`ps`、`base64`、`grep`、`egrep`、`fgrep`、`sha256sum`、`sha1sum`、`md5sum`、`tree`、`date`、`hostname`、`lsof`、`pgrep`、`tput`、`ss`、`fd`、`fdfind`、`aki`、`rg`、`jq`、`uniq`、`history`、`arch`、`ifconfig`、`pyright`、`find`（禁止 `-delete`/`-exec`/`-execdir`/`-ok`/`-okdir`/`-fprint*`/`-fls`/`-files0-from`）、`printf`（禁止任何 `-flag`）、`test`（禁止 `-v`/`-R`/`-a`/`-o`）。
- **所有 Git 只读子命令：** `git status`、`git log`、`git diff`、`git show`、`git blame`、`git branch`、`git tag`、`git remote`、`git ls-files`、`git ls-remote`、`git config --get`、`git rev-parse`、`git describe`、`git stash list`、`git reflog`、`git shortlog`、`git cat-file`、`git for-each-ref`、`git worktree list` 等。
- **所有 gh 只读子命令：** `gh pr view`、`gh pr list`、`gh pr diff`、`gh pr checks`、`gh pr status`、`gh issue view`、`gh issue list`、`gh issue status`、`gh run view`、`gh run list`、`gh workflow list`、`gh workflow view`、`gh repo view`、`gh release view`、`gh release list`、`gh api`（GET）、`gh auth status` 等。
- **Docker 只读子命令：** `docker ps`、`docker images`、`docker logs`、`docker inspect`。

事实来源：`src/tools/BashTool/readOnlyValidation.ts`（`READONLY_COMMANDS`、`READONLY_NOARGS`、`READONLY_EXACT`、`COMMAND_ALLOWLIST`）以及 `src/utils/shell/readOnlyCommandValidation.ts`（`GIT_READ_ONLY_COMMANDS`、`GH_READ_ONLY_COMMANDS`、`DOCKER_READ_ONLY_COMMANDS`、`RIPGREP_READ_ONLY_COMMANDS`、`PYRIGHT_READ_ONLY_COMMANDS`）。如果用户位于该仓库且不确定某条命令是否已被覆盖，请搜索这些文件，而非凭猜测判断。

5. **选择模式形式。** 使用能够覆盖观察到的使用场景且范围最窄的模式：
   - 如果用户运行多种变体（`git log`、`git log --oneline`、`git log main..HEAD`），则使用 `Bash(git log *)` — 注意 `*` 前的空格，这是确保前缀匹配正确工作的必要条件。
   - 如果常见的是单一精确调用，则使用不带通配符的 `Bash(foo)`。
   - 对于 MCP，原样使用完整工具名称（无需通配符；它们本身已足够具体）。
   - 切勿将模式放宽至与上述规则相冲突的程度（禁止任意代码执行，禁止修改或副作用）。

6. **排序优先级。** 按出现次数从高到低排序。删除出现次数少于约 3 次的项——不值得列入白名单。将列表上限控制在前 20 项左右，以便用户可以快速浏览。

7. **向用户展示优先级列表**，以 Markdown 表格形式呈现，包含以下列：排名、模式、出现次数、简要说明。示例：

| # | 模式 | 出现次数 | 备注 |
|---|------|--------|------|
| 1 | `Bash(git status *)` | 142 | 仓库状态检查 |
| 2 | `Bash(gh pr view *)` | 87 | PR 查看 |
| 3 | `mcp__slack__slack_read_thread` | 54 | Slack 线程阅读 |

8. **合并至当前项目的 `.claude/settings.json` 文件中**（非 `~/.claude/settings.json`，也非 `.claude/settings.local.json`）。如文件不存在则创建之。保留现有键及 `permissions.allow` 中的已有条目；避免重复；不得删除任何内容；不得重新排列无关字段。9. **汇报进展。** 告知用户您添加了哪些内容（数量及几个示例），哪些内容已存在于白名单中，以及您跳过了哪些内容及其原因（例如：“删除了 `rm` 和 `git push` —— 不是只读命令；删除了 `cat`/`ls`/`git status` —— 已自动放行，无需添加规则”）。

请勿向 `permissions.deny` 或 `permissions.ask` 中添加任何内容。请勿修改其他任何设置字段。
