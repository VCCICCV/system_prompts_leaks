# Anthropic 命令行工具 (`ant`)

`ant` 命令行工具将 Claude API 的每个资源都作为 shell 子命令公开。与 `curl` 相比：请求体通过类型化的标志或管道传递的 YAML 构建，而非手写 JSON；`@path` 可以将文件内容内联到任意字符串字段中；`--transform` 使用 GJSON 路径提取字段（无需 `jq`）；列表端点会自动分页（可通过 `--max-items N` 限制总结果数；`--limit` 仅设置服务器的每页大小）；而 `beta:` 前缀会自动设置正确的 `anthropic-beta` 头。

## 何时使用 CLI 而不是 SDK

**控制平面用 CLI，数据平面用 SDK。** 代理和环境是相对静态的资源，您可以通过 `ant` 进行定义、配置和调试——将它们保存为文件并放在代码库中，通过 `ant apply` 同步（手动或在 CI 中），并在终端中进行检查。会话则是动态的，由您的应用通过 SDK 驱动——按任务创建，流式接收事件，响应工具调用，并集成到您的产品中。两者都调用相同的 API；区分的关键在于调用的位置，而不是功能的差异。

| | 控制平面 -> `ant` | 数据平面 -> SDK |
|---|---|---|
| 资源 | 代理、环境、技能、保险库、文件 | 会话、事件 |
| 调用频率 | 每次部署时或临时调用 | 每个任务/每一步 |
| 所在位置 | 代码库中的 `agents/`、`environments/`、`claude-lock.json`，以及 CI 和终端 | 应用程序代码 |
| 典型调用 | `ant apply`、`list`、`retrieve`、`archive`、`--debug` | `sessions.create()`、`events.stream()`、`events.send()` |

## 安装与认证

```sh
# macOS
brew install anthropics/tap/ant
xattr -d com.apple.quarantine "$(brew --prefix)/bin/ant"

# Linux / WSL - 从 github.com/anthropics/anthropic-cli/releases 下载对应版本
curl -fsSL "https://github.com/anthropics/anthropic-cli/releases/download/v${VERSION}/ant_${VERSION}_$(uname -s | tr A-Z a-z)_$(uname -m | sed -e s/x86_64/amd64/ -e s/aarch64/arm64/).tar.gz" \
  | sudo tar -xz -C /usr/local/bin ant

# 或者从源码编译（Go 1.25+）
go install github.com/anthropics/anthropic-cli/cmd/ant@latest
```

**认证**——CLI 解析凭证的方式与 SDK 相同（优先匹配第一个）：显式标志、`ANTHROPIC_API_KEY`、`ANTHROPIC_AUTH_TOKEN`、`ANTHROPIC_PROFILE` 选定的或当前活动的配置文件、工作负载身份联合的环境变量，最后是磁盘上的默认配置文件。您可以通过 `ANTHROPIC_BASE_URL` 或 `--base-url` 来覆盖 API 主机。

- **API 密钥**：在环境中设置 `ANTHROPIC_API_KEY`。
- **OAuth 配置文件**（无需管理静态密钥）：运行 `ant auth login` 会打开浏览器，交换获取短期令牌，并将配置文件存储在 `$ANTHROPIC_CONFIG_DIR` 下（Linux/macOS 默认为 `~/.config/anthropic/`，Windows 为 `%APPDATA%\Anthropic`——`configs/<profile>.json` 存储设置，`credentials/<profile>.json` 存储令牌）。后续的 `ant`（以及 SDK）调用会自动使用该配置——登录后直接使用 `Anthropic()` 客户端即可，但直接读取 `ANTHROPIC_API_KEY` 的脚本则无法正常工作。Claude Code 和 Claude Agent SDK 也遵循相同的配置文件解析规则。`ant auth status` 会显示当前使用的凭证来源和配置文件（仅报告状态，不要将其退出码作为健康检查的依据）；`ant auth logout` 会清除当前活动的配置文件（`--all` 则清除所有配置文件）。在没有浏览器的远程主机上，运行 `ant auth login --no-browser` 会打印授权 URL，并允许您在终端中输入验证码。
- **非交互式工作负载**（CI、服务器、容器）：交互式登录适用于您本地机器上的开发——请改用工作负载身份联合（详情请参阅通过 `shared/live-sources.md` 提供的认证文档）。

