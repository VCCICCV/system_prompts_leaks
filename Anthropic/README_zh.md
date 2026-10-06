# Anthropic — 哪个文件对应哪个产品？

**此文件夹中的纯 `claude-<model>.md` 文件是 claude.ai 的系统提示词**——即在 Claude 网页版/移动端应用（claude.ai）中提供给相应模型的提示词。它们不是 API 提示词（Claude API 不会注入系统提示词），也不是 Claude Code 的提示词。

| 文件模式 | 产品 |
|---|---|
| `claude-fable-5.md`、`claude-opus-4.8.md`、`claude-sonnet-5.md` 等 | 该模型对应的 **claude.ai** 应用系统提示词 |
| `claude-*-no-tools.md` | 禁用工具的 claude.ai |
| `claude-code/` | Claude Code（CLI/代理框架） |
| `claude-design.md` | Claude Design |
| `claude-cowork/` | Claude Cowork |
| `claude-for-excel.md`、`claude-for-word.md`、`claude-in-powerpoint.md` | Microsoft 365 中的 Claude |
| `claude-in-chrome.md` | Chrome 扩展中的 Claude |
| `claude-mobile-ios.md` | claude.ai iOS 应用 |
| `anthropic_reminders.md`、`sonnet-4.6-reminders.md`、`research_instructions.md`、`visualize.md` | claude.ai 注入的片段（提醒、研究指令、生成物） |
| `official/` | Anthropic 官方发布的提示词（发布说明版本——比上述实际提供的提示词更短） |
