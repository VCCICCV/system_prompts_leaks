---
name: keybindings-help
description: |-
  当用户需要自定义键盘快捷键、重新绑定按键、添加组合键绑定，或修改 ~/.claude/keybindings.json 文件时使用。示例：“重新绑定 Ctrl+S”、“添加一个组合键快捷方式”、“更改提交键”、“自定义键位绑定”。
user-invocable: false
---
# 快捷键技能

通过创建或修改 `~/.claude/keybindings.json` 文件，您可以自定义键盘快捷键。

## 重要提示：写入前请先阅读

**务必先读取 `~/.claude/keybindings.json` 文件**（该文件可能尚未存在）。请将更改与现有绑定合并，切勿直接替换整个文件。

- 对于已有文件的修改，请使用“编辑”工具。
- 只有在文件尚不存在时，才使用“写入”工具。

## 文件格式

```json
{
  "$schema": "https://www.schemastore.org/claude-code-keybindings.json",
  "$docs": "https://code.claude.com/docs/en/keybindings",
  "bindings": [
    {
      "context": "Chat",
      "bindings": {
        "ctrl+e": "chat:externalEditor"
      }
    }
  ]
}
```

请始终包含 `$schema` 和 `$docs` 字段。

## 键位语法

**修饰键**（用 `+` 连接）：
- `ctrl`（别名：`control`）
- `alt`（别名：`opt`、`option`）——请注意，在终端中 `alt` 和 `meta` 是等效的
- `shift`
- `meta`（别名：`cmd`、`command`）

**特殊键**：`escape`/`esc`、`enter`/`return`、`tab`、`space`、`backspace`、`delete`、`up`、`down`、`left`、`right`

**组合键**：以空格分隔的多个按键序列，例如 `ctrl+k ctrl+s`（两次按键之间有1秒的超时）。

**示例**：`ctrl+shift+p`、`alt+enter`、`ctrl+k ctrl+n`

## 取消默认快捷键绑定

将某个键设置为 `null` 即可移除其默认绑定：

```json
{
  "context": "Chat",
  "bindings": {
    "ctrl+s": null
  }
}
```

## 用户绑定与默认绑定的交互方式

- 用户绑定是**累加式**的——它们会追加到默认绑定之后。
- 若要将某个绑定移动到其他键上：需同时取消旧键的绑定（设置为 `null`）并添加新绑定。
- 只有当用户希望更改某个上下文中的绑定时，才需要在用户配置文件中添加该上下文。

## 常见模式

### 重新绑定某个键
若要将外部编辑器的快捷键从 `ctrl+g` 改为 `ctrl+e`：

```json
{
  "context": "Chat",
  "bindings": {
    "ctrl+g": null,
    "ctrl+e": "chat:externalEditor"
  }
}
```

### 添加组合键绑定
```json
{
  "context": "Global",
  "bindings": {
    "ctrl+k ctrl+t": "app:toggleTodos"
  }
}
```

## 行为规则

1. 仅包含用户希望更改的上下文（最小化覆盖）。
2. 确保所有动作和上下文均来自以下已知列表。
3. 如果用户选择的键与保留快捷键或常用工具（如 tmux 的 `ctrl+b` 和 screen 的 `ctrl+a`）冲突，系统会主动发出警告。
4. 为现有动作添加新绑定时，新绑定为累加式（除非显式取消绑定，否则原有默认绑定仍有效）。
5. 若要完全替换某个默认绑定，必须先取消旧键的绑定，再添加新键的绑定。

## 验证机制

Claude Code 在加载 `~/.claude/keybindings.json` 时会进行验证；任何警告都会记录到调试日志中。编辑文件后，请对照以下规则再次检查，并修正所有不符合项。

### 常见问题及解决方法

| 问题 | 原因 | 解决方法 |
| --- | --- | --- |
| `keybindings.json 必须包含 "bindings" 数组` | 缺少外层对象 | 将绑定包裹在 `{ "bindings": [...] }` 中 |
| `"bindings" 必须为数组` | `bindings` 不是数组 | 将 `"bindings"` 设置为数组：`[{ context: ..., bindings: ... }]` |
| `未知上下文 "X"` | 拼写错误或无效上下文名称 | 使用“可用上下文”表中的准确上下文名称 |
| `Y 个绑定中存在重复键 "X"` | 同一上下文中同一键被定义两次 | 删除重复项；JSON 只保留最后一个值 |
| `"X" 可能无法正常工作：...` | 键与终端或操作系统预留快捷键冲突 | 更换其他键（参见“保留快捷键”部分） |
| `键 "X" 对应的动作无效` | 动作值不是字符串或 null | 动作必须是字符串，如 `"app:help"`，或设置为 `null` 以取消绑定 |

