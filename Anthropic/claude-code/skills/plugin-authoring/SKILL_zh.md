---
name: plugin-authoring
description: |-
  制作一个插件：在 Claude Code 中（终端或桌面 Code 选项卡内），创建一个实时窗格、面板、状态栏、提示或钩子，并将其编写为可在当前会话中热重载的函数钩子插件。在编写或调试钩子模块之前先加载它。
---
在哪里编写。将每个模块写入 `/Users/asgeirtj/.claude/dev-mods/6c004cfe-1d2e-49dc-904a-37f8db6d0a99` 下的独立子文件夹中：`/Users/asgeirtj/.claude/dev-mods/6c004cfe-1d2e-49dc-904a-37f8db6d0a99/<mod-name>/`，直接编写三个文件：

- `.claude-plugin/plugin.json`: `{ "name": "<mod-name>", "version": "0.1.0", "description": "<一句话>" }`
- `hooks/hooks.json`: `{ "modules": ["./register.tsx"] }`，路径相对该文件
- `hooks/register.tsx`（或 `.ts`）：钩子模块，`export const register: Register = (on, options) => { ... }`，类型从 `'claude-code'` 导入

如果模块在 `$.state` 中保存值，则还需第四个文件：`types/index.d.ts`，即类型契约，在模块名称下以 `interface PluginState` 声明每个值，并在 `plugin.json` 中指定 `"types": "./types/index.d.ts"`。模块从 `'../types'` 导入其值的类型，且 `claude plugin validate` 会确保模块声明的每个 `$.state` 键都符合该契约。

类型文件的位置。引擎会自动写入，无需手动运行任何命令。在模块加载之前：`/private/tmp/claude-501/bundled-skills/2.1.288/90955cd459120f476f04c5ab6d943d7b/plugin-authoring/types/claude-code.d.ts`，这是当前构建的 API 和内置工具，随技能加载时生成（此进程的专属文件夹：重启后，下次加载该技能时会生成并命名一个新的）。一旦引擎加载了模块（本会话启用了热重载，或指定了 `--plugin-dir` 文件夹），`<mod folder>/.claude-plugin/types/` 将为编辑器提供相同内容：`claude-code/index.d.ts` 是 API；`claude-code-tools/index.d.ts` 用于使 `e.tool === "Bash"` 类型更精确；`claude-code-mcp/index.d.ts` 是模块上次重新加载时连接的 MCP 工具；每个 `dependencies` 插件的契约；以及一个 `tsconfig.json`，供模块自身扩展，以便使用 `tsc -p <mod folder>` 对其进行类型检查。API 文件包含所有事件的输入与结果、$ 上的每个名词和方法及其文档注释与示例，以及每个元素的 props，总计约 14,000 行：可通过 grep 搜索所需名称（如 `'tool.call'`、`Pane: {`、`export type ToolCallResult`），并阅读匹配到的声明。

回合结束时会发生什么。通过 Skill 工具或其斜杠命令加载此技能时，引擎会开始监视 `/Users/asgeirtj/.claude/dev-mods/6c004cfe-1d2e-49dc-904a-37f8db6d0a99`。首次在此目录写入文件时，引擎会在当前回合期间向用户提出一个问题：“是否为本会话启用热重载？”先显示“这是如何工作的？”，然后是“为本会话启用”和“暂不启用”。该问题即为开关，仅由用户本人回答，不受权限模式、规则或钩子的影响。选择“为本会话启用”后，该文件夹将加入会话的插件文件夹列表，回合结束后模块将完整加载，后续每次编辑完成后，执行该编辑的回合结束时都会重新加载模块。您的回复将在下一个回合开始时以通知形式呈现，选项包括：已启用，附上加载结果；已拒绝（模块仍存在，下次会话启动时会加载；用户可再次请求该问题）；仍在等待（用户发出新提示则取消该问题，引擎将在该回合结束时再次询问）；或已禁用，并说明原因（无人可被询问，如在 `claude -p` 模式下；组织政策限制；工作区不可信）。若进程重启，已启用的文件夹会自动重新加载；否则，其监视将在下次加载此技能时启动，而文件夹内已存在的清单将在该回合结束时触发该问题。

