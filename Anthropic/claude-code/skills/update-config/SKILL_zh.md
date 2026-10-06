---
name: update-config
description: '使用此技能可通过 settings.json 配置 Claude Code 框架。自动化行为（“从现在起每当 X 发生时”、“每次 X 时”、“只要 X 发生时”、“在 X 之前/之后”）需要在 settings.json 中配置钩子——这些行为由框架执行，而非 Claude 本身，因此无法通过记忆或偏好来实现。此外，还可用于：权限管理（“允许 X”、“添加权限”、“将权限移至”）、环境变量设置（“将 X 设置为 Y”）、钩子排查，或对 settings.json/settings.local.json 文件的任何修改。示例：“允许运行 npm 命令”、“在全局设置中添加 bq 权限”、“将权限移至用户设置”、“将 DEBUG 设置为 true”、“当 Claude 停止时显示 X”。对于主题、模型等简单设置，建议使用 /config 命令。'
---
# 更新配置技能

通过更新 settings.json 文件来修改 Claude Code 的配置。

## 何时需要使用钩子（而非内存）

如果用户希望在某个事件发生时自动执行某些操作，就需要在 settings.json 中配置一个**钩子**。仅靠内存或偏好设置无法触发自动化行为。

**以下场景需要使用钩子：**
- “压缩前询问要保留的内容” → PreCompact 钩子
- “写入文件后运行 Prettier 格式化工具” → PostToolUse 钩子，匹配 Write|Edit 操作
- “运行 Bash 命令时记录日志” → PreToolUse 钩子，匹配 Bash 操作
- “代码变更后始终运行测试” → PostToolUse 钩子

**可用的钩子事件：** PreToolUse、PostToolUse、PreCompact、PostCompact、Stop、Notification、SessionStart

## 重要提示：先读后写

**在修改设置之前，务必先读取现有的配置文件。** 将新设置与现有设置合并，切勿直接替换整个文件。

## 重要提示：遇到模糊需求时使用 AskUserQuestion

当用户的请求存在歧义时，请使用 AskUserQuestion 来明确：
- 要修改哪个配置文件（用户级、项目级还是本地级）
- 是向现有数组中添加内容，还是完全替换
- 当有多个选项时，具体选择哪些值

## 决策：/config 命令 vs 直接编辑

对于以下简单设置，建议使用 `/config` 斜杠命令：
- `theme`、`editorMode`、`verbose`、`model`
- `language`、`alwaysThinkingEnabled`
- `permissions.defaultMode`

而对于以下情况，则应直接编辑 settings.json：
- 钩子（PreToolUse、PostToolUse 等）
- 复杂的权限规则（allow/deny 数组）
- 环境变量
- MCP 服务器配置
- 插件配置

## 工作流程

1. **明确意图** - 如果请求存在歧义，请先询问用户
2. **读取现有文件** - 使用 Read 工具读取目标配置文件
3. **谨慎合并** - 保留原有设置，尤其是数组中的内容
4. **编辑文件** - 使用 Edit 工具进行修改（若文件不存在，先请用户创建）
5. **确认更改** - 告知用户已修改的内容

## 数组合并（重要！）

在向权限数组或钩子数组中添加内容时，**应与现有内容合并**，而不要直接替换：

**错误做法**（会覆盖原有权限）：
```json
{ "permissions": { "allow": ["Bash(npm *)"] } }
```

**正确做法**（保留原有内容并新增）：
```json
{
  "permissions": {
    "allow": [
      "Bash(git *)",      // 原有
      "Edit(.claude)",    // 原有
      "Bash(npm *)"       // 新增
    ]
  }
}
```

## 配置文件位置

根据作用范围选择合适的文件：

| 文件 | 作用范围 | 是否纳入 Git | 适用场景 |
|------|----------|--------------|----------|
| `~/.claude/settings.json` | 全局 | 不适用 | 所有项目的个人偏好设置 |
| `.claude/settings.json` | 项目级 | 可提交 | 团队范围的钩子、权限、插件配置 |
| `.claude/settings.local.json` | 项目级 | 忽略于 Git | 该项目的个人自定义设置 |

加载顺序为：用户级 → 项目级 → 本地级（后加载的配置会覆盖先加载的配置）。

## 配置模式参考

### 权限设置
```json
{
  "permissions": {
    "allow": ["Bash(npm run test)"]、"Edit(.claude)"、"Read"],
    "deny": ["Bash(rm -rf *)"],
    "ask": ["Edit(//etc/*)"],
    "defaultMode": "default" | "plan" | "acceptEdits" | "dontAsk",
    "additionalDirectories": ["/extra/dir"]
  }
}
```

**权限规则语法：**
- 完全匹配：`"Bash(npm run test)"`
- 前缀通配符：`"Bash(git *)"` - 匹配 `git`、`git status`、`git commit` 等
- 仅指定工具：`"Read"` - 允许所有 Read 操作

### 环境变量
```json
{
  "env": {
    "DEBUG": "true",
    "MY_API_KEY": "value"
  }
}
```

### 模型与代理
```json
{
  "model": "sonnet"、"fable"、"opus"、"haiku" 或完整模型 ID,
  "agent": "agent-name",
  "alwaysThinkingEnabled": true
}
```

### 归属信息（提交与 PR）
```json
{
  "attribution": {
    "commit": "自定义提交尾注",
    "pr": "自定义 PR 描述"
  }
}
```

将 `commit` 或 `pr` 设置为空字符串 `""` 可隐藏归属信息。

### MCP 服务器管理
```json
{
  "enableAllProjectMcpServers": true,
  "enabledMcpjsonServers": ["server1", "server2"],
  "disabledMcpjsonServers": ["blocked-server"]
}
```

### 插件
```json
{
  "enabledPlugins": {
    "formatter@anthropic-tools": true
  }
}
```
插件语法：`plugin-name@source`，其中 source 可以是 `claude-code-marketplace`、`claude-plugins-official` 或 `builtin`。

### 其他设置
- `language`: 首选响应语言（例如：“japanese”）
- `cleanupPeriodDays`: 自动清理前保留对话记录的天数（默认：30 天；最小值：1 天）
- `respectGitignore`: 是否尊重 `.gitignore` 文件（默认：true）
- `spinnerTipsEnabled`: 在加载动画中显示提示
- `timeFormat`: UI 中显示时间的时钟格式：“auto”（默认）、“12-hour”、“24-hour”、“24-hour-utc”，或 strftime 格式字符串，如 “%H:%M”
- `timeZone`: UI 中显示时间的 IANA 时区，例如 “UTC”（默认：系统时区）
- `spinnerVerbs`: 自定义加载动画中的动词（`{ "mode": "append" | "replace", "verbs": [...] }`）
- `spinnerTipsOverride`: 覆盖加载动画提示（`{ "excludeDefault": true, "tips": ["自定义提示"] }`）
- `syntaxHighlightingDisabled`: 禁用 diff 高亮显示


## 钩子配置

钩子会在 Claude Code 生命周期的特定节点执行命令。

### 钩子结构
```json
{
  "hooks": {
    "EVENT_NAME": [
      {
        "matcher": "ToolName|OtherTool",
        "hooks": [
          {
            "type": "command",
            "command": "your-command-here",
            "timeout": 60,
            "statusMessage": "正在运行..."
          }
        ]
      }
    ]
  }
}
```

### 钩子事件

| 事件 | 匹配器 | 用途 |
|-------|---------|---------|
| PermissionRequest | 工具名称 | 权限提示之前执行 |
| PreToolUse | 工具名称 | 工具调用之前执行，可阻止调用 |
| PostToolUse | 工具名称 | 工具成功调用后执行 |
| PostToolUseFailure | 工具名称 | 工具调用失败后执行 |
| Notification | 通知类型 | 发生通知时执行 |
| Stop | - | 当 Claude 停止时执行（包括清屏、恢复、压缩等） |
| PreCompact | "manual"/"auto" | 压缩之前执行 |
| PostCompact | "manual"/"auto" | 压缩之后执行（接收摘要） |
| UserPromptSubmit | - | 用户提交提示时执行 |
| SessionStart | - | 会话开始时执行 |

**常用工具匹配器：** `Bash`、`Write`、`Edit`、`Read`、`Glob`、`Grep`

### 钩子类型

**1. 命令钩子** - 执行一个 shell 命令：
```json
{ "type": "command", "command": "prettier --write $FILE", "timeout": 30 }
```

**2. 提示钩子** - 使用 LLM 对条件进行评估：
```json
{ "type": "prompt", "prompt": "这安全吗？$ARGUMENTS" }
```
仅适用于工具相关事件：PreToolUse、PostToolUse、PermissionRequest。

**3. 代理钩子** - 运行带有工具的代理：
```json
{ "type": "agent", "prompt": "验证测试是否通过：$ARGUMENTS" }
```
仅适用于工具相关事件：PreToolUse、PostToolUse、PermissionRequest。

### 钩子输入（stdin JSON）
```json
{
  "session_id": "abc123",
  "tool_name": "Write",
  "tool_input": { "file_path": "/path/to/file.txt", "content": "..." },
  "tool_response": { "success": true }  // 仅在 PostToolUse 事件中提供
}
```

### 钩子 JSON 输出

钩子可以返回 JSON 来控制行为：

```json
{
  "systemMessage": "在 UI 中向用户显示的警告",
  "continue": false,
  "stopReason": "阻止时显示的消息",
  "suppressOutput": false,
  "decision": "block",
  "reason": "决策原因",
  "hookSpecificOutput": {
    "hookEventName": "PostToolUse",
    "additionalContext": "注入回模型的上下文"
  }
}
```

**字段：**
- `systemMessage` - 向用户显示消息（所有钩子）
- `continue` - 设置为 `false` 以阻止/停止（默认：true）
- `stopReason` - 当 `continue` 为 false 时显示的消息
- `suppressOutput` - 在会话记录中隐藏标准输出（默认：false）
- `decision` - 对于 PostToolUse/Stop/UserPromptSubmit 钩子，值为 "block"（PreToolUse 钩子已弃用，请改用 hookSpecificOutput.permissionDecision）
- `reason` - 决策的解释
- `hookSpecificOutput` - 事件特定的输出（必须包含 `hookEventName`）：
  - `additionalContext` - 注入到模型上下文中的文本
  - `permissionDecision` - "allow"、"deny" 或 "ask"（仅适用于 PreToolUse 钩子）
  - `permissionDecisionReason` - 权限决策的理由（仅适用于 PreToolUse 钩子）
  - `updatedInput` - 修改后的工具输入（仅适用于 PreToolUse 钩子）

### 常见模式

**写入后自动格式化：**
```json
{
  "hooks": {
    "PostToolUse": [{
      "matcher": "Write|Edit",
      "hooks": [{
        "type": "command",
        "command": "jq -r '.tool_response.filePath // .tool_input.file_path' | { read -r f; prettier --write \"$f\"; } 2>/dev/null || true"
      }]
    }]
  }
}
```

**记录所有 Bash 命令：**
```json
{
  "hooks": {
    "PreToolUse": [{
      "matcher": "Bash",
      "hooks": [{
        "type": "command",
        "command": "jq -r '.tool_input.command' >> ~/.claude/bash-log.txt"
      }]
    }]
  }
}
```

**显示消息给用户的停止钩子：**

命令必须输出带有 `systemMessage` 字段的 JSON：
```bash
# 示例命令，输出：{"systemMessage": "会话已完成！"}
echo '{"systemMessage": "会话已完成！"}'
```

**代码变更后运行测试：**
```json
{
  "hooks": {
    "PostToolUse": [{
      "matcher": "Write|Edit",
      "hooks": [{
        "type": "command",
        "command": "jq -r '.tool_input.file_path // .tool_response.filePath' | grep -E '\\.(ts|js)$' && npm test || true"
      }]
    }]
  }
}
```


## 构建一个钩子（含验证）

给定事件、匹配器、目标文件和期望的行为，请按以下流程操作。每一步都能捕获不同类型的错误——一个默默无闻的钩子比没有钩子更糟糕。

1. **去重检查。** 读取目标文件。如果同一事件+匹配器上已存在钩子，则显示现有命令，并询问：保留、替换还是并列添加？

2. **针对本项目构建命令——不要假设。** 钩子从标准输入接收 JSON 数据。构建一个命令，它应该：
   - 安全地提取所需的有效载荷——使用 `jq -r` 将其赋值给带引号的变量，或 `{ read -r f; ... "$f"; }`，切勿使用未加引号的 `| xargs`（会在空格处分割）
   - 以该项目实际运行的方式调用底层工具（npx/bunx/yarn/pnpm？Makefile 目标？全局安装？）
   - 跳过工具无法处理的输入（格式化工具通常有 `--ignore-unknown`；如果没有，则按文件扩展名进行防护）
   - 暂时保持原始状态——先不加 `|| true`，也不抑制标准错误输出。待管道测试通过后再做封装。

3. **对原始命令进行管道测试。** 构造钩子将接收到的标准输入负载，并直接通过管道传递：
   - `Pre|PostToolUse` 针对 `Write|Edit`：`echo '{"tool_name":"Edit","tool_input":{"file_path":"<本仓库中的真实文件>"}}' | <cmd>`
   - `Pre|PostToolUse` 针对 `Bash`：`echo '{"tool_name":"Bash","tool_input":{"command":"ls"}}' | <cmd>`
   - `Stop`/`UserPromptSubmit`/`SessionStart`：大多数命令不读取标准输入，因此只需 `echo '{}' | <cmd>` 即可。

   检查退出码及副作用（文件是否确实被格式化，测试是否确实运行）。若失败，则出现真实错误——修正问题（包管理器不对？工具未安装？jq 路径错误？）并重新测试。成功后，再在外层加上 `2>/dev/null || true`（除非用户希望该检查是阻塞式的）。

4. **编写 JSON。** 合并到目标文件中（JSON 结构请参见上文“钩子结构”部分）。若首次创建 `.claude/settings.local.json`，则将其加入 .gitignore——Write 工具不会自动忽略它。

5. **一次性验证语法与模式：**

   `jq -e '.hooks.<event>[] | select(.matcher == "<matcher>") | .hooks[] | select(.type == "command") | .command' <目标文件>`

   退出码为0且打印出您的命令，即为正确；退出码为4表示匹配器不匹配；退出码为5表示JSON格式错误或嵌套层级不对。如果settings.json文件损坏，该文件中的所有设置都会被静默禁用——请同时修复其中已存在的任何格式问题。

6. **验证钩子是否触发**——仅适用于可通过当前回合触发的`Pre|PostToolUse`事件（例如：通过“编辑”触发`Write|Edit`，通过Bash触发`Bash`）。`Stop`/`UserPromptSubmit`/`SessionStart`等事件在当前回合之外触发，请跳至步骤7。

   对于`PostToolUse`/`Write|Edit`上的格式化程序：通过“编辑”引入一个可检测到的违规（如连续两行空行、缩进错误、缺少分号——即该格式化程序会修正的内容；但不要使用尾随空格，因为“编辑”会在写入前将其去除），然后重新读取内容，确认钩子已将其“修复”。对于其他类型的钩子：暂时在settings.json中相应命令前添加`echo "$(date) hook fired" >> /tmp/claude-hook-check.txt;`，触发匹配的工具（对`Write|Edit`使用“编辑”，对`Bash`使用一个无害的`true`），并检查标记文件。

   **务必清理**——无论验证成功与否，都需恢复违规内容，并移除标记前缀。

   **若验证失败，但管道测试和`jq -e`均通过**：说明设置监听器未监控`.claude/`目录——它只监听本会话启动时已存在设置文件的目录。钩子配置本身是正确的。请告知用户手动打开一次“/hooks”菜单（这会重新加载配置）或重启应用——您无法自行执行此操作；“/hooks”是用户界面菜单，打开后会结束当前回合。

7. **交接。**告知用户钩子已生效（或因监听器的限制需要打开“/hooks”菜单或重启）。引导用户前往“/hooks”查看、编辑或后续禁用该钩子。UI仅在钩子执行出错或运行较慢时显示“已执行N个钩子”；而静默成功的状态则按设计不予显示。

## 示例工作流

### 添加钩子

用户：“让 Claude 写完代码后自动格式化”

1. **明确**：使用哪种格式化工具？（prettier、gofmt 等）
2. **读取**：`.claude/settings.json`（若不存在则创建）
3. **合并**：添加到现有钩子中，不要覆盖
4. **结果**：
```json
{
  "hooks": {
    "PostToolUse": [{
      "matcher": "Write|Edit",
      "hooks": [{
        "type": "command",
        "command": "jq -r '.tool_response.filePath // .tool_input.file_path' | { read -r f; prettier --write \"$f\"; } 2>/dev/null || true"
      }]
    }]
  }
}
```

### 添加权限

用户：“允许运行 npm 命令而不弹出提示”

1. **读取**：现有权限
2. **合并**：将 `Bash(npm *)` 添加到允许列表中
3. **结果**：与现有允许项合并

### 环境变量

用户：“设置 DEBUG=true”

1. **决定**：是全局用户设置还是项目设置？
2. **读取**：目标文件
3. **合并**：添加到 env 对象中
```json
{ "env": { "DEBUG": "true" } }
```

## 需要避免的常见错误

1. **替换而非合并**——始终保留原有设置
2. **写错文件**——当作用域不明确时，请询问用户
3. **JSON 格式错误**——修改后务必验证语法
4. **忘记先读取**——写入前务必先读取

## 钩子排查指南

如果钩子未执行：
1. **检查设置文件**——读取 ~/.claude/settings.json 或 .claude/settings.json
2. **验证 JSON 语法**——无效的 JSON 会导致静默失败
3. **检查匹配器**——是否与工具名称匹配？（如“Bash”、“Write”、“Edit”）
4. **检查钩子类型**——是“command”、“prompt”还是“agent”？
5. **测试命令**——手动运行钩子命令，确认其是否正常工作
6. **启用调试模式**——运行 `claude --debug` 查看钩子执行日志


