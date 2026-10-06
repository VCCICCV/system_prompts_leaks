# OpenAI — 哪个文件对应哪个产品？

**`gpt-<版本>-thinking/instant.md` 文件是 ChatGPT 应用的系统提示词**——即 chatgpt.com 为该模型提供的内容。以 `-api.md` 结尾的文件是 OpenAI 在原始 API 调用时注入的隐藏系统消息（未公开文档）。`Codex/` 目录则对应 Codex 的命令行界面/代码助手。

| 文件模式 | 产品 |
|---|---|
| `gpt-5.6-sol-extra-high.md`、`gpt-5.5-thinking.md`、`gpt-5.5-instant.md` 等 | 该模型对应的 **ChatGPT** 应用系统提示词 |
| `chatgpt-4.5.md`、`chatgpt-atlas.md`、`chatgpt-gpt-5-agent-mode.md` | ChatGPT 应用（较旧的记录 / Atlas 浏览器 / 助手模式）|
| `gpt-*-api.md` | 在 **API** 调用时注入的隐藏系统消息 |
| `Codex/` | Codex 命令行界面 / 编码助手 |
| `gpt-4o.md` | ChatGPT 4o（包含弃用自处理协议，第226行及以后）|
| `gpt-5-*-personality.md`、`gpt-5.1-*.md` | ChatGPT 的不同人格变体 |
| `tool-*.md` | ChatGPT 针对特定工具的片段 |
| `Old/` | 已被取代的版本 · `deprecated/` —— 已废弃的功能 |