重新加载即为模块的全新加载：`register` 会再次运行，`session.start` 也会再次触发。`$.state`（会话级）和 `$.store`（跨会话）中的值属于宿主，保持不变；模块自身的变量则会重新初始化。

## 一段话概括一个模块`on(event, matcher?, hook)` 用于添加一个钩子，每个钩子的签名都是 `($, e, next)`：`$` 是引擎接口，每次调用时先指定名词再调用方法；`e` 是事件的输入，是一个普通的冻结值；`next(e)` 会依次执行其后的插件以及引擎自身的逻辑，并返回事件的结果。如果钩子在不调用 `next` 的情况下返回，则仅由该钩子自行处理；若调用 `next({ ...e, x })`，则会改写后续所有组件看到的内容。该模块运行在独立的环境中，没有 DOM 也没有 Node.js 环境：`$` 可以访问模块外部的所有资源。JSX 会针对全局的 `h` 进行编译，而元素则来自绘制表面自己的组件表，例如 `const { Box, Text, Button } = $.ui.resolve(e)`，其中 `e.surface` 可以是 `terminal`、`desktop`、`vscode` 或 `mobile`。

## 从需求到形态

每个示例都构成一个完整的钩子模块；配合上述两个 JSON 文件，以及它所定义的契约（如有），即可作为一个可在当前构建中加载、验证并进行类型检查的模块。

| 人请求的内容 | 是什么 | 显示方式 |
| --- | --- | --- |
| 一个窗格、面板、侧边栏、实时视图 | `$.ui.open({ id, title })`，由 `{ component: 'Pane', requestId: id }` 上的 `ui.render` 钩子绘制；由用户操作（输入的命令、点击的按钮）打开时，宽度可自适应；未请求时打开（如从 `session.start` 或定时器触发），则在终端宽度达到 144 列时显示，并在此之下等待 | `/private/tmp/claude-501/bundled-skills/2.1.288/90955cd459120f476f04c5ab6d943d7b/plugin-authoring/examples/pane.tsx`，其契约文件为 `/private/tmp/claude-501/bundled-skills/2.1.288/90955cd459120f476f04c5ab6d943d7b/plugin-authoring/examples/pane-state.d.ts` |
| 提示词上方的一条横带或一行内容 | 对 `{ component: 'AbovePrompt' }` 的 `ui.render` 钩子返回一棵树，或无内容时调用 `next(e)` | `/private/tmp/claude-501/bundled-skills/2.1.288/90955cd459120f476f04c5ab6d943d7b/plugin-authoring/examples/band.tsx`，其契约文件为 `/private/tmp/claude-501/bundled-skills/2.1.288/90955cd459120f476f04c5ab6d943d7b/plugin-authoring/examples/band-state.d.ts` |
| 状态栏的一项内容 | 任何钩子中调用 `$.ui.status(text)`；传入 `undefined` 可清空状态栏 | `/private/tmp/claude-501/bundled-skills/2.1.288/90955cd459120f476f04c5ab6d943d7b/plugin-authoring/examples/tool-call.ts` |
| 一个提示气泡 | 任何钩子中调用 `$.ui.toast(text)` | `/private/tmp/claude-501/bundled-skills/2.1.288/90955cd459120f476f04c5ab6d943d7b/plugin-authoring/examples/band.tsx` |
| 拦截、改写或响应工具调用 | `on('tool.call', { tool }, hook)`：返回 `{ deny }`，调用 `next({ ...e, ... })`，或 `await next(e)` 并根据结果采取行动 | `/private/tmp/claude-501/bundled-skills/2.1.288/90955cd459120f476f04c5ab6d943d7b/plugin-authoring/examples/tool-call.ts` |
| 修改或响应提示词 | `on('prompt.submit', hook)`：调用 `next({ ...e, text })` | `/private/tmp/claude-501/bundled-skills/2.1.288/90955cd459120f476f04c5ab6d943d7b/plugin-authoring/examples/band.tsx` |
| 更改系统提示词 | `on('prompt.compose', hook)`：对 `next(e)` 的 `{ sections }` 进行补充（`scope: 'session'`）、替换或删除 | 其文档见类型定义 |
| 播放声音 | 任何钩子中调用 `$.audio.play({ asset })`，音效资源为插件内的文件 | 其文档见类型定义 |
| 一个斜杠命令 | 在 `session.start` 中调用 `$.command.register({ name, description })`，由 `command.run` 钩子返回 `{ text }` 来响应 | `/private/tmp/claude-501/bundled-skills/2.1.288/90955cd459120f476f04c5ab6d943d7b/plugin-authoring/examples/pane.tsx` |
| 绘制时读取的值 | `atom(ref, initial)`，绘制时通过 `read($, atom)` 读取，由处理器或其他事件调用 `update($, atom, fn)` 更新；写入后会重新渲染所有读取者；每个值均在契约中声明 | `/private/tmp/claude-501/bundled-skills/2.1.288/90955cd459120f476f04c5ab6d943d7b/plugin-authoring/examples/pane-state.d.ts` |
| 定时器、模型调用的工具、子代理类型、模型调用、文件、进程相关操作 | `$.clock`、`$.tool`、`$.agent`、`$.model`、`$.fs`、`$.process` | `/private/tmp/claude-501/bundled-skills/2.1.288/90955cd459120f476f04c5ab6d943d7b/plugin-authoring/reference.md` |