## 完整的 Settings JSON 模式

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "type": "object",
  "properties": {
    "$schema": {
      "description": "Claude Code 设置的 JSON Schema 引用",
      "type": "string"
    },
    "apiKeyHelper": {
      "description": "输出认证值的脚本路径",
      "type": "string"
    },
    "proxyAuthHelper": {
      "description": "输出 Proxy-Authorization 头字段值（EAP）的 Shell 命令",
      "type": "string"
    },
    "awsCredentialExport": {
      "description": "导出 AWS 凭证的脚本路径",
      "type": "string"
    },
    "awsAuthRefresh": {
      "description": "刷新 AWS 认证的脚本路径",
      "type": "string"
    },
    "gcpAuthRefresh": {
      "description": "用于刷新 GCP 认证的命令（例如：gcloud auth application-default login）",
      "type": "string"
    },
    "processWrapper": {
      "description": "background-agent 监控器及其托管的会话和工作进程，以及 Claude Code 企业启动器文档中列出的其他受管理后台进程所使用的公司级启动器 argv 前缀。等同于 CLAUDE_CODE_PROCESS_WRAPPER 环境变量；当该变量被设置时，优先使用其值。按以下优先级顺序生效：受管理设置、通过 --settings/SDK 提供的设置文件以及用户设置；项目设置和本地设置将被忽略。",
      "type": "string"
    },
    "policyHelper": {
      "description": "在启动时计算受管理设置的可执行程序。仅从管理员控制的策略来源中生效。",
      "type": "object",
      "properties": {
        "path": {
          "description": "辅助可执行程序的绝对路径",
          "type": "string"
        },
        "timeoutMs": {
          "type": "integer",
          "minimum": 1000,
          "maximum": 9007199254740991
        },
        "refreshIntervalMs": {
          "anyOf": [
            {
              "type": "number",
              "const": 0
            },
            {
              "type": "integer",
              "minimum": 60000,
              "maximum": 9007199254740991
            }
          ]
        }
      },
      "required": [
        "path"
      ]
    },
    "fileSuggestion": {
      "description": "@提及时的自定义文件建议配置",
      "type": "object",
      "properties": {
        "type": {
          "type": "string",
          "const": "command"
        },
        "command": {
          "type": "string"
        }
      },
      "required": [
        "type",
        "command"
      ]
    },
    "respectGitignore": {
      "description": "文件选择器是否应尊重 .gitignore 文件（默认：true）。注意：.ignore 文件始终会被尊重。",
      "type": "boolean"
    },
    "cleanupPeriodDays": {
      "description": "聊天记录自动清理前的保留天数（默认：30 天）。最小值为 1 天。如需长期保留，请使用较大值；若要完全禁用会话记录写入，请使用 --no-session-persistence。",
      "type": "integer",
      "exclusiveMinimum": 0,
      "maximum": 9007199254740991
    },
    "desktopSessionCleanupPeriodDays": {
      "description": "由桌面端界面（Claude Desktop、Cowork）创建或最后写入的会话记录的保留期限上限（以天为单位），此类记录不受 cleanupPeriodDays 清理规则约束。默认值为 0，表示无上限：此类记录将一直保留，直到被其他方式删除。与 cleanupPeriodDays 不同，此处允许设置为 0，因为此设置仅限制豁免删除，并不禁止写入。该上限为硬性限制：它同时也设定了主动归档的宽限期，因此发布标记后的宽限期内也不会保留超过上限的文件。当 cleanupPeriodDays 受组织策略管控时，此设置将被忽略。若该上限等于或低于 cleanupPeriodDays，则实际上会取消豁免：这些记录将按照常规的 cleanupPeriodDays 时间表进行老化处理，最终保留期限以两者中较长者为准。",
      "type": "integer",
      "minimum": 0,
      "maximum": 9007199254740991
    },
    "syncClaudeAiSkills": {
      "description": "设置为 false 可关闭 claude.ai 上已启用技能的同步功能。在您的用户设置（或受管理设置）中：不再下载任何内容，之前同步的技能（~/.claude/skills/synced）将无法运行，且在之后启动的所有会话中均被隐藏，并在下次启动时移至 ~/.claude/skills/.trash（将在 cleanupPeriodDays 后被删除；重新启用后会重新下载，但不会恢复）。在 .claude/settings.local.json 或通过 --settings 指定时：仅在该工作区或调用范围内停止下载并屏蔽已同步技能（不会移动）。不读取项目设置（.claude/settings.json）。仅当设置为 false 时才会生效——该功能在服务器端对您的账户始终启用，因此设置为 true 并不能提前开启。当该功能开启时，已同步技能在所有会话中可用，每 10 分钟重新同步一次，您在 claude.ai 上禁用它们后即被移除。仅在使用 Claude 账户登录时适用。",
      "type": "boolean"
    },
    "syncClaudeAiPlugins": {
      "description": "设置为 false 可关闭 claude.ai 上已启用插件的同步功能。在您的用户设置（或受管理设置）中：不再下载任何内容，之前同步的插件（~/.claude/plugins/synced）将在之后启动的所有会话中被隐藏，并在下次启动时移至 ~/.claude/plugins/.trash（将在 cleanupPeriodDays 后被删除；重新启用后会重新下载，但不会恢复）。在 .claude/settings.local.json 或通过 --settings 指定时：仅在该工作区或调用范围内停止下载并屏蔽已同步插件（不会移动）。不读取项目设置（.claude/settings.json）。仅当设置为 false 时才会生效——该功能在服务器端对您的账户始终启用，因此设置为 true 并不能提前开启。当该功能开启时，已同步插件会在每个会话中加载，行为与您自行安装的插件相同（同名插件以您手动安装的为准），每次启动时都会重新同步，您在 claude.ai 上禁用它们后即被移除。仅在使用 Claude 账户登录时适用。",
      "type": "boolean"
    },
    "skillListingMaxDescChars": {
      "description": "发送给 Claude 的技能列表中每项技能描述的最大字符数（默认：1536）。超过此长度的描述将被截断。提高此值将导致每轮对话的上下文成本增加。",
      "type": "integer",
      "exclusiveMinimum": 0,
      "maximum": 9007199254740991
    },
    "skillListingBudgetFraction": {
      "description": "为发送给 Claude 的技能列表预留的上下文窗口比例（以字符数计，默认：0.01 = 1%）。当列表超出此比例时，描述将被缩短以适应。提高此值将导致每轮对话的上下文成本增加。",
      "type": "number",
      "exclusiveMinimum": 0,
      "maximum": 1
    },
    "wslInheritsWindowsSettings": {
      "description": "当在仅限管理员访问的 Windows 来源中设置为 true 时——即 HKLM SOFTWARE/Policies/ClaudeCode 注册表项或 C:/Program Files/ClaudeCode/managed-settings.json——WSL 除了读取 /etc/claude-code 外，还会从完整的 Windows 策略链（HKLM、C:/Program Files/ClaudeCode 通过 DrvFs、HKCU）中读取受管理设置。Windows 来源具有优先权。此外，在 HKCU 中也必须设置此标志，才能使 HKCU 策略在 WSL 上生效（双重确认：管理员启用策略链，用户确认 HKCU）。在原生 Windows 系统上，此标志无效。",
      "type": "boolean"
    },
    "env": {
      "description": "为 Claude Code 会话设置的环境变量",
      "type": "object",
      "propertyNames": {
        "type": "string"
      },
      "additionalProperties": {
        "type": "string"
      }
    },
    "attribution": {
      "description": "自定义提交和 PR 的署名文本。若未设置，各字段将采用标准的 Claude Code 署名格式。"      "type": "object",
      "properties": {
        "commit": {
          "description": "用于 Git 提交的署名文本，包括所有尾注。空字符串将隐藏署名。",
          "type": "string"
        },
        "pr": {
          "description": "用于拉取请求描述的署名文本。空字符串将隐藏署名。",
          "type": "string"
        },
        "sessionUrl": {
          "description": "是否在通过网页或远程控制会话创建的提交和拉取请求中附加 claude.ai 会话链接（默认：true）。设置为 false 可省略 Claude-Session 尾注和 PR 正文中的链接。",
          "type": "boolean"
        }
      },
      "additionalProperties": {}
    },
    "includeCoAuthoredBy": {
      "description": "已弃用：请改用 attribution。是否在提交和拉取请求中包含 Claude 的共同作者署名（默认为 true）。",
      "type": "boolean"
    },
    "includeGitInstructions": {
      "description": "在 Claude 的系统提示中包含内置的提交与拉取请求工作流说明（默认：true）。",
      "type": "boolean"
    },
    "permissions": {
      "description": "工具使用权限配置",
      "type": "object",
      "properties": {
        "allow": {
          "description": "允许操作的权限规则列表。",
          "type": "array",
          "items": {
            "type": "string"
          }
        },
        "deny": {
          "description": "禁止操作的权限规则列表。",
          "type": "array",
          "items": {
            "type": "string"
          }
        },
        "ask": {
          "description": "始终需要确认的权限规则列表。",
          "type": "array",
          "items": {
            "type": "string"
          }
        },
        "defaultMode": {
          "description": "当 Claude Code 需要访问时的默认权限模式（'manual' 可作为 'default' 的别名）。",
          "type": "string",
          "enum": [
            "acceptEdits",
            "auto",
            "bypassPermissions",
            "default",
            "dontAsk",
            "plan"
          ]
        },
        "disableBypassPermissionsMode": {
          "description": "禁用绕过权限提示的功能。",
          "type": "string",
          "enum": [
            "disable"
          ]
        },
        "blockReadsOutsideWorkingDirectories": {
          "description": "在任何权限模式下，均拒绝文件工具在工作目录之外的读取操作（Read、Grep、Glob、LSP）；只要任一设置来源将其设为 true 即生效。此外，当用户在首次出现的工作目录外读取提示中选择“阻止”时也会启用此选项。",
          "type": "boolean"
        },
        "disableAutoMode": {
          "description": "禁用自动模式。",
          "type": "string",
          "enum": [
            "disable"
          ]
        },
        "additionalDirectories": {
          "description": "纳入权限范围的额外目录列表。",
          "type": "array",
          "items": {
            "type": "string"
          }
        }
      },
      "additionalProperties": {}
    },
    "model": {
      "description": "覆盖 Claude Code 默认使用的模型。",
      "type": "string"
    },
    "fallbackModel": {
      "description": "当主模型过载或不可用时按顺序尝试的备用模型。每个元素可接受模型名称或别名；“default”会展开为默认模型。CLI 中的 --fallback-model 优先级更高。",
      "type": "array",
      "items": {
        "type": "string"
      }
    },
    "availableModels": {
      "description": "允许用户选择的模型白名单。支持模型系列别名（如“opus”允许任意 opus 版本）、版本前缀（如“opus-4-5”仅允许该版本）以及完整模型 ID。若未定义，则所有模型均可选；若为空数组，则仅默认模型可用。通常由企业管理员在托管设置中配置。",
      "type": "array",
      "items": {
        "type": "string"
      }
    },
    "enforceAvailableModels": {
      "description": "当此选项为 true 且 availableModels 为非空数组时，默认模型的选择也将受到限制：如果用户层级的默认模型不在 availableModels 中，则默认模型将改为指向 availableModels 中的第一个允许项。若 availableModels 未设置或为空数组，则此选项无效。通常由企业管理员在托管设置中配置。",
      "type": "boolean"
    },
    "modelOverrides": {
      "description": "覆盖从 Anthropic 模型 ID（如“claude-opus-4-6”）到特定提供商模型 ID（如 Bedrock 推理配置 ARN）的映射。通常由企业管理员在托管设置中配置。",
      "type": "object",
      "propertyNames": {
        "type": "string"
      },
      "additionalProperties": {
        "type": "string"
      }
    },
    "modelPicker": {
      "description": "自定义 /model 选择器：提供一份带有自定义标签的有序模型列表，独立于内置模型阵容及 Claude Code 的版本发布。availableModels 仍适用于这些行。仅在托管设置、--settings/SDK 和用户设置中生效（不适用于项目检出）；具有最高优先级的 modelPicker 定义将完全覆盖其他来源（各来源之间不会合并）。通常由企业管理员在托管设置中配置。",
      "type": "object",
      "properties": {
        "options": {
          "description": "以指定顺序显示在 /model 选择器中的行。",
          "type": "array",
          "items": {
            "type": "object",
            "properties": {
              "model": {
                "description": "要选择的模型，原样输入：可以是别名（如“opus”）、Anthropic 模型 ID 或提供商格式的 ID（Vertex、Bedrock、网关）。与 --model 接受的值相同。",
                "type": "string"
              },
              "label": {
                "description": "行标题。默认为模型名称。",
                "type": "string"
              },
              "description": {
                "description": "行副标题。默认为通用描述。",
                "type": "string"
              },
              "behavesAs": {
                "description": "对于当前版本 Claude Code 不认识的模型：指定一个它认识的模型 ID（如“claude-opus-4-8”），其客户端侧的处理方式——提示配置、能力与努力的默认值——将被应用于该模型。这不会改变行的标签或发送的模型 ID。若未指定，对于当前版本未知的模型，除非 Claude Code 更新，否则不会在模型列表中显示。",
                "type": "string"
              }
            },
            "required": [
              "model"
            ]
          }
        },
        "replaceBuiltInOptions": {
          "description": "当为 true 时，选择器仅显示默认行和这些选项——内置阵容、网关发现的模型以及 ANTHROPIC_CUSTOM_MODEL_OPTION 将被隐藏。当为 false 或未设置时，这些选项将在内置阵容之后添加。",
          "type": "boolean"
        }
      },
      "required": [
        "options"
      ]
    },
    "modelPricing": {
      "description": "按照贵组织的合同费率而非标价计算费用。影响 Claude Code 报告的所有支出数据——/cost、状态栏、SDK 的 total_cost_usd、--max-budget-usd，以及 OpenTelemetry 的成本指标和事件——这些数据仍为美元估算值，而非发票（/model 中的每 Mtok 价格标签仍为标价）。"overrides" 将模型 ID 映射为其每百万 token 的美元费率（输入、输出、缓存读取、缓存写入——四项均需填写，每项范围 0 至 10000；缓存写入同时计费 5 分钟与 1 小时的写入）。匹配的行将按所列金额精确收费；不会额外收取快速模式或美国数据驻留的附加费用。一个关键 Claude代码本身使用内置模型的 ID，例如“claude-sonnet-4-6”，或其第一方、Bedrock（可带任意或不带区域前缀）、Vertex 或 Foundry 的 ID，这些 ID 覆盖了该模型的所有版本和所有提供商形式；任何其他键——网关模型别名，或 Claude Code 本身未使用的拼写——仅与该模型 ID 匹配（不区分大小写），且此类精确匹配优先于内置行。在 Bedrock 上，应用推理配置文件与其底层模型匹配。无效的行或乘数将被报告并跳过；其余规则仍然适用。“multiplier”位于 (0, 10] 范围内，会按比例调整所有计算出的成本，无论是否已被覆盖（0.85 表示价格的 85%，1.2 表示 120%）。此设置仅在受管理的配置中生效（由服务器管理、MDM/操作系统策略或 managed-settings.json），或者——当上述来源均未设置时——由负责管理模型提供商的宿主应用提供；在用户、项目、本地及 --settings 来源中则会被忽略。
      “type”: “object”,
      “properties”: {
        “multiplier”: {
          “type”: “number”,
          “exclusiveMinimum”: 0,
          “maximum”: 10
        },
        “overrides”: {
          “type”: “object”,
          “propertyNames”: {
            “type”: “string”
          },
          “additionalProperties”: {
            “type”: “object”,
            “properties”: {
              “input”: {
                “type”: “number”，
                “minimum”: 0，
                “maximum”: 10000
              },
              “output”: {
                “type”: “number”，
                “minimum”: 0，
                “maximum”: 10000
              },
              “cacheRead”: {
                “type”: “number”，
                “minimum”: 0，
                “maximum”: 10000
              },
              “cacheWrite”: {
                “type”: “number”，
                “minimum”: 0，
                “maximum”: 10000
              }
            },
            “required”: [
              “input”，
              “output”，
              “cacheRead”，
              “cacheWrite”
            ]
          }
        }
      }
    },
    “enableAllProjectMcpServers”: {
      “description”: “是否自动批准项目中的所有 MCP 服务器”，
      “type”: “boolean”
    },
    “enabledMcpjsonServers”: {
      “description”: “来自 .mcp.json 的已批准 MCP 服务器列表”，
      “type”: “array”，
      “items”: {
        “type”: “string”
      }
    },
    “disabledMcpjsonServers”: {
      “description”: “来自 .mcp.json 的被拒绝 MCP 服务器列表”，
      “type”: “array”，
      “items”: {
        “type”: “string”
      }
    },
    “disableClaudeAiConnectors”: {
      “description”: “当任一设置来源为真时，claude.ai 的 MCP 云连接器将不会被自动获取或连接。只有通过显式方式传递的网关自动获取的连接器（例如通过 --mcp-config 或 SDK 的 mcpServers 选项）仍遵循正常的 MCP 配置信任流程。任何来源为真即生效：项目可以选择退出，但项目级别的假无法覆盖用户级别的真。”，
      “type”: “boolean”
    },
    “skillOverrides”: {
      “description”: “按技能名称键入的每项技能列表覆盖。‘name-only’仅列出技能名称而不显示描述；‘user-invocable-only’对模型隐藏该技能但保留 /name；‘off’则两者均隐藏。若未设置，则默认为启用。”，
      “type”: “object”，
      “propertyNames”: {
        “type”: “string”
      },
      “additionalProperties”: {
        “type”: “string”，
        “enum”: [
          “on”，
          “name-only”，
          “user-invocable-only”，
          “off”
        ]
      }
    },
    “disableBundledSkills”: {
      “description”: “禁用随 Claude Code 一同提供的技能和工作流：捆绑的技能和工作流将被完全移除；内置的斜杠命令仍可输入，但对模型不可见。插件、.claude/skills/ 和 .claude/commands/ 不受影响。等同于 CLAUDE_CODE_DISABLE_BUNDLED_SKILLS=1。”，
      “type”: “boolean”
    },
    “managedMcpServers”: {
      “description”: “组织为每位用户提供的一组 MCP 服务器，以服务器名称为键，每个条目格式与 .mcp.json 相同；仅接受 ‘http’ 和 ‘sse’ 类型的服务器（不接受指定运行程序的配置，也不允许使用 ${VAR} 引用）。此设置仅在受管理的配置中生效；用户无法移除这些服务器，deniedMcpServers 依然适用，且无需在 allowedMcpServers 中另行添加。在第三方部署的 Claude Desktop 的 Code 标签页或 Cowork 会话中，此设置不会被读取，因为此时 Claude Desktop 会自行提供并锁定会话的 MCP 服务器。”，
      “type”: “object”，
      “propertyNames”: {
        “type”: “string”
      },
      “additionalProperties”: {
        “type”: “object”，
        “propertyNames”: {
          “type”: “string”
        },
        “additionalProperties”：{}
      }
    },
    “allowedMcpServers”: {
      “description”: “企业允许用户使用的 MCP 服务器白名单。该设置适用于用户自行添加的服务器（用户、项目和本地配置，--mcp-config，代理前端信息，插件，以及 claude.ai 连接器）；企业直接提供的服务器（managedMcpServers，以及不使用 ${VAR} 扩展的 managed-mcp.json 条目）无需列入白名单即可使用；而使用 ${VAR} 扩展的 managed-mcp.json 条目仍需在此列表中进行核验。若未定义，则所有服务器均被允许。若为空数组，则用户不得使用任何自建服务器。黑名单优先——若某服务器同时出现在黑白名单中，则按黑名单处理。”，
      “type”: “array”，
      “items”: {
        “type”: “object”，
        “properties”: {
          “serverName”: {
            “description”: “允许用户配置的 MCP 服务器名称”，
            “type”: “string”，
            “pattern”: “^[a-zA-Z0-9_-]+$”
          },
          “serverCommand”: {
            “description”: “用于匹配允许的 stdio 服务器的命令数组 [command, ...args]，必须完全一致”，
            “minItems”: 1，
            “type”: “array”，
            “items”: {
              “type”: “string”
            }
          },
          “serverUrl”: {
            “description”: “支持通配符的 URL 模式（如 ‘https://*.example.com/*’），用于允许的远程 MCP 服务器”，
            “type”: “string”
          }
        }
      }
    },
    “deniedMcpServers”: {
      “description”: “企业明确禁止的 MCP 服务器黑名单。若某服务器被列入黑名单，则在包括企业在内的所有范围内均被禁止。黑名单优先于白名单——若某服务器同时出现在黑白名单中，则按黑名单处理。”，
      “type”: “array”，
      “items”: {
        “type”: “object”，
        “properties”: {
          “serverName”: {
            “description”: “被明确禁止的 MCP 服务器名称”，
            “type”: “string”，
            “minLength”: 1
          },
          “serverCommand”: {
            “description”: “用于匹配被禁止的 stdio 服务器的命令数组 [command, ...args]，必须完全一致”，
            “minItems”: 1，
            “type”: “array”，
            “items”: {
              “type”: “string”
            }
          },
          “serverUrl”: {
            “description”: “支持通配符的 URL 模式（如 ‘https://*.example.com/*’），用于被禁止的远程 MCP 服务器”，
            “type”: “string”
          }
        }
      }
    },
    “hooks”: {
      “description”: “在工具执行前后运行的自定义命令”，
      “type”: “object”，
      “propertyNames”: {
        “type”: “string”，
        “enum”: [
          “PreToolUse”，
          “PostToolUse”，
          “PostToolUseFailure”，
          “PostToolBatch”，
          “Notification”，
          “UserPromptSubmit”，
          “UserPromptExpansion”，
          “SessionStart”，
          “SessionEnd”，
          “Stop”，
          “StopFailure”，
          “SubagentStart”，
          “SubagentStop”，
          “PreCompact”，
          “PostCompact”，
          “PreModelSwitch”，
          “PostModelSwitch”，
          “PermissionRequest”，
          “PermissionDenied”，
          “Setup”，
          “TeammateIdle”，
          "任务已创建",
          "任务已完成",
          "需求获取",
          "需求获取结果",
          "配置变更",
          "创建工作树",
          "移除工作树",
          "指令已加载",
          "当前目录已更改",
          "文件已更改",
          "目录已添加",
          "消息显示"
        ]
      },
      "额外属性": {
        "类型": "数组",
        "项": {
          "类型": "对象",
          "属性": {
            "匹配器": {
              "描述": "用于匹配的字符串模式（例如工具名称，如“Write”）",
              "类型": "字符串"
            },
            "钩子": {
              "描述": "当匹配器匹配时要执行的一系列钩子",
              "类型": "数组",
              "项": {
                "任意之一": [
                  {
                    "类型": "对象",
                    "属性": {
                      "类型": {
                        "描述": "Shell 命令类型的钩子",
                        "类型": "字符串",
                        "常量值": "command"
                      },
                      "命令": {
                        "描述": "要执行的 Shell 命令",
                        "类型": "字符串"
                      },
                      "参数列表": {
                        "描述": "用于 exec 形式的参数列表。当此字段存在时，`command` 会被解析为可执行文件，并直接使用这些参数启动——不经过 Shell。路径占位符（如 ${CLAUDE_PLUGIN_ROOT}）会按元素逐个替换为普通字符串，因此带有引号、$ 或反引号的路径不会进入 Shell 解析器。当此字段不存在时，`command` 会通过 Shell 执行（POSIX 系统上为 bash，Windows 上在没有 Git Bash 的情况下为 PowerShell）。",
                        "类型": "数组",
                        "项": {
                          "类型": "字符串"
                        }
                      },
                      "条件": {
                        "描述": "权限规则语法，用于筛选该钩子何时运行（例如“Bash(git *)”）。仅当工具调用与模式匹配时才会运行，避免为不匹配的命令启动钩子。",
                        "类型": "字符串"
                      },
                      "Shell 解释器": {
                        "描述": "Shell 解释器。'bash' 使用您的 $SHELL 环境变量（bash/zsh/sh）；'powershell' 使用 pwsh。默认为 bash（在没有 Git Bash 的 Windows 上则为 PowerShell）。",
                        "类型": "字符串",
                        "枚举值": [
                          "bash",
                          "powershell"
                        ]
                      },
                      "超时时间": {
                        "描述": "该特定命令的超时时间，单位为秒",
                        "类型": "数字",
                        "严格大于0"
                      },
                      "状态消息": {
                        "描述": "在钩子运行期间于加载动画中显示的自定义状态信息",
                        "类型": "字符串"
                      },
                      "仅执行一次": {
                        "描述": "如果为真，钩子仅执行一次并在执行后被移除",
                        "类型": "布尔值"
                      },
                      "异步执行": {
                        "描述": "如果为真，钩子将在后台运行且不阻塞主线程",
                        "类型": "布尔值"
                      },
                      "异步唤醒": {
                        "描述": "如果为真，钩子将在后台运行，并在退出码为2（阻塞型错误）时唤醒模型。同时隐含异步执行。",
                        "类型": "布尔值"
                      }
                    },
                    "必填项": [
                      "类型",
                      "命令"
                    ]
                  },
                  {
                    "类型": "对象",
                    "属性": {
                      "类型": {
                        "描述": "LLM 提示词类型的钩子",
                        "类型": "字符串",
                        "常量值": "prompt"
                      },
                      "提示词": {
                        "描述": "要提交给 LLM 进行评估的提示语。请使用 $ARGUMENTS 占位符来表示钩子输入的 JSON 数据。",
                        "类型": "字符串"
                      },
                      "条件": {
                        "描述": "权限规则语法，用于筛选该钩子何时运行（例如“Bash(git *)”）。仅当工具调用与模式匹配时才会运行，避免为不匹配的命令启动钩子。",
                        "类型": "字符串"
                      },
                      "超时时间": {
                        "描述": "该特定提示词评估的超时时间，单位为秒",
                        "类型": "数字",
                        "严格大于0"
                      },
                      "使用的模型": {
                        "描述": "用于该提示词钩子的模型（例如“claude-sonnet-5”）。若未指定，则使用默认的小型快速模型。",
                        "类型": "字符串"
                      },
                      "遇到阻塞时继续": {
                        "描述": "设置当决策结果为“阻塞”时的继续值。默认为假（回合结束）。是否允许继续取决于事件本身的“阻塞”语义。在 PostToolUse 事件中，原因会反馈给 Claude 并继续本轮对话。",
                        "类型": "布尔值"
                      },
                      "状态消息": {
                        "描述": "在钩子运行期间于加载动画中显示的自定义状态信息",
                        "类型": "字符串"
                      },
                      "仅执行一次": {
                        "描述": "如果为真，钩子仅执行一次并在执行后被移除",
                        "类型": "布尔值"
                      }
                    },
                    "必填项": [
                      "类型",
                      "提示词"
                    ]
                  },
                  {
                    "类型": "对象",
                    "属性": {
                      "类型": {
                        "描述": "代理验证类型的钩子",
                        "类型": "字符串",
                        "常量值": "agent"
                      },
                      "提示词": {
                        "描述": "用于描述需要验证内容的提示语（例如“验证单元测试已运行并全部通过”）。请使用 $ARGUMENTS 占位符来表示钩子输入的 JSON 数据。",
                        "类型": "字符串"
                      },
                      "条件": {
                        "描述": "权限规则语法，用于筛选该钩子何时运行（例如“Bash(git *)”）。仅当工具调用与模式匹配时才会运行，避免为不匹配的命令启动钩子。",
                        "类型": "字符串"
                      },
                      "超时时间": {
                        "描述": "代理执行的超时时间，单位为秒（默认60秒）",
                        "类型": "数字",
                        "严格大于0"
                      },
                      "使用的模型": {
                        "描述": "用于该代理钩子的模型（例如“claude-sonnet-5”）。若未指定，则使用 Haiku 模型。",
                        "类型": "字符串"
                      },
                      "状态消息": {
                        "描述": "在钩子运行期间于加载动画中显示的自定义状态信息",
                        "类型": "字符串"
                      },
                      "仅执行一次": {
                        "描述": "如果为真，钩子仅执行一次并在执行后被移除",
                        "类型": "布尔值"
                      }
                    },
                    "required": [
                      "type",
                      "prompt"
                    ]
                  },
                  {
                    "type": "object",
                    "properties": {
                      "type": {
                        "description": "HTTP 钩子类型",
                        "type": "string",
                        "const": "http"
                      },
                      "url": {
                        "description": "用于 POST 钩子输入 JSON 的 URL",
                        "type": "string",
                        "format": "uri"
                      },
                      "if": {
                        "description": "用于过滤该钩子何时触发的权限规则语法（例如：\"Bash(git *)\"）。仅当工具调用匹配该模式时才会触发，避免为不匹配的命令创建钩子。",
                        "type": "string"
                      },
                      "timeout": {
                        "description": "此特定请求的超时时间，单位为秒",
                        "type": "number",
                        "exclusiveMinimum": 0
                      },
                      "headers": {
                        "description": "要包含在请求中的额外头信息。值中可使用 $VAR_NAME 或 ${VAR_NAME} 语法引用环境变量（例如：\"Authorization\": \"Bearer $MY_TOKEN\"）。仅会替换 allowedEnvVars 列表中列出的变量。",
                        "type": "object",
                        "propertyNames": {
                          "type": "string"
                        },
                        "additionalProperties": {
                          "type": "string"
                        }
                      },
                      "allowedEnvVars": {
                        "description": "可在头信息值中进行插值的环境变量名称显式列表。只有此处列出的变量会被解析；其他所有 $VAR 引用将被替换为空字符串。启用环境变量插值时必需。",
                        "type": "array",
                        "items": {
                          "type": "string"
                        }
                      },
                      "statusMessage": {
                        "description": "钩子运行期间在加载动画中显示的自定义状态消息",
                        "type": "string"
                      },
                      "once": {
                        "description": "如果为 true，钩子仅执行一次并在执行后被移除",
                        "type": "boolean"
                      }
                    },
                    "required": [
                      "type",
                      "url"
                    ]
                  },
                  {
                    "type": "object",
                    "properties": {
                      "type": {
                        "description": "MCP 工具钩子类型",
                        "type": "string",
                        "const": "mcp_tool"
                      },
                      "server": {
                        "description": "要调用的已配置 MCP 服务器名称",
                        "type": "string"
                      },
                      "tool": {
                        "description": "该服务器上要调用的工具名称",
                        "type": "string"
                      },
                      "input": {
                        "description": "传递给 MCP 工具的参数。字符串值支持从钩子输入 JSON 中进行 ${path} 插值（例如：\"${tool_input.file_path}\"）。",
                        "type": "object",
                        "propertyNames": {
                          "type": "string"
                        },
                        "additionalProperties": {}
                      },
                      "if": {
                        "description": "用于过滤该钩子何时触发的权限规则语法（例如：\"Bash(git *)\"）。仅当工具调用匹配该模式时才会触发，避免为不匹配的命令创建钩子。",
                        "type": "string"
                      },
                      "timeout": {
                        "description": "此特定工具调用的超时时间，单位为秒",
                        "type": "number",
                        "exclusiveMinimum": 0
                      },
                      "statusMessage": {
                        "description": "钩子运行期间在加载动画中显示的自定义状态消息",
                        "type": "string"
                      },
                      "once": {
                        "description": "如果为 true，钩子仅执行一次并在执行后被移除",
                        "type": "boolean"
                      }
                    },
                    "required": [
                      "type",
                      "server",
                      "tool"
                    ]
                  }
                ]
              }
            }
          },
          "required": [
            "hooks"
          ]
        }
      }
    },
    "worktree": {
      "description": "Git 工作树配置：CLI 的 --worktree 标志、EnterWorktree 和代理隔离，以及 Claude Code Desktop 在本机上用于 SSH 会话工作树的位置。",
      "type": "object",
      "properties": {
        "symlinkDirectories": {
          "description": "为了避免磁盘空间膨胀，需要从主仓库符号链接到工作树的目录。必须显式配置——默认情况下不会符号链接任何目录。常见示例：\"node_modules\"、\".cache\"、\".bin\"",
          "type": "array",
          "items": {
            "type": "string"
          }
        },
        "sparsePaths": {
          "description": "通过 git sparse-checkout（锥形模式）在创建工作树时包含的目录。在大型 monorepo 中速度显著提升——只有列出的路径会被写入磁盘。",
          "type": "array",
          "items": {
            "type": "string"
          }
        },
        "baseRef": {
          "description": "新工作树基于哪个引用分支。'fresh'（默认）基于 origin/<默认分支>，以获得一个干净的工作树。'head' 基于当前本地 HEAD，因此未推送的提交和特性分支状态都会保留。适用于 --worktree、EnterWorktree 和代理隔离。",
          "type": "string",
          "enum": [
            "fresh",
            "head"
          ]
        },
        "bgIsolation": {
          "description": "此仓库中后台会话的隔离模式。'worktree'（默认）在调用 EnterWorktree 之前会阻止对主工作区的编辑和写入操作。'none' 允许后台任务直接编辑工作副本。",
          "type": "string",
          "enum": [
            "worktree",
            "none"
          ]
        },
        "location": {
          "description": "Claude Code Desktop 在本机上创建 SSH 会话工作树的目录位置（绝对路径或以 ~/ 开头的路径），而不是 <项目>/.claude/worktrees。桌面应用会从 SSH 主机用户设置中读取该位置；桌面应用的 SSH 连接设置中选择的路径具有优先权。CLI（--worktree、EnterWorktree、代理隔离）目前尚未读取此设置。",
          "type": "string"
        }
      }
    },
    "disableAllHooks": {
      "description": "禁用所有钩子和 statusLine 的执行",
      "type": "boolean"
    },
    "disableAgentView": {
      "description": "禁用代理视图（`claude agents`、`--bg`、/background、按需守护进程）。通常在托管设置中启用。等同于 CLAUDE_CODE_DISABLE_AGENT_VIEW=1。",
      "type": "boolean"
    },
    "disableRemoteControl": {
      "description": "禁用远程控制（claude.ai/code、`claude remote-co`ntrol`、`--remote-control`/`--rc`、自动启动以及会话内切换）。通常在托管设置中进行配置。",
      "type": "boolean"
    },
    "disableWorkflows": {
      "description": "禁用工作流功能（也可通过 CLAUDE_CODE_DISABLE_WORKFLOWS 环境变量控制）。",
      "type": "boolean"
    },
    "disableArtifact": {
      "description": "已弃用：请使用 enableArtifact: false。该选项仍会被识别——设置为 true 时禁用 Artifact 工具；设置为 false 时则被忽略。",
      "type": "boolean"
    },
    "enableArtifact": {
      "description": "启用或禁用 Artifact 工具。若在托管设置、命令行参数 `--settings` 或用户设置中任一处将其关闭，则优先生效；项目级和本地设置仅能将其关闭。在功能可用后，未设置时默认为开启。",
      "type": "boolean"
    },
    "enableWorkflows": {
      "description": "为此用户启用或禁用工作流功能。未设置时，在功能可用后将按其所在套餐的默认值决定。",
      "type": "boolean"
    },
    "workflowSizeGuideline": {
      "description": "Claude 自动生成的工作流的建议规模指南：\"small\" 目标是少于 5 个代理，\"medium\" 少于 10 个，\"large\" 少于 50 个，\"unrestricted\" 则不设限制。未设置时默认为 \"medium\"，Pro 套餐下默认为 \"small\"。此处设置的值（包括来自托管设置的值）优先于 `/config` 中的“动态工作流规模”选项，并且当设置文件提供该键时，`/config` 中的对应行将被隐藏。这仅为建议，并非强制性限制。",
      "type": "string",
      "enum": [
        "unrestricted",
        "small",
        "medium",
        "large"
      ]
    },
    "workflowKeywordTriggerEnabled": {
      "description": "启用“ultracode”关键词触发器：在提示词中包含该关键词即可激活工作流工具。设置为 false 可禁用该触发器。默认值：true。",
      "type": "boolean"
    },
    "disableSkillShellExecution": {
      "description": "禁用来自用户、项目或插件来源的技能及自定义斜杠命令中的内联 Shell 执行。命令将被占位符替代，而不会被执行。",
      "type": "boolean"
    },
    "defaultShell": {
      "description": "输入框中 ! 命令的默认 Shell。所有平台上默认为 'bash'（无 Windows 自动切换）。",
      "type": "string",
      "enum": [
        "bash",
        "powershell"
      ]
    },
    "bashEditDiffEnabled": {
      "description": "Bash 工具是否显示 Bash 命令所修改文件的差异（PostToolUse Bash 钩子会在 tool_response 中获取被修改文件的列表）。设置为 false 可关闭此功能。默认值：当 Bash 工具处理文件编辑时为开启。除自动模式和 bypassPermissions 模式外，仅用户设置、命令行标志或策略设置可将其开启。",
      "type": "boolean"
    },
    "bashOutputMaxChars": {
      "description": "成功执行的 Bash 或 PowerShell 命令，Claude 内联接收的输出字符数上限（默认 30000；取值范围为 4000–128000）。超出部分将保存至文件，Claude 仅接收简短预览及文件路径。设置后，该值还将取代 BASH_MAX_OUTPUT_LENGTH，后者仅用于限制回读窗口的大小。",
      "type": "integer",
      "exclusiveMinimum": 0,
      "maximum": 9007199254740991
    },
    "taskOutputMaxChars": {
      "description": "TaskOutput 工具内联传递给 Claude 的后台任务输出字符数上限（默认 32000；取值范围为 4000–128000）。超过部分将截取最新内容，并附上完整输出文件的路径；仍在运行的 Shell 命令则返回截至该长度的前部分内容。设置后，该值还将取代 TASK_MAX_OUTPUT_LENGTH，后者仅用于限制该窗口的大小。",
      "type": "integer",
      "exclusiveMinimum": 0,
      "maximum": 9007199254740991
    },
    "respondToBashCommands": {
      "description": "输入框中 ! bash 命令执行后，Claude 是否作出响应。设置为 false 时，仅将命令输出加入上下文而不生成回复。默认值：true。",
      "type": "boolean"
    },
    "allowManagedHooksOnly": {
      "description": "当设置为 true 且在托管设置中配置时，仅执行来自托管设置的钩子，用户、项目及本地钩子将被忽略。",
      "type": "boolean"
    },
    "allowedHttpHookUrls": {
      "description": "HTTP 钩子可访问的 URL 模式白名单。支持使用 * 作为通配符（如 \"https://hooks.example.com/*\"）。设置后，URL 不匹配的 HTTP 钩子将被阻止。未设置时，默认允许所有 URL；设置为空数组时，则禁止所有 HTTP 钩子。多个设置源中的数组会合并（语义与 allowedMcpServers 相同）。",
      "type": "array",
      "items": {
        "type": "string"
      }
    },
    "httpHookAllowedEnvVars": {
      "description": "HTTP 钩子可在请求头中插入的环境变量名称白名单。设置后，每个钩子实际允许使用的环境变量集为该列表的交集。未设置时，则不限制。多个设置源中的数组会合并（语义与 allowedMcpServers 相同）。",
      "type": "array",
      "items": {
        "type": "string"
      }
    },
    "allowManagedPermissionRulesOnly": {
      "description": "当设置为 true 且在托管设置中配置时，用户、项目、本地及 --settings 文件中的权限规则，以及 --allowedTools 中的允许规则均被忽略；只有托管设置可通过 settings.json 添加允许规则。命令行或当前会话中的 --disallowedTools 及其他拒绝或询问规则仍然有效。",
      "type": "boolean"
    },
    "allowManagedMcpServersOnly": {
      "description": "当设置为 true 且在托管设置中配置时，allowedMcpServers 仅从托管设置中读取。deniedMcpServers 仍会从所有来源合并，因此用户仍可自行屏蔽某些服务器。用户仍可添加自己的 MCP 服务器，但仅适用管理员定义的白名单。",
      "type": "boolean"
    },
    "allowAllClaudeAiMcps": {
      "description": "当设置为 true 且在托管设置中配置时，claude.ai 云端的 MCP 连接器将与 managed-mcp.json 一同加载，而不会因其独占控制机制而被屏蔽。默认为关闭，以维持该锁定机制。仅从托管设置中读取。",
      "type": "boolean"
    },
    "strictPluginOnlyCustomization": {
      "description": "当在托管设置中配置时，将针对指定界面阻止非插件来源的自定义配置。数组形式可锁定特定界面（如 [\"skills\", \"hooks\"]）；设置为 true 时锁定全部四个界面；设置为 false 时则无任何效果。被阻止的内容包括：~/.claude/{surface}/、.claude/{surface}/（项目级）、settings.json 中的钩子、.mcp.json。未被阻止的内容包括：托管（policySettings）来源、插件提供的自定义配置。可与 strictKnownMarketplaces 配合使用，实现端到端的管理控制——插件由市场白名单管控，其余一切在此处被阻断。",
      "anyOf": [
        {
          "type": "boolean"
        },
        {
          "type": "array",
          "items": {
            "type": "string",
            "enum": [
              "skills",
              "agents",
              "hooks",
              "mcp"
            ]
          }
        }
      ]
    },
    "statusLine": {
      "description": "自定义状态栏显示配置",
      "type": "object",
      "properties": {
        "type": {
          "type": "string",
          "const": "command"
        },
        "command": {
          "type": "string"
        },
        "padding": {
          "type": "number"
        },
        "refreshInterval": {
          "description": "除事件驱动更新外，每 N 秒重新执行一次状态栏命令",
          "type": "number",
          "minimum": 1
        },
        "hideVimModeIndicator": {
          "description": "隐藏提示词下方的内置 `-- INSERT --` / `-- VISUAL --` 模式指示器。当您的状态栏脚本自行渲染 `vim.mode` 时，请使用此选项。",
          "type": "boolean"
        }
      },
      "required": [
        "type",
        "command"
      ]
    },
    "prUrlTemplate": {
      "description": "用于页脚链接徽章和内联消息中的 PR 链接的 URL 模板。检测到的 Git PR 会作为第一个页脚链接徽章显示。占位符：{host} {owner} {repo} {number} {url}。示例：\"https://reviews.example.com/{owner}/{repo}/pull/{number}\"",
      "type": "字符串"
    },
    "footerLinksRegexes": {
      "description": "当正则表达式匹配回话输出（工具结果和助手回复）时，会显示额外的可点击页脚徽章。仅从用户、标志和托管设置中读取；在项目 .claude/settings.json 和本地 .claude/settings.local.json 中会被忽略。最多显示 5 个徽章；较旧的徽章会被新的匹配项替换，/clear 命令会移除它们。可用于将项目 CLI 打印的 ID 显示为会话链接。",
      "type": "数组",
      "items": {
        "default": {
          "type": "invalid-entry-stripped"
        },
        "anyOf": [
          {
            "type": "对象",
            "properties": {
              "type": {
                "description": "配置变体。此客户端支持 \"regex\"：匹配回话输出，并根据命名捕获组构建 URL。其他变体的条目会被保留，但在运行时会被跳过。",
                "type": "字符串",
                "const": "regex"
              },
              "pattern": {
                "description": "用于匹配回话输出（工具结果和助手文本）的正则表达式",
                "type": "字符串"
              },
              "url": {
                "description": "链接目标。{name} 占位符由命名正则捕获组填充，例如 (?<id>...) -> {id}。值会被 URL 编码；模板中必须使用字面量指定来源。协议必须是 https、http，或被识别的编辑器或工作区深度链接协议：vscode、vscode-insiders、cursor、windsurf、zed、jetbrains、idea、slack、linear、notion、figma。",
                "type": "字符串"
              },
              "label": {
                "description": "徽章文本。{name} 占位符由命名捕获组填充；默认为完整匹配内容。",
                "type": "字符串"
              }
            },
            "required": [
              "type",
              "pattern",
              "url"
            ],
            "additionalProperties": {}
          },
          {
            "type": "对象",
            "properties": {
              "type": {
                "description": "用于标识本客户端不支持的配置变体的标记；该条目会被原样保留并在运行时跳过。",
                "type": "字符串"
              }
            },
            "required": [
              "type"
            ],
            "additionalProperties": {}
          }
        ]
      }
    },
    "subagentStatusLine": {
      "description": "在代理面板中显示的自定义子代理状态行；通过标准输入接收以 JSON 格式的行上下文数据",
      "type": "对象",
      "properties": {
        "type": {
          "type": "字符串",
          "const": "command"
        },
        "command": {
          "type": "字符串"
        }
      },
      "required": [
        "type",
        "command"
      ]
    },
    "enabledPlugins": {
      "description": "启用的插件，格式为 plugin-id@marketplace-id。示例：{ \"formatter@anthropic-tools\": true }。也支持带有版本约束的扩展格式。设置优先级顺序为：用户 < 项目 < 本地 < 标志 < 策略，因此要禁用项目设置中已启用的插件，请在 .claude/settings.local.json 中将其设为 false——在 ~/.claude/settings.json 中设为 false 会被项目设置覆盖。",
      "type": "对象",
      "propertyNames": {
        "type": "字符串"
      },
      "additionalProperties": {
        "anyOf": [
          {
            "type": "数组",
            "items": {
              "type": "字符串"
            }
          },
          {
            "type": "布尔值"
          },
          {
            "not": {}
          }
        ]
      }
    },
    "prependPlugins": {
      "description": "托管插件（plugin@marketplace id，且 managed enabledPlugins 设置为 true 的插件），其钩子按列出的顺序最先、最外层执行：列表中第一个插件会在任何其他插件之前看到每个事件，并在其之后看到每个结果。未在此处或 appendPlugins 中列出的托管插件会紧随这些插件之后；用户、项目和市场插件则在其后；接着是 appendPlugins；最后是内置插件。捆绑的 sec-default@builtin 会自动置于最外层（在启用了托管设置的机器上，以及面向 Team 和 Enterprise 组织时），除非设置了此列表，在这种情况下请将 sec-default@builtin 列在它应处的位置，或将其排除。任何不是已启用托管插件的 ID 都会被跳过；同时出现在两个键中的 ID 会被前置。仅在托管设置中生效（或在无托管设置的机器上，对您自己的插件则在用户设置中生效）；在项目、本地和 --settings 来源中会被忽略。",
      "type": "数组",
      "items": {
        "type": "字符串"
      }
    },
    "appendPlugins": {
      "description": "托管插件（plugin@marketplace id，且 managed enabledPlugins 设置为 true 的插件），其钩子在所有插件中最后、最内层执行，按列出的顺序排列：列表中最后一个插件位于内置插件之上，看到每个事件时已是其他插件处理完毕的状态。仅在托管设置中生效（或在无托管设置的机器上，对您自己的插件则在用户设置中生效）；在项目、本地和 --settings 来源中会被忽略。",
      "type": "数组",
      "items": {
        "type": "字符串"
      }
    },
    "extraKnownMarketplaces": {
      "description": "为此仓库提供的额外市场。通常在仓库的 .claude/settings.json 中使用，以确保团队成员拥有所需的插件来源。",
      "type": "对象",
      "propertyNames": {
        "type": "字符串"
      },
      "additionalProperties": {
        "type": "对象",
        "properties": {
          "source": {
            "description": "获取市场的来源",
            "anyOf": [
              {
                "type": "对象",
                "properties": {
                  "source": {
                    "type": "字符串",
                    "const": "url"
                  },
                  "url": {
                    "描述": "指向 marketplace.json 文件的直接 URL",
                    "类型": "字符串",
                    "格式": "uri"
                  },
                  "headers": {
                    "描述": "自定义 HTTP 头部（例如用于身份验证）",
                    "类型": "对象",
                    "propertyNames": {
                      "类型": "字符串"
                    },
                    "additionalProperties": {
                      "类型": "字符串"
                   }
                  },
                  "headersHelper": {
                    "描述": "一个打印 HTTP 头部 JSON 对象的命令（例如短期有效的认证令牌）。其输出会覆盖 `headers`，并且与 `headers` 一样，会被从此市场下载的同源归档继承。该命令在固定目录（Claude 配置主目录，而非会话目录）下运行，因此请提供可通过 PATH 查找到的简单命令或绝对路径；每次刷新此市场时都会重新执行。",
                    "类型": "字符串",
                    "最大长度": 500
                  }
                },
                "required": [
                  "source",
                  "url"
                ]
              },
              {
                "type": "对象",
                "properties": {
                  "source": {
                    "类型": "字符串",
                    "const": "github"
                  },
                  "repo": {
                    "描述": "GitHub 仓库，格式为 owner/repo。仅在托管设置策略列表（strictKnownMarketplaces / blockedMarketplaces）中允许使用 owner-wildcard 形式 \"owner/*\" 匹配该所有者下的每一个仓库。在其他所有地方（市场添加、extraKnownMarketplaces、known_marketplaces.json），该值必须指定一个单一的仓库——通配符会被原样解析，导致克隆失败。",
                    "type": "string"
                  },
                  "ref": {
                    "description": "要使用的 Git 分支或标签（例如：\"main\"、\"v1.0.0\"）。默认为仓库的默认分支。",
                    "type": "string"
                  },
                  "path": {
                    "description": "仓库内 marketplace.json 的路径（默认为 .claude-plugin/marketplace.json）",
                    "type": "string"
                  },
                  "sparsePaths": {
                    "description": "通过 Git 稀疏检出（cone 模式）包含的目录。适用于市场位于子目录中的 monorepo。示例：[\".claude-plugin\", \"plugins\"]。若省略，则会克隆整个仓库。",
                    "type": "array",
                    "items": {
                      "type": "string"
                    }
                  },
                  "skipLfs": {
                    "description": "无实际作用；保留此选项是为了兼容现有配置。Claude Code 自带的 Git 始终不会下载 Git LFS 内容：无论是否设置此项，市场仓库中受 LFS 跟踪的文件都会以指针文件形式检出，并且在添加或更新市场时会显示有多少个这样的文件。如需获取其内容，请在 ~/.claude/plugins/marketplaces/ 下的市场工作区运行 `git lfs pull`。",
                    "type": "boolean"
                  }
                },
                "required": [
                  "source",
                  "repo"
                ]
              },
              {
                "type": "object",
                "properties": {
                  "source": {
                    "type": "string",
                    "const": "git"
                  },
                  "url": {
                    "description": "完整的 Git 仓库 URL",
                    "type": "string"
                  },
                  "ref": {
                    "description": "要使用的 Git 分支或标签（例如：\"main\"、\"v1.0.0\"）。默认为仓库的默认分支。",
                    "type": "string"
                  },
                  "path": {
                    "description": "仓库内 marketplace.json 的路径（默认为 .claude-plugin/marketplace.json）",
                    "type": "string"
                  },
                  "sparsePaths": {
                    "description": "通过 Git 稀疏检出（cone 模式）包含的目录。适用于市场位于子目录中的 monorepo。示例：[\".claude-plugin\", \"plugins\"]。若省略，则会克隆整个仓库。",
                    "type": "array",
                    "items": {
                      "type": "string"
                    }
                  },
                  "skipLfs": {
                    "description": "无实际作用；保留此选项是为了兼容现有配置。Claude Code 自带的 Git 始终不会下载 Git LFS 内容：无论是否设置此项，市场仓库中受 LFS 跟踪的文件都会以指针文件形式检出，并且在添加或更新市场时会显示有多少个这样的文件。如需获取其内容，请在 ~/.claude/plugins/marketplaces/ 下的市场工作区运行 `git lfs pull`。",
                    "type": "boolean"
                  }
                },
                "required": [
                  "source",
                  "url"
                ]
              },
              {
                "type": "object",
                "properties": {
                  "source": {
                    "type": "string",
                    "const": "npm"
                  },
                  "package": {
                    "description": "包含 marketplace.json 的 NPM 包",
                    "type": "string"
                  }
                },
                "required": [
                  "source",
                  "package"
                ]
              },
              {
                "type": "object",
                "properties": {
                  "source": {
                    "type": "string",
                    "const": "file"
                  },
                  "path": {
                    "description": "本地 marketplace.json 文件的路径",
                    "type": "string"
                  }
                },
                "required": [
                  "source",
                  "path"
                ]
              },
              {
                "type": "object",
                "properties": {
                  "source": {
                    "type": "string",
                    "const": "directory"
                  },
                  "path": {
                    "description": "包含 .claude-plugin/marketplace.json 的本地目录",
                    "type": "string"
                  }
                },
                "required": [
                  "source",
                  "path"
                ]
              },
              {
                "description": "用于 ~/.claude/skills/ 自动加载（@skills-dir 插件）的策略列表哨兵。在 strictKnownMarketplaces 中：可将其扫描重新开启（默认情况下，任何白名单都会阻止它）。在 blockedMarketplaces 中：关闭扫描功能，但不额外限制市场来源。仅在这两个受管理的设置列表（areLocalPluginDirsAllowedByPolicy）中有意义；known_marketplaces.json 和市场添加等功能会忽略它。",
                "type": "object",
                "properties": {
                  "source": {
                    "type": "string",
                    "const": "skills-dir"
                  }
                },
                "required": [
                  "source"
                ]
              },
              {
                "type": "object",
                "properties": {
                  "source": {
                    "type": "string",
                    "const": "hostPattern"
                  },
                  "hostPattern": {
                    "description": "用于匹配从任何市场来源类型中提取的主机/域名的正则表达式模式。对于 GitHub 来源，匹配 github.com。对于 Git 来源（SSH 或 HTTPS），从 URL 中提取主机名。在 strictKnownMarketplaces 中使用，以允许来自特定主机的所有市场（例如：\"^github\\.mycompany\\.com$\"）。",
                    "type": "string"
                  }
                },
                "required": [
                  "source",
                  "hostPattern"
                ]
              },
              {
                "type": "object",
                "properties": {
                  "source": {
                    "type": "string",
                    "const": "pathPattern"
                  },
                  "pathPattern": {
                    "description": "对文件和目录来源的 .path 字段进行匹配的正则表达式模式。在 strictKnownMarketplaces 中使用，以便在网络来源的 hostPattern 限制之外，同时允许基于文件系统的市场。使用 \".*\" 可允许所有文件系统路径，或使用更窄的模式（例如：\"^/opt/approved/\"）来限制到特定目录。",
                    "type": "string"
                  }
                },
                "required": [
                  "source",
                  "pathPattern"
                ]
              },
              {
                "description": "直接在 settings.json 中定义的内联市场清单。协调器会将一个合成的 marketplace.json 写入缓存；diffMarketplaces 通过比较存储的 source（插件数组包含在此对象内，因此编辑会表现为 sourceChanged）来检测修改。",
                "type": "object",
                "properties": {
                  "source": {
                    "type": "string",
                    "const": "settings"
                  },
                  "name": {
                    "description": "市场名称。必须与 extraKnownMarketplaces 键匹配（强制执行）；合成清单将以此名称写入。验证规则与 PluginMarketplaceSchema 相同，并且会拒绝保留名称——validateOfficialNameSource 会在磁盘写入之后运行，此时已无法进行修正。",
                    "type": "string",
                    "minLength": 1
                  },
                  "plugins": {
                    "description": "在 settings.json 中内联声明的插件条目",
                    "type": "array",
                    "items": {
                      "type": "object",
                      "properties": {
                        "name": {
                          "description": "插件在目标仓库中的名称",
                          "type": "string",
                          "minLength": 1
                        },
                        "source": {
                          "description": "插件的获取来源。必须是远程源——相对路径没有可供解析的市场仓库。",
                          "anyOf": [
                            {
                              "description": "相对于市场根目录（包含 .claude-plugin/ 的目录，而非 .claude-plugin/ 本身）的插件根目录路径",
                              "type": "string",
                              "pattern": "^\\.\\/.*"
                            },
                            {
                              "description": "以 NPM 包作为插件来源",
                              "type": "object",
                              "properties": {
                                "source": {
                                  "type": "string",
                                  "const": "npm"
                                },
                                "package": {
                                  "description": "包名（或 URL，或本地路径，或其他可作为包传递给 `npm` 的内容）",
                                  "anyOf": [
                                    {
                                      "type": "string"
                                    },
                                    {
                                      "type": "string"
                                    }
                                  ]
                                },
                                "version": {
                                  "description": "指定版本或版本范围（例如 ^1.0.0、~2.1.0）",
                                  "type": "string"
                                },
                                "registry": {
                                  "description": "自定义 NPM 注册表 URL（默认使用系统默认注册表，通常是 npmjs.org）",
                                  "type": "string",
                                  "format": "uri"
                                }
                              },
                              "required": [
                                "source",
                                "package"
                              ]
                            },
                            {
                              "type": "object",
                              "properties": {
                                "source": {
                                  "type": "string",
                                  "const": "url"
                                },
                                "url": {
                                  "description": "完整的 Git 仓库 URL（https:// 或 git@ 格式）",
                                  "type": "string"
                                },
                                "ref": {
                                  "description": "要使用的 Git 分支或标签（例如 \"main\"、\"v1.0.0\"）。默认为仓库的默认分支。",
                                  "type": "string"
                                },
                                "sha": {
                                  "description": "要使用的特定提交 SHA 值",
                                  "type": "string",
                                  "minLength": 40,
                                  "maxLength": 40,
                                  "pattern": "^[a-f0-9]{40}$"
                                }
                              },
                              "required": [
                                "source",
                                "url"
                              ]
                            },
                            {
                              "type": "object",
                              "properties": {
                                "source": {
                                  "type": "string",
                                  "const": "github"
                                },
                                "repo": {
                                  "description": "GitHub 仓库，格式为 owner/repo",
                                  "type": "string"
                                },
                                "ref": {
                                  "description": "要使用的 Git 分支或标签（例如 \"main\"、\"v1.0.0\"）。默认为仓库的默认分支。",
                                  "type": "string"
                                },
                                "sha": {
                                  "description": "要使用的特定提交 SHA 值",
                                  "type": "string",
                                  "minLength": 40,
                                  "maxLength": 40,
                                  "pattern": "^[a-f0-9]{40}$"
                                }
                              },
                              "required": [
                                "source",
                                "repo"
                              ]
                            },
                            {
                              "description": "插件位于较大仓库的子目录中（monorepo）。仅克隆并提取指定的子目录，其余部分不会下载。",
                              "type": "object",
                              "properties": {
                                "source": {
                                  "type": "string",
                                  "const": "git-subdir"
                                },
                                "url": {
                                  "description": "Git 仓库：可以是 GitHub 的 owner/repo 简写、https:// 或 git@ 格式的 URL",
                                  "type": "string"
                                },
                                "path": {
                                  "description": "仓库中包含插件的子目录（例如 \"tools/claude-plugin\"）。采用稀疏克隆（--filter=tree:0）方式，以最大限度地减少大型 monorepo 的带宽消耗。",
                                  "type": "string",
                                  "minLength": 1
                                },
                                "ref": {
                                  "description": "要使用的 Git 分支或标签（例如 \"main\"、\"v1.0.0\"）。默认为仓库的默认分支。",
                                  "type": "string"
                                },
                                "sha": {
                                  "description": "要使用的特定提交 SHA 值",
                                  "type": "string",
                                  "minLength": 40,
                                  "maxLength": 40,
                                  "pattern": "^[a-f0-9]{40}$"
                                }
                              },
                              "required": [
                                "source",
                                "url",
                                "path"
                              ]
                            },
                            {
                              "description": "以 HTTPS 协议获取的 ZIP 压缩包形式分发的插件——适用于在任何静态文件服务器或制品仓库（如 S3、GitLab、nginx）上托管，客户端无需 Git 或 npm。认证方式：使用该条目自身的 `headers` / `headersHelper`（绑定到此 URL），当压缩包与其来源同域时，会与外层 URL 源市场中的 headers（静态或由 `headersHelper` 生成）叠加。",
                              "type": "object",
                              "properties": {
                                "source": {
                                  "type": "string",
                                  "const": "archive"
                                },
                                "url": {
                                  "description": "包含插件的 ZIP 压缩包的 HTTPS URL。插件根目录（即包含 .claude-plugin/ 的目录）可以位于压缩包的顶层，也可以嵌套在最外层目录下——仅允许有一层包装目录，且该层会被剥离。",
                                  "type": "string",
                                  "format": "uri"
                                },
                                "sha256": {
                                  "description": "压缩包的 SHA-256 摘要。若已设置，则每次下载都会与其进行校验，不匹配时安装将被拒绝。此外，当 plugin.json 和市场条目均未声明 `version` 时，该摘要也可用作版本标识。建议填写。请注意，更新信号是版本字符串（plugin.json 中的版本，若无则取条目版本，若两者都无则以此摘要为准）——仅更改摘要而未声明版本不会触发更新。",
                                  "type": "string",
                                  "pattern": "^[0-9a-fA-F]{64}$"
                                }
                              },
                              "required": [
                                "source",
                                "url"
                              ]
                            },
                            {
                              "description": "由本地安装的工具生成的插件目录（例如，为当前选定的 SDK 渲染插件的 IDE）。Claude Code 会执行该命令，复制其输出的目录，并在启动时后台重新执行以检测变更。",
                              "type": "object",
                              "properties": {
                                "source": {
                                  "type": "string",
                                  "const": "command"
                                },
                                "command": {
                                  "description": "Shell 命令，应在标准输出中精确输出一行插件目录的绝对路径，并以退出码 0 结束。该命令必须在退出前确保该目录中存在完整的插件；目录会被复制到插件缓存中，因此每次运行时输出的路径可能不同（每次安装和更新时都会重新解析，且每会话会在后台解析一次）。该命令将在用户的主目录下，通过平台 Shell（macOS/Linux 上为 sh，Windows 上为 cmd.exe）以 Claude Code 的子进程环境执行。",
                                  "type": "string",
                                  "minLength": 1,
                                  "maxLength": 500
                                },
                                "timeout": {
                                  "description": "等待命令完成的秒数，超时后将放弃（默认：60 秒）",
                                  "type": "integer",
                                  "exclusiveMinimum": 0,
                                  "maximum": 600
                                },
                                "mode": {
                                  "description": "copy（默认）：输出的目录会被复制到插件缓存并按内容哈希存储，因此后续可删除该目录。link：缓存条目直接链接到输出目录（不复制，无大小限制；仅限 macOS/Linux）——适用于大型导出；此时目录必须在 Claude Code 运行期间保持有效，且只有输出路径发生变化才会被视为新内容。",
                                  "type": "string",
                                  "enum": [
                                    "copy",
                                    "link"
                                  ]
                                }
                              },
                              "required": [
                                "source",
                                "command"
                              ]
                            },
                            {
                              "description": "用于标记本 Claude Code 版本无法识别的源类型，或已知类型但其字段校验失败的情况（此时 `error` 字段会记录原因）。绝不会由用户手动创建——PluginMarketplaceSchema 会将无法解析的源重写为此类型，以使条目仍保留在 marketplace.plugins 中（detectDelistedPlugins 不应将其视为已移除）。尝试安装时会在 cachePlugin 阶段失败，并给出明确的错误提示。",
                              "type": "object",
                              "properties": {
                                "source": {
                                  "type": "string",
                                  "const": "unsupported"
                                },
                                "error": {
                                  "type": "string"
                                }
                              },
                              "required": [
                                "source"
                              ]
                            }
                          ]
                        },
                        "description": {
                          "type": "string"
                        },
                        "version": {
                          "type": "string"
                        },
                        "strict": {
                          "type": "boolean"
                        },
                        "headers": {
                          "description": "下载该条目 `archive` 源时发送的 HTTP 头部。",
                          "type": "object",
                          "propertyNames": {
                            "type": "string"
                          },
                          "additionalProperties": {
                            "type": "string"
                          }
                        },
                        "headersHelper": {
                          "description": "用于打印下载该条目 `archive` 源时所需 HTTP 头部的 JSON 对象的命令。仅在用户显式安装或更新此插件时执行。与目录条目不同，在此处编写的条目无需设置 `strict: false`：它是在设置文件中声明的，而设置文件中没有可用于内联的清单字段。项目设置中的声明并非由运营方编写，因此请求路由和客户端身份相关的头部名称仍会在此处被过滤。请使用绝对路径。",
                          "type": "string",
                          "maxLength": 500
                        }
                      },
                      "required": [
                        "name",
                        "source"
                      ]
                    }
                  },
                  "owner": {
                    "type": "object",
                    "properties": {
                      "name": {
                        "description": "插件作者或组织的显示名称",
                                        "type": "string",
                      "minLength": 1
                    },
                    "email": {
                      "description": "用于支持或反馈的联系邮箱",
                      "type": "string"
                    },
                    "url": {
                      "description": "网站、GitHub 个人主页或组织的 URL",
                      "type": "string"
                    }
                  },
                  "required": [
                    "name"
                  ]
                }
              },
              "required": [
                "source",
                "name",
                "plugins"
              ]
            }
          },
          "installLocation": {
            "description": "存放市场清单的本地缓存路径（如未提供则自动生成）",
            "type": "string"
          },
          "autoUpdate": {
            "description": "启动时是否自动更新此市场及其已安装的插件",
            "type": "boolean"
          }
        },
        "required": [
          "source"
        ]
      }
    },
    "additionalMarketplaces": {
      "description": "extraKnownMarketplaces 的别名：此键将被完全按照 extraKnownMarketplaces 的形式读取。请勿在同一文件中同时设置这两个键——若两者均出现，此键将被忽略并发出警告。Claude Code 在更新文件时可能会将此键重写为 extraKnownMarketplaces。较旧版本的客户端会忽略此键，因此在旧版 Claude Code 仍共享相同设置的情况下，建议使用 extraKnownMarketplaces。",
      "type": "object",
      "propertyNames": {
        "type": "string"
      },
      "additionalProperties": {
        "type": "object",
        "properties": {
          "source": {
            "description": "从何处获取市场信息",
            "anyOf": [
              {
                "type": "object",
                "properties": {
                  "source": {
                    "type": "string",
                    "const": "url"
                  },
                  "url": {
                    "description": "指向 marketplace.json 文件的直接 URL",
                    "type": "string",
                    "format": "uri"
                  },
                  "headers": {
                    "description": "自定义 HTTP 头（例如用于身份验证）",
                    "type": "object",
                    "propertyNames": {
                      "type": "string"
                    },
                    "additionalProperties": {
                      "type": "string"
                    }
                  },
                  "headersHelper": {
                    "description": "一个打印 HTTP 头 JSON 对象的命令（例如短期认证令牌）。其输出会覆盖 `headers`，并且与 `headers` 一样，会被从此市场下载的同源归档继承。该命令将在固定目录（Claude 配置主目录，而非会话目录）下运行，因此请提供可通过 PATH 查找到的简单命令或绝对路径；在后续刷新此市场时，该命令将被重新执行。",
                    "type": "string",
                    "maxLength": 500
                  }
                },
                "required": [
                  "source",
                  "url"
                ]
              },
              {
                "type": "object",
                "properties": {
                  "source": {
                    "type": "string",
                    "const": "github"
                  },
                  "repo": {
                    "description": "GitHub 仓库，格式为 owner/repo。仅在受管理设置策略列表（strictKnownMarketplaces / blockedMarketplaces）中，“owner/*”这种所有者通配符形式才会匹配该所有者下的所有仓库。其他任何地方（添加市场、extraKnownMarketplaces、known_marketplaces.json）都必须指定单个仓库——通配符将被视为字面值，导致克隆失败。",
                    "type": "string"
                  },
                  "ref": {
                    "description": "要使用的 Git 分支或标签（例如 'main'、'v1.0.0'）。默认为仓库的默认分支。",
                    "type": "string"
                  },
                  "path": {
                    "description": "仓库内 marketplace.json 的路径（默认为 .claude-plugin/marketplace.json）",
                    "type": "string"
                  },
                  "sparsePaths": {
                    "description": "通过 git sparse-checkout（锥形模式）包含的目录。适用于市场位于子目录中的 monorepo。示例：['.claude-plugin', 'plugins']。若省略，则克隆整个仓库。",
                    "type": "array",
                    "items": {
                      "type": "string"
                    }
                  },
                  "skipLfs": {
                    "description": "无实际作用；保留此选项是为了兼容现有设置。Claude Code 自带的 Git 永远不会下载 Git LFS 内容：无论是否设置此项，市场仓库中受 LFS 跟踪的文件都会以指针文件的形式检出，并且在添加或更新市场时会报告此类文件的数量。若需获取其内容，请在 ~/.claude/plugins/marketplaces/ 下的市场工作区中运行 `git lfs pull`。",
                    "type": "boolean"
                  }
                },
                "required": [
                  "source",
                  "repo"
                ]
              },
              {
                "type": "object",
                "properties": {
                  "source": {
                    "type": "string",
                    "const": "git"
                  },
                  "url": {
                    "description": "完整的 Git 仓库 URL",
                    "type": "string"
                  },
                  "ref": {
                    "description": "要使用的 Git 分支或标签（例如 'main'、'v1.0.0'）。默认为仓库的默认分支。",
                    "type": "string"
                  },
                  "path": {
                    "description": "仓库内 marketplace.json 的路径（默认为 .claude-plugin/marketplace.json）",
                    "type": "string"
                  },
                  "sparsePaths": {
                    "description": "通过 git sparse-checkout（锥形模式）包含的目录。适用于市场位于子目录中的 monorepo。示例：['.claude-plugin', 'plugins']。若省略，则克隆整个仓库。",
                    "type": "array",
                    "items": {
                      "type": "string"
                    }
                  },
                  "skipLfs": {
                    "description": "无实际作用；保留此选项是为了兼容现有设置。Claude Code 自带的 Git 永远不会下载 Git LFS 内容：无论是否设置此项，市场仓库中受 LFS 跟踪的文件都会以指针文件的形式检出，并且在添加或更新市场时会报告此类文件的数量。若需获取其内容，请在 ~/.claude/plugins/marketplaces/ 下的市场工作区中运行 `git lfs pull`。",
                    "type": "boolean"
                  }
                },
                "required": [
                  "source",
                  "url"
                ]
              },
              {
                "type": "object",
                "properties": {
                  "source": {
                    "type": "string",
                    "const": "npm"
                  },
                  "package": {
                    "description": "包含 marketplace.json 的 NPM 包",
                    "type": "string"
                  }
                },
                "required": [
                  "sour"ce",
                  "package"
                ]
              },
              {
                "类型": "对象",
                "属性": {
                  "source": {
                    "类型": "字符串",
                    "常量值": "file"
                  },
                  "path": {
                    "描述": "指向 marketplace.json 的本地文件路径",
                    "类型": "字符串"
                  }
                },
                "必填项": [
                  "source",
                  "path"
                ]
              },
              {
                "类型": "对象",
                "属性": {
                  "source": {
                    "类型": "字符串",
                    "常量值": "directory"
                  },
                  "path": {
                    "描述": "包含 .claude-plugin/marketplace.json 的本地目录",
                    "类型": "字符串"
                  }
                },
                "必填项": [
                  "source",
                  "path"
                ]
              },
              {
                "描述": "用于 ~/.claude/skills/ 自动加载的策略列表哨兵（@skills-dir 插件）。在 strictKnownMarketplaces 模式下：可选择重新启用扫描功能（默认情况下，任何白名单都会阻止该功能）。在 blockedMarketplaces 模式下：关闭扫描功能，但不额外限制市场。此设置仅在这两种受管理的配置列表中有效（areLocalPluginDirsAllowedByPolicy）；known_marketplaces.json 和 marketplace add 等命令会忽略此项。",
                "类型": "对象",
                "属性": {
                  "source": {
                    "类型": "字符串",
                    "常量值": "skills-dir"
                  }
                },
                "必填项": [
                  "source"
                ]
              },
              {
                "类型": "对象",
                "属性": {
                  "source": {
                    "类型": "字符串",
                    "常量值": "hostPattern"
                  },
                  "hostPattern": {
                    "描述": "用于匹配从任何市场来源类型中提取的主机/域名的正则表达式模式。对于 GitHub 来源，匹配 github.com；对于 Git 来源（SSH 或 HTTPS），则从 URL 中提取主机名。在 strictKnownMarketplaces 模式下，可用于允许来自特定主机的所有市场（例如：“^github\\.mycompany\\.com$”）。",
                    "类型": "字符串"
                  }
                },
                "必填项": [
                  "source",
                  "hostPattern"
                ]
              },
              {
                "类型": "对象",
                "属性": {
                  "source": {
                    "类型": "字符串",
                    "常量值": "pathPattern"
                  },
                  "pathPattern": {
                    "描述": "用于与文件和目录来源的 .path 字段进行匹配的正则表达式模式。在 strictKnownMarketplaces 模式下，可用于在对网络来源施加 hostPattern 限制的同时，允许基于文件系统的市场。使用“.*”可允许所有文件系统路径，或使用更窄的模式（如“^/opt/approved/”）以限制到特定目录。",
                    "类型": "字符串"
                  }
                },
                "必填项": [
                  "source",
                  "pathPattern"
                ]
              },
              {
                "描述": "直接在 settings.json 中定义的内联市场清单。协调器会将一个合成的 marketplace.json 写入缓存；diffMarketplaces 通过比较存储的 source（plugins 数组位于此对象内，因此编辑会显示为 sourceChanged）来检测修改。",
                "类型": "对象",
                "属性": {
                  "source": {
                    "类型": "字符串",
                    "常量值": "settings"
                  },
                  "name": {
                    "描述": "市场名称。必须与 extraKnownMarketplaces 键匹配（强制执行）；合成清单将以该名称写入。验证规则与 PluginMarketplaceSchema 相同，并且还会拒绝保留名称——validateOfficialNameSource 会在磁盘写入之后运行，此时已无法进行清理。",
                    "类型": "字符串",
                    "最小长度": 1
                  },
                  "plugins": {
                    "描述": "在 settings.json 中内联声明的插件条目",
                    "类型": "数组",
                    "元素": {
                      "类型": "对象",
                      "属性": {
                        "name": {
                          "描述": "插件在目标仓库中的名称",
                          "类型": "字符串",
                          "最小长度": 1
                        },
                        "source": {
                          "描述": "获取插件的来源。必须是远程来源——相对路径没有可供解析的市场仓库。",
                          "任意一种": [
                            {
                              "描述": "相对于市场根目录（即包含 .claude-plugin/ 的目录，而非 .claude-plugin/ 本身）的插件根目录路径",
                              "类型": "字符串",
                              "格式": "^\\.\\/.*"
                            },
                            {
                              "描述": "以 NPM 包作为插件来源",
                              "类型": "对象",
                              "属性": {
                                "source": {
                                  "类型": "字符串",
                                  "常量值": "npm"
                                },
                                "package": {
                                  "描述": "包名（或 URL，或本地路径，或其他可作为包传递给 `npm` 的内容）",
                                  "任意一种": [
                                    {
                                      "类型": "字符串"
                                    },
                                    {
                                      "类型": "字符串"
                                    }
                                  ]
                                },
                                "version": {
                                  "描述": "指定版本或版本范围（如 ^1.0.0、~2.1.0）",
                                  "类型": "字符串"
                                },
                                "registry": {
                                  "描述": "自定义 NPM 注册表 URL（默认使用系统默认注册表，通常是 npmjs.org）",
                                  "类型": "字符串",
                                  "格式": "uri"
                                }
                              },
                              "必填项": [
                                "source",
                                "package"
                              ]
                            },
                            {
                              "类型": "对象",
                              "属性": {
                                "source": {
                                  "类型": "字符串",
                                  "常量值": "url"
                                },
                                "url": {
                                  "描述": "完整的 Git 仓库 URL（https:// 或 git@）",
                                  "类型": "字符串"
                                },
                                "ref": {
                                  "描述": "要使用的 Git 分支或标签（如 “main”、“v1.0.0”）。默认使用仓库的默认分支。",
                                  "类型":字符串"
                                },
                                "sha": {
                                  "description": "要使用的特定提交 SHA",
                                  "type": "字符串",
                                  "minLength": 40,
                                  "maxLength": 40,
                                  "pattern": "^[a-f0-9]{40}$"
                                }
                              },
                              "required": [
                                "source",
                                "url"
                              ]
                            },
                            {
                              "type": "object",
                              "properties": {
                                "source": {
                                  "type": "字符串",
                                  "const": "github"
                                },
                                "repo": {
                                  "description": "GitHub 仓库，格式为 owner/repo",
                                  "type": "字符串"
                                },
                                "ref": {
                                  "description": "要使用的 Git 分支或标签（例如：\"main\"、\"v1.0.0\"）。默认为仓库的默认分支。",
                                  "type": "字符串"
                                },
                                "sha": {
                                  "description": "要使用的特定提交 SHA",
                                  "type": "字符串",
                                  "minLength": 40,
                                  "maxLength": 40,
                                  "pattern": "^[a-f0-9]{40}$"
                                }
                              },
                              "required": [
                                "source",
                                "repo"
                              ]
                            },
                            {
                              "description": "插件位于较大仓库（monorepo）的子目录中。仅会克隆并构建指定的子目录，其余部分不会被下载。",
                              "type": "object",
                              "properties": {
                                "source": {
                                  "type": "字符串",
                                  "const": "git-subdir"
                                },
                                "url": {
                                  "description": "Git 仓库：可使用 GitHub 的 owner/repo 简写、https:// 或 git@ 格式的 URL",
                                  "type": "字符串"
                                },
                                "path": {
                                  "description": "仓库中包含插件的子目录（例如：\"tools/claude-plugin\"）。为减少 monorepo 的带宽占用，采用稀疏克隆（--filter=tree:0）方式仅克隆该子目录。",
                                  "type": "字符串",
                                  "minLength": 1
                                },
                                "ref": {
                                  "description": "要使用的 Git 分支或标签（例如：\"main\"、\"v1.0.0\"）。默认为仓库的默认分支。",
                                  "type": "字符串"
                                },
                                "sha": {
                                  "description": "要使用的特定提交 SHA",
                                  "type": "字符串",
                                  "minLength": 40,
                                  "maxLength": 40,
                                  "pattern": "^[a-f0-9]{40}$"
                                }
                              },
                              "required": [
                                "source",
                                "url",
                                "path"
                              ]
                            },
                            {
                              "description": "插件以 ZIP 压缩包的形式通过 HTTPS 下载分发——适用于托管在任何静态文件服务器或制品仓库（如 S3、GitLab、nginx）上，客户端无需 Git 或 npm。认证方式为：由该条目自身的 `headers` / `headersHelper`（绑定到此 URL）提供，当压缩包与其来源相同（同源）时，会与外部 marketplace 的 headers（静态或由 `headersHelper` 生成）叠加使用。",
                              "type": "object",
                              "properties": {
                                "source": {
                                  "type": "字符串",
                                  "const": "archive"
                                },
                                "url": {
                                  "description": "包含插件的 ZIP 压缩包的 HTTPS URL。插件根目录（即存放 .claude-plugin/ 的目录）可以位于压缩包的顶层，也可以嵌套在第一层子目录中——系统会剥离最外层的一层包装目录。",
                                  "type": "字符串",
                                  "format": "uri"
                                },
                                "sha256": {
                                  "description": "压缩包的 SHA-256 摘要。若设置了该值，则每次下载都会进行校验，若校验失败则安装会被拒绝。同时，当 plugin.json 和 marketplace 条目均未声明 `version` 时，该摘要也可用作版本标识。建议设置。请注意，更新信号是版本字符串（plugin.json 中的 version，若无则取条目的 version，再无则以此摘要为准）——仅更改摘要而未变更版本号时，不会触发更新。",
                                  "type": "字符串",
                                  "pattern": "^[0-9a-fA-F]{64}$"
                                }
                              },
                              "required": [
                                "source",
                                "url"
                              ]
                            },
                            {
                              "description": "插件目录由本地安装的工具生成（例如：IDE 为当前选定的 SDK 渲染其插件）。Claude Code 会执行该命令，复制其输出的目录，并在启动时后台重新执行该命令以获取最新变更。",
                              "type": "object",
                              "properties": {
                                "source": {
                                  "type": "字符串",
                                  "const": "command"
                                },
                                "command": {
                                  "description": "Shell 命令，应在标准输出中打印插件目录的绝对路径（且仅打印一行），并以退出码 0 结束。该命令必须在退出前确保该目录中存在一个完整的插件；由于目录会被复制到插件缓存中，因此每次运行时打印的路径可能会变化（每次安装和更新时都会重新解析，且在每个会话期间会在后台重新解析一次）。该命令将在平台的 Shell 环境下执行（macOS/Linux 上为 sh，Windows 上为 cmd.exe），从用户的主目录开始，并使用 Claude Code 的子进程环境。",
                                  "type": "字符串",
                                  "minLength": 1,
                                  "maxLength": 500
                                },
                                "timeout": {
                                  "description": "等待命令完成的超时时间（单位：秒；默认值：60）",
                                  "type": "整数",
                                  "exclusiveMinimum": 0,
                                  "maximum": 600
                                },
                                "mode": {
                                  ""description": "copy（默认）：打印出的目录会被复制到插件缓存中，并进行内容哈希处理，因此之后可以将其删除。link：缓存条目会直接链接到打印出的目录（不复制，无大小限制；适用于 macOS/Linux），用于大型导出；此时目录必须在 Claude Code 运行期间保持有效，而不同的打印路径则表示内容已更新。",
                                  "type": "string",
                                  "enum": [
                                    "copy",
                                    "link"
                                  ]
                                }
                              },
                              "required": [
                                "source",
                                "command"
                              ]
                            },
                            {
                              "description": "用于标识当前 Claude Code 版本无法识别的来源类型，或已知但字段校验失败的来源类型（此时 `error` 字段会给出原因）。此对象绝不会由用户手动创建——PluginMarketplaceSchema 会将无法解析的来源重写为此格式，以确保该条目仍保留在 marketplace.plugins 中（detectDelistedPlugins 不应将其视为已移除）。安装尝试会在 cachePlugin 阶段因可操作的错误信息而失败。",
                              "type": "object",
                              "properties": {
                                "source": {
                                  "type": "string",
                                  "const": "unsupported"
                                },
                                "error": {
                                  "type": "string"
                                }
                              },
                              "required": [
                                "source"
                              ]
                            }
                          ]
                        },
                        "description": {
                          "type": "string"
                        },
                        "version": {
                          "type": "string"
                        },
                        "strict": {
                          "type": "boolean"
                        },
                        "headers": {
                          "description": "下载此条目 `archive` 来源时发送的 HTTP 头部。",
                          "type": "object",
                          "propertyNames": {
                            "type": "string"
                          },
                          "additionalProperties": {
                            "type": "string"
                          }
                        },
                        "headersHelper": {
                          "description": "用于打印下载此条目 `archive` 来源时所需 HTTP 头部的 JSON 对象的命令。仅在用户明确安装或更新此插件时运行。与目录条目不同，在此处声明的条目无需设置 `strict: false`：它是在设置文件中声明的，而设置文件没有可用于内联的清单字段。项目设置中的声明并非由操作者编写，因此请求路由和客户端身份标识头名称仍会在那里被过滤。请使用绝对路径。",
                          "type": "string",
                          "maxLength": 500
                        }
                      },
                      "required": [
                        "name",
                        "source"
                      ]
                    }
                  },
                  "owner": {
                    "type": "object",
                    "properties": {
                      "name": {
                        "description": "插件作者或组织的显示名称",
                        "type": "string",
                        "minLength": 1
                      },
                      "email": {
                        "description": "用于支持或反馈的联系邮箱",
                        "type": "string"
                      },
                      "url": {
                        "description": "网站、GitHub 个人主页或组织的 URL",
                        "type": "string"
                      }
                    },
                    "required": [
                      "name"
                    ]
                  }
                },
                "required": [
                  "source",
                  "name",
                  "plugins"
                ]
              }
            ]
          },
          "installLocation": {
            "description": "存储市场清单的本地缓存路径（未提供时自动生成）",
            "type": "string"
          },
          "autoUpdate": {
            "description": "是否在启动时自动更新此市场及其已安装的插件",
            "type": "boolean"
          }
        },
        "required": [
          "source"
        ]
      }
    },
    "strictKnownMarketplaces": {
      "description": "企业严格允许的市场来源列表。当在托管设置中启用时，只有这些来源才能被添加为市场。条目需完全匹配，但 GitHub 条目可以使用所有者通配符形式 {\"source\":\"github\",\"repo\":\"owner/*\"}，以允许该所有者下的所有仓库。检查在下载之前进行，因此被阻止的来源永远不会接触文件系统。注意：这只是一个策略性限制——它并不会注册市场。如需为用户预先注册允许的市场，请同时设置 extraKnownMarketplaces。",
      "type": "array",
      "items": {
        "anyOf": [
          {
            "type": "object",
            "properties": {
              "source": {
                "type": "string",
                "const": "url"
              },
              "url": {
                "description": "指向 marketplace.json 文件的直接 URL",
                "type": "string",
                "format": "uri"
              },
              "headers": {
                "description": "自定义 HTTP 头部（例如用于身份验证）",
                "type": "object",
                "propertyNames": {
                  "type": "string"
                },
                "additionalProperties": {
                  "type": "string"
                }
              },
              "headersHelper": {
                "description": "用于打印 HTTP 头部 JSON 对象的命令（例如短期有效的认证令牌）。其输出会覆盖 `headers`，并且与 `headers` 一样，会被从此市场下载的同源归档继承。该命令从固定目录（Claude 的配置主目录，而非会话目录）运行，因此请提供可通过 PATH 查找到的简单命令或绝对路径；在后续刷新此市场时会重新执行。",
                "type": "string",
                "maxLength": 500
              }
            },
            "required": [
              "source",
              "url"
            ]
          },
          {
            "type": "object",
            "properties": {
              "source": {
                "type": "string",
                "const": "github"
              },
              "repo": {
                "description": "GitHub 仓库，格式为 owner/repo。仅在托管设置的策略列表（strictKnownMarketplaces / blockedMarketplaces）中，所有者通配符形式 \"owner/*\" 才会匹配该所有者下的所有仓库。其他任何地方（添加市场、extraKnownMarketplaces、known_marketplaces.json）都必须指定单个仓库——通配符会被原样解析，导致克隆失败。",
                "type": "string"
              },
              "ref": {
                "description": "要使用的 Git 分支或标签（例如 \"main\"、\"v1.0.0\"）。默认为仓库的默认分支。",
                "type": "string"
              },
              "path": {
                "description": "仓库内 marketplace.json 的路径（默认为 .claude-plugin/marketplace.json）",
                "type": "string"
              },
              "sparsePaths": {
                "description": "通过 git sparse-checkout 包含的目录（cone 模式）。适用于市场位于子目录中的 monorepo。例如：[\".claude-plugin\", \"plugins\"]。若省略，则克隆整个仓库。",
                "type": "array",
                "items": {
                  "type": "string"
                }
              },
              "skipLfs": {
                "description": "无实际作用；保留此选项以确保现有配置继续有效。Claude Code 自带的 Git 始终不会下载 Git LFS 内容：无论是否设置此项，市场仓库中受 LFS 跟踪的文件都会以指针文件形式检出，并且在添加或更新市场时会显示此类文件的数量。如需获取其内容，请在 ~/.claude/plugins/marketplaces/ 下的市场工作区运行 `git lfs pull`。",
                "type": "boolean"
              }
            },
            "required": [
              "source",
              "repo"
            ]
          },
          {
            "type": "object",
            "properties": {
              "source": {
                "type": "string",
                "const": "git"
              },
              "url": {
                "description": "完整的 Git 仓库 URL",
                "type": "string"
              },
              "ref": {
                "description": "要使用的 Git 分支或标签（例如：\"main\"、\"v1.0.0\"）。默认为仓库的默认分支。",
                "type": "string"
              },
              "path": {
                "description": "仓库内 marketplace.json 的路径（默认为 .claude-plugin/marketplace.json）",
                "type": "string"
              },
              "sparsePaths": {
                "description": "通过 git sparse-checkout 包含的目录（cone 模式）。适用于市场位于子目录中的 monorepo。例如：[\".claude-plugin\", \"plugins\"]。若省略，则克隆整个仓库。",
                "type": "array",
                "items": {
                  "type": "string"
                }
              },
              "skipLfs": {
                "description": "无实际作用；保留此选项以确保现有配置继续有效。Claude Code 自带的 Git 始终不会下载 Git LFS 内容：无论是否设置此项，市场仓库中受 LFS 跟踪的文件都会以指针文件形式检出，并且在添加或更新市场时会显示此类文件的数量。如需获取其内容，请在 ~/.claude/plugins/marketplaces/ 下的市场工作区运行 `git lfs pull`。",
                "type": "boolean"
              }
            },
            "required": [
              "source",
              "url"
            ]
          },
          {
            "type": "object",
            "properties": {
              "source": {
                "type": "string",
                "const": "npm"
              },
              "package": {
                "description": "包含 marketplace.json 的 NPM 包",
                "type": "string"
              }
            },
            "required": [
              "source",
              "package"
            ]
          },
          {
            "type": "object",
            "properties": {
              "source": {
                "type": "string",
                "const": "file"
              },
              "path": {
                "description": "本地 marketplace.json 文件的路径",
                "type": "string"
              }
            },
            "required": [
              "source",
              "path"
            ]
          },
          {
            "type": "object",
            "properties": {
              "source": {
                "type": "string",
                "const": "directory"
              },
              "path": {
                "description": "包含 .claude-plugin/marketplace.json 的本地目录",
                "type": "string"
              }
            },
            "required": [
              "source",
              "path"
            ]
          },
          {
            "description": "用于 ~/.claude/skills/ 自动加载的策略列表哨兵（@skills-dir 插件）。在 strictKnownMarketplaces 策略下：可重新启用扫描（默认情况下，任何白名单都会阻止它）。在 blockedMarketplaces 策略下：关闭扫描，但不额外限制市场来源。仅在这两个受管理的设置列表中（areLocalPluginDirsAllowedByPolicy）有意义；known_marketplaces.json 和 marketplace add 等命令会忽略此项。",
            "type": "object",
            "properties": {
              "source": {
                "type": "string",
                "const": "skills-dir"
              }
            },
            "required": [
              "source"
            ]
          },
          {
            "type": "object",
            "properties": {
              "source": {
                "type": "string",
                "const": "hostPattern"
              },
              "hostPattern": {
                "description": "用于匹配从任何市场来源类型中提取的主机/域名的正则表达式模式。对于 GitHub 来源，匹配 github.com。对于 Git 来源（SSH 或 HTTPS），从 URL 中提取主机名。在 strictKnownMarketplaces 策略下，可用于允许来自特定主机的所有市场（例如：\"^github\\.mycompany\\.com$\"）。",
                "type": "string"
              }
            },
            "required": [
              "source",
              "hostPattern"
            ]
          },
          {
            "type": "object",
            "properties": {
              "source": {
                "type": "string",
                "const": "pathPattern"
              },
              "pathPattern": {
                "description": "用于匹配文件和目录来源的 .path 字段的正则表达式模式。在 strictKnownMarketplaces 策略下，可用于在网络来源的 hostPattern 限制之外，同时允许基于文件系统的市场。使用 \".*\" 可允许所有文件系统路径，或使用更窄的模式（例如：\"^/opt/approved/\"）来限制到特定目录。",
                "type": "string"
              }
            },
            "required": [
              "source",
              "pathPattern"
            ]
          },
          {
            "description": "直接在 settings.json 中定义的内联市场清单。协调器会将一个合成的 marketplace.json 写入缓存；diffMarketplaces 通过比较存储的 source（插件数组在此对象内，因此编辑会显示为 sourceChanged）来检测修改。",
            "type": "object",
            "properties": {
              "source": {
                "type": "string",
                "const": "settings"
              },
              "name": {
                "description": "市场名称。必须与 extraKnownMarketplaces 键匹配（强制执行）；合成清单将以该名称写入。验证规则与 PluginMarketplaceSchema 相同，并且会拒绝保留名称——validateOfficialNameSource 在磁盘写入后运行，已无法进行修正。",
                "type": "string",
                "minLength": 1
              },
              "plugins": {
                "description": "在 settings.json 中内联声明的插件条目",
                "type": "array",
                "items": {
                  "type": "object",
                  "properties": {
                    "name": {
                      "description": "插件在目标仓库中显示的名称",
                      "type": "string",
                      "minLength": 1
                    },
                    "source": {
                      "description": "获取插件的来源。必须是远程来源——相对路径没有对应的市场仓库来 r与之相悖。",
                      "anyOf": [
                        {
                          "description": "插件根目录的路径，相对于市场根目录（即包含 .claude-plugin/ 的目录，而非 .claude-plugin/ 本身）",
                          "type": "string",
                          "pattern": "^\\.\\/.*"
                        },
                        {
                          "description": "以 NPM 包作为插件来源",
                          "type": "object",
                          "properties": {
                            "source": {
                              "type": "string",
                              "const": "npm"
                            },
                            "package": {
                              "description": "包名（或 URL、本地路径，或其他可作为包传递给 `npm` 的内容）",
                              "anyOf": [
                                {
                                  "type": "string"
                                },
                                {
                                  "type": "string"
                                }
                              ]
                            },
                            "version": {
                              "description": "特定版本或版本范围（如 ^1.0.0、~2.1.0）",
                              "type": "string"
                            },
                            "registry": {
                              "description": "自定义 NPM 注册表 URL（默认使用系统默认注册表，通常是 npmjs.org）",
                              "type": "string",
                              "format": "uri"
                            }
                          },
                          "required": [
                            "source",
                            "package"
                          ]
                        },
                        {
                          "type": "object",
                          "properties": {
                            "source": {
                              "type": "string",
                              "const": "url"
                            },
                            "url": {
                              "description": "完整的 Git 仓库 URL（https:// 或 git@ 格式）",
                              "type": "string"
                            },
                            "ref": {
                              "description": "要使用的 Git 分支或标签（如 \"main\"、\"v1.0.0\"）。默认为仓库的默认分支。",
                              "type": "string"
                            },
                            "sha": {
                              "description": "要使用的特定提交 SHA 值",
                              "type": "string",
                              "minLength": 40,
                              "maxLength": 40,
                              "pattern": "^[a-f0-9]{40}$"
                            }
                          },
                          "required": [
                            "source",
                            "url"
                          ]
                        },
                        {
                          "type": "object",
                          "properties": {
                            "source": {
                              "type": "string",
                              "const": "github"
                            },
                            "repo": {
                              "description": "GitHub 仓库，格式为 owner/repo",
                              "type": "string"
                            },
                            "ref": {
                              "description": "要使用的 Git 分支或标签（如 \"main\"、\"v1.0.0\"）。默认为仓库的默认分支。",
                              "type": "string"
                            },
                            "sha": {
                              "description": "要使用的特定提交 SHA 值",
                              "type": "string",
                              "minLength": 40,
                              "maxLength": 40,
                              "pattern": "^[a-f0-9]{40}$"
                            }
                          },
                          "required": [
                            "source",
                            "repo"
                          ]
                        },
                        {
                          "description": "插件位于一个大型仓库的子目录中（monorepo）。仅克隆并提取指定的子目录，其余部分不会被下载。",
                          "type": "object",
                          "properties": {
                            "source": {
                              "type": "string",
                              "const": "git-subdir"
                            },
                            "url": {
                              "description": "Git 仓库：可以是 GitHub 的 owner/repo 简写、https:// 或 git@ 格式的 URL",
                              "type": "string"
                            },
                            "path": {
                              "description": "仓库内包含插件的子目录（如 \"tools/claude-plugin\"）。采用稀疏克隆（--filter=tree:0）方式，以减少大型 monorepo 的带宽消耗。",
                              "type": "string",
                              "minLength": 1
                            },
                            "ref": {
                              "description": "要使用的 Git 分支或标签（如 \"main\"、\"v1.0.0\"）。默认为仓库的默认分支。",
                              "type": "string"
                            },
                            "sha": {
                              "description": "要使用的特定提交 SHA 值",
                              "type": "string",
                              "minLength": 40,
                              "maxLength": 40,
                              "pattern": "^[a-f0-9]{40}$"
                            }
                          },
                          "required": [
                            "source",
                            "url",
                            "path"
                          ]
                        },
                        {
                          "description": "插件以 ZIP 压缩包形式分发，通过 HTTPS 下载——适用于任何静态文件服务器或制品仓库（如 S3、GitLab、nginx），客户端无需 Git 或 npm。认证机制：由该条目自身的 `headers` / `headersHelper`（绑定到此 URL）提供，当压缩包与其所在源同域时，会叠加覆盖外部市场来源的 headers（无论是静态配置还是通过 `headersHelper` 生成的）。",
                          "type": "object",
                          "properties": {
                            "source": {
                              "type": "string",
                              "const": "archive"
                            },
                            "url": {
                              "description": "包含插件的 ZIP 压缩包的 HTTPS URL。插件根目录（即存放 .claude-plugin/ 的目录）可以位于压缩包的顶层，也可以嵌套在最外层的一级目录中——最外层的包裹目录会被剥离。",
                              "type": "string",
                              "format": "uri"
                            },
                            "sha256": {
                              "description": "压缩包的 SHA-256 摘要。若设置了该值，则每次下载都会与其进行校验，不匹配时安装将被拒绝。同时，在 plugin.json 和市场条目均未声明 `version` 时，它也将作为版本标识。建议设置。请注意，更新信号是版本字符串（plugin.json 中的版本，否则为条目的版本，否则“此摘要）——在声明版本时仅更改摘要不会触发更新。”
                              “类型”： “字符串”，
                              “模式”： “^[0-9a-fA-F]{64}$”
                            }
                          },
                          “必要属性”： [
                            “source”，
                            “url”
                          ]
                        },
                        {
                          “描述”： “由本地安装的工具（例如，为当前所选 SDK 渲染插件的 IDE）生成的插件目录。Claude Code 会执行该命令，复制其输出的目录，并在启动时在后台重新运行该命令以检测变更。”，
                          “类型”： “对象”，
                          “属性”： {
                            “source”： {
                              “类型”： “字符串”，
                              “常量”： “command”
                            },
                            “command”： {
                              “描述”： “Shell 命令，在标准输出上打印插件目录的绝对路径（且仅一行），并以退出码 0 结束。该命令必须在退出前确保该目录中包含完整的插件；目录会被复制到插件缓存中，因此每次运行时打印的路径可能会变化（每次安装和更新时都会重新解析，且每个会话期间会在后台重新解析一次）。该命令通过平台 Shell 执行（macOS/Linux 上为 sh，Windows 上为 cmd.exe），从用户的主目录以 Claude Code 的子进程环境运行。”，
                              “类型”： “字符串”，
                              “最小长度”： 1，
                              “最大长度”： 500
                            },
                            “timeout”： {
                              “描述”： “等待命令完成的秒数，超时则放弃（默认：60 秒）”，
                              “类型”： “整数”，
                              “严格最小值”： 大于 0，
                              “最大值”： 600
                            },
                            “mode”： {
                              “描述”： “copy（默认）：打印的目录会被复制到插件缓存中并进行内容哈希，因此之后可以删除该目录。link：缓存条目直接链接到打印的目录（不复制，无大小限制；适用于 macOS/Linux 系统）——用于大型导出；此时目录必须在 Claude Code 运行期间保持有效，而不同的打印路径才是新内容的信号。”，
                              “类型”： “字符串”，
                              “枚举”： [
                                “copy”，
                                “link”
                              ]
                            }
                          },
                          “必要属性”： [
                            “source”，
                            “command”
                          ]
                        },
                        {
                          “描述”： “用于标识本 Claude Code 版本无法识别的来源类型，或已知类型但其字段校验失败的情况（此时 `error` 字段会记录原因）。绝不会由用户手动创建——PluginMarketplaceSchema 会将无法解析的来源重写为此类型，以使该条目仍保留在 marketplace.plugins 中（detectDelistedPlugins 不应将其视为已移除）。尝试安装时会在 cachePlugin 阶段失败，并给出可操作的错误信息。”，
                          “类型”： “对象”，
                          “属性”： {
                            “source”： {
                              “类型”： “字符串”，
                              “常量”： “unsupported”
                            },
                            “error”： {
                              “类型”： “字符串”
                            }
                          },
                          “必要属性”： [
                            “source”
                          ]
                        }
                      ]
                    },
                    “description”： {
                      “类型”： “字符串”
                    },
                    “version”： {
                      “类型”： “字符串”
                    },
                    “strict”： {
                      “类型”： “布尔值”
                    },
                    “headers”： {
                      “描述”： “下载该条目 `archive` 来源时发送的 HTTP 头信息。”，
                      “类型”： “对象”，
                      “属性名”： “类型”： “字符串”，
                      “额外属性”： “类型”： “字符串”
                    },
                    “headersHelper”： {
                      “描述”： “打印用于下载该条目 `archive` 来源的 HTTP 头信息 JSON 对象的命令。仅在用户明确安装或更新此插件时运行。与目录条目不同，此处编写的条目无需设置 `strict: false`：它是在设置文件中声明的，而设置文件没有可用于内联的清单字段。项目设置中的声明并非由操作者编写，因此请求路由和客户端身份头名称仍会在那里被过滤。请使用绝对路径。”，
                      “类型”： “字符串”，
                      “最大长度”： 500
                    }
                  },
                  “必要属性”： [
                    “name”，
                    “source”
                  ]
                }
              },
              “owner”： {
                “类型”： “对象”，
                “属性”： {
                  “name”： {
                    “描述”： “插件作者或组织的显示名称”，
                    “类型”： “字符串”，
                    “最小长度”： 1
                  },
                  “email”： {
                    “描述”： “用于支持或反馈的联系邮箱”，
                    “类型”： “字符串”
                  },
                  “url”： {
                    “描述”： “网站、GitHub 个人主页或组织主页的 URL”，
                    “类型”： “字符串”
                  }
                },
                “必要属性”： [
                  “name”
                ]
              }
            },
            “必要属性”： [
              “source”，
              “name”，
              “plugins”
            ]
          }
        ]
      }
    },
    “allowedMarketplaces”： {
      “描述”： “strictKnownMarketplaces 的别名（仅限管理设置）：此键的读取方式与拼写为 strictKnownMarketplaces 完全相同。请勿在同一文件中同时设置这两个键——若两者同时出现，则此键将被忽略并发出警告。早于该别名的客户端会忽略此键，因此当白名单还需兼容较旧版本的 Claude Code 时，请继续使用 strictKnownMarketplaces。”，
      “类型”： “数组”，
      “元素”： {
        “任意之一”： [
          {
            “类型”： “对象”，
            “属性”： {
              “source”： {
                “类型”： “字符串”，
                “常量”： “url”
              },
              “url”： {
                “描述”： “指向 marketplace.json 文件的直接 URL”，
                “类型”： “字符串”，
                “格式”： “uri”
              },
              “headers”： {
                “描述”： “自定义 HTTP 头信息（例如用于身份验证）”，
                “类型”： “对象”，
                “属性名”： “类型”： “字符串”，
                “额外属性”： “类型”： “字符串”
              },
              “headersHelper”： {
                “描述”： “打印 HTTP 头信息 JSON 对象的命令（例如短期有效的认证令牌）。其输出会覆盖 `headers`，并且像 `headers` 一样，会被从此市场下载的同源归档继承。该命令从固定目录（Claude 配置主目录，而非会话目录）运行，因此请提供一个可通过PATH 或绝对路径；在后续刷新该市场时会重新执行。
                “类型”: “字符串”，
                “最大长度”: 500
              }
            },
            “必填”: [
              “来源”,
              “网址”
            ]
          },
          {
            “类型”: “对象”，
            “属性”: {
              “来源”: {
                “类型”: “字符串”，
                “常量”: “github”
              },
              “仓库”: {
                “描述”: “GitHub 仓库，格式为 owner/repo。仅在受管理设置策略列表（strictKnownMarketplaces / blockedMarketplaces）中，owner-通配符形式“owner/*”会匹配该所有者下的所有仓库。其他任何地方（市场添加、extraKnownMarketplaces、known_marketplaces.json）都必须指定单个仓库——通配符会被原样解析，导致克隆失败。”，
                “类型”: “字符串”
              },
              “引用”: {
                “描述”: “要使用的 Git 分支或标签（例如，“main”、“v1.0.0”）。默认为仓库的默认分支。”，
                “类型”: “字符串”
              },
              “路径”: {
                “描述”: “仓库内 marketplace.json 的路径（默认为 .claude-plugin/marketplace.json）。”，
                “类型”: “字符串”
              },
              “稀疏路径”: {
                “描述”: “通过 git sparse-checkout（锥形模式）包含的目录。适用于市场位于子目录中的 monorepo。示例：[“.claude-plugin”, “plugins”]。若省略，则克隆整个仓库。”，
                “类型”: “数组”，
                “元素”: {
                  “类型”: “字符串”
                }
              },
              “跳过 LFS”: {
                “描述”: “无实际作用；保留此选项是为了兼容现有设置。Claude Code 自带的 Git 永远不会下载 Git LFS 内容：无论是否启用此项，市场仓库中被 LFS 跟踪的文件都会以指针文件的形式检出，并且在添加或更新市场时会显示有多少这样的文件。如需获取其内容，请在 ~/.claude/plugins/marketplaces/ 下的市场工作区运行 `git lfs pull`。”，
                “类型”: “布尔值”
              }
            },
            “必填”: [
              “来源”,
              “仓库”
            ]
          },
          {
            “类型”: “对象”，
            “属性”: {
              “来源”: {
                “类型”: “字符串”，
                “常量”: “git”
              },
              “网址”: {
                “描述”: “完整的 Git 仓库 URL。”，
                “类型”: “字符串”
              },
              “引用”: {
                “描述”: “要使用的 Git 分支或标签（例如，“main”、“v1.0.0”）。默认为仓库的默认分支。”，
                “类型”: “字符串”
              },
              “路径”: {
                “描述”: “仓库内 marketplace.json 的路径（默认为 .claude-plugin/marketplace.json）。”，
                “类型”: “字符串”
              },
              “稀疏路径”: {
                “描述”: “通过 git sparse-checkout（锥形模式）包含的目录。适用于市场位于子目录中的 monorepos。示例：[“.claude-plugin”, “plugins”]。若省略，则克隆整个仓库。”，
                “类型”: “数组”，
                “元素”: {
                  “类型”: “字符串”
                }
              },
              “跳过 LFS”: {
                “描述”: “无实际作用；保留此选项是为了兼容现有设置。Claude Code 自带的 Git 永远不会下载 Git LFS 内容：无论是否启用此项，市场仓库中被 LFS 跟踪的文件都会以指针文件的形式检出，并且在添加或更新市场时会显示有多少这样的文件。如需获取其内容，请在 ~/.claude/plugins/marketplaces/ 下的市场工作区运行 `git lfs pull`。”，
                “类型”: “布尔值”
              }
            },
            “必填”: [
              “来源”,
              “网址”
            ]
          },
          {
            “类型”: “对象”，
            “属性”: {
              “来源”: {
                “类型”: “字符串”，
                “常量”: “npm”
              },
              “包”: {
                “描述”: “包含 marketplace.json 的 NPM 包。”，
                “类型”: “字符串”
              }
            },
            “必填”: [
              “来源”,
              “包”
            ]
          },
          {
            “类型”: “对象”，
            “属性”: {
              “来源”: {
                “类型”: “字符串”，
                “常量”: “file”
              },
              “路径”: {
                “描述”: “本地 marketplace.json 文件的路径。”，
                “类型”: “字符串”
              }
            },
            “必填”: [
              “来源”,
              “路径”
            ]
          },
          {
            “类型”: “对象”，
            “属性”: {
              “来源”: {
                “类型”: “字符串”，
                “常量”: “directory”
              },
              “路径”: {
                “描述”: “包含 .claude-plugin/marketplace.json 的本地目录。”，
                “类型”: “字符串”
              }
            },
            “必填”: [
              “来源”,
              “路径”
            ]
          },
          {
            “描述”: “用于 ~/.claude/skills/ 自动加载（@skills-dir 插件）的策略列表哨兵。在 strictKnownMarketplaces 中：可将其扫描重新开启（默认情况下，任何白名单都会阻止它）。在 blockedMarketplaces 中：关闭扫描功能，但不对市场进行其他限制。仅在这两个受管理设置列表中（areLocalPluginDirsAllowedByPolicy）有意义；known_marketplaces.json、市场添加等功能均忽略此项。”，
            “类型”: “对象”，
            “属性”: {
              “来源”: {
                “类型”: “字符串”，
                “常量”: “skills-dir”
              }
            },
            “必填”: [
              “来源”
            ]
          },
          {
            “类型”: “对象”，
            “属性”: {
              “来源”: {
                “类型”: “字符串”，
                “常量”: “hostPattern”
              },
              “主机模式”: {
                “描述”: “用于匹配从任何市场来源类型中提取的主机/域名的正则表达式模式。对于 GitHub 来源，匹配 github.com。对于 Git 来源（SSH 或 HTTPS），从 URL 中提取主机名。在 strictKnownMarketplaces 中使用，以允许来自特定主机的所有市场（例如，“^github\\.mycompany\\.com$”）。”，
                “类型”: “字符串”
              }
            },
            “必填”: [
              “来源”,
              “主机模式”
            ]
          },
          {
            “类型”: “对象”，
            “属性”: {
              “来源”: {
                “类型”: “字符串”，
                “常量”: “pathPattern”
              },
              “路径模式”: {
                “描述”: “对文件和目录来源的 .path 字段进行匹配的正则表达式模式。在 strictKnownMarketplaces 中使用，以允许基于文件系统的市场同时应用针对网络来源的主机模式限制。使用“.*”可允许所有文件系统路径，或使用更窄的模式（例如“^/opt/approved/”）来限制到特定目录。”，
                “类型”: “字符串”
              }
            },
            “必填”: [
              “来源”,
              “路径模式”
            ]
          },
          {
            “描述”: “直接在 settings.json 中定义的内联市场清单。协调器会将一个合成的 marketplace.json 写入缓存；diffMarketplaces 通过比较存储的源（插件数组在此对象内，因此编辑会显示为 sourceChanged）来检测修改。”，
            “类型”: “对象”，
            “属性”"s": {
              "source": {
                "type": "string",
                "const": "settings"
              },
              "name": {
                "description": "市场名称。必须与 extraKnownMarketplaces 键匹配（强制执行）；合成清单将以此名称写入。验证规则与 PluginMarketplaceSchema 相同，并且会拒绝保留名称——validateOfficialNameSource 在磁盘写入之后才运行，已无法进行修复。",
                "type": "string",
                "minLength": 1
              },
              "plugins": {
                "description": "在 settings.json 中内联声明的插件条目",
                "type": "array",
                "items": {
                  "type": "object",
                  "properties": {
                    "name": {
                      "description": "插件在目标仓库中的名称",
                      "type": "string",
                      "minLength": 1
                    },
                    "source": {
                      "description": "插件的获取来源。必须是远程源——相对路径没有可供解析的市场仓库。",
                      "anyOf": [
                        {
                          "description": "相对于市场根目录（包含 .claude-plugin/ 的目录，而非 .claude-plugin/ 自身）的插件根目录路径",
                          "type": "string",
                          "pattern": "^\\.\\/.*"
                        },
                        {
                          "description": "以 NPM 包作为插件来源",
                          "type": "object",
                          "properties": {
                            "source": {
                              "type": "string",
                              "const": "npm"
                            },
                            "package": {
                              "description": "包名（或 URL、本地路径，或其他可作为包传递给 `npm` 的内容）",
                              "anyOf": [
                                {
                                  "type": "string"
                                },
                                {
                                  "type": "string"
                                }
                              ]
                            },
                            "version": {
                              "description": "特定版本或版本范围（如 ^1.0.0、~2.1.0）",
                              "type": "string"
                            },
                            "registry": {
                              "description": "自定义 NPM 注册表 URL（默认使用系统默认注册表，通常是 npmjs.org）",
                              "type": "string",
                              "format": "uri"
                            }
                          },
                          "required": [
                            "source",
                            "package"
                          ]
                        },
                        {
                          "type": "object",
                          "properties": {
                            "source": {
                              "type": "string",
                              "const": "url"
                            },
                            "url": {
                              "description": "完整的 Git 仓库 URL（https:// 或 git@）",
                              "type": "string"
                            },
                            "ref": {
                              "description": "要使用的 Git 分支或标签（如 \"main\"、\"v1.0.0\"）。默认为仓库的默认分支。",
                              "type": "string"
                            },
                            "sha": {
                              "description": "要使用的特定提交 SHA 值",
                              "type": "string",
                              "minLength": 40,
                              "maxLength": 40,
                              "pattern": "^[a-f0-9]{40}$"
                            }
                          },
                          "required": [
                            "source",
                            "url"
                          ]
                        },
                        {
                          "type": "object",
                          "properties": {
                            "source": {
                              "type": "string",
                              "const": "github"
                            },
                            "repo": {
                              "description": "GitHub 仓库，格式为 owner/repo",
                              "type": "string"
                            },
                            "ref": {
                              "description": "要使用的 Git 分支或标签（如 \"main\"、\"v1.0.0\"）。默认为仓库的默认分支。",
                              "type": "string"
                            },
                            "sha": {
                              "description": "要使用的特定提交 SHA 值",
                              "type": "string",
                              "minLength": 40,
                              "maxLength": 40,
                              "pattern": "^[a-f0-9]{40}$"
                            }
                          },
                          "required": [
                            "source",
                            "repo"
                          ]
                        },
                        {
                          "description": "插件位于大型仓库的子目录中（monorepo）。仅克隆指定的子目录，其余部分不会下载。",
                          "type": "object",
                          "properties": {
                            "source": {
                              "type": "string",
                              "const": "git-subdir"
                            },
                            "url": {
                              "description": "Git 仓库：GitHub owner/repo 简写、https:// 或 git@ URL",
                              "type": "string"
                            },
                            "path": {
                              "description": "仓库中包含插件的子目录（如 \"tools/claude-plugin\"）。采用稀疏克隆（--filter=tree:0）以最大限度地减少 monorepo 的带宽消耗。",
                              "type": "string",
                              "minLength": 1
                            },
                            "ref": {
                              "description": "要使用的 Git 分支或标签（如 \"main\"、\"v1.0.0\"）。默认为仓库的默认分支。",
                              "type": "string"
                            },
                            "sha": {
                              "描述": "要使用的特定提交 SHA 值",
                              "类型": "字符串",
                              "最小长度": 40，
                              "最大长度": 40，
                              "模式": "^[a-f0-9]{40}$"
                            }
                          },
                          "required": [
                            "source",
                            "url",
                            "path"
                          ]
                        },
                        {
                          "描述": "插件以 ZIP 压缩包形式通过 HTTPS 下载分发——适用于任何静态文件服务器或制品仓库（S3、GitLab、nginx），客户端无需 Git 或 npm。认证方式：该条目的自有 `headers` / `headersHelper`（绑定到此 URL），叠加在外部 url-source 市场的 headers 上（st`headersHelper` 生成的）时，归档与其源共享同一来源。",
                          "type": "object",
                          "properties": {
                            "source": {
                              "type": "string",
                              "const": "archive"
                            },
                            "url": {
                              "description": "包含插件的 ZIP 归档的 HTTPS URL。插件根目录（即包含 .claude-plugin/ 的目录）可以位于归档的顶层，也可以嵌套在归档内一层——如果存在单层包裹目录，则该目录会被剥离。",
                              "type": "string",
                              "format": "uri"
                            },
                            "sha256": {
                              "description": "归档的 SHA-256 摘要。当设置此字段时，每次下载都会与之校验；若不匹配则拒绝安装。此外，当 plugin.json 和市场条目均未声明 `version` 时，它也会用作版本标识。建议设置此字段。请注意，更新信号是版本字符串（plugin.json 中的版本，若无则取条目版本，再无则以此摘要为准）——仅更改摘要而未声明版本不会触发更新。",
                              "type": "string",
                              "pattern": "^[0-9a-fA-F]{64}$"
                            }
                          },
                          "required": [
                            "source",
                            "url"
                          ]
                        },
                        {
                          "description": "由本地安装的工具生成的插件目录（例如，IDE 为当前选定的 SDK 渲染的插件）。Claude Code 会执行该命令，复制其输出的目录，并在启动时在后台重新执行以获取变更。",
                          "type": "object",
                          "properties": {
                            "source": {
                              "type": "string",
                              "const": "command"
                            },
                            "command": {
                              "description": "Shell 命令，会在标准输出中打印插件目录的绝对路径（且仅一行），并以退出码 0 结束。该命令必须在退出前确保该目录中存在完整的插件；目录会被复制到插件缓存中，因此每次运行时打印的路径可能会变化（每次安装和更新时都会重新解析，且每个会话期间会在后台解析一次）。该命令将在用户的主目录下，使用 Claude Code 的子进程环境，通过平台 Shell 执行（macOS/Linux 上为 sh，Windows 上为 cmd.exe）。",
                              "type": "string",
                              "minLength": 1,
                              "maxLength": 500
                            },
                            "timeout": {
                              "description": "等待命令完成的秒数，超时后将放弃（默认：60 秒）",
                              "type": "integer",
                              "exclusiveMinimum": 0,
                              "maximum": 600
                            },
                            "mode": {
                              "description": "copy（默认）：打印的目录会被复制到插件缓存中并进行内容哈希，因此之后可删除该目录。link：缓存条目直接链接到打印的目录（不复制，无大小限制；适用于 macOS/Linux 系统），适合大型导出；此时目录必须在 Claude Code 运行期间保持有效，且只有打印的路径发生变化才会被视为内容更新。",
                              "type": "string",
                              "enum": [
                                "copy",
                                "link"
                              ]
                            }
                          },
                          "required": [
                            "source",
                            "command"
                          ]
                        },
                        {
                          "description": "用于标记本 Claude Code 版本无法识别的来源类型，或已知类型但其字段验证失败的情况（此时 `error` 字段会记录原因）。绝不会由用户手动创建——PluginMarketplaceSchema 会将无法解析的来源重写为此类型，以使条目仍保留在 marketplace.plugins 中（detectDelistedPlugins 不应将其视为已移除）。尝试安装时会在 cachePlugin 阶段失败，并给出可操作的错误信息。",
                          "type": "object",
                          "properties": {
                            "source": {
                              "type": "string",
                              "const": "unsupported"
                            },
                            "error": {
                              "type": "string"
                            }
                          },
                          "required": [
                            "source"
                          ]
                        }
                      ]
                    },
                    "description": {
                      "type": "string"
                    },
                    "version": {
                      "type": "string"
                    },
                    "strict": {
                      "type": "boolean"
                    },
                    "headers": {
                      "description": "下载该条目 `archive` 来源时发送的 HTTP 头部。",
                      "type": "object",
                      "propertyNames": {
                        "type": "string"
                      },
                      "additionalProperties": {
                        "type": "string"
                      }
                    },
                    "headersHelper": {
                      "description": "打印用于下载该条目 `archive` 来源的 HTTP 头部 JSON 对象的命令。仅在用户明确安装或更新此插件时执行。与目录条目不同，在此处编写的条目无需设置 `strict: false`：它是在设置文件中声明的，而设置文件中没有可内联的清单字段。项目设置中的声明并非由运营方编写，因此请求路由和客户端身份相关的头部名称仍会在此处被过滤。请使用绝对路径。",
                      "type": "string",
                      "maxLength": 500
                    }
                  },
                  "required": [
                    "name",
                    "source"
                  ]
                }
              },
              "owner": {
                "type": "object",
                "properties": {
                  "name": {
                    "description": "插件作者或组织的显示名称",
                    "type": "string",
                    "minLength": 1
                  },
                  "email": {
                    "description": "用于支持或反馈的联系邮箱",
                    "type": "string"
                  },
                  "url": {
                    "description": "网站、GitHub 个人主页或组织的 URL",
                    "type": "string"
                  }
                },
                "required": [
                  "name"
                ]
              }
            },
            "required": [
              "source",
              "name",
              "plugins"
            ]
          }
        ]
      }
    },
    "blockedMarketplaces": {
      "description": "企业级市场来源黑名单。当在托管设置中配置时，这些来源将被禁止添加为市场。条目需完全匹配，例外情况是 GitHub 条目可以使用所有者通配符形式 {\"source\":\"github\",\"repo\":\"owner/*\"} 来阻止该所有者下的所有仓库。检查过程ppens 在下载之前，因此被屏蔽的来源永远不会触及文件系统。",
      "type": "array",
      "items": {
        "anyOf": [
          {
            "type": "object",
            "properties": {
              "source": {
                "type": "string",
                "const": "url"
              },
              "url": {
                "description": "指向 marketplace.json 文件的直接 URL",
                "type": "string",
                "format": "uri"
              },
              "headers": {
                "description": "自定义 HTTP 头（例如用于身份验证）",
                "type": "object",
                "propertyNames": {
                  "type": "string"
                },
                "additionalProperties": {
                  "type": "string"
                }
              },
              "headersHelper": {
                "description": "一个命令，用于输出 HTTP 头的 JSON 对象（例如短期的认证令牌）。其输出会覆盖 `headers` 字段，并且与 `headers` 一样，会被从此市场下载的同源归档继承。该命令在固定目录下运行（Claude 配置主目录，而非会话目录），因此请提供通过 PATH 路径找到的简单命令或绝对路径；每次刷新此市场时都会重新执行。",
                "type": "string",
                "maxLength": 500
              }
            },
            "required": [
              "source",
              "url"
            ]
          },
          {
            "type": "object",
            "properties": {
              "source": {
                "type": "string",
                "const": "github"
              },
              "repo": {
                "description": "GitHub 仓库，格式为 owner/repo。仅在受管理设置策略列表（strictKnownMarketplaces / blockedMarketplaces）中，“owner/*”这种所有者通配符形式才会匹配该所有者下的所有仓库。其他任何地方（市场添加、extraKnownMarketplaces、known_marketplaces.json）都必须指定单个仓库——通配符将被视为字面值，导致克隆失败。",
                "type": "string"
              },
              "ref": {
                "description": "要使用的 Git 分支或标签（例如“main”、“v1.0.0”）。默认使用仓库的默认分支。",
                "type": "string"
              },
              "path": {
                "description": "仓库内 marketplace.json 的路径（默认为 .claude-plugin/marketplace.json）",
                "type": "string"
              },
              "sparsePaths": {
                "description": "通过 git sparse-checkout（cone 模式）指定要包含的目录。适用于市场位于子目录中的 monorepo 项目。示例：[“.claude-plugin”, “plugins”]。若省略，则会克隆整个仓库。",
                "type": "array",
                "items": {
                  "type": "string"
                }
              },
              "skipLfs": {
                "description": "无实际作用，保留此字段是为了兼容现有配置。Claude Code 自带的 Git 不会下载 Git LFS 内容：无论是否设置此项，市场仓库中的 LFS 追踪文件都会以指针文件的形式检出，而市场新增或更新时会报告这些文件的数量。如需获取其内容，请在 ~/.claude/plugins/marketplaces/ 下的市场工作区中执行 `git lfs pull`。",
                "type": "boolean"
              }
            },
            "required": [
              "source",
              "repo"
            ]
          },
          {
            "type": "object",
            "properties": {
              "source": {
                "type": "string",
                "const": "git"
              },
              "url": {
                "description": "完整的 Git 仓库 URL",
                "type": "string"
              },
              "ref": {
                "description": "要使用的 Git 分支或标签（例如“main”、“v1.0.0”）。默认使用仓库的默认分支。",
                "type": "string"
              },
              "path": {
                "description": "仓库内 marketplace.json 的路径（默认为 .claude-plugin/marketplace.json）",
                "type": "string"
              },
              "sparsePaths": {
                "description": "通过 git sparse-checkout（cone 模式）指定要包含的目录。适用于市场位于子目录中的 monorepo 项目。示例：[“.claude-plugin”, “plugins”]。若省略，则会克隆整个仓库。",
                "type": "array",
                "items": {
                  "type": "string"
                }
              },
              "skipLfs": {
                "description": "无实际作用，保留此字段是为了兼容现有配置。Claude Code 自带的 Git 不会下载 Git LFS 内容：无论是否设置此项，市场仓库中的 LFS 追踪文件都会以指针文件的形式检出，而市场新增或更新时会报告这些文件的数量。如需获取其内容，请在 ~/.claude/plugins/marketplaces/ 下的市场工作区中执行 `git lfs pull`。",
                "type": "boolean"
              }
            },
            "required": [
              "source",
              "url"
            ]
          },
          {
            "type": "object",
            "properties": {
              "source": {
                "type": "string",
                "const": "npm"
              },
              "package": {
                "description": "包含 marketplace.json 的 NPM 包",
                "type": "string"
              }
            },
            "required": [
              "source",
              "package"
            ]
          },
          {
            "type": "object",
            "properties": {
              "source": {
                "type": "string",
                "const": "file"
              },
              "path": {
                "description": "本地 marketplace.json 文件的路径",
                "type": "string"
              }
            },
            "required": [
              "source",
              "path"
            ]
          },
          {
            "type": "object",
            "properties": {
              "source": {
                "type": "string",
                "const": "directory"
              },
              "path": {
                "description": "包含 .claude-plugin/marketplace.json 的本地目录",
                "type": "string"
              }
            },
            "required": [
              "source",
              "path"
            ]
          },
          {
            "description": "用于 ~/.claude/skills/ 自动加载（@skills-dir 插件）的策略列表哨兵。在 strictKnownMarketplaces 中：可选择重新启用扫描（默认情况下，任何白名单都会阻止它）。在 blockedMarketplaces 中：关闭扫描功能，同时不对市场进行其他限制。此选项仅在上述两个受管理设置列表中有效（areLocalPluginDirsAllowedByPolicy）；known_marketplaces.json 或市场添加等功能均忽略此项。",
            "type": "object",
            "properties": {
              "source": {
                "type": "string",
                "const": "skills-dir"
              }
            },
            "required": [
              "source"
            ]
          },
          {
            "type": "object",
            "properties": {
              "source": {
                "type": "string",
                "const": "hostPattern"
              },
              "hostPattern": {
                "description": "用于匹配从任何市场来源类型中提取的主机名或域名的正则表达式模式。对于 GitHub 来源，匹配的是 github.com。对于 Git 来源（SSH 或 HTTPS），则从 URL 中提取主机名。可在 strictKnownMarketplaces 中使用，以允许来自特定主机的所有市场（例如“^github\\.mycompany\\.com$”）。",
                "type": "string"
              }
            },
            "r"required": [
              "source",
              "hostPattern"
            ]
          },
          {
            "type": "object",
            "properties": {
              "source": {
                "type": "string",
                "const": "pathPattern"
              },
              "pathPattern": {
                "description": "用于匹配文件和目录来源的 .path 字段的正则表达式模式。在 strictKnownMarketplaces 中使用，以允许基于文件系统的市场与针对网络来源的 hostPattern 限制共存。使用 \".*\" 可允许所有文件系统路径；若需限制到特定目录，可使用更具体的模式（如 \"^/opt/approved/\")。",
                "type": "string"
              }
            },
            "required": [
              "source",
              "pathPattern"
            ]
          },
          {
            "description": "直接在 settings.json 中定义的内联市场清单。协调器会将一个合成的 marketplace.json 写入缓存；diffMarketplaces 通过比较存储的 source 来检测编辑（插件数组包含在此对象中，因此编辑会显示为 sourceChanged）。",
            "type": "object",
            "properties": {
              "source": {
                "type": "string",
                "const": "settings"
              },
              "name": {
                "description": "市场名称。必须与 extraKnownMarketplaces 键一致（强制执行）；合成清单将以该名称写入。验证规则与 PluginMarketplaceSchema 相同，并且还会拒绝保留名称——validateOfficialNameSource 在磁盘写入之后运行，已无法进行修正。",
                "type": "string",
                "minLength": 1
              },
              "plugins": {
                "description": "在 settings.json 中内联声明的插件条目",
                "type": "array",
                "items": {
                  "type": "object",
                  "properties": {
                    "name": {
                      "description": "插件在目标仓库中的名称",
                      "type": "string",
                      "minLength": 1
                    },
                    "source": {
                      "description": "插件的获取来源。必须是远程来源——相对路径没有可供解析的市场仓库。",
                      "anyOf": [
                        {
                          "description": "相对于市场根目录（即包含 .claude-plugin/ 的目录，而非 .claude-plugin/ 本身）的插件根目录路径",
                          "type": "string",
                          "pattern": "^\\.\\/.*"
                        },
                        {
                          "description": "以 NPM 包作为插件来源",
                          "type": "object",
                          "properties": {
                            "source": {
                              "type": "string",
                              "const": "npm"
                            },
                            "package": {
                              "description": "包名（或 URL、本地路径，或其他可作为包传递给 `npm` 的内容）",
                              "anyOf": [
                                {
                                  "type": "string"
                                },
                                {
                                  "type": "string"
                                }
                              ]
                            },
                            "version": {
                              "description": "具体版本或版本范围（如 ^1.0.0、~2.1.0）",
                              "type": "string"
                            },
                            "registry": {
                              "description": "自定义 NPM 注册表 URL（默认使用系统默认注册表，通常是 npmjs.org）",
                              "type": "string",
                              "format": "uri"
                            }
                          },
                          "required": [
                            "source",
                            "package"
                          ]
                        },
                        {
                          "type": "object",
                          "properties": {
                            "source": {
                              "type": "string",
                              "const": "url"
                            },
                            "url": {
                              "description": "完整的 Git 仓库 URL（https:// 或 git@）",
                              "type": "string"
                            },
                            "ref": {
                              "description": "要使用的 Git 分支或标签（如 \"main\"、\"v1.0.0\"）。默认使用仓库的默认分支。",
                              "type": "string"
                            },
                            "sha": {
                              "description": "要使用的特定提交 SHA 值",
                              "type": "string",
                              "minLength": 40,
                              "maxLength": 40,
                              "pattern": "^[a-f0-9]{40}$"
                            }
                          },
                          "required": [
                            "source",
                            "url"
                          ]
                        },
                        {
                          "type": "object",
                          "properties": {
                            "source": {
                              "type": "string",
                              "const": "github"
                            },
                            "repo": {
                              "description": "GitHub 仓库，格式为 owner/repo",
                              "type": "string"
                            },
                            "ref": {
                              "description": "要使用的 Git 分支或标签（如 \"main\"、\"v1.0.0\"）。默认使用仓库的默认分支。",
                              "type": "string"
                            },
                            "sha": {
                              "description": "要使用的特定提交 SHA 值",
                              "type": "string",
                              "minLength": 40,
                              "maxLength": 40,
                              "pattern": "^[a-f0-9]{40}$"
                            }
                          },
                          "required": [
                            "source",
                            "repo"
                          ]
                        },
                        {
                          "description": "插件位于较大仓库的子目录中（monorepo）。仅克隆指定的子目录，其余部分不会下载。",
                          "type": "object",
                          "properties": {
                            "source": {
                              "type": "string",
                              "const": "git-subdir"
                            },
                            "url": {
                              "description": "Git 仓库：可使用 GitHub 的 owner/repo 简写、https:// 或 git@ 格式的 URL",
                              "type": "string"
                            },
                            "path": {
                              "description": "仓库中包含插件的子目录（如 \"tools/claude-plugin\"）。采用部分克隆（--filter=tree:0）方式稀疏克隆，以最大限度地减少 monorepo 的带宽消耗。",
                              "type": "string",
                              "minLength": 1
                           },
                            "ref": {
                              "description": "要使用的 Git 分支或标签（例如“main”、“v1.0.0”）。默认为仓库的默认分支。",
                              "type": "string"
                            },
                            "sha": {
                              "description": "要使用的特定提交 SHA 值",
                              "type": "string",
                              "minLength": 40,
                              "maxLength": 40,
                              "pattern": "^[a-f0-9]{40}$"
                            }
                          },
                          "required": [
                            "source",
                            "url",
                            "path"
                          ]
                        },
                        {
                          "description": "通过 HTTPS 获取的 ZIP 压缩包形式分发的插件——适用于在任何静态文件服务器或制品仓库（如 S3、GitLab、nginx）上托管，客户端无需 Git 或 npm。认证方式：该条目自身的 `headers` / `headersHelper`（绑定到此 URL），当压缩包与其来源同域时，会叠加在包含它的 URL 源市场中的头部信息之上（无论是静态头部还是由 `headersHelper` 生成的头部）。",
                          "type": "object",
                          "properties": {
                            "source": {
                              "type": "string",
                              "const": "archive"
                            },
                            "url": {
                              "description": "包含插件的 ZIP 压缩包的 HTTPS URL。插件根目录（即存放 .claude-plugin/ 的目录）可以位于压缩包的顶层，也可以嵌套在第一层子目录中——如果仅有一层包装目录，则会被剥离。",
                              "type": "string",
                              "format": "uri"
                            },
                            "sha256": {
                              "description": "压缩包的 SHA-256 摘要。若已设置，则每次下载都会与之校验，不匹配时安装将被拒绝。同时，在 plugin.json 和市场条目均未声明 `version` 时，它也用作版本标识。建议设置。请注意，更新信号是版本字符串（plugin.json 中的版本，若无则取条目版本，若两者都无则以此摘要为准）——仅更改摘要而未声明版本不会触发更新。",
                              "type": "string",
                              "pattern": "^[0-9a-fA-F]{64}$"
                            }
                          },
                          "required": [
                            "source",
                            "url"
                          ]
                        },
                        {
                          "description": "由本地安装的工具生成的插件目录（例如，根据当前选定的 SDK 渲染其插件的 IDE）。Claude Code 会执行该命令，复制其输出的目录，并在启动时后台重新执行以获取变更。",
                          "type": "object",
                          "properties": {
                            "source": {
                              "type": "string",
                              "const": "command"
                            },
                            "command": {
                              "description": "Shell 命令，应在标准输出中精确输出一行插件目录的绝对路径并返回退出码 0。该命令必须在退出前确保该目录中存在完整的插件；目录会被复制到插件缓存中，因此每次运行时输出的路径可能不同（每次安装和更新时都会重新解析，且每会话会在后台解析一次）。该命令将在用户的主目录下，使用 Claude Code 的子进程环境，通过平台 Shell 执行（macOS/Linux 上为 sh，Windows 上为 cmd.exe）。",
                              "type": "string",
                              "minLength": 1,
                              "maxLength": 500
                            },
                            "timeout": {
                              "description": "等待命令完成的秒数，超时后将放弃（默认：60秒）",
                              "type": "integer",
                              "exclusiveMinimum": 0,
                              "maximum": 600
                            },
                            "mode": {
                              "description": "copy（默认）：输出的目录会被复制到插件缓存中并进行内容哈希，因此后续可删除该目录。link：缓存条目直接链接到输出的目录（不复制，无大小限制；仅限 macOS/Linux）——适用于大型导出；此时目录必须在 Claude Code 运行期间保持有效，只有输出的路径发生变化才会被视为内容更新。",
                              "type": "string",
                              "enum": [
                                "copy",
                                "link"
                              ]
                            }
                          },
                          "required": [
                            "source",
                            "command"
                          ]
                        },
                        {
                          "description": "用于标记本 Claude Code 版本无法识别的源类型，或已知类型但其字段验证失败的情况（此时 `error` 字段会记录原因）。绝不会由用户手动创建——PluginMarketplaceSchema 会将无法解析的源重写为此类型，以使条目仍保留在 marketplace.plugins 中（detectDelistedPlugins 不应将其视为已移除）。尝试安装时会在 cachePlugin 阶段失败，并给出可操作的错误提示。",
                          "type": "object",
                          "properties": {
                            "source": {
                              "type": "string",
                              "const": "unsupported"
                            },
                            "error": {
                              "type": "string"
                            }
                          },
                          "required": [
                            "source"
                          ]
                        }
                      ]
                    },
                    "description": {
                      "type": "string"
                    },
                    "version": {
                      "type": "string"
                    },
                    "strict": {
                      "type": "boolean"
                    },
                    "headers": {
                      "description": "下载该条目 `archive` 类型源时发送的 HTTP 头部信息。",
                      "type": "object",
                      "propertyNames": {
                        "type": "string"
                      },
                      "additionalProperties": {
                        "type": "string"
                      }
                    },
                    "headersHelper": {
                      "description": "打印用于下载该条目 `archive` 类型源的 HTTP 头部信息 JSON 对象的命令。仅在用户明确安装或更新此插件时执行。与目录条目不同，此处编写的条目无需设置 `strict: false`：它是在设置文件中声明的，而设置文件中没有可用于内联的清单字段。项目设置中的声明并非由运营方编写，因此请求路由和客户端身份的头部名称仍会在那里被过滤。请使用绝对路径。",
                      "type": "string",
                      "maxLength": 500
                    }
                  },
                  "required": [
                    "name",
                    "source"
                  ]
                }
              },
              "owner": {
                "type": "object",
                "properties": {
                  "name": {
                    "description": "插件作者或组织的显示名称",
                    "type": "字符串",
                    "最小长度": 1
                  },
                  "email": {
                    "description": "用于支持或反馈的联系邮箱",
                    "type": "字符串"
                  },
                  "url": {
                    "description": "网站、GitHub 个人主页或组织的 URL",
                    "type": "字符串"
                  }
                },
                "required": [
                  "name"
                ]
              }
            },
            "required": [
              "source",
              "name",
              "plugins"
            ]
          }
        ]
      }
    },
    "disableCommandPluginSources": {
      "description": "控制 `command` 插件源，该插件源的插件目录是通过在本机运行市场声明的命令生成的。true：从 command 源安装、更新或重新解析插件的行为将被完全禁止（即该命令永远不会执行）。false：明确允许。未设置时：遵循 allowManagedHooksOnly 的规则——如果某个组织限制仅在受管设置中执行钩子，则 command 源也会被禁用。此设置仅在受管设置中生效。",
      "类型": "布尔值"
    },
    "disableSideloadFlags": {
      "description": "当设置为 true（且在受管设置中启用）时，启动时会拒绝使用 --plugin-dir、--plugin-url、--agents 以及非 SDK 的 --mcp-config CLI 标志。这将关闭通过 CLI 标志绕过 strictKnownMarketplaces 限制的途径。可与 allowedMcpServers 配合使用，以实现针对每个服务器的 MCP 控制；但此设置不会影响其他 MCP 入口（如 SDK 的 setMcpServers、claude mcp add、.mcp.json）。此外，它还会阻止内部使用这些标志启动 CLI 的情况（详见设置文档）。此设置仅在受管设置中生效，在用户、项目或本地设置中会被忽略。",
      "类型": "布尔值"
    },
    "pluginSuggestionMarketplaces": {
      "描述": "其插件可能作为上下文安装建议（基于相关性的提示）出现的市场名称列表。若未在此白名单中列出，任何市场声明的建议都不会显示；内置的第一方前端设计提示不受影响。此设置仅在受管设置中生效（策略范围）；在用户、项目和本地设置中该键将被忽略。只有当市场已在本机注册，并且其注册来源也在受管设置中声明时，名称才会生效——要么作为该名称的 extraKnownMarketplaces 条目，要么作为 strictKnownMarketplaces 中的一项。以白名单名称注册但来源不同的市场将被忽略。官方市场不受来源要求限制：仅列出其名称即可生效，因为该名称只能从 Anthropic 官方来源注册。",
      "类型": "数组",
      "元素类型": "字符串"
    },
    "forceLoginMethod": {
      "描述": "强制指定登录方式：Claude Pro/Max 使用 \"claudeai\"，Console 计费使用 \"console\"，云网关 OIDC 设备流程使用 \"gateway\"。",
      "类型": "字符串",
      "枚举": [
        "claudeai",
        "console",
        "gateway"
      ]
    },
    "forceLoginGatewayUrl": {
      "描述": "与 forceLoginMethod: \"gateway\" 一起使用时，用于在登录过程中预填并自动连接的云网关 URL。仅在管理员控制的受管设置中生效（MDM / managed-settings.json / 策略助手）；在用户、项目及远程下发的设置中会被忽略。",
      "类型": "字符串",
      "最小长度": 1
    },
    "gatewayInternalNetworks": {
      "描述": "您的云网关所在 IPv4 CIDR 块（最多 4 个，每个块大小为 /8 至 /32，且不重叠）：即贵组织为其内部网络分配的公网地址段，使 /login 能够访问位于该段内的网关。每个地址段必须完全位于私有地址空间之外，因为在私有空间内，/login 无需此键即可接受网关。只有当本机在该连接上的地址也位于同一地址段内时，/login 才会通过直连方式接受位于已列地址段内的网关，因此登录操作必须在地址位于该段内的机器上进行（不能通过代理、VPN 池、容器或 NAT 段外的设备登录）。此设置旨在防止设置文件被复制，但并不能证明位置。仅在管理员控制的受管设置中生效（MDM / managed-settings.json / 策略助手）；在用户、项目及远程下发的设置中会被忽略。",
      "类型": "数组",
      "元素类型": "字符串"
    },
    "parentSettingsBehavior": {
      "描述": "控制 SDK 的父级层级（Options.managedSettings / --managed-settings）是否叠加在当前管理员层级之下。\"first-wins\"（默认）：父级设置被舍弃——仅保留管理员层级作为策略来源。\"merge\"：父级设置中的限制性内容与管理员层级的设置合并，取两者中更严格的值。若不存在管理员层级，则此设置无效（父级设置仍为唯一的策略层级，且仅保留其中的限制性内容）。",
      "类型": "字符串",
      "枚举": [
        "first-wins",
        "merge"
      ]
    },
    "managedSourcesBehavior": {
      "描述": "控制受管设置来源的组合方式。\"first-wins\"（默认）：优先级最高的来源（服务器管理 > MDM（managed plist / HKLM）> managed-settings.json）单独构成受管层级。\"merge\"：所有存在的来源按固定优先级深度合并——服务器管理 > MDM > managed-settings.json；标量取最高来源的值（如 restrictiv e布尔值或枚举——allowManaged*Only 锁、disable* 开关、沙盒锁系列——取各来源中最严格的值），数组则合并，例外情况包括 fallbackModel、allowedMcpServers、availableModels、strictKnownMarketplaces 和 allowedChannelPlugins 的限制性白名单，以及 sandbox.credentials.awsPairs 和 sandbox.ripgrep（由最先设置的来源完全拥有）、modelOverrides（由最先设置的来源提供完整映射，若其优先级低于设置 availableModels 的来源，则该映射将被舍弃）、managedMcpServers（服务器名称合并；两个来源同时设置某名称时，取较高来源的完整条目），还有仅取自最高来源的键：认证密钥 forceLoginOrgUUID、forceLoginMethod、forceLoginGatewayUrl 和 gatewayInternalNetworks，凭证辅助工具 apiKeyHelper、awsAuthRefresh、awsCredentialExport、gcpAuthRefresh、otelHeadersHelper 和 proxyAuthHelper，modelPicker、permissions.defaultMode、parentSettingsBehavior 以及 policyHelper 配置（env 保持各自的键级合并）。仅采用优先级最高的来源；请确保所有较低来源均由管理员控制，否则较低来源可能会贡献诸如 permissions.allow 之类的条目。HKCU 和 --managed-settings 不参与合并。",
      "类型": "字符串",
      "枚举": [
        "first-wins",
        "merge"
      ]
    },
    "forceLoginOrgUUID": {
      "描述": "OAuth 登录时需验证的组织 UUID。可接受单个 UUID 字符串或 UUID 数组（任一匹配均可）。在受管设置中启用后，若认证账户不属于所列组织，则登录失败。",
      "联合类型": [
        {
          "类型": "字符串"
        },
        {
          "类型": "数组",
          "元素类型": "字符串"
        }
      ]
    },
    "forceRemoteSettingsRefresh": {
      "描述": "在受管设置中启用后，CLI 将阻塞启动，直到成功获取最新的远程受管设置；若获取失败，则退出程序。",
      "类型": "布尔值"
    },
    "otelHeadersHelper": {
      "描述": "输出 OpenTelemetry 头信息的脚本路径。",
      "类型": "字符串"
    },
    "outputStyle": {
      "描述": "控制助手回复的输出样式。",
      "类型": "字符串"
    },
    "viewMode": {
      "描述": "启动时的默认对话视图模式。",
      "t"type": "字符串",
      "enum": [
        "默认",
        "详细",
        "聚焦"
      ]
    },
    "language": {
      "description": "Claude 回答和语音听写所使用的首选语言（例如“日语”、“西班牙语”）",
      "type": "字符串"
    },
    "skipWebFetchPreflight": {
      "description": "对于具有严格安全策略的企业环境，跳过 WebFetch 黑名单检查",
      "type": "布尔值"
    },
    "sandbox": {
      "type": "对象",
      "properties": {
        "enabled": {
          "type": "布尔值"
        },
        "failIfUnavailable": {
          "description": "如果 sandbox.enabled 为 true 但沙箱无法启动（缺少依赖项或平台不支持），则在启动时以错误退出。当该选项为 false 时（默认值），会显示警告，并且命令将在未启用沙箱的情况下运行。此设置适用于需要强制启用沙箱的托管配置部署。",
          "type": "布尔值"
        },
        "autoAllowBashIfSandboxed": {
          "type": "布尔值"
        },
        "allowUnsandboxedCommands": {
          "description": "允许通过参数 dangerouslyDisableSandbox 在沙箱外执行命令。当该选项为 false 时，参数 dangerouslyDisableSandbox 将被完全忽略，所有命令都必须在沙箱内运行。默认值：true。",
          "type": "布尔值"
        },
        "network": {
          "type": "对象",
          "properties": {
            "allowedDomains": {
              "type": "数组",
              "items": {
                "type": "字符串"
              }
            },
            "deniedDomains": {
              "description": "始终被阻止的域名，即使它们也匹配 allowedDomains 中的规则。支持与 allowedDomains 相同的通配符语法。来自所有设置来源的 deniedDomains 规则都会合并，不受 allowManagedDomainsOnly 设置的影响。",
              "type": "数组",
              "items": {
                "type": "字符串"
              }
            },
            "strictAllowlist": {
              "description": "当该选项为 true 时，沙箱运行时会确定性地拒绝所有不在 allowedDomains 列表中的主机，而不是弹出提示。此设置仅对沙箱内的命令生效——像 WebFetch 这样的进程内工具不受此设置限制。仅在用户、托管/策略或 CLI（--settings）设置中生效，项目设置（.claude/settings.json 和 .claude/settings.local.json）会被忽略。",
              "type": "布尔值"
            },
            "allowManagedDomainsOnly": {
              "description": "当该选项为 true 且在托管设置中启用时，仅允许来自托管设置的 allowedDomains 和 WebFetch(domain:...) 允许规则生效。用户、项目、本地及标志设置中的域名规则将被忽略。来自所有来源的 deniedDomains 规则仍然有效。",
              "type": "布尔值"
            },
            "allowUnixSockets": {
              "description": "仅限 macOS：允许访问的 Unix 套接字路径。在 Linux 上被忽略（seccomp 无法按路径过滤）。",
              "type": "数组",
              "items": {
                "type": "字符串"
              }
            },
            "allowAllUnixSockets": {
              "description": "如果为 true，则允许所有 Unix 套接字（在两个平台上均禁用拦截功能）。",
              "type": "布尔值"
            },
            "allowLocalBinding": {
              "type": "布尔值"
            },
            "allowMachLookup": {
              "description": "仅限 macOS：允许查找的额外 XPC/Mach 服务名称。支持后缀通配符匹配（如“com.apple.coresimulator.*”）。这是使用 XPC 进行通信的工具（如 iOS 模拟器或 Playwright）所必需的。",
              "type": "数组",
              "items": {
                "type": "字符串"
              }
            },
            "httpProxyPort": {
              "type": "数字"
            },
            "socksProxyPort": {
              "type": "数字"
            },
            "tlsTerminate": {
              "description": "[实验性] 启用进程内 TLS 终止功能，以便每请求过滤器能够查看 HTTPS 请求体。提供 CA 证书和密钥，或者两者都省略，让沙箱运行时为本次会话生成一个临时证书。在原生 Windows 系统上，临时 CA 无法通过沙箱的信任检查，因此省略路径时将使用由沙箱运行时管理的持久性 CA（通过 /sandbox install 配置并信任）；而配置的路径会原封不动地传递给沙箱运行时，后者会在沙箱初始化时拒绝无效或不完整的证书/密钥对。此设置仅在用户、托管/策略或 CLI（--settings）设置中生效，项目设置（.claude/settings.json 和 .claude/settings.local.json）会被忽略。",
              "type": "对象",
              "properties": {
                "caCertPath": {
                  "type": "字符串",
                  "最小长度"：1
                },
                "caKeyPath": {
                  "type": "字符串",
                  "最小长度"：1
                }
              }
            }
          }
        },
        "filesystem": {
          "type": "对象",
          "properties": {
            "allowWrite": {
              "description": "允许在沙箱内写入的额外路径。与 Edit(...) 允许权限规则中的路径合并。",
              "type": "数组",
              "items": {
                "type": "字符串"
              }
            },
            "denyWrite": {
              "description": "禁止在沙箱内写入的额外路径。与 Edit(...) 拒绝权限规则中的路径合并。",
              "type": "数组",
              "items": {
                "type": "字符串"
              }
            },
            "denyRead": {
              "description": "禁止在沙箱内读取的额外路径。与 Read(...) 拒绝权限规则中的路径合并。",
              "type": "数组",
              "items": {
                "type": "字符串"
              }
            },
            "allowRead": {
              "description": "重新允许在 denyRead 区域内读取的路径。对于匹配的路径，其优先级高于 denyRead。",
              "type": "数组",
              "items": {
                "type": "字符串"
              }
            },
            "allowManagedReadPathsOnly": {
              "description": "当该选项为 true（在托管设置中启用）时，仅使用 policySettings 中的 allowRead 路径。",
              "type": "布尔值"
            },
            "disabled": {
              "description": "仅限 macOS 和 Linux/WSL：完全跳过文件系统隔离，同时保留网络和 seccomp 隔离。在原生 Windows 上被忽略，在该平台上沙箱进程以独立用户身份运行，本身没有任何权限，因此跳过文件系统规则只会撤销所有访问权限，而不会放宽限制——所以那里的文件系统隔离仍然开启。沙箱内的命令可无限制地读写宿主文件系统；网络出口仍受限于 network.allowedDomains。此设置适用于那些目标是控制出口而非隔离文件系统的部署场景。它不会改变 Bash 的提示行为：sandbox.autoAllowBashIfSandboxed 是独立的，默认仍为 true，若要保持对沙箱内命令的提示，请将其设为 false。对于沙箱内的命令，它会取消 filesystem.denyRead 和 credentials.files 拒绝条目的读取保护，因为这两项均由被关闭的文件系统层强制执行；而 credentials.files 掩码条目（哨兵绑定）以及 credentials.envVars 的拒绝/掩码规则则不受影响。此设置仅在用户、托管/策略或 CLI（--settings）设置中生效，项目设置（.claude/settings.json 和 .claude/settings.local.json）会被忽略。如果托管设置配置了 sandbox.filesystem 或列出了任何 sandbox.credentials.files 拒绝条目，则只有托管设置才能设置此项：部署了文件系统限制的管理员不得允许用户可写的文件将其关闭。（sandbox.credentials.envVars 和 credentials.files 掩码条目并不会锁定该项）— 环境变量清理和哨兵绑定与文件系统层无关，并且在该设置下仍然有效。）未设置时，文件系统隔离保持开启状态。",
              "type": "boolean"
            }
          }
        },
        "credentials": {
          "type": "object",
          "properties": {
            "files": {
              "description": "要保护的凭据文件或目录。`deny` 会阻止沙箱内对该文件的读取；`mask` 则会在沙箱内用哨兵值替换（可选择整文件替换或仅替换 `extract` 捕获的部分），并在代理处注入真实值。在 macOS 和 Windows 上，`mask` 会退化为 `deny`。",
              "type": "array",
              "items": {
                "type": "object",
                "properties": {
                  "path": {
                    "description": "凭据文件或目录的路径。解析方式与 `sandbox.filesystem.*` 路径相同：绝对路径、已展开的 `~` 或相对于设置文件根目录的相对路径（项目设置为项目根目录，用户设置为 `~/.claude`）。",
                    "type": "string",
                    "minLength": 1
                  },
                  "mode": {
                    "description": "此路径的访问模式。`deny` 会阻止沙箱内对该文件的读取；`mask` 则向沙箱内的命令提供一份被哨兵替换的副本（可选择整文件替换，或仅替换由 `extract` 捕获的部分），并在出口处由主机代理将哨兵替换为真实值以注入到 `injectHosts` 中。在 macOS 和 Windows 上，`mask` 目前会退化为 `deny`。",
                    "type": "string",
                    "enum": [
                      "deny",
                      "mask"
                    ]
                  },
                  "extract": {
                    "description": "当模式为 `mask` 时，用于结构化遮蔽的可选正则表达式。该正则表达式全局应用于文件；每次匹配中捕获组 1 即为凭据值，只有这些被捕获的部分会被替换成哨兵——文件其余部分则保持原样，以便解析该文件的工具（如 .netrc、JSON、YAML）仍能正常运行。若未设置 `extract`，则整个文件内容将被单一哨兵值替换（整文件遮蔽，适用于仅包含单个秘密的文件）。如果正则表达式未匹配任何内容，则行为由 `onExtractNoMatch` 决定（默认为 `warn`）。对于 `deny` 模式，该选项虽可接受但会被忽略。",
                    "type": "string"
                  },
                  "onExtractNoMatch": {
                    "description": "当 `extract` 在文件中未匹配到任何内容时——或者在使用 `decode` 时，若无任何候选凭据通过验证时——应如何处理。`warn`（默认）会在沙箱内发出一条 stderr 警告，并使文件保持原样可读（开放失败，适用于可能确实不存在的凭据）；`deny` 会将该条目降级为 `deny` 模式，使文件不可读（关闭失败）——在 `sandbox.filesystem.disabled` 模式下，则视为 `error`，因为该模式下读取被拒绝的操作会被直接丢弃；`error` 会在沙箱初始化时直接终止，直到配置修复后才会继续运行。该选项仅在模式为 `mask` 且设置了 `extract` 或 `decode` 时有意义，否则会被忽略。",
                    "type": "string",
                    "enum": [
                      "warn",
                      "deny",
                      "error"
                    ]
                  },
                  "decode": {
                    "description": "用于 `mask` 模式的可选编码凭据格式。`jwt`：候选凭据通过内置的 JWT 正则表达式（或显式设置的 `extract` 模式）定位，在遮蔽前会先验证其是否为有效的 JWT，并用一个结构上合法的假 JWT 替换，以确保沙箱内客户端对令牌的解析功能不受影响。若无任何候选凭据通过验证，则行为由 `onExtractNoMatch` 决定（默认为 `warn`）。对于 `deny` 模式，该选项虽可接受但会被忽略。",
                    "type": "string",
                    "enum": [
                      "jwt"
                    ]
                  },
                  "maskClaims": {
                    "description": "在每个解码后的凭据值中，需要遮蔽的顶级负载声明名称，而不是替换整个令牌。每个具有字符串值的指定声明都会被单独替换为哨兵，并基于修改后的负载重新构建令牌；其余声明则保持不变，以便解析令牌并读取非敏感声明的工具仍能正常工作。需配合 `decode` 使用。若在所有通过验证的令牌中均未找到任何指定声明，则行为由 `onExtractNoMatch` 决定（默认为 `warn`）。该选项仅在模式为 `mask` 时有意义，对于 `deny` 模式则会被忽略。",
                    "type": "array",
                    "items": {
                      "type": "string"
                    }
                  },
                  "maskDuplicates": {
                    "description": "若为真，则在正则表达式匹配范围之外，所有与被捕获凭据值完全相同的出现位置也会被替换为对应的哨兵——适用于那些未被正则覆盖但重复出现的秘密值（例如被粘贴到注释中）。该功能按原始子串匹配，因此较短或常见的值可能会破坏无关内容；适用于较长且高熵的秘密值。默认为 false。该选项仅在模式为 `mask` 且设置了 `extract` 或 `decode` 时有意义，否则会被忽略。",
                    "type": "boolean"
                  },
                  "injectHosts": {
                    "description": "用于限定代理注入该凭据的具体目标范围的可选参数。仅在模式为 `mask` 时有意义，对于 `deny` 模式则会被忽略。若未设置，则默认为 `network.allowedDomains`——即在所有可访问的主机上注入该凭据。每个目标必须可通过 `network.allowedDomains` 访问（沙箱运行时会进行验证）。",
                    "type": "array",
                    "items": {
                      "type": "string"
                    }
                  }
                },
                "required": [
                  "path",
                  "mode"
                ]
              }
            },
            "envVars": {
              "description": "要保护的环境变量。`deny` 会为沙箱中的命令取消设置该变量；`mask` 则会在沙箱内用哨兵值替换，并在代理处注入真实值。",
              "type": "array",
              "items": {
                "type": "object",
                "properties": {
                  "name": {
                    "description": "环境变量的名称。",
                    "type": "string",
                    "pattern": "^[A-Za-z_][A-Za-z0-9_]*$"
                  },
                  "mode": {
                    "description": "该环境变量的访问模式。`deny` 会为沙箱中的命令取消设置该变量；`mask` 则向沙箱中的命令提供哨兵值，并在出口处由主机代理将哨兵替换为真实值以注入到 `injectHosts` 中。",
                    "type": "string",
                    "enum": [
                      "deny",
                      "mask"
                    ]
                  },
                  "extract": {
                    "description": "当模式为 `mask` 时，用于结构化遮蔽的可选正则表达式。该正则表达式全局应用于变量值；每次匹配中捕获组 1 即为凭据值，只有这些被捕获的部分会被替换成哨兵——变量值的其余部分则保持原样，以便解析该值的工具（如 `DATABASE_URL` 连接字符串，或复合的 `KEY:SECRET` 对）在沙箱内仍能正常运行。若未设置 `extract`，则整个变量值将被单一哨兵值替换（整值遮蔽，适用于纯令牌）。如果正则表达式未匹配到任何内容，则行为由 `onExtractNoMatch` 决定（默认为 `warn`）。该选项不能与 `decode` 同时使用（解码路径不会参考它）。对于 `deny` 模式，该选项虽可接受但会被忽略。",
                    "type": "string"
                  },
                  "onExtractNoMatch": {
                    "description": "当 `extract` 在变量值中未匹配到任何内容时，应如何处理。`warn`（默认）会在沙箱内发出一条 stderr 警告，并让该变量不加遮蔽地通过（开放失败，适用于可能确实不存在的凭据）它可能合法地不存在）；`deny` 会在沙箱内清空该变量（默认关闭）；`error` 会在沙箱初始化时中止，直到配置修复前不会运行任何内容。此选项仅在模式为 `mask` 且设置了 `extract` 而未设置 `decode` 时有意义。对于带有 `decode` 的掩码条目，运行时会走解码路径，而不会检查此字段，因此无法执行默认关闭策略——此时 `deny` 和 `error` 都会被拒绝，仅接受 `warn`。在其他所有情况下，该字段虽被接受但会被忽略。
                    "type": "string",
                    "enum": [
                      "warn",
                      "deny",
                      "error"
                    ]
                  },
                  "decode": {
                    "description": "可选的编码凭据格式，用于 `mask` 模式。`jwt`：验证变量的完整值是否确实为 JWT，并将其替换为结构上有效的伪造 JWT，以使沙箱内的客户端令牌解析继续正常工作；代理会在出站时交换整个伪造令牌。如果验证失败，则变量保持未掩码状态，并在标准错误输出中发出警告（默认开放）。不能与 `extract` 同时使用——解码路径从不检查此字段。对于 `deny` 模式，此选项虽被接受但会被忽略。
                    "type": "string",
                    "enum": [
                      "jwt"
                    ]
                  },
                  "maskClaims": {
                    "description": "在解码后的值中需要掩码的顶层载荷声明名称，而不是替换整个令牌。每个具有字符串值的指定声明都会被赋予一个哨兵标记，并基于修改后的载荷重新构建令牌；其余声明则保持不变，以便读取声明的客户端继续正常工作。需要配合 `decode` 使用。如果没有匹配的声明，则变量保持未掩码状态，并在标准错误输出中发出警告（默认开放）。仅当模式为 `mask` 时有意义；对于 `deny` 模式，此选项虽被接受但会被忽略。
                    "type": "array",
                    "items": {
                      "type": "string"
                    }
                  },
                  "injectHosts": {
                    "description": "可选的限制范围，用于指定代理在哪些位置注入此凭据。仅当模式为 `mask` 时有意义；对于 `deny` 模式，此选项虽被接受但会被忽略。若未设置，则默认为 `network.allowedDomains`——凭据将在每个可达主机处注入。每个条目都必须可通过 `network.allowedDomains` 访问（沙箱运行时会进行验证）。
                    "type": "array",
                    "items": {
                      "type": "string"
                    }
                  }
                },
                "required": [
                  "name",
                  "mode"
                ]
              }
            },
            "allowPlaintextInject": {
              "description": "允许在纯 HTTP 代理路径上进行哨兵→真实值的替换。默认为 false：在未终止 TLS 的情况下，上游身份未经验证，凭据将以明文传输。仅适用于可信网络的测试环境。仅在用户、托管/策略或 CLI（`--settings`）设置中生效——项目设置（.claude/settings.json 和 .claude/settings.local.json）将被忽略。
              "type": "boolean"
            },
            "awsPairs": {
              "description": "将已掩码的环境变量显式分组为 AWS 凭证对，用于 SigV4 重新签名，适用于非标准变量名。传统的 AWS_ACCESS_KEY_ID / AWS_SECRET_ACCESS_KEY / AWS_SESSION_TOKEN 三元组在被掩码时会自动配对。仅在用户、托管/策略或 CLI（`--settings`）设置中生效——项目设置（.claude/settings.json 和 .claude/settings.local.json）将被忽略。只有当某个成员的环境变量作为整体值的 `mask` 条目被转发时才可用（带有 `extract` 或 `decode` 的条目不符合条件——重新签名需要完整的原始值）。如果一对中的密钥 ID 或秘密成员不可用，则该对不会参与重新签名，会被丢弃；除非其命名的是传统 AWS 变量，此时会作为惰性抑制器被转发，以确保隐式自动配对被覆盖。如果仅会话令牌成员不可用，该对仍会参与重新签名，但不会添加 x-amz-security-token（临时凭证请求将在上游失败，直到该条目被修复）。
              "type": "array",
              "items": {
                "type": "object",
                "properties": {
                  "accessKeyIdVar": {
                    "description": "存放 AWS 访问密钥 ID 的已掩码环境变量名称。
                    "type": "string",
                    "pattern": "^[A-Za-z_][A-Za-z0-9_]*$"
                  },
                  "secretAccessKeyVar": {
                    "description": "存放 AWS 秘密访问密钥的已掩码环境变量名称。
                    "type": "string",
                    "pattern": "^[A-Za-z_][A-Za-z0-9_]*$"
                  },
                  "sessionTokenVar": {
                    "description": "存放 AWS 会话令牌（临时凭证）的已掩码环境变量名称，可选。若设置，代理会在重新签名的请求中将真实令牌作为 x-amz-security-token 发送，并在客户端未提供的情况下将其加入签名头集。
                    "type": "string",
                    "pattern": "^[A-Za-z_][A-Za-z0-9_]*$"
                  }
                },
                "required": [
                  "accessKeyIdVar",
                  "secretAccessKeyVar"
                ]
              }
            },
            "sigv4": {
              "description": "针对代理无法重新签名的 AWS SigV4 请求形状（流式、预签名、SigV4A），当这些请求引用了已掩码的凭证对时所采用的策略：`deny`（默认）或 `passthrough`。仅在用户、托管/策略或 CLI（`--settings`）设置中生效——项目设置（.claude/settings.json 和 .claude/settings.local.json）将被忽略。
              "type": "object",
              "properties": {
                "streaming": {
                  "description": "针对 AWS 分块流式上传（x-amz-content-sha256: STREAMING-*）的策略：每一块的签名都依赖于种子签名，因此重新签名需要重写整个请求体。`deny`（默认）会以 403 错误响应关闭连接；`passthrough` 则会原样转发未签名的请求（上游会拒绝其签名）。
                  "type": "string",
                  "enum": [
                    "deny",
                    "passthrough"
                  ]
                },
                "presigned": {
                  "description": "针对预签名 URL（查询参数中包含 X-Amz-Algorithm/X-Amz-Signature，无 Authorization 头）的策略：签名直接嵌入在 URL 中。`deny`（默认）或 `passthrough`。
                  "type": "string",
                  "enum": [
                    "deny",
                    "passthrough"
                  ]
                },
                "sigv4a": {
                  "description": "针对 SigV4A（AWS4-ECDSA-P256-SHA256）非对称签名的策略：由于没有共享密钥 HMAC 可供重新计算，只能选择 `deny`（默认）或 `passthrough`。
                  "type": "string",
                  "enum": [
                    "deny",
                    "passthrough"
                  ]
                }
              }
            }
          }
        },
        "ignoreViolations": {
          "type": "object",
          "propertyNames": {
            "type": "string"
          },
          "additionalProperties": {
            "type": "array",
            "items": {
              "type": "string"
            }
          }
        },
        "enableWeakerNestedSandbox": {
          "type": "boolean"
        },
        "enableWeakerNetworkIsolation": {
          "description": "仅限 macOS：允许沙箱访问 com.apple.trustd.agent。这对于基于 Go 的 CLI 工具（gh、gcloud、terraform 等）在使用 httpProxyPort 时验证 TLS 证书是必需的。“一种中间人代理和自定义 CA。**降低安全性**——通过 trustd 服务打开了潜在的数据外泄通道。默认值：false”，
          “类型”： “布尔型”
        },
        “allowAppleEvents”： {
          “描述”： “仅限 macOS：允许沙盒命令发送 Apple 事件（并查找 appleeventsd Mach 服务）。`open`、`osascript` 以及基于浏览器的打开 URL 的认证流程需要此功能。**取消代码执行隔离**——沙盒命令可以在没有用户提示的情况下以非沙盒模式启动其他应用，并且可以对正在运行的应用（例如终端）进行脚本控制，但需遵守用户针对各应用的 TCC 自动化权限设置。此设置仅在用户、托管/策略或 CLI（--settings）配置中生效——项目设置文件（.claude/settings.json 和 .claude/settings.local.json）将被忽略。默认值：false”，
          “类型”： “布尔型”
        },
        “excludedCommands”： {
          “类型”： “数组”，
          “items”： {
            “类型”： “字符串”
          }
        },
        “ripgrep”： {
          “描述”： “用于内置 ripgrep 支持的自定义 ripgrep 配置。仅在用户、托管/策略或 CLI（--settings）配置中生效——项目设置文件（.claude/settings.json 和 .claude/settings.local.json）将被忽略。”，
          “类型”： “对象”，
          “properties”： {
            “command”： {
              “类型”： “字符串”
            },
            “args”： {
              “类型”： “数组”，
              “items”： {
                “类型”： “字符串”
              }
            }
          },
          “required”： [
            “command”
          ]
        },
        “bwrapPath”： {
          “描述”： “仅限 Linux/WSL：指向 bwrap（bubblewrap）二进制文件的绝对路径。会覆盖通过 PATH 自动检测的结果。仅在管理员控制的托管设置中生效。”，
          “类型”： “字符串”
        },
        “socatPath”： {
          “描述”： “仅限 Linux/WSL：用于沙盒网络代理的 socat 二进制文件的绝对路径。会覆盖通过 PATH 自动检测的结果。仅在管理员控制的托管设置中生效。”，
          “类型”： “字符串”
        }
      },
      “additionalProperties”： {}
    },
    “feedbackSurveyRate”： {
      “描述”： “会话质量调查在符合条件时出现的概率（0–1）。0.05 是一个合理的初始值。”，
      “类型”： “数字”，
      “minimum”： 0，
      “maximum”： 1
    },
    “feedbackDrafts”： {
      “描述”： “由模型草拟的反馈（SendFeedback 工具）。‘notify’（默认）会在草稿排队时显示一行通知；‘quiet’仅显示页脚计数器；‘off’则完全禁用该工具，使草稿永不排队。”，
      “类型”： “字符串”，
      “enum”： [
        “notify”，
        “quiet”，
        “off”
      ]
    },
    “spinnerTipsEnabled”： {
      “描述”： “是否在加载动画中显示提示信息”，
      “类型”： “布尔型”
    },
    “spinnerVerbs”： {
      “描述”： “自定义加载动画中的动词。mode：‘append’在默认动词基础上追加，‘replace’则仅使用您提供的动词。”，
      “类型”： “对象”，
      “properties”： {
        “mode”： {
          “类型”： “字符串”，
          “enum”： [
            “append”，
            “replace”
          ]
        },
        “verbs”： {
          “类型”： “数组”，
          “items”： {
            “类型”： “字符串”
          }
        }
      },
      “required”： [
        “mode”，
        “verbs”
      ]
    },
    “spinnerTipsOverride”： {
      “描述”： “将贵组织自己的提示加入到加载动画的提示轮换中。tips：可以是字符串，也可以是 {id, text, cooldownSessions?, priority?} 对象；tipsFile：包含相同格式的 JSON 文件；label：在您的提示前显示的前缀；excludeDefault：若为 true，则仅显示您的提示（默认为 false）。”，
      “类型”： “对象”，
      “properties”： {
        “excludeDefault”： {
          “类型”： “布尔型”
        },
        “tips”： {
          “类型”： “数组”，
          “items”： {
            “anyOf”： [
              {
                “类型”： “字符串”
              },
              {
                “description”： “{ id：稳定 ID（字母、数字、“.”、“_”、“-”；最多 64 个字符），text：提示内容（最多 500 字符，单行），cooldownSessions?：再次显示前需等待的会话数（默认 0），priority?：在从未显示过的提示中用于破除平局的权重（默认 0） }”，
                “类型”： “对象”，
                “properties”： {},
                “additionalProperties”： {}
              }
            ]
          }
        },
        “tipsFile”： {
          “描述”： “指向包含提示数组的 JSON 文件的绝对路径或 ~/ 下的本地路径；仅在用户、--settings 以及磁盘上的托管设置中生效。每个 CLI 进程只读取一次（重启后才会加载修改后的文件）。”，
          “类型”： “字符串”
        },
        “label”： {
          “描述”： “在加载动画中您的提示前显示的前缀（默认为‘Tip’）”，
          “类型”： “字符串”
        }
      },
      “additionalProperties”： {}
    },
    “syntaxHighlightingDisabled”： {
      “描述”： “是否在差异比较中禁用语法高亮”，
      “类型”： “布尔型”
    },
    “spellcheck”： {
      “描述”： “在输入提示时，利用已安装的 aspell、hunspell 或 ispell 对拼写错误的单词进行下划线标注（除非‘enabled’为真，否则不启用；若未安装任何拼写检查器，则无任何效果）。仅从用户、标志位及托管设置中读取（以优先级最高的来源为准）；在项目 .claude/settings.json 和 .claude/settings.local.json 中将被忽略。”，
      “类型”： “对象”，
      “properties”： {
        “enabled”： {
          “描述”： “开启提示输入的拼写检查（默认：假）”，
          “类型”： “布尔型”
        },
        “checker”： {
          “描述”： “要使用的拼写检查器：‘aspell’、‘hunspell’、‘ispell’，或‘auto’（默认）——即在 PATH 中找到的第一个可用检查器。”，
          “类型”： “字符串”
        },
        “language”： {
          “描述”： “使用的词典，原样传递给检查器（aspell --lang，hunspell -d，ispell -d），例如‘en_GB’；名称因检查器而异（仅允许字母、数字以及‘_’‘-’‘.’‘,’）。默认值：检查器自身的默认词典。”，
          “类型”： “字符串”
        },
        “color”： {
          “描述”： “拼写错误单词的颜色（同时会加上下划线）：可以是终端颜色名，如‘red’或‘magenta’，也可以是‘#rrggbb’、‘rgb(r,g,b)’、‘ansi256(n)’或‘ansi:<name>’。默认值：主题的错误颜色。”，
          “类型”： “字符串”
        }
      },
      “additionalProperties”： {}
    },
    “terminalTitleFromRename”： {
      “描述”： “/rename 是否更新终端标签页标题（默认为真）。设为假可保留自动生成的主题标题。”，
      “类型”： “布尔型”
    },
    “promptCacheTtl”： {
      “描述”： “主对话（交互式、-p 和 SDK 轮次，以及与其内联运行的助手）的提示缓存 TTL：‘5m’或‘1h’。未设置时为自动模式：Claude 订阅在用量限制内为 1 小时，API 密钥、Bedrock、Vertex 或 Foundry 则为 5 分钟。1 小时缓存的写入费用更高；缓存在较长时间的中断后仍能保持活跃状态。环境变量 CLAUDE_CODE_PROMPT_CACHE_TTL 具有优先权。”，
      “类型”： “字符串”，
      “enum”： [
        “5m”，
        “1h”
      ]
    },
    “subagentPromptCacheTtl”： {
      “描述”： “主对话之外的所有内容——子代理、工作流、后台请求及辅助请求——的提示缓存 TTL：‘5m’或‘1h’。未设置时为自动模式（5 分钟，除非 ENABLE_PROMPT_CACHING_1H=1）。环境变量 CLAUDE_CODE_SUBAGENT_PROMPT_CACHE_TTL 具有优先权。”，
      “类型”： “字符串”，
      “enum”： [
        “5m”，
        “1h”
      ]
    },
    “alwaysThinkingEnabled”： {
      “描述”： “当为假时，思考功能被禁用。当未设置或为真时，支持的模型会自动启用思考功能。”，
      “类型”： “布尔型”
    },
    “effortLevel”： {
      “描述”： “支持模型的持久化努力等级。”，
      “类型”： “字符串”，    "enum": [
        "低",
        "中",
        "高",
        "超高"
      ]
    },
    "maxEffortLevel": {
      "description": "最大努力级别。任何高于该级别的设置（包括 /effort 或 /model 选择、--effort 参数、CLAUDE_CODE_EFFORT_LEVEL 环境变量或模型的默认值）都会被限制到该级别，适用于所有提供商，包括 Bedrock、Vertex 和 Foundry。它会与组织针对各模型设定的努力上限取较低值；在多个设置文件之间，最低值生效，且 modelSettings.<model>.maxEffortLevel 会按模型覆盖该设置。此限制在客户端强制执行：通过 CLAUDE_CODE_EXTRA_BODY 提供的努力级别不会被钳制。",
      "type": "string",
      "enum": [
        "低",
        "中",
        "高",
        "超高",
        "最大"
      ]
    },
    "modelSettings": {
      "description": "按规范模型名称索引的各模型设置。",
      "type": "object",
      "propertyNames": {
        "type": "string"
      },
      "additionalProperties": {
        "type": "object",
        "properties": {
          "effortLevel": {
            "description": "该模型的持久化努力级别。",
            "type": "string",
            "enum": [
              "低",
              "中",
              "高",
              "超高"
            ]
          },
          "maxEffortLevel": {
            "description": "该模型的最大努力级别。在同一设置文件中，它会覆盖顶级的 maxEffortLevel 设置（“最大”则豁免该限制）；在不同设置文件之间，适用的最低值生效。其键名与 effortLevel 一致：规范模型名称也适用于其带日期后缀、[1m] 标记以及 Bedrock 和 Vertex 的拼写形式。",
            "type": "string",
            "enum": [
              "低",
              "中",
              "高",
              "超高",
              "最大"
            ]
          }
        },
        "additionalProperties": {}
      }
    },
    "ultracode": {
      "description": "为当前会话启用 ultracode：超高努力级别并结合持续的动态工作流编排。该设置仅限于会话范围——通常通过 --settings 参数或 apply_flag_settings 控制请求提供；交互式切换不会使其持久化。需要启用工作流功能，并使用支持超高努力级别的模型。",
      "type": "boolean"
    },
    "autoCompactWindow": {
      "description": "自动折叠窗口大小",
      "type": "integer",
      "minimum": 100000,
      "maximum": 1000000
    },
    "advisorModel": {
      "description": "用于服务器端顾问工具的顾问模型。",
      "type": "string"
    },
    "fastMode": {
      "description": "当为 true 时，快速模式启用；当缺失或为 false 时，快速模式关闭。",
      "type": "boolean"
    },
    "fastModePerSessionOptIn": {
      "description": "当为 true 时，快速模式不会跨会话持久化。每次会话开始时，快速模式均处于关闭状态。",
      "type": "boolean"
    },
    "promptSuggestionEnabled": {
      "description": "当为 false 时，提示建议功能关闭；当缺失或为 true 时，提示建议功能开启。",
      "type": "boolean"
    },
    "emojiCompletionEnabled": {
      "description": "当为 false 时，:emoji: 缩写补全功能（即建议弹窗及 :name: 的内联替换）关闭；当缺失或为 true 时，该功能开启。",
      "type": "boolean"
    },
    "showClearContextOnPlanAccept": {
      "description": "当为 true 时，计划确认对话框会提供“清除上下文”选项。默认为 false。",
      "type": "boolean"
    },
    "askUserQuestionTimeout": {
      "description": "在 Claude 提问后，若无用户响应，系统将在多长时间后自动继续处理已选答案。默认为从不自动继续——只有显式设置为 60 秒/5 分钟/10 分钟时才会启用自动继续功能。",
      "type": "string",
      "enum": [
        "60秒",
        "5分钟",
        "10分钟",
        "永不"
      ]
    },
    "dialogExpiry": {
      "description": "转发至远程客户端的权限/用户对话框在等待答复期间可停留的最长时间，以及处于 HELD 状态的跨会话消息在等待批准前的最长时限，超过该时间两者将自动转为安全的无操作默认状态（取消或拒绝）。默认为 5 分钟，以匹配长期沿用的远程对话超时机制；“永不”则禁用该超时限制。仅本地的权限提示（无远程客户端）不受影响。环境变量 CLAUDE_CODE_USER_DIALOG_TIMEOUT_MS 若已设置，则会覆盖此配置。仅从可信来源读取（绝不在已提交的代码库设置文件中使用）。",
      "type": "string",
      "enum": [
        "60秒",
        "5分钟",
        "10分钟",
        "永不"
      ]
    },
    "agent": {
      "description": "用于主线程的代理名称（内置或自定义）。应用该代理的系统提示、工具限制及所选模型。",
      "type": "string"
    },
    "companyAnnouncements": {
      "description": "启动时显示的公司公告（若提供多条，则随机选取一条）",
      "type": "array",
      "items": {
        "type": "string"
      }
    },
    "pluginConfigs": {
      "description": "按插件配置，包括 MCP 服务器的用户配置，以插件 ID（plugin@marketplace 格式）为键。",
      "type": "object",
      "propertyNames": {
        "type": "string"
      },
      "additionalProperties": {
        "anyOf": [
          {
            "type": "object",
            "properties": {
              "mcpServers": {
                "description": "按服务器名称索引的 MCP 服务器用户配置值",
                "type": "object",
                "propertyNames": {
                  "type": "string"
                },
                "additionalProperties": {
                  "type": "object",
                  "propertyNames": {
                    "type": "string"
                  },
                  "additionalProperties": {
                    "anyOf": [
                      {
                        "type": "string"
                      },
                      {
                        "type": "number"
                      },
                      {
                        "type": "boolean"
                      },
                      {
                        "type": "array",
                        "items": {
                          "type": "string"
                        }
                      }
                    ]
                  }
                }
              },
              "options": {
                "description": "来自插件清单 userConfig 的非敏感选项值，按选项名称索引。敏感值则存入安全存储。",
                "type": "object",
                "propertyNames": {
                  "type": "string"
                },
                "additionalProperties": {
                  "anyOf": [
                    {
                      "type": "string"
                    },
                    {
                      "type": "number"
                    },
                    {
                      "type": "boolean"
                    },
                    {
                      "type": "array",
                      "items": {
                        "type": "string"
                      }
                    }
                  ]
                }
              }
            }
          },
          {
            "not": {}
          }
        ]
      }
    },
    "remote": {
      "description": "云端会话配置",
      "type": "object",
      "properties": {
        "defaultEnvironmentId": {
          "description": "云端会话使用的默认环境 ID",
          "type": "string"
        }
      }
    },
    "autoUpdatesChannel": {
      "description": "自动更新的发布通道（最新版、稳定版或候选版）",
      "type": "string",
      "enum": [
        "最新版",
        "稳定版",
        "候选版"
      ]
    },
    "minimumVersion": {
      "description": "保持使用的最低版本——防止切换到稳定版通道时降级",
      "type": "string"
    },
    "requiredMinimumVersion": {
      "description": "启动所需的最低 Claude Code 版本。如果运行的版本较旧，Claude Code 将在启动时退出，并提示用户更新。此设置仅在受管理（策略）配置中生效。",
      "type": "string"
    },
    "requiredMaximumVersion": {
      "description": "允许启动的最大 Claude Code 版本。如果运行的版本较新，Claude Code 将在启动时退出，并提示用户安装经批准的版本。此设置仅在受管理（策略）配置中生效。",
      "type": "string"
    },
    "plansDirectory": {
      "description": "计划文件的自定义目录，相对于项目根目录。未设置时，默认为 ~/.claude/plans/。",
      "type": "string"
    },
    "tui": {
      "description": "终端用户界面渲染器。\"fullscreen\" 使用无闪烁的备用屏幕渲染器，并支持虚拟化回滚（等同于 CLAUDE_CODE_NO_FLICKER=1）。\"default\" 使用经典的主屏幕渲染器。",
      "type": "string",
      "enum": [
        "default",
        "fullscreen"
      ]
    },
    "voice": {
      "description": "语音模式设置（长按说话 / 点击切换听写）",
      "type": "object",
      "properties": {
        "enabled": {
          "type": "boolean"
        },
        "mode": {
          "description": "'hold'（默认）：长按说话。'tap'：点击开始，点击停止并提交。",
          "type": "string",
          "enum": [
            "hold",
            "tap"
          ]
        },
        "autoSubmit": {
          "description": "在长按说话模式下，松开时自动提交提示语。",
          "type": "boolean"
        }
      }
    },
    "channelsEnabled": {
      "description": "组织级通道通知的启用选项（具有 claude/channel 能力的 MCP 服务器推送入站消息）。claude.ai Teams/Enterprise：默认关闭。控制台：默认开启，除非存在受管理设置。设为 true 可允许使用；用户随后可通过 --channels 选择服务器。",
      "type": "boolean"
    },
    "allowedChannelPlugins": {
      "description": "组织级通道插件白名单。设置后将替换默认的 Anthropic 白名单——由管理员决定哪些插件可以推送入站消息。未设置时则使用默认值。需同时设置 channelsEnabled: true。",
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "marketplace": {
            "type": "string"
          },
          "plugin": {
            "type": "string"
          }
        },
        "required": [
          "marketplace",
          "plugin"
        ]
      }
    },
    "prefersReducedMotion": {
      "description": "为提升无障碍性，减少或禁用动画效果（如加载指示器的闪烁、闪光特效等）。",
      "type": "boolean"
    },
    "timeFormat": {
      "description": "UI 中显示时间的格式：\"auto\"（默认，遵循系统区域设置）、\"12-hour\"、\"24-hour\"、\"24-hour-utc\"（如 \"18:05Z\"），或 strftime 格式字符串（如 \"%H:%M\"；包含 \"%\" 的任意值；其他值视为 \"auto\"）。格式字符串会全局替换时间显示；消息时间戳仅显示该格式，因此若需显示日期，请包含 %Y-%m-%d。/config 提供预设选项；此处设置自定义格式。",
      "anyOf": [
        {
          "type": "string",
          "enum": [
            "auto",
            "12-hour",
            "24-hour",
            "24-hour-utc"
          ]
        },
        {
          "type": "string"
        }
      ]
    },
    "timeZone": {
      "description": "UI 中显示时间所使用的 IANA 时区，例如 \"UTC\" 或 \"Europe/Dublin\"。默认为系统时区。未知时区名称将回退至系统时区。",
      "type": "string"
    },
    "autoMemoryEnabled": {
      "description": "为此项目启用自动记忆功能。设为 false 时，Claude 不会读取或写入自动记忆目录。",
      "type": "boolean"
    },
    "autoMemoryDirectory": {
      "description": "自动记忆存储的自定义目录路径。支持使用 ~/ 前缀展开到主目录。出于安全考虑，若已在 projectSettings（签入的 .claude/settings.json 文件）中设置，则忽略此处设置。未设置时，默认为 ~/.claude/projects/<sanitized-cwd>/memory/。",
      "type": "string"
    },
    "autoDreamEnabled": {
      "description": "启用后台记忆整合（自动梦境）。启用后将覆盖服务器端默认设置。",
      "type": "boolean"
    },
    "showThinkingSummaries": {
      "description": "请求 API 端提供思考摘要，并在对话及转录视图（Ctrl+O）中显示。显式设置可覆盖您所在环境的默认行为。",
      "type": "boolean"
    },
    "skipDangerousModePermissionPrompt": {
      "description": "用户是否已接受绕过权限模式的提示框。",
      "type": "boolean"
    },
    "disableAutoMode": {
      "description": "禁用自动模式。",
      "type": "string",
      "enum": [
        "disable"
      ]
    },
    "sshConfigs": {
      "description": "远程环境的 SSH 连接配置。通常由企业管理员在受管理设置中配置，以预先为团队成员设置 SSH 连接。",
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "id": {
            "description": "此 SSH 配置的唯一标识符，用于在不同设置来源间匹配配置。",
            "type": "string"
          },
          "name": {
            "description": "SSH 连接的显示名称。",
            "type": "string"
          },
          "sshHost": {
            "description": "SSH 主机，格式为 \"user@hostname\" 或 \"hostname\"，或 ~/.ssh/config 中的主机别名。",
            "type": "string"
          },
          "sshPort": {
            "description": "SSH 端口（默认：22）。",
            "type": "integer",
            "minimum": -9007199254740991,
            "maximum": 9007199254740991
          },
          "sshIdentityFile": {
            "description": "SSH 私钥文件的路径。",
            "type": "string"
          },
          "startDirectory": {
            "description": "远程主机上的默认工作目录。支持波浪号展开（如 ~/projects）。未指定时，默认为远程用户的主目录。可通过 `claude ssh <config> [dir]` 命令中的位置参数 [dir] 覆盖此设置。",
            "type": "string"
          }
        },
        "required": [
          "id",
          "name",
          "sshHost"
        ]
      }
    },
    "claudeMd": {
      "description": "以组织级内存形式注入的 CLAUDE.md 样式指令。仅在受管理/策略设置中生效。",
      "type": "string"
    },
    "claudeMdExcludes": {
      "description": "要排除加载的 CLAUDE.md 文件的 glob 模式或绝对路径。模式使用 picomatch 对绝对文件路径进行匹配。仅适用于用户、项目和本地内存类型（受管理/策略文件不可被排除）。示例：\"/home/user/monorepo/CLAUDE.md\"、\"**/code/CLAUDE.md\"、\"**/some-dir/.claude/rules/**\"。",
      "type": "array",
      "items": {
        "type": "string"
      }
    },
    "pluginTrustMessage": {
      "description": "在安装前显示的插件信任警告中追加的自定义提示信息。仅从策略设置（managed-settings.json / MDM）中读取。对企业管理员而言，可用于添加组织特定的背景说明（如：“我们内部市场中的所有插件均已审核并批准。”）。",
      "type": "string"
    },
    "theme": {
      "description": "UI 的颜色主题。",
      "anyOf": [
        {
          "type": "string",
          "enum": [
            "auto",
            "dark",
            "light",
            "light-daltonized",
            "dark-daltonized",
            "light-ansi",
            "dark-ansi"
          ]
        },
        {
          "type": "string",
          "pattern": "^custom:.*"
        }
      ]
    },
    "editorMode": {
      "description": "提示输入的键绑定模式",
      "type": "string",
      "enum": [
        "normal",
        "vim"
      ]
    },
    "keybindingFlavor": {
      "description": "已弃用：不再有任何作用。提示的单词编辑键始终遵循 Bash（readline）的约定。",
      "type": "string",
      "enum": [
        "classic",
        "readline"
      ]
    },
    "vimInsertModeRemaps": {
      "description": "Vim 插入模式下的键序列重映射，例如 {\"jj\": \"<Esc>\"}。每个键必须是连续输入的两个可打印字符；唯一支持的目标是 \"<Esc>\"（返回正常模式）。当 editorMode 为 \"vim\" 时生效。",
      "type": "object",
      "propertyNames": {
        "type": "string"
      },
      "additionalProperties": {}
    },
    "verbose": {
      "description": "显示完整的工具输出，而非截断的摘要",
      "type": "boolean"
    },
    "preferredNotifChannel": {
      "description": "首选的操作系统通知通道",
      "type": "string",
      "enum": [
        "auto",
        "iterm2",
        "terminal_bell",
        "iterm2_with_bell",
        "kitty",
        "ghostty",
        "notifications_disabled"
      ]
    },
    "autoCompactEnabled": {
      "description": "在上下文填满时自动压缩对话",
      "type": "boolean"
    },
    "precomputeCompactionEnabled": {
      "description": "在需要之前在后台预先计算压缩摘要。仅在自动压缩开启时生效。",
      "type": "boolean"
    },
    "switchModelsOnFlag": {
      "description": "当安全防护标记某条消息时，自动切换到其他模型以继续对话。关闭时，会暂停当前会话。",
      "type": "boolean"
    },
    "autoContinueAtUsageLimit": {
      "description": "当 claude.ai 的使用限制导致会话停止时，等待限制重置后自动继续任务。关闭时，限制对话框会将等待作为选项之一。",
      "type": "boolean"
    },
    "autoScrollEnabled": {
      "description": "自动将对话视图滚动至底部（仅限全屏模式）",
      "type": "boolean"
    },
    "wheelScrollAccelerationEnabled": {
      "description": "在快速滚动时逐步提升鼠标滚轮的滚动速度（仅限全屏模式）",
      "type": "boolean"
    },
    "fileCheckpointingEnabled": {
      "description": "在编辑前对文件进行快照，以便 /rewind 能够恢复它们",
      "type": "boolean"
    },
    "showTurnDuration": {
      "description": "在助手每轮回复后显示“处理耗时 N 分 N 秒”",
      "type": "boolean"
    },
    "showMessageTimestamps": {
      "description": "在每条消息上标注其到达时间",
      "type": "boolean"
    },
    "terminalProgressBarEnabled": {
      "description": "在长时间操作期间发出 OSC 9;4 进度序列",
      "type": "boolean"
    },
    "todoFeatureEnabled": {
      "description": "启用待办事项/任务跟踪面板",
      "type": "boolean"
    },
    "teammateMode": {
      "description": "生成的队友的执行方式（tmux、iterm2、进程内、自动）",
      "type": "string",
      "enum": [
        "auto",
        "tmux",
        "iterm2",
        "in-process"
      ]
    },
    "remoteControlAtStartup": {
      "description": "每次会话启动时自动开启远程控制桥接",
      "type": "boolean"
    },
    "isolatePeerMachines": {
      "description": "在通过远程控制让 SendMessage 访问另一台机器上的对等会话之前，需获得明确批准",
      "type": "boolean"
    },
    "daemonColdStart": {
      "description": "当没有后台服务运行时：'transient' 会在本次登录会话中临时启动一个；'ask' 则会提示是否将其持久安装",
      "type": "string",
      "enum": [
        "transient",
        "ask"
      ]
    },
    "crossSessionInbound": {
      "description": "来自其他会话的跨会话对等消息（SendMessage 来自您的其他会话）：'accept' 会直接送达，'hold' 会将其暂存供您审核而不让 Claude 处理，'refuse' 则会让本会话退出该功能。显式设置的值始终优先。未设置时（模式一致）：只有当发送会话的权限模式类别与您匹配时（bypass↔bypass 或 prompting↔prompting），消息才会自动送达；不匹配的发送者消息会被暂存等待您的批准；若发送者未声明任何类别，则该消息仅在本会话绕过权限提示时才会被暂存。",
      "类型": "字符串",
      "枚举": [
        "接受",
        "保留",
        "拒绝"
      ]
    },
    "autoUploadSessions": {
      "描述": "将本地会话镜像到 claude.ai，仅可查看（无远程控制）",
      "类型": "布尔值"
    },
    "inputNeededNotifEnabled": {
      "描述": "当有权限提示或问题等待处理时，推送通知至移动端",
      "类型": "布尔值"
    },
    "agentPushNotifEnabled": {
      "描述": "允许 Claude 推送主动式移动通知",
      "类型": "布尔值"
    },
    "skipAutoPermissionPrompt": {
      "描述": "用户是否已接受自动模式的同意对话框",
      "类型": "布尔值"
    },
    "useAutoModeDuringPlan": {
      "描述": "在计划模式下，当自动模式可用时是否采用自动模式的语义（默认：true）",
      "类型": "布尔值"
    },
    "autoMode": {
      "描述": "自动模式分类器提示的自定义配置",
      "类型": "对象",
      "属性": {
        "allow": {
          "描述": "自动模式分类器‘允许’部分的规则。在相应位置插入字面字符串‘$defaults’以继承内置规则。",
          "类型": "数组",
          "项的类型": "字符串"
        },
        "soft_deny": {
          "描述": "自动模式分类器‘软阻止’部分的规则——即用户意图可以解除的破坏性/不可逆操作。在相应位置插入字面字符串‘$defaults’以继承内置规则。",
          "类型": "数组",
          "项的类型": "字符串"
        },
        "hard_deny": {
          "描述": "自动模式分类器‘硬阻止’部分的规则——即用户意图无法解除的安全边界。在相应位置插入字面字符串‘$defaults’以继承内置规则。",
{
        "type": "array",
        "items": {
          "type": "string"
        }
      },
      "environment": {
        "description": "自动模式分类器环境部分的条目。包含字面字符串“$defaults”以继承该位置的内置条目。",
        "type": "array",
        "items": {
          "type": "string"
        }
      },
      "classifyAllShell": {
        "description": "当为真时，在自动模式激活期间，所有 Bash/PowerShell 的允许规则均被暂停，因此所有 Shell 命令都会通过分类器进行路由（安全性更高，但会增加分类器的调用次数）。默认值：假。",
        "type": "布尔值"
      }
    }
  },
  "disableDeepLinkRegistration": {
    "description": "阻止 claude-cli:// 协议处理器在操作系统中的注册。"
      "类型": "字符串",
      "枚举": [
        "禁用"
      ]
    },
    "语音启用": {
      "描述": "启用语音模式（按住说话的听写功能）",
      "类型": "布尔值"
    },
    "默认视图": {
      "描述": "默认的转录视图：聊天（仅显示 SendUserMessage 检查点）或完整转录",
      "类型": "字符串",
      "枚举": [
        “聊天”, 
        “转录”
      ]
    },
    “axScreenReader”: {
      “描述”: “渲染对屏幕阅读器友好的输出（纯文本，无装饰性边框或动画）。该设置会被 CLAUDE_AX_SCREEN_READER 环境变量和 --ax-screen-reader 命令行参数覆盖。”,
      “类型”: “布尔值”
    }
  },
  “额外属性”: {}
}
