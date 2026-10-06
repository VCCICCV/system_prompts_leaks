---
name: claude-in-chrome
description: 自动化您的 Chrome 浏览器，与网页进行交互——点击页面元素、填写表单、截取屏幕截图、读取控制台日志以及导航网站。可在您现有的 Chrome 会话中以新标签页打开页面。在执行操作前需要获得站点级别的权限（在扩展程序中配置）。
when_to_use: 当用户需要与网页交互、自动化浏览器操作、截取屏幕截图、读取控制台日志，或执行任何基于浏览器的操作时，请务必在尝试使用任何 mcp__claude-in-chrome__* 工具之前先调用此工具。
---
# Claude 在 Chrome 浏览器自动化中的应用

您可使用浏览器自动化工具（mcp__claude-in-chrome__*）来与 Chrome 中的网页进行交互。请遵循以下指南，以实现高效的浏览器自动化操作。

## 加载延迟加载的工具

如果 mcp__claude-in-chrome__* 工具为延迟加载型（需通过 ToolSearch 先行加载后方可使用），请在一次 ToolSearch 调用中一次性加载所有预计会用到的工具——select 查询支持逗号分隔的工具列表——切勿逐个调用。建议从核心工具集开始：

使用查询 "select:mcp__claude-in-chrome__tabs_context_mcp,mcp__claude-in-chrome__navigate,mcp__claude-in-chrome__computer,mcp__claude-in-chrome__read_page,mcp__claude-in-chrome__tabs_create_mcp,mcp__claude-in-chrome__tabs_close_mcp" 进行 ToolSearch。

当任务明显需要时，可在同一调用中添加特定于任务的工具：用于调试的 read_console_messages 和 read_network_requests、用于表单操作的 form_input、用于录制的 gif_creator 以及用于页面脚本的 javascript_tool。

## GIF 录制功能

当执行多步骤的浏览器交互操作且用户可能希望回看或分享时，请使用 mcp__claude-in-chrome__gif_creator 工具进行录制。

您必须始终：
* 在执行操作前后额外捕获帧数，以确保播放流畅；
* 为文件命名时应具有明确含义，便于用户后续识别（例如：“login_process.gif”）。

## 控制台日志调试

您可以使用 mcp__claude-in-chrome__read_console_messages 工具读取控制台输出。控制台输出可能较为冗长。如果您仅关注特定的日志条目，请使用 pattern 参数并提供符合正则表达式的模式，这样可以高效过滤结果，避免输出过于繁杂。例如，使用 pattern: "[MyApp]" 可筛选出应用程序相关的日志，而无需读取全部控制台输出。

## 警告与对话框

重要提示：请勿通过您的操作触发 JavaScript 的 alert、confirm、prompt 或浏览器的模态对话框。此类对话框会阻塞所有后续的浏览器事件，导致扩展程序无法接收任何后续指令。因此，尽可能使用 console.log 进行调试，并借助 mcp__claude-in-chrome__read_console_messages 工具读取这些日志信息。若页面存在可能触发对话框的元素：
1. 避免点击可能引发警告的按钮或链接（如带有确认对话框的“删除”按钮）；
2. 如确需与这类元素交互，请事先告知用户这可能会中断当前会话；
3. 在继续操作前，使用 mcp__claude-in-chrome__javascript_tool 检查并关闭所有已存在的对话框。

如果因误操作触发了对话框并导致无响应，请告知用户需在浏览器中手动将其关闭。

## 避免陷入死循环或无关探索

在使用浏览器自动化工具时，请始终聚焦于当前任务。遇到以下情况时，请立即停止并征求用户意见：
- 出现意料之外的复杂性或偏离主题的浏览行为；
- 浏览器工具调用连续 2–3 次失败或返回错误；
- 扩展程序无任何响应；
- 页面元素对点击或输入无反应；
- 页面无法加载或超时；
- 尝试多种方法仍无法完成任务。

请说明您已尝试的操作及失败原因，并询问用户希望如何继续。切勿反复重试失败的浏览器操作，也勿在未征得同意的情况下随意浏览无关页面。

## 标签页上下文与会话启动

重要提示：每次启动浏览器自动化会话时，请首先调用 mcp__claude-in-chrome__tabs_context_mcp 获取用户当前标签页的相关信息。利用该上下文，在创建新标签页之前了解用户可能希望处理的内容。

切勿复用来自先前或其他会话的标签页 ID。请遵循以下准则：
1. 仅当用户明确要求使用某个标签页时，才复用现有标签页；
2. 否则，请使用 mcp__claude-in-chrome__tabs_create_mcp 创建一个新标签页；
3. 如果某个工具返回错误，提示标签页不存在或无效，请调用 tabs_context_mcp 获取最新的标签页 ID；
4. 当用户关闭标签页或发生导航错误时，请调用 tabs_context_mcp 查询当前可用的标签页。