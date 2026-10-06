---
name: artifact_pdf
metadata: { "不包含在提示中": 假 }
description: 构建、修订或操作固定版式 PDF（报告、指南、单页文档、可打印文档）。当构建任务的工件类型为 PDF、文档构建的输出格式为 PDF，或任务涉及读取、合并、拆分、裁剪或填充现有 PDF（包括可填写的 AcroForms）时，请使用此功能。涵盖打印 CSS HTML 源文件的编写、渲染、页面布局与验证流程、现有 PDF 的处理与表单填写，以及在 workspace/your_files 下的交付。
---
# PDF 产物

PDF 是基于 HTML 和打印 CSS 编写，并通过共享的捕获引擎进行渲染生成的。位于 `.src/` 目录下的源文件是未来每次修订的可编辑基准；PDF 二进制文件始终会被重新生成，绝不会被直接修改。

| 任务 | 首先阅读 |
|---|---|
| 任何 PDF 的构建或编辑 | `/opt/hatch/skills/artifacts/pdf/references/workflow.md`（工作流程：编写、渲染、门控检查、验证循环） |
| 设计与排版 | `/opt/hatch/skills/artifacts/pdf/references/visual.md` |
| 文字内容：大纲、标题、语气、文稿的朗读反馈 | `/opt/hatch/skills/artifacts/references/prose.md`（通用） |
| 文档中的数据图表 | `/opt/hatch/skills/artifacts/references/charts.md`（通用） |
| 文档中的地点或地图 | `/opt/hatch/skills/artifacts/references/maps.md`（通用） |
| 阅读、合并、拆分或提取现有 PDF；填写 PDF 表单 | `/opt/hatch/skills/artifacts/pdf/references/existing-pdfs.md` |

## 脚本

| 脚本 | 功能 |
|---|---|
| `/opt/hatch/skills/artifacts/scripts/render_audit.mjs`（通用） | 将 HTML 源文件渲染为 PDF 和 PNG，并执行渲染门控检查；相关参数在 `workflow.md` 中说明 |
| `/opt/hatch/skills/artifacts/scripts/validate_pdf.sh` | 检查文件完整性、页面元数据、data URI 嵌入及全分辨率栅格化；持续运行直至通过所有检查 |

## 验证

请参照 `/opt/hatch/skills/artifacts/testing/SKILL.md`：先执行门控检查，然后在提供链接之前仔细查看每一张验证用的 PNG 图像。