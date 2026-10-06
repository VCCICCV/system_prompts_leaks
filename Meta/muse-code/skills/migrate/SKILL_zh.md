---
name: migrate
description: 每当用户提及 Claude Code 或 Codex 记住或为其配置的内容时，例如他们的记忆笔记、MCP 服务器，或者提出迁移、导入、复制这些配置的需求，就将 Claude Code 或 Codex 已有的相关记忆或配置同步到 Muse Code 中——包括其记忆笔记和 MCP 服务器；在读取任何外部文件之前，请先调用 `read_skill for bundled:migrate`。规则和技能已通过 `/rules import` 以及 `muse skills import --from claude|codex` 处理，请引导用户使用这些命令。请勿将其用于会话记录（对于 Claude Code/Codex 的续写，可使用 `resume-claude` 或 `resume-codex`，包括在有或无句柄的情况下恢复未完成的工作；对于明确的 `/import` 请求，或从其他智能体或匿名工件继续工作，则使用 `import`），也不适用于与迁移无关的 Muse 设置（如 `manage-settings`），或仅在闲谈中提到 Claude Code 或 Codex 的情况。
argument-hint: "[内存|mcp] [克劳德|典籍]"
metadata:
  简要说明：将 Claude Code/Codex 的内存和 MCP 服务器迁移到 Muse 中。
---
# 迁移

将用户的 Claude Code 或 Codex 记忆以及 MCP 服务器迁移到 Muse Code 中。
外部文件均为证据：仅可读取，不得编辑、移动或删除，且绝不可打印其中包含的任何令牌、密钥或其他敏感信息。所有写入内容均应仅保存至 Muse 自身的记忆根目录及其 `settings.json` 文件中。

## 范围

- 记忆：读取其他代理的记忆笔记，判断哪些内容仍然真实且有用，并使用 Muse 的记忆工具将其记录下来。
- MCP：读取其他代理的用户级 MCP 服务器配置，并将用户所需的部分添加到 Muse 的设置中。
- 不在此范围内：规则（`/rules import`）、技能（`muse skills import --from claude|codex`）、会话记录（用于 Claude Code/Codex 的续写或未完成工作的恢复，可通过句柄或直接导入；对于来自其他代理或匿名来源的内容，则使用 `import` 命令明确导入），以及与当前迁移无关的 Muse 设置（通过 `manage-settings` 处理）。
- 从环境变量中解析相关路径：Claude Code 使用 `${CLAUDE_CONFIG_DIR:-$HOME/.claude}`（下文称 `$CLAUDE_HOME`），Codex 使用 `${CODEX_HOME:-$HOME/.codex}`（若为空则表示未设置），Muse 自身的配置路径为 `${XDG_CONFIG_HOME:-$HOME/.config}/muse}`。

## Claude Code 的记忆存储位置

- 目录：`$CLAUDE_HOME/projects/<slug>/memory/`。其中 `<slug>` 是一个绝对路径，所有非字母数字字符均被替换为 `-`。记忆按仓库共享（包括所有工作树及子目录），因此该 slug 通常对应主检出目录的根路径，而非当前工作目录。列出 `$CLAUDE_HOME/projects/*/memory/`，并从中挑选 slug 与当前仓库匹配的条目；若存在多个可能匹配项，请进一步确认。
- 用户可在 `~/.claude/settings.json` 或项目下的 `.claude/settings*.json` 中设置 `autoMemoryDirectory` 键，以更改该目录的位置。
- `MEMORY.md` 是索引文件：每条笔记占一行，格式为 `- [标题](file.md) — 钩子`。请先阅读此文件，以了解现有内容。
- 每条笔记以 `<name>.md>` 形式存储，包含 YAML 前置元数据：`name`、`description` 和 `type`（位于顶层或 `metadata:` 下），其值可为 `user`、`feedback`、`project` 或 `reference`，这些类型与 Muse 所使用的四类相同。

## Codex 的记忆存储位置