> **最常见的认证陷阱：** 只有在未设置 API 密钥时才会使用配置文件。过期的导出 `ANTHROPIC_API_KEY` 会静默覆盖所有配置文件——请求将路由到该密钥所限定的组织/工作空间。运行 `ant auth status` 可查看最终生效的来源；在依赖配置文件之前，请先取消设置该密钥（或在单次命令中：`env -u ANTHROPIC_API_KEY ant ...`）。务必将其**彻底清除**——即使设置为 `ANTHROPIC_API_KEY=""`，它仍会占据优先级并以空密钥进行认证。同样的遮蔽效应也适用于 Claude Code：在执行 `ant auth login` 后，Claude Code 可能会提示配置文件与其自身 `/login` 凭证之间存在认证冲突——请保留其中之一（使用配置文件并在 Claude Code 中执行 `/logout`，或运行 `ant auth logout` 以保留 Claude Code 的登录状态）。

**命名配置文件**——交互式登录生成的令牌仅绑定到一个组织和工作空间，API 也只会显示属于该工作空间的资源。如果您创建的代理、会话或文件“消失”了，通常是因为令牌的作用域与创建它们的工作空间不一致（可通过 `ant auth status` 查看当前活动的工作空间）。若需在多个工作空间间切换，建议每个工作空间使用一个独立的配置文件：

```sh
ant auth login --profile <name>                  # 如果配置文件不存在则创建；浏览器中选择组织/工作空间
ant auth login --profile <name> --workspace-id wrkspc_01...   # 直接绑定，跳过选择界面
ant profile activate <name>                      # 切换默认配置文件
ant --profile <name> models list                 # 一次性使用；等价于：ANTHROPIC_PROFILE=<name> ant models list
ant profile list                                 # 查看配置文件列表
ant profile set workspace_id wrkspc_01... --profile <name>    # 编辑配置项（workspace_id、base_url、organization_id 等）
```

`ant profile set` 用于编辑现有配置文件的配置，不会新建配置文件，也不会重新绑定已颁发的凭据；如需为新的目标绑定令牌，请在该配置文件下再次运行 `ant auth login`。如果将 `ANTHROPIC_PROFILE` 指向一个不存在的配置文件，将会报错，而不会回退到其他配置。刷新令牌最终会硬性失效（不会因使用而延长有效期）——当某个原本可用的配置文件开始认证失败时，请先重新运行一次 `ant auth login`，再排查其他问题。

**作用域**——配置文件的 OAuth 作用域是在登录时指定的（通过 `--scope`），并会持久化保存在配置文件中（`scope` 也是 `profile set` 的配置项；与其他配置修改一样，更改后需重新登录才能生效）。特权作用域（例如用于组织管理接口的 `org:admin`）**不在默认作用域集合中**：请显式传递所需的所有作用域（`ant auth login --profile admin --scope "... org:admin"`），且服务器仅在您的角色确实拥有该权限时才会授予。由于作用域会随配置文件生成的每个令牌一起携带，建议将特权操作分配给专用配置文件（如 `admin`），日常推理则使用非特权配置文件，并通过 `--profile` 或 `ANTHROPIC_PROFILE` 进行切换。可运行 `ant auth login --help` 查看当前支持的作用域列表，运行 `ant auth status` 查看当前有效令牌所携带的作用域。

如需将当前凭证传递给子进程或原生 HTTP 脚本，可按如下方式操作：

```sh
# 原始访问令牌——用于 curl 的 Authorization 头
curl https://api.anthropic.com/v1/messages \
  -H "Authorization: Bearer $(ant auth print-credentials --access-token)" \
  -H "anthropic-version: 2023-06-01" \
  -H "anthropic-beta: oauth-2025-04-20" \
  -H "content-type: application/json" \
  -d '{"model": "claude-opus-5-5", "max_tokens": 1024, "messages": [{"role": "user", "content": "Hello"}]}'

# .env 格式——设置 ANTHROPIC_AUTH_TOKEN（以及 ANTHROPIC_BASE_URL，若配置文件中有）。
# 输出为纯 KEY=value（不含 export），因此可使用 `set -a` 自动导出供子进程使用：
set -a; eval "$(ant auth print-credentials --env)"; set +a
python my_script.py   # SDK 将自动读取 ANTHROPIC_AUTH_TOKEN
```

