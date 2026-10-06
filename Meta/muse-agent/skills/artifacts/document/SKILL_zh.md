---
name: artifact_document
metadata: { "不包含在提示中": 假 }
description: 创建、读取、编辑或操作 Word 文档（.docx）和 Word 模板（.dotx）。当构建任务的工件类型为文档且默认输出为 .docx，或者任务提及 Word 文档、.docx 或 .dotx 时使用；也可用于从文档中提取或重新组织内容、插入或替换图片、执行查找与替换操作，或处理修订（标记更改）和批注。涵盖 python-docx 的文档生成、对现有文件的原始 OOXML 编辑、文档结构与格式设置，以及渲染验证。不适用于 PDF、电子表格或 Google 文档。
---
# Word 文档相关工件

通过预装的 `python-docx` 库，借助生成脚本创建 .docx 文件。请将该生成脚本保留在 `.src/` 目录下：它是未来修订时可编辑的源文件，二进制文档始终由此重新生成。

| 任务 | 首先阅读 |
|---|---|
| 设计与结构 | `/opt/hatch/skills/artifacts/document/references/visual.md` |
| 文字内容：大纲、标题、语气、文稿通读 | `/opt/hatch/skills/artifacts/references/prose.md`（共用） |
| 内容格式化（表格、列表、强调） | `/opt/hatch/skills/artifacts/references/markdown.md`（共用） |
| 文档中包含地点或地图 | `/opt/hatch/skills/artifacts/references/maps.md`（共用） |
| 编辑现有或已上传的 .docx/.dotx 文件，处理修订记录与批注，提取/读取内容，以及处理旧版 .doc 文件 | `/opt/hatch/skills/artifacts/document/references/editing.md` |

务必为 `document.core_properties.title` 设置一个非空值，使用便于人类识别的文件名和显眼的标题，并且切勿伪造文档结构：列表应使用真实的编号（切勿直接使用项目符号字符），目录中需要显示的内容必须使用真正的标题样式，分隔线应使用段落底边框（切勿使用单行表格），并在段落内使用换行符时应拆分为独立段落，而非在同一个运行（run）中插入换行。

## 验证

请参照 `/opt/hatch/skills/artifacts/testing/SKILL.md`：使用无界面的 LibreOffice 将文档渲染为 PDF，再用 pdftoppm 进行光栅化，并在提供链接前逐页检查图像内容。
