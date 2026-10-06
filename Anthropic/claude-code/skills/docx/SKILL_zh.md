---
name: docx
description: "每当用户需要创建、读取、编辑或处理 Word 文档（.docx）或 Word 模板（.dotx）时，均可使用此技能。触发条件包括：任何提及 Microsoft Word 文档的表述，例如“Word 文档”、“word document”、“.docx”、“.dotx”、“microsoft doc”等。此外，在从 .docx 或 .dotx 文件中提取或重新组织内容、在文档中插入或替换图片、在 Word 文件中执行查找与替换、处理修订或批注，或将内容转换为格式整洁的 Word 文档时，也应使用此技能。如果用户要求以 Word 格式或 .docx 文件的形式交付成果（用于下载、发送电子邮件或打印），则应使用此技能。然而，如果用户仅要求提供“文档”“页面”“报告”“备忘录”或“笔记”，且未指定文件格式，而会话中提供了 Claude 自带的专用文档或页面技能或连接器，则应优先使用该技能，即使最终用户会通过电子邮件发送或打印。请勿将此技能用于 PDF、电子表格、Google 文档，或与文档生成无关的编程任务。"
license: 专有。LICENSE.txt 文件中包含完整条款。
---
# DOCX 的创建、编辑与分析

`.docx` 是一个由 XML 文件组成的 ZIP 压缩包。请根据具体任务选择合适的方法：

| 任务 | 方法 |
|---|---|
| **创建** 新文档 | 编写 `docx`（npm）脚本——参见下方注意事项 |
| **编辑** 现有文档 | 先 `unzip`，编辑 `word/document.xml`，再 `zip`（docx-js 无法打开现有文件） |
| **读取** 内容 | `pandoc -t markdown file.docx` |

> 下文中的脚本路径均相对于本技能的目录。

## 使用 docx-js 创建文档——常见问题

`docx` 已预装，请勿先运行 `npm install`；直接编写脚本并 `require('docx')` 即可。仅当 `require` 失败时才执行 `npm install docx`。API 使用方法已内置，以下是需要注意的陷阱：

- **页面尺寸默认为 A4。** 如需设置为美国信纸尺寸，请使用 `page: { size: { width: 12240, height: 15840 } }`（单位：DXA；1440 = 1 英寸）。
- **横向页面：** 传入纵向尺寸，并设置 `orientation: PageOrientation.LANDSCAPE`——docx-js 会在内部交换宽度和高度。
- **表格需要双重宽度设置：** 既要在表格上设置 `columnWidths`，也要在每个单元格上设置 `width`，且单位均为 `WidthType.DXA`（使用百分比会导致 Google 文档显示异常）。各列宽度之和必须等于表格总宽度。
- **表格底纹：** 使用 `ShadingType.CLEAR`，切勿使用 `SOLID`（会渲染为黑色）。
- **列表：** 切勿直接插入 `•`，应使用带有 `LevelFormat.BULLET` 的 `numbering` 配置。
- **`ImageRun` 必须指定 `type:`**（如 `"png"`、`"jpg"` 等）。
- **`PageBreak` 必须位于 `Paragraph` 内部。**
- **切勿使用 `\n`**——应使用单独的 `Paragraph` 元素。
- **目录：** 标题必须使用内置的 `HeadingLevel.*`；自定义标题样式需设置 `outlineLevel`，否则不会出现在目录中。
- **不要用表格充当水平线**——应使用段落底部边框代替。
- **点线填充/同行右对齐：** 在 `TextRun` 中使用 `PositionalTab`（`alignment: PositionalTabAlignment.RIGHT`，`leader: PositionalTabLeader.DOT`），而非直接使用 `.` 或空格填充。

## 验证输出结果

编写完 `.docx` 后，将其渲染并查看：

```bash
python scripts/office/soffice.py --headless --convert-to pdf output.docx
pdftoppm -jpeg -r 100 output.pdf page
ls page-*.jpg   # 然后读取这些图片
```

`pdftoppm` 会按页数位数对页码进行零填充（如 `page-01.jpg`…`page-12.jpg`）。

## 编辑现有文档

旧版 `.doc` 文件需先转换：`python scripts/office/soffice.py --headless --convert-to docx file.doc`。

