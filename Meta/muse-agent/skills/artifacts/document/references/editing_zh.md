# 编辑现有的 Word 文档

`.docx` 文件（以及 `.dotx` 模板，处理方式相同）是一个由 XML 部件组成的 ZIP 压缩包。根据编辑需求选择相应的方法：

| 编辑操作 | 路径 |
|---|---|
| 对您生成的文档进行内容修改 | 修改 `.src/` 生成器并重新生成 |
| 对已上传文档进行简单内容修改 | 使用 `python-docx` 打开、编辑并保存 |
| 跟踪修订（红字）、批注、保留格式的局部编辑 | 直接操作原始 XML：解压、编辑 `word/document.xml`、再打包（`python-docx` 无法表达此类操作） |
| 读取或提取内容 | `muse.read` 可直接打开 `.docx` 文件（转换为分页的 Markdown 格式）；当需要结构化处理时，可使用 `python-docx` 进行迭代，因为 Markdown 会将内容扁平化 |
| 旧版 `.doc` 文件 | 先转换：`soffice --headless --convert-to docx file.doc`，然后按上述方法处理 |

## 原始 XML 的往返流程

```bash
unzip -q doc.docx -d unpacked/
find unpacked -type l -delete   # 来自外部的 ZIP 条目可能是符号链接；在操作文件树之前先移除
# 在 unpacked/word/document.xml 中就地编辑
(cd unpacked && rm -f ../out.docx && zip -Xr ../out.docx .)
```

- 切勿对 XML 进行美化或重新缩进：新增的空白文本节点会影响 `xml:space` 敏感的内容，并改变渲染效果。
- 从解压后的目录内部重新打包，使各部件路径相对于归档根目录（如 `word/document.xml`，而非 `unpacked/word/...`）；Word 不接受带有前缀路径的归档。
- 先用 `rm -f` 删除输出文件：`zip` 会向现有归档追加内容，因此若未删除，即使从文件树中删除的部件仍会保留在归档中。`-X` 参数用于去除 UID/GID 和时间戳等额外字段。
- 使用 Python 提取时，应拒绝符号链接条目（通过 `stat.S_ISLNK(info.external_attr >> 16)` 判断），并在提取前检查条目是否解析到目标目录之外。

## 运行片段化问题

Word 会将可见文本拆分为多个 `<w:r>` 运行（包含修订 ID、拼写检查标记和编辑历史），因此文档中你能读到的词组，在 XML 中往往并非连续字符串，导致查找替换操作可能被忽略。在进行字符串编辑前，应将 `<w:rPr>` 序列完全相同的相邻运行合并（两者都不存在也视为匹配）；这一条件可确保渲染结果不变。`rsid*` 属性和 `<w:proofErr>` 元素均为纯元数据，可安全删除。切勿跨不同的 `<w:ins>` 或 `<w:del>` 包装进行合并：这会改写跟踪修订的结构，并导致不同修订版本被合并。

## 跟踪修订（红字标注）

将更改的运行包裹在 `<w:ins>` 或 `<w:del>` 中，每个标签都带有 `w:id`、`w:author` 和 `w:date` 属性。

- 在 `<w:del>` 内，文本元素为 `<w:delText>`，而非 `<w:t>`；域指令则变为 `<w:delInstrText>`。
- 已删除段落的标记（`<w:pPr><w:rPr><w:del .../></w:rPr></w:pPr>`）表示“将该段落与下一段合并”。删除整个段落则是在此基础上，再为每个运行加上 `<w:del>` 包装；仅其中任一部分都属于不同的编辑。
- 在 `w:rPr` 中，`<w:del/>` 子元素必须位于其他子元素之前；rPr 的子元素顺序受模式约束。
- 若要拒绝其他作者的插入内容，可将自己的 `<w:del>` 嵌套在其 `<w:ins>` 内，切勿编辑或解开对方的包装。若要恢复对方的删除，则应在对方的 `<w:del>` 后添加自己的 `<w:ins>`。变更以（类型、作者、日期、文本）为标识，因此改写他人包装会被视为全新的变更。
- 隐性失败的情况是编辑未包裹在任何包装内：在最终视图中不可见，且未被记录。完成红字标注后，请再次检查 XML，确认所有与原文不同的文本均位于您创建的 `<w:ins>` 或 `<w:del>` 标签内。此规范主要适用于文档正文；页眉、页脚和脚注为独立部件，若曾编辑过，需单独检查。

要返回一份已全部接受修订且格式干净的文档，可以使用无界面模式运行 LibreOffice，并通过一个 StarBasic 宏调用 `.uno:AcceptAllTrackedChanges` 来执行修订接受操作，随后解压输出文件并检查其中是否仍存在 `w:ins` 或 `w:del` 标签。需要注意两个问题：一是 soffice 在保存后可能会挂起（超时并不意味着失败，应以输出内容而非退出状态作为判断依据）；二是一段被完全删除的段落后若紧接一个空的分隔段落，该段落可能仅被清空而保留下来，在自动编号时会显示为一个孤立的空项目符号。这个项目符号是已接受修订视图下的产物，段落的删除情况应以 XML 内容为准。

## 注释

注释分布在六个相互关联的位置：`word/comments.xml`、`word/commentsExtended.xml`、`word/commentsIds.xml`、`word/commentsExtensible.xml`，以及它们在 `word/_rels/document.xml.rels` 中对应的四个 `Relationship` 条目，还有在 `[Content_Types].xml` 中的四个 `Override` 条目。ID 链条如下：`w:comment` 携带 `w:id` 和一个段落 ID `w14:paraId`；`commentsExtended` 以该段落 ID 为键；`commentsIds` 将段落 ID 映射到一个持久 ID `durableId`；`commentsExtensible` 则以该持久 ID 为键。

- 单独的部件本身不会显示任何内容：需要配合带有 `<w:commentRangeStart w:id="N"/>` … `<w:commentRangeEnd w:id="N"/>` 的锚点，以及一个 `<w:commentReference w:id="N"/>` 运行。范围标记是 `<w:p>` 的直接子元素，绝不会位于 `<w:r>` 内部。
- 回复的 `w15:commentEx` 会将 `w15:paraIdParent` 设置为父评论的段落 ID，其范围标记则嵌套在父评论的范围内。
- ID 上限：`w14:paraId` 必须为小于 `0x80000000` 的十六进制值，`durableId` 必须小于 `0x7FFFFFFF`；生成时可使用 `randint(0, 0x7FFFFFFE)` 并格式化为 `%08X`。`durableId` 在 `numbering.xml` 中为十进制，在其他地方则为十六进制。
- 在每个新部件的根元素上预先声明完整的 Word 命名空间集（并包含 `mc:Ignorable`），以避免后续添加的子元素因未声明的前缀而报错。

## 验证

每次编辑完成后，都会经过 `/opt/hatch/skills/artifacts/testing/SKILL.md` 中的渲染验证环节：使用无界面模式的 LibreOffice 将文档转换为 PDF，再用 pdftoppm 将其栅格化，并逐一检查每一页的图像，确认无误后再返回链接。