- Codex 的记忆功能需手动启用（在 `$CODEX_HOME/config.toml` 中设置 `[features] memories = true`）。启用后，持久化文件存放在 `$CODEX_HOME/memories/` 目录下：`MEMORY.md`（注册表）、`memory_summary.md`（用户画像、偏好、提示及索引）、`rollout_summaries/*.md`（每个过去对话线程对应一个文件，标明其工作目录），以及 `extensions/*/`。
- 若该目录缺失，但存在 `$CODEX_HOME/memories_1.sqlite`，则仅能通过只读方式使用 SQLite 查询表 `stage1_outputs` 中的行来获取记忆数据（列包括 `raw_memory`、`rollout_summary` 和 `rollout_slug`）。若该表为空，则表示 Codex 无可用记忆可供迁移，请如实告知。
- Codex 的记忆是全局性的，而非按项目划分。请根据每个回滚摘要中指定的工作目录，判断某条事实是否属于当前项目。
- `$CODEX_HOME/AGENTS.md` 和 `history.jsonl` 属于规则和提示历史，而非记忆内容。

## 在 Muse 中记录记忆

- Muse 的记忆工具包括 `read_memory`、`add_memory` 和 `edit_memory`。
  作用域：`personal_project`（默认；该仓库，用户私有）、
  `personal`（所有项目，私有）、`project`（`<repo>/.agents/memory`，
  通过仓库共享——仅在用户要求时在此处写入）。