## 检查并查看引擎拒绝的内容

`claude plugin validate <mod folder>` 会按照引擎的方式读取清单和模块源码，并报告模块中的钩子、调用以及所有会被引擎拒绝的部分。对插件进行类型检查：加载完成后运行 `tsc -p <mod folder>`；若尚未编写任何代码（首次编写或使用 `claude -p` 时），则使用类型文件头部的 `tsconfig.json`（置于模块文件夹外），并将该文件及模块的 `hooks` 添加到 `include` 中。至少编写一个 `*.test.ts` 文件来测试预期行为，并运行 `claude plugin test <mod folder>`。随后向用户提供插件的完整路径及其加载方式：启用时无需额外操作；若未启用，则需在终端中通过 `claude --plugin-dir` 指定该路径方可加载；否则将不会加载任何内容，直到设置更改为止。

当会话热加载一个插件文件夹时，对话记录会显示一条简略的日志，标明插件名称、事件以及钩子失败或模块加载失败的原因，并指出某个钩子的树结构未通过验证、引擎因此绘制了自动生成的树：`<plugin>: ui.render (<Component>) 被拒绝： <reason>; 引擎绘制了自己的树`，其中 `<plugin>` 是那个本应绘制该树的插件，而其他情况则归为 `hooks`。在其他会话中，这些日志行只会单独出现在调试日志中，格式为 `<plugin>: <line>`。调试日志（`claude --debug`）会为引擎拒绝的每一次发生及其结果都记录一行；在每次会话中，这样的树都会在日志中以 `ui.render (<Component>): 某个钩子返回的树未通过验证` 开头，后接具体原因。

`/private/tmp/claude-501/bundled-skills/2.1.288/90955cd459120f476f04c5ab6d943d7b/plugin-authoring/reference.md` 是详细文档：包含了完整的事件列表与流式事件、`ui.render` 的深入说明、`$.state` 合约、`--plugin-dir` 和 `CLAUDE_CODE_PLUGIN_DIRS`、`userConfig` 选项，以及测试内容。