### 示例验证警告（调试日志）

```
[keybindings] 发现 2 处验证问题
[keybindings] [error] 未知上下文 "chat" — 有效上下文：Global, Chat, Autocomplete, ...
[keybindings] [warning] "ctrl+c" 可能无法正常工作：终端中断信号 (SIGINT)
```

**错误** 会阻止绑定生效，必须予以修复。**警告** 表示可能存在冲突，但绑定仍可能正常工作。

## 保留快捷键

### 不可重新绑定（错误）
- `ctrl+c` — 不可重新绑定 - 用于中断/退出（硬编码）
- `ctrl+d` — 不可重新绑定 - 用于退出（硬编码）
- `ctrl+m` — 不可重新绑定 - 在终端中与 Enter 键功能相同（均发送回车符 CR）
- `ctrl+[` — 不可重新绑定 - 在终端中与 Escape 键功能相同
- `ctrl+i` — 不可重新绑定 - 在终端中与 Tab 键功能相同
- `ctrl+h` — 不可重新绑定 - 在终端中与 Backspace 键功能相同
- `capslock` — 大写锁定键不会传递给终端应用程序

### 终端保留（错误/警告）
- `ctrl+z` — Unix 进程挂起（SIGTSTP）（可能存在冲突）
- `ctrl+\` — 终端退出信号（SIGQUIT）（无法正常工作）

### macOS 保留（错误）
- `cmd+c` — macOS 系统复制
- `cmd+v` — macOS 系统粘贴
- `cmd+x` — macOS 系统剪切
- `cmd+q` — macOS 退出应用程序
- `cmd+w` — macOS 关闭窗口/标签页
- `cmd+tab` — macOS 应用程序切换器
- `cmd+space` — macOS Spotlight 搜索

## 可用上下文

| 上下文 | 描述 |
| --- | --- |
| `Global` | 无论焦点在何处，全局生效 |
| `Chat` | 当聊天输入框获得焦点时 |
| `Autocomplete` | 当自动补全菜单可见时 |
| `Confirmation` | 当显示确认/权限对话框时 |
| `Help` | 当帮助覆盖层打开时 |
| `Transcript` | 查看对话记录时 |
| `HistorySearch` | 搜索命令历史（Ctrl+R）时 |
| `Task` | 当任务/代理在前台运行时 |
| `ThemePicker` | 当主题选择器打开时 |
| `Settings` | 当设置菜单打开时 |
| `Tabs` | 当标签页导航处于活动状态时 |
| `Attachments` | 在选择对话框中浏览图片附件时 |
| `Footer` | 当页脚指示器获得焦点时 |
| `AbovePrompt` | 当提示上方的插件面板或其中的按钮获得键盘焦点时 |
| `AbovePromptInput` | 当提示上方的插件输入框获得键盘焦点时 |
| `AbovePromptSelect` | 当提示上方的插件选择框获得键盘焦点时 |
| `Pane` | 当插件面板获得键盘焦点时 |
| `PaneField` | 当插件面板中的输入框或选择框获得键盘焦点时 |
| `MessageSelector` | 当消息选择器（回退）打开时 |
| `DiffDialog` | 当差异对比对话框打开时 |
| `DiffPanel` | 当差异对比侧边栏面板打开时 |
| `ModelPicker` | 当模型选择器打开时 |
| `EffortSlider` | 当努力程度滑块打开时 |
| `Select` | 当选择/列表组件获得焦点时 |
| `Plugin` | 当插件对话框打开时 |
| `Scroll` | 当可滚动视图获得焦点时（全屏布局） |
| `Agents` | 当代理视图（`claude agents`）打开时 |

## 可用操作

| 操作 | 默认快捷键 | 上下文 |
| --- | --- | --- |
| `app:interrupt` | `ctrl+c` | 全局 |
| `app:exit` | `ctrl+d` | 全局 |
| `app:toggleTodos` | `ctrl+t` | 全局 |
| `app:toggleTranscript` | `ctrl+o` | 全局 |
| `app:toggleBrief` | `ctrl+shift+b` | 全局 |
| `app:toggleReplTab` | 无 | 全局 |
| `app:toggleDiffNoiseFilter` | 无 | 全局 |
| `app:diffFileListUp` | `meta+up` | 全局 |
| `app:diffFileListDown` | `meta+down` | 全局 |
| `app:toggleDiffPreSession` | 无 | 全局 |
| `app:cycleDiffBase` | `ctrl+x b` | 差异面板 |
| `app:toggleTerminal` | 无 | 全局 |
| `app:redraw` | 无 | 全局 |
| `app:openArtifact` | `ctrl+]` | 全局 |
| `history:search` | `ctrl+r` | 全局 |
| `history:previous` | `up` | 聊天 |
| `history:next` | `down` | 聊天 |
| `chat:cancel` | `escape` | 聊天 |
| `chat:killAgents` | `ctrl+x ctrl+k` | 聊天 |
| `chat:cycleMode` | `shift+tab` | 聊天 |
| `chat:modelPicker` | `meta+p` | 聊天 |
| `chat:fastMode` | `meta+o` | 聊天 |
| `chat:thinkingToggle` | `meta+t` | 聊天 |
| `chat:workflowKeywordToggle` | `meta+w` | 聊天 |
| `chat:submit` | `enter` | 聊天 |
| `chat:queueSubmit` | `ctrl+x enter` | 聊天 |
| `chat:newline` | `ctrl+j` | 聊天 |
| `chat:undo` | `ctrl+_`, `ctrl+-`, `ctrl+shift+-`, `ctrl+shift+_` | 聊天 |
| `chat:externalEditor` | `ctrl+x ctrl+e`, `ctrl+g` | 聊天 |
| `chat:stash` | `ctrl+s` | 聊天 |
| `chat:imagePaste` | `ctrl+v` | 聊天 |
| `chat:clearInput` | `ctrl+l` | 聊天 |
| `chat:clearScreen` | `cmd+k` | 聊天 |
| `autocomplete:accept` | `tab` | 自动补全 |
| `autocomplete:dismiss` | `escape` | 自动补全 |
| `autocomplete:previous` | `up` | 自动补全 |
| `autocomplete:next` | `down` | 自动补全 |
| `confirm:yes` | `y`, `enter` | 确认 |
| `confirm:no` | `escape`, `n`, `escape` | 设置 |
| `confirm:previous` | `up` | 确认 |
| `confirm:next` | `down` | 确认 |
| `confirm:nextField` | `tab` | 确认 |
| `confirm:previousField` | 无 | 确认 |
| `confirm:cycleMode` | `shift+tab` | 确认 |
| `confirm:toggle` | `space` | 确认 |
| `tabs:next` | `tab`, `right` | 标签页 |
| `tabs:previous` | `shift+tab`, `left` | 标签页 |
| `transcript:toggleShowAll` | `ctrl+e` | 文本记录 |
| `transcript:exit` | `ctrl+c`, `escape`, `q` | 文本记录 |
| `historySearch:next` | `ctrl+r` | 历史搜索 |
| `historySearch:accept` | `escape`, `tab` | 历史搜索 |
| `historySearch:cancel` | `ctrl+c` | 历史搜索 |
| `historySearch:execute` | `enter` | 历史搜索 |
| `historySearch:cycleScope` | `ctrl+s` | 历史搜索 |
| `task:background` | `ctrl+x ctrl+b`, `ctrl+b` | 任务 |
| `theme:toggleSyntaxHighlighting` | `ctrl+t` | 主题选择器 |
| `theme:editCustom` | `ctrl+e` | 主题选择器 |
| `help:dismiss` | `escape` | 帮助 |
| `attachments:next` | `right` | 附件 |
| `attachments:previous` | `left` | 附件 |
| `attachments:remove` | `backspace`, `delete` | 附件 |
| `attachments:exit` | `down`, `escape` | 附件 |
| `footer:up` | `up`, `ctrl+p` | 页脚 |
| `footer:down` | `down`, `ctrl+n` | 页脚 |
| `footer:next` | `right` | 页脚 |
| `footer:previous` | `left` | 页脚 |
| `footer:openSelected` | `enter` | 页脚 |
| `footer:clearSelection` | `escape` | 页脚 |
| `footer:close` | `x` | 页脚 |
| `footer:dismiss` | `backspace`, `delete` | 页脚 |
| `abovePrompt:toggle` | `ctrl+x ctrl+a` | 聊天 |
| `abovePrompt:focus` | `ctrl+x tab` | 聊天 |
| `abovePrompt:next` | `tab`, `right`, `tab`, `down`, `tab`, `tab` | 提示上方 |
| `abovePrompt:previous` | `shift+tab`, `left`, `shift+tab`, `up`, `shift+tab`, `shift+tab` | 提示上方 |
| `abovePrompt:press` | `enter`, `space`, `enter`, `enter`, `enter` | 提示上方 |
| `abovePrompt:leave` | `escape`, `escape`, `escape`, `escape` | 提示上方 |
| `abovePrompt:highlightNext` | `down` | 提示上方选择 |
| `abovePrompt:highlightPrevious` | `up` | 提示上方选择 |
| `pane:scrollUp` | `up`, `up` | 提示上方 |
| `pane:scrollDown` | `down`, `down` | 提示上方 |
| `pane:pageUp` | `pageup`, `pageup` | 提示上方 |
| `pane:pageDown` | `pagedown`, `pagedown` | 提示上方 |
| `pane:top` | `home`, `home` | 提示上方 |
| `pane:bottom` | `end`, `end` | 提示上方 |
| `pane:grow` | `ctrl+x left`, `ctrl+x up` | 面板 |
| `pane:shrink` | `ctrl+x right`, `ctrl+x down` | 面板 |
| `pane:close` | `ctrl+x x`, `ctrl+x x` | 面板 |
| `pane:next` | 无 | 面板 |
| `pane:previous` | 无 | 面板 |
| `messageSelector:up` | `up`, `k`, `ctrl+p` | 消息选择器 |
| `messageSelector:down` | `down`, `j`, `ctrl+n` | 消息选择器 |
| `messageSelector:top` | `ctrl+up`, `shift+up`, `meta+up`, `shift+k` | 消息选择器 |
| `messageSelector:bottom` | `ctrl+down`, `shift+down`, `meta+down`, `shift+j` | 消息选择器 |
| `messageSelector:select` | `enter` | 消息选择器 |
| `diff:dismiss` | `escape` | 差异对话框 |
| `diff:previousSource` | `left` | 差异对话框 |
| `diff:nextSource` | `right` | 差异对话框 |
| `diff:back` | 无 | 差异对话框 |
| `diff:viewDetails` | `enter` | 差异对话框 |
| `diff:previousFile` | `up`, `k` | 差异对话框 |
| `diff:nextFile` | `down`, `j` | 差异对话框 |
| `modelPicker:decreaseEffort` | `left` | 模型选择器 |
| `modelPicker:increaseEffort` | `right` | 模型选择器 |
| `modelPicker:thisSessionOnly` | `s` | 模型选择器 |
| `effortSlider:thisSessionOnly` | `s` | 效力滑块 |
| `select:next` | `down`, `j`, `ctrl+n`, `down`, `j`, `ctrl+n` | 设置 |
| `select:previous` | `up`, `k`, `ctrl+p`, `up`, `k`, `ctrl+p` | 设置 |
| `select:pageUp` | `pageup` | 选择 |
| `select:pageDown` | `pagedown` | 选择 |
| `select:first` | `home` | 选择 |
| `select:last` | `end` | 选择 |
| `select:accept` | `space`, `enter`, `enter` | 设置 |
| `select:cancel` | `escape` | 选择 |
| `plugin:toggle` | `space` | 插件 |
| `plugin:install` | `i` | 插件 |
| `plugin:favorite` | `f` | 插件 |
| `permission:toggleDebug` | 无 | 确认 |
| `settings:search` | `/` | 设置 |
| `settings:retry` | `r` | 设置 |
| `settings:periodDay` | `d` | 设置 |
| `settings:periodWeek` | `w` | 设置 |
| `settings:sortByTokens` | `t` | 设置 |
| `voice:pushToTalk` | `space` | 聊天 |
| `scroll:previousPrompt` | `ctrl+up`, `ctrl+up` | 文本记录 |
| `scroll:nextPrompt` | `ctrl+down`, `ctrl+down` | 文本记录 |
| `scroll:pageUp` | `pageup`, `pageup` | 滚动 |
| `scroll:pageDown` | `pagedown`, `pagedown` | 滚动 |
| `scroll:lineUp` | `ctrl+p`, `k`, `up`, `滚轮上` | 文本记录 |
| `scroll:lineDown` | `ctrl+n`, `j`, `down`, `滚轮下` | 文本记录 |
| `scroll:top` | `g`, `home`, `ctrl+home`, `g`, `home` | 文本记录 |
| `scroll:bottom` | `shift+g`, `end`, `ctrl+end`, `shift+g`, `end` | 文本记录 |
| `scroll:halfPageUp` | `ctrl+u`, `ctrl+u` | 设置 |
| `scroll:halfPageDown` | `ctrl+d`, `ctrl+d` | 设置 |
| `scroll:fullPageUp` | `ctrl+b`, `b`, `shift+空格`, `b` | 文本记录 |
| `scroll:fullPageDown` | `ctrl+f`, `空格`, `空格` | 文本记录 |
| `selection:copy` | `ctrl+shift+c`, `cmd+c` | 滚动 |
| `selection:clear` | 无 | 未知 |
| `selection:extendLeft` | `shift+left` | 滚动 |
| `selection:extendRight` | `shift+right` | 滚动 |
| `selection:extendUp` | `shift+up` | 滚动 |
| `selection:extendDown` | `shift+down` | 滚动 |
| `selection:extendLineStart` | `shift+home` | 滚动 |
| `selection:extendLineEnd` | `shift+end` | 滚动 |
| `agents:switchView` | `ctrl+s` | 代理 |
| `agents:togglePin` | `ctrl+t` | 代理