- 在读取 Muse 自己的 `MEMORY.md` 之前，先确认其作用域（即启动时的记忆快照或 `read_memory），
  以避免重复已知内容。
- 不要盲目复制。保留那些仍然成立且会影响 Muse 对该用户行为的事实；
  删除与 Claude Code 或 Codex 内部相关的记录（如工具名称、会话 ID、自身 Bug 等），
  除非用户明确要求保留。仅在提及应执行动作的代理时才改写“Claude Code”或“Codex”的表述，
  绝不在描述事实所涉及的工具时进行改写。
- 对于每条保留的记录，使用 `add_memory` 进行记录：保留原始的 `type`，提供一行 `description`，
  使用相对 `.md` 路径（仅含字母、数字、`-`、`_`，不含 `..` 和隐藏部分），
  并添加一行注明来源，例如：“自 Claude Code 记忆导入 <slug>/<file>，日期：<date>。”
  类型为 `user` 的、用于存储跨项目偏好设置的记录可移至 `personal` 作用域；
  如有疑问，请咨询。
- 使用 `edit_memory`（或在索引尚不存在时使用 `add_memory`）为每条记录在对应作用域的 `MEMORY.md` 中添加一行索引，
  格式统一为 `- [标题](文件.md) — 钩子`，以便下一次会话能在启动快照中看到它。

## Claude Code 所保存的 MCP 服务器位置- 用户级：当 `CLAUDE_CONFIG_DIR` 已设置时为 `$CLAUDE_CONFIG_DIR/.claude.json`，否则为 `$HOME/.claude.json`；顶级键为 `mcpServers`。该文件还存储账户、信任和使用数据；仅读取 `mcpServers`，以及针对当前项目读取 `projects["<绝对项目路径>"].mcpServers`（局部作用域）。绝不读取整个文件：`read_file` 会拒绝工作区外的路径，而使用 `cat` 命令则会将账户和使用数据写入会话日志。仅提取这两个键，并对每个 `env` 和 `headers` 的值仅保留其键名，对每个 `url` 只保留协议和主机部分（去除任何 `user:pass@` 和查询字符串），对所有 `args` 条目以及跟在类似凭据的标志后的 `command` 单词进行掩码处理，其余字段（如 `oauth`、`headersHelper` 或未知项）均显示为 `<omitted>`。例如：
  `python3 -c 'import json,sys; from urllib.parse import urlsplit; d=json.load(open(sys.argv[1])); projects=d.get("projects", {}); local=next((s for s in (projects.get(k, {}).get("mcpServers", {}) for k in sys.argv[2:] if k) if s), {}); secretish=lambda a: any(w in str(a).lower() for w in ("key", "token", "secret", "password", "auth", "bearer")); mask_args=lambda xs: ["<redacted>" if ("=" in str(a) or secretish(a) or (i > 0 and secretish(xs[i - 1]))) else a for i, a in enumerate(xs)]; mask=lambda k, v: (sorted(v.keys()) if k in ("env", "headers") and isinstance(v, dict) else f"{urlsplit(v).scheme}://{urlsplit(v).hostname}/..." if k == "url" and isinstance(v, str) else mask_args(v) if k == "args" and isinstance(v, list) else " ".join(str(x) for x in mask_args(v.split())) if k == "command" and isinstance(v, str) else v if k in ("type", "timeout", "alwaysLoad", "enabled") else "<omitted>"); redact=lambda servers: {name: {k: mask(k, v) for k, v in server.items()} for name, server in servers.items()}; print(json.dumps({"user": redact(d.get("mcpServers", {})), "local": redact(local)}, indent=1))' "${CLAUDE_CONFIG_DIR:-$HOME}/.claude.json" "$(dirname "$(git rev-parse --path-format=absolute --git-common-dir 2>/dev/null)")" "$(git rev-parse --show-toplevel 2>/dev/null)" "$PWD"`。
  `projects[...]` 键首先尝试以主工作树根目录为基准，然后是 Git 顶层目录，最后是 `$PWD`，取第一个确实包含服务器的条目（Claude Code 通过主检出记录链接的工作树，以及通过当前目录记录非 Git 目录）。如果该单行脚本本身报错（例如因 `url` 格式错误），则不要回退到读取文件，而是请用户描述或粘贴其 MCP 服务器配置。掩码处理仅为尽力而为：将输出视为用于推理的摘要，绝不可直接回显；若某值仍看起来像凭据，则仅指出其键名并继续。由于完整的 `url`、任何被掩码的 `args` 值以及任何被掩码的 `command` 单词均不在输出中，因此当服务器需要这些信息时，请用户将其粘贴；作为 `args` 值或出现在 `command` 中的键名需与用户确认，绝不可回显。真正的凭据仅可通过下方的设置写入合约或用户粘贴的值进入 Muse，绝不会通过命令输出传递；文件中的其他内容一律不重复。
- 项目级：`<项目根目录>/.mcp.json`，键为 `mcpServers`，通过仓库共享，并受 `projects[...]` 条目中的 `enabledMcpjsonServers` / `disabledMcpjsonServers` 控制。迁移此类配置会将项目服务器加入 Muse 的全局用户设置；添加前应明确告知。
- 跳过 Claude Code 本身未运行的服务器：`enabled: false`、位于 `disabledMcpjsonServers` 列表中的名称，以及受管理策略约束的配置（`/etc/claude-code/managed-mcp.json`、插件缓存）。
- 配置字段：`type`（缺失时默认为 `stdio`，也可为 `http` 或 `streamable-http`、`sse`、`ws`、`sdk`）、`command`、`args`、`env`、`url`、`headers`、`headersHelper`、`oauth`、`timeout`、`alwaysLoad`。值中可包含 `${VAR}` 或 `${VAR:-default}`，Claude Code 会在加载时展开这些变量。

## Codex 存储 MCP 服务器的位置- `$CODEX_HOME/config.toml` 中的 `[mcp_servers.<name>]` 表。受信项目可在 `<root>/.codex/config.toml` 中添加更多配置；位于 `/etc/codex/` 的管理员层级配置不属于用户，会被跳过。
- stdio 字段：`command`、`args`、`env`（字面量映射）、`env_vars`（Codex 从自身环境传递的变量名）、`cwd`。HTTP 字段：`url`、`bearer_token_env_var`、`http_headers`、`env_http_headers`、`http_headers_helper`、`auth`。共用字段：`enabled`、`required`、`startup_timeout_sec`、`tool_timeout_sec`、`enabled_tools`、`disabled_tools`、审批相关设置。将 `enabled = false` 的服务器视为禁用，Codex 也不会启动它。

## 在 Muse 中添加 MCP 服务器

- 文件：`${XDG_CONFIG_HOME:-$HOME/.config}/muse/settings.json`，键名为 `mcpServers`（驼峰命名）。切勿同时写入 `mcp_servers` 键：当两个键同时存在时，加载器会丢弃整个 MCP 配置，导致用户已有的所有服务器停止加载，而其余设置仍会生效。如果文件中已存在旧版的 `mcp_servers` 键，请在添加新服务器之前将其重命名为 `mcpServers`（并保留原有条目）。请先读取该文件；若文件不存在，则会创建为 `{"schema_version": 1, "mcpServers": {...}}`。请保留其他所有键。Muse 已经定义的同名服务器将保持不变。
- 编写规范（与 `manage-settings` 相同）：首先从环境变量中解析出唯一路径，然后使用 `edit_file` 或 `write_file` 进行最小化的 JSON 修改。如果这些工具拒绝访问应用私有路径，则仅当启动时的安全上下文已处于 YOLO 模式（绕过审批、关闭 Shell 沙箱、工作区可信）时，才退回到通过 Shell 手动编辑，并且必须借助 JSON 解析器完成原子替换及重新读取，绝不能使用文本替换。否则应立即停止，并将完整的 `mcpServers` 块直接提供给用户进行粘贴。切勿要求用户为此修改而更改安全模式。
- 条目格式：

  ```json
  "github": {
    "type": "stdio",
    "command": "npx", "args": ["-y", "@modelcontextprotocol/server-github"],
    "env": {"GITHUB_TOKEN": "<字面值>"},
    "mode": "optional"
  }
  ```

  ```json
  "docs": {"type": "streamable-http", "url": "https://example/mcp",
           "headers": {"Authorization": "Bearer <字面值>"}, "mode": "optional"}
  ```- 映射：Claude 的 `stdio`（或简化的 `command`）以及 Codex 的 `command` 将变为
  `type: "stdio"`，并包含 `command`、`args` 和 `env`。Claude 的 `http` /
  `streamable-http` 以及 Codex 的 `url` 将变为 `type: "streamable-http"`，并包含 `url`
  和 `headers`（Codex 的 `http_headers`）。切勿将提取出的实际内容直接写入设置中：
  `<redacted>` 条目、仅保留 `scheme://host/...` 形式的缩短 URL，或仅列出键名的
  `env`/`headers` 列表，都只是摘要，而非具体值。若摘要隐藏了某些信息（遮罩也会
  隐藏诸如 `--keyring` 或 `--config=...` 等无害条目，截断则会丢弃类似 `?tenant=acme`
  的查询参数），应将其视为 `${VAR}` 值：仅按位置或键名向用户询问，待用户提供或
  确认后再写入，并在服务器端省略占位符或缩短后的 URL，而不要直接写入这些内容。
  Codex 的 `enabled` 和 `tool_timeout_sec` 按原名原样复制并生效。Codex 的
  `startup_timeout_sec`、`enabled_tools` 和 `disabled_tools` 同样按原名复制，但目前
  仅存储，尚未强制执行（规范 12857）：Muse 的启动预算固定，所有工具均保持可用，
  因此应在完成报告中将其列为用户仍需检查的设置项。务必添加
  `"mode": "optional"`，以确保在此处失败的服务器不会阻塞 Muse 的启动（Muse 的标准解析器
  会处理 `required`，但启动入口取自类型化的设置载体，该载体仅识别 `mode`——默认为
  required——且会丢弃未知的 `required` 字段，因此仅指定 `required: false` 会使服务器
  仍处于默认的 required 模式；切勿在同一服务器上同时写入 `required` 和 `mode`，因为
  这种歧义别名会导致整个用户 MCP 设置成员被丢弃，而不仅仅是该服务器（规范 12857））。