```bash
unzip -q doc.docx -d unpacked/
find unpacked -type l -delete   # 删除符号链接条目——来自外部的 DOCX 文件不可信
python scripts/merge_runs.py unpacked/   # 合并碎片化的文本块，便于搜索
# 直接编辑 unpacked/word/document.xml——切勿重新格式化或美化
(cd unpacked && rm -f ../out.docx && zip -Xr ../out.docx .)
python scripts/office/validate.py out.docx --original doc.docx   # 进行 XSD 检查；--auto-repair 可修复常见问题
# 需要批注功能？添加 --author "<你的批注用户名>" 参数，以确保每处修改都被记录
```

Word 会将文本拆分到多个 `<w:r>` 运行中（包含修订 ID 和拼写检查标记），因此你在文档中看到的短语，在 XML 中往往并非连续的字符串。`merge_runs.py` 会在不改变内容和渲染效果的前提下，合并 `word/document.xml` 中相邻且格式相同的运行；它也支持直接接收 `.docx` 文件（`python scripts/merge_runs.py doc.docx -o merged.docx`）。**修订跟踪**：在进行修订标记时，请使用 `--author "<您使用的修订人姓名>"` 参数进行验证（需配合 `--original` 使用）——该参数会报告所有未被 `<w:ins>` 或 `<w:del>` 标签包裹的修改文本，这类错误容易发生且在“已接受”视图中不可见。请为文本片段加上带有 `w:id`、`w:author` 和 `w:date` 属性的 `<w:ins>` 或 `<w:del>` 标签。在 `<w:del>` 中，文本元素应为 `<w:delText>`，而非 `<w:t>`。一个被删除的段落标记（`<w:pPr><w:rPr><w:del w:id=".." w:author=".." w:date=".."/></w:rPr></w:pPr>`）表示“将本段与下一段合并”——因此，要彻底删除一个段落，除了在每个文本片段外层加上 `<w:del>` 标签之外，还需删除该段落标记。`<w:del/>` 必须位于 rPr 的其他子元素之前；其顺序由架构强制约束。

要生成一份已接受所有修订的干净文档：`python scripts/accept_changes.py in.docx out.docx`。

接受一个被删除的段落标记后，应将其与下方的段落合并；因此，如果某个段落的所有文本片段都被删除，则该段落将完全消失。Word 会正确执行此操作，但 `accept_changes.py` 和 `pandoc --track-changes=accept` 并不总是如此。两者都存在同样的问题——它们会移除被删除的文本，却保留已被清空的段落，当该段落设置了自动编号时，就会显示为一个孤立的空项目符号：

- `pandoc --track-changes=accept` 从不合并这些段落。
- `accept_changes.py`（基于 LibreOffice）能正确合并，但当被删除的段落后紧跟着一个空的分隔段落时则无法正常工作。

无论在哪种视图中出现的空项目符号，都是该视图的呈现结果，而非文档本身的缺陷。请在 XML 中检查段落是否已被正确删除。

## 评论

添加评论需要六个相互关联的文件。请使用辅助脚本——如果您还需要编辑 `document.xml`，建议使用目录模式（可省去解压和重新打包的步骤）；否则可直接对 `.docx` 文件操作：

```bash
# 针对已解压的目录（当同时需要插入锚点时优先使用）
python scripts/comment.py unpacked/ "费用与开支上限过低"
python scripts/comment.py unpacked/ "同意" --parent 0

# 直接针对 .docx 文件
python scripts/comment.py contract.docx "这个上限太低了" -o annotated.docx
```

该脚本会生成 `comments.xml`、`commentsExtended.xml`、`commentsIds.xml`、`commentsExtensible.xml`，以及关系文件和内容类型覆盖文件。评论 ID 会自动生成。随后，脚本会输出一段 `<w:commentRangeStart>`/`<w:commentRangeEnd>`/`<w:commentReference>` 代码片段，供您添加到 `word/document.xml` 中，以使评论锚定到特定文本——在您插入这些锚点之前，评论虽然存在但不可见。

## 依赖项

`docx`（npm，已预装——仅当 `require('docx')` 报错时才需安装）· `pandoc` · LibreOffice (`soffice`) · `pdftoppm`（Poppler）