OAuth 令牌应放在 `Authorization: Bearer` 头中（而非 `x-api-key:`），**并加上 `anthropic-beta: oauth-2025-04-20` 头**——将使用 API 密钥的原始 curl/httpx 脚本转换为 OAuth 令牌时，只需更改请求头，而无需更换密钥。Beta 头的要求因端点而异（某些端点在没有该头的情况下也能正常工作；但 `/v1/messages` 端点则不行），因此请始终携带该头，以免在切换端点时导致请求失败。通过环境变量传递的令牌有效期较短且不会自动刷新，因此对于长时间运行的脚本，请在令牌过期前重新执行 `print-credentials` 命令（`print-credentials` 本身会在必要时刷新令牌）。如果同时设置了 `ANTHROPIC_API_KEY` 和 `ANTHROPIC_AUTH_TOKEN`，SDK 会同时发送两者，API 将拒绝该请求——在执行 `--env` 输出之前，请先取消设置 `ANTHROPIC_API_KEY`。

**防误操作提示：** 不带任何参数运行 `ant auth print-credentials` 会输出完整的凭据 JSON，而不是单纯的令牌字符串；如果直接将此 JSON 放入 `Authorization` 头中，可能会导致空响应或 HTTP/2 协议错误。在需要放入请求头时，请始终使用 `--access-token` 选项（它总是读取命名或活动的配置文件；即使设置了 `ANTHROPIC_API_KEY`，也不会覆盖凭据的打印结果）。

## 命令结构

```
ant <资源>[:<子资源>] <操作> [选项]
```

Beta 资源（代理、会话、环境、部署、技能、保险库、记忆存储）位于 `beta:` 命名空间下——CLI 会自动添加正确的 `anthropic-beta` 头，除非您使用 `--beta <header>` 显式覆盖。对于自托管环境，`ant beta:worker poll/run` 和 `ant beta:environments:work stats/stop` 用于驱动和监控工作队列——详情请参阅 `shared/managed-agents-self-hosted-sandboxes.md`。

```sh
ant models list
ant messages create --model claude-opus-5-5 --max-tokens 1024 --message '{role: user, content: "Hello"}'
ant beta:agents retrieve --agent-id agent_01...
ant beta:sessions:events list --session-id session_01...
```

`ant --help` 列出所有资源；在任意子命令后追加 `--help` 可查看其可用选项。

## 全局选项

| 选项 | 用途 |
| --- | --- |
| `--format` | `auto`（默认：TTY 时美观输出，管道输入时紧凑输出）、`json`、`jsonl`、`yaml`、`pretty`、`raw`、`explore`（交互式 TUI） |
| `--transform` | 对响应应用 GJSON 路径表达式（在列表端点上对每项生效）。当使用 `--format raw` 时，该选项不生效。 |
| `-r`, `--raw-output` | 如果经过变换后的结果是字符串，则以无引号形式输出（类似 jq 的行为）。与 `--transform` 配合使用，可用于提取标量值。 |
| `--max-items` | 限制自动分页的列表端点返回的总结果数（不同于 `--limit`，后者指服务器的单页大小）。 |
| `--format-error` / `--transform-error` | 类似于 `--format`/`--transform`，但应用于错误响应。`-r` 不适用于错误路径——如需无引号输出错误标量，请使用 `--format-error yaml`。 |
| `--base-url` | 覆盖 API 主机地址 |
| `--debug` | 将完整的 HTTP 请求与响应打印到标准错误流（API 密钥已脱敏） |

## 输出 —— `--transform` + `--format`