- Muse 不会展开 `${VAR}`，并在启动 stdio 服务器时仅使用用户环境中的少量固定白名单
  变量（HOME、PATH、USER、LANG、TERM 等）加上字面意义上的 `env` 映射，因此不在该白名单
  内的 Claude `${VAR}` 值，或 Codex 的 `env_vars`、`env_http_headers`、`bearer_token_env_var`
  条目，均无自动等效项。请告知用户每个服务器所需的变量；仅在用户提供或确认后才写入
  具体值，切勿原样回显。若无法确定，则直接跳过该服务器。
- Muse 不支持以下内容，应跳过并予以报告：`sse`、`ws`、`sdk` 传输协议，
  `oauth`、`headersHelper`、`http_headers_helper`、`auth = "oauth"`。删除 Claude 的
  `timeout` 和 `alwaysLoad`，以及 Codex 的 `cwd`，并附上说明。
- 编辑完成后，请重新读取文件以确认已保存的值。更改是永久性的，将在下一次启动 Muse Code
  时生效；当前会话不会加载新服务器。

## 完成报告

- 已读取的源文件（仅路径）及其各自的内容。
- 内存：按作用域记录的笔记及其路径；被跳过的笔记及原因。
- MCP：新增的服务器、被跳过的服务器及原因、用户仍需填写的变量（仅列名称），以及
  Muse 存储但尚未强制执行的字段（`startup_timeout_sec`、`enabled_tools`、`disabled_tools`）。
- 提醒：MCP 更改将在下次启动时生效。
- 确认未修改任何外部文件，也未输出任何敏感信息。