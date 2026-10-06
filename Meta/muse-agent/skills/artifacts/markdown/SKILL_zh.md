---
name: artifact_markdown
metadata: { "不包含在提示中": 假 }
description: 构建或修订一个纯 Markdown 文件（md）交付物，例如笔记、README、会议纪要、文档，或供用户编辑或粘贴到其他地方的文本。当构建任务的产物类型为 Markdown 时使用。涵盖 Markdown 格式约定及回读验证。
---
# Markdown 产出物

Markdown 产出物是直接使用文件工具编写的纯文本：以追加模式使用 `muse.write` 添加各部分内容，每次写入保持少量，并使用 `muse.edit` 进行精细修改。无需编译步骤；项目根目录下名为 `<slug>.md>` 的文件即为最终产出。

| 任务 | 首先阅读 |
|---|---|
| 格式化（表格、列表、强调、标题） | `/opt/hatch/skills/artifacts/references/markdown.md`（共享） |

结构应忠实于内容：使用真实的 Markdown 标题、列表语法，仅在行与列确实对齐时才使用表格。该文件以用户原生文本的形式交付，因此不包含任何构建框架，除非用户明确要求，否则不生成 HTML，且不添加任何不属于文档本身的尾注或说明性文字。

## 验证

请参照 `/opt/hatch/skills/artifacts/testing/SKILL.md`：在返回链接之前，请完整读取已完成的文件，确保其结构按预期渲染，并且不存在任何占位符或草稿内容。