`--transform` 接受 [GJSON 路径](https://github.com/tidwall/gjson/blob/master/SYNTAX.md)。在列表端点上，该选项**逐项**执行，而非作用于整个响应包。

```sh
ant beta:agents list --transform '{id,name,model}' --format jsonl
```

**提取标量供 Shell 使用：** 将 `--transform` 与 `-r`（`--raw-output`——按 jq 风格无引号输出字符串）配合使用：

```sh
AGENT_ID=$(ant beta:agents create --name "My Agent" --model '{id: claude-sonnet-5-5}' \
  --transform id -r)
```

## 输入 —— 选项、标准输入、`@文件`

**选项**——标量字段可直接映射。结构化字段支持宽松 YAML 语法（键无需加引号）或严格 JSON 格式。可重复的选项会构建数组（每个 `--tool`、`--event`、`--message` 都会追加一个元素）：

```sh
ant beta:agents create \
  --name "Research Agent" \
  --model '{id: claude-opus-5-5}' \
  --tool '{type: agent_toolset_20260401}' \
  --tool '{type: custom, name: search_docs, input_schema: {type: object, properties: {query: {type: string}}}}'
```**标准输入** - 通过管道传递完整的 JSON 或 YAML 请求体。请求体与命令行参数合并；发生冲突时以命令行参数为准（对于数组字段，任何命令行参数都会**完全替换**标准输入中的数组，而不会追加）。若要禁用请求体内的 Shell 展开，请对 Here Document 的分隔符加单引号：`<<'YAML'`。

```sh
ant beta:agents create <<'YAML'
name: Research Agent
model: claude-opus-5-5
system: |
  You are a research assistant. Cite sources for every claim.
tools:
  - type: agent_toolset_20260401
YAML
```

**`@文件` 引用** - 可将文件内容内联到任意字符串类型的字段中。在结构化命令行参数值中，请对路径加引号。二进制文件会自动进行 Base64 编码；也可通过 `@file://`（文本）或 `@data://`（Base64）强制指定编码方式。若需表示字面意义上的 `@`，请使用转义符 `\@`。

```sh
ant beta:agents create --name "Researcher" --model '{id: claude-sonnet-5-5}' --system @./prompts/researcher.txt

ant messages create --model claude-opus-5-5 --max-tokens 1024 \
  --message '{role: user, content: [
    {type: document, source: {type: base64, media_type: application/pdf, data: "@./scan.pdf"}},
    {type: text, text: "Extract the text from this scanned document."}
  ]}' \
  --transform 'content.0.text' -r
```

原生接受文件路径的命令行参数（如 `beta:files upload` 中的 `--file`）可以直接使用不带 `@` 的文件路径。

## 版本控制下的托管代理资源（`ant apply`）

这是定义代理、环境、技能、记忆存储和部署的推荐流程：在仓库中为每个资源创建一个文件（或技能目录），并通过 `ant apply` 进行同步（需要 `ant` 1.30.0 或更高版本——可通过 `ant --version` 检查）。该命令会输出执行计划，创建或更新有差异的资源，并将每个资源的 ID 记录在 `claude-lock.json` 文件中。有关字段参考，请参阅 `shared/managed-agents-core.md`；关于 `--force`、`--prune`、`--lock-file`、重命名或删除的文件以及 CI 配置等选项，请参阅 `shared/live-sources.md` 中的 `ant apply` 页面（文档面向终端用户编写；以下规则同样适用）。

```
agents/summarizer.md          # YAML 前置元数据为代理配置，Markdown 正文为系统提示
environments/cloud.yaml       # 环境创建请求体
skills/pr-summary/SKILL.md    # 技能是一个目录，根目录下包含 SKILL.md
memory_stores/notes.yaml
deployments/nightly.md        # 前置元数据为部署创建请求体，Markdown 正文为每次运行的起始消息
claude-lock.json              # 由 ant apply 写入，提交至版本库
```

```markdown
---
# agents/summarizer.md
name: Summarizer
model: claude-sonnet-5-5
tools:
  - type: agent_toolset_20260401
---

You are a helpful assistant that writes concise summaries.
```

```yaml
# environments/cloud.yaml
name: summarizer-env
config: {type: cloud, networking: {type: unrestricted}}
```

```sh
ant apply --dry-run -v agents/summarizer.md environments/cloud.yaml   # 打印包含所有字段的计划，但不执行任何操作
ant apply agents/summarizer.md environments/cloud.yaml               # 打印计划并提示确认：(y)es / (n)o / (d)etails —— 需要交互式终端
ant apply                                                          # 后续：同步仓库中所有已记录在 claude-lock.json 的文件
```

- **为您编写的文件命名；仅当用户要求应用整个目录树时，才传递`.`或一个目录。** 目录会递归遍历至任意深度，所有看起来像是资源的文件都会被应用：任何在顶级包含`type:`字段的文件、直接位于`agents/`、`environments/`、`memory_stores/`或`deployments/`目录中的文件，或者以这些目录命名的文件（如`environment_staging.yaml`），以及任何包含`SKILL.md`的目录。Claude Code 插件、conda（`environment.yml`）和 Kubernetes（`deployments/`）使用相同的命名规范，克隆下来的仓库中可能包含用户从未阅读过的文件。
  
- **在没有终端（即编码代理的 Shell）的情况下，`ant apply` 会打印计划并退出；只有加上`--yes`才会执行。** 如果您是为用户运行此命令的编码代理，该标志代表用户的批准，而非您的：请先向用户展示试运行计划，待其同意后再添加`--yes`（或回答提示）。计划还会覆盖`claude-lock.json`中已追踪的内容：如果计划会创建或更改您未编写的内容，或者您需要的路径上存在非您编写的文件，请立即停止并询问；切勿自行添加`--force`或`--prune`。

- **通过路径引用其他资源，而非 ID**（相对于声明该资源的文件而言）：在代理配置中写`skills: [../skills/pr-summary]`；在部署配置中写`agent: ../agents/summarizer.md`和`environment_id: ../environments/cloud.yaml`。`ant apply` 还会按依赖顺序应用您传递的文件所引用的所有内容，并补全 ID。对于这些文件未管理的资源，请直接填写其 ID（如`agent_01...`、`env_01...`）；其他内容则按原样传递。

- **提交`claude-lock.json`**（首次运行时会在您执行命令的目录下生成该文件——建议使用仓库根目录）。后续运行将基于该文件更新相同资源，而不会创建重复项。以其他方式（控制台、`ant beta:agents create`、SDK）创建的资源无法被纳入管理：为其编写描述文件会导致创建第二个副本。

- **要变更某个资源，编辑其文件后再次运行`ant apply`**（代理会获得新版本，所有引用它的部分也会在同一轮中更新）。

- **在用户自己的仓库中进行 CI：** 从包含`claude-lock.json`的目录（通常是仓库根目录）运行，并指定项目中已有的资源目录，而非`.`（递归遍历`.`还会应用仓库中其他位置的类似文件）：在拉取请求时运行`ant apply --dry-run agents environments`，仅在推送到默认分支时运行`ant apply --yes agents environments`（此时合并即视为批准），然后提交`claude-lock.json`。

- **不受管理的内容：** 秘钥库与凭据（通过`ant beta:vaults`、`ant beta:vaults:credentials`或 SDK 操作）、上传的文件、会话。

**一次性部署**仍可使用`ant beta:agents create <<'YAML'`（参见上方“输入”部分）和`ant beta:agents update --agent-id ... --version N`；您需自行维护这些资源的 ID。

使用`claude-lock.json`中的 ID 启动会话（每个`resources`键对应的值即计划中打印的文件路径）：

```sh
AGENT_ID=$(jq -r '.resources["./agents/summarizer.md"].id' claude-lock.json)
ENV_ID=$(jq -r '.resources["./environments/cloud.yaml"].id' claude-lock.json)
SID=$(ant beta:sessions create --agent "$AGENT_ID" --environment-id "$ENV_ID" --title "Task" --transform id -r)
ant beta:sessions:events send --session-id "$SID" \
  --event '{type: user.message, content: [{type: text, text: "总结 X"}]}'
ant beta:sessions:events list --session-id "$SID" --transform 'content.0.text' -r
ant beta:sessions:events stream --session-id "$SID"   # 实时事件流
```

### 将终端连接到会话（`ant beta:sessions connect`）

`ant beta:sessions connect <会话 ID>` 会将您的终端连接到现有会话：加载会话记录、实时跟踪，并允许您介入——发送消息、中断，或对正在等待批准的工具调用进行允许/拒绝操作。按下 Ctrl+C 即可断开连接；会话将继续运行，重新连接时会加载完整历史记录。若会话状态为`terminated`或已归档，则为只读模式。
```sh
ant beta:sessions connect sesn_011CZkZAtmR3yMPDzynEDxu7          # 终端视图
ant beta:sessions connect sesn_011CZkZAtmR3yMPDzynEDxu7 --web    # 控制台会话查看器，本地提供服务
```

| 快捷键 | 功能 |
|---|---|
| Enter | 将输入作为 `user.message` 发送（Alt+Enter / Ctrl+J 插入换行） |
| Esc | 中断正在运行的代理（`user.interrupt`） |
| Ctrl+O | 切换详细信息：工具输入/结果、Token 使用量、状态事件（`--verbose` / `-v` 启动时展开显示） |
| PgUp / PgDn | 滚动；向上滚动时暂停跟随，End 键恢复 |
| Ctrl+C（或空输入时 Ctrl+D） | 断开连接 |

当调用等待批准时（`always_ask`，或 `auto` 且未作出判断），输入行会变为 **允许工具调用？**，并提供 **是** / **否** / **否，并告知代理原因** 选项——CLI 会发送 `user.tool_confirmation`，并将您输入的原因作为 `deny_message` 一并发送。在多代理会话中，终端视图仅跟踪主线程（包括协调器与子代理之间的消息）。

`--web` 会在本地服务器 `127.0.0.1` 上提供控制台的会话查看器，打印 URL 并自动打开浏览器（使用 `--no-browser` 可跳过此步骤）。该 URL 在两分钟内有效一次（重新加载该标签页即可；若要在其他地方打开，需再次执行命令）。页面仅与本地的 `ant` 进程通信，由其发起 API 调用，因此凭据不会离开 CLI；服务器将持续运行，直到按下 Ctrl+C 停止。与终端视图不同，浏览器查看器会跟踪多代理会话中的所有线程。

除 `--web` 外，均需要交互式终端——脚本场景请使用下方的 `ant beta:sessions:events stream` / `send`。

### 交互式会话循环（先流后发）

`ant beta:sessions:events stream` 仅推送流开启 *之后* 发出的事件——因此请在发送启动消息 **之前** 先开启流，以免错过早期事件。可使用进程替换将流保留在文件描述符上，先发送再读取：

```sh
exec {stream}< <(ant beta:sessions:events stream --session-id "$SID" \
  --transform '{type,text:content.#(type=="text").text,err:error.message}' --format yaml)

ant beta:sessions:events send --session-id "$SID" > /dev/null <<'YAML'
events:
  - type: user.message
    content:
      - type: text
        text: 总结一下仓库的 README
YAML

type=
while IFS= read -r -u "$stream" line; do
  case "$line" in
    type:\ session.status_idle) break ;;
    type:\ session.error)
      IFS= read -r -u "$stream" next || next=
      case "$next" in err:\ *) msg=${next#err: } ;; *) msg=未知 ;; esac
      printf '\n[错误：%s]\n' "$msg"; break ;;
    type:\ *) type=${line#type: } ;;
    text:*)
      [[ $type == agent.message ]] || continue
      val=${line#text: }
      case "$val" in '|-'|'|') ;; *) printf '%s' "$val" ;; esac ;;
    \ \ *)
      if [[ $type == agent.message ]]; then printf '%s\n' "${line#  }"; fi ;;
  esac
done
exec {stream}<&-
```

此方法适用于交互式探索和演示。对于需要响应 `agent.tool_use` / `agent.custom_tool_use` 事件的应用程序代码，可在连接中断后重新建立连接，或针对 `events.list` 进行去重处理，请使用 SDK——详情参见 `shared/managed-agents-client-patterns.md`。

## 脚本编写模式

在列表端点使用 `--transform id -r` 会逐行输出裸 ID——可与 `xargs` 组合使用，或使用 `--max-items N` 来限制结果集，而无需通过 `head` 管道：

```sh
FIRST=$(ant beta:agents list --transform id -r --max-items 1)
ant beta:agents:versions list --agent-id "$FIRST" --transform '{version,created_at}' --format jsonl
```

错误格式化与成功路径一致（注意：`-r` 不适用于错误输出——此处可使用 `--format-error yaml` 获取未加引号的标量）：

```sh
ant beta:agents retrieve --agent-id bogus --transform-error error.message --format-error yaml 2>&1
```

Shell 补全：`ant @completion {zsh|bash|fish|powershell}`。

如需获取完整且始终最新的参考文档（包括各端点的标志选项），请在 `shared/live-sources.md` 中使用 WebFetch 打开 **Anthropic CLI** 的网址。