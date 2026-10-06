# 处理现有PDF

读取、重新组织并填写用户已有的PDF文件（上传至`~/workspace/your_files/`或作为附件）。新的PDF交付物仍然以HTML格式编写并渲染；本参考仅用于操作已存在的PDF文件。Cell中自带Poppler，但不包含`pypdf`，因此在以下任务中提到该库时，请按需安装：  
`pip install --break-system-packages pypdf`。

| 任务 | 路径 |
|---|---|
| 读取文本 | `muse.read`可直接打开PDF（转换为Markdown，并分页）；仅在需要每字坐标时使用`pdftotext -bbox-layout`生成XML格式 |
| 查看页面 | 使用`pdfppm -png -r 150 file.pdf page`生成PNG图像后再读取；`-f N -l M`指定范围，`-r 300`用于处理细小文字 |
| 列出或提取嵌入式图像 | `pdfimages -list file.pdf`；`pdfimages -all file.pdf out/img` |
| 合并文档 | `pdfunite a.pdf b.pdf out.pdf` |
| 按页拆分 / 提取指定范围 | `pdfseparate -f 2 -l 5 file.pdf page-%d.pdf`，然后用`pdfunite`合并保留的页面 |
| 填写表单、加盖水印、裁剪、使用已知密码解密 | `pypdf`（按需安装） |

文本提取无法还原版面布局：对于任何视觉相关的信息（对齐方式、元素间的相对位置、数值是否确实填入相应框内），应查看渲染后的页面图像，而非仅依赖提取的文本。

## 填写表单

首先确认PDF是否包含真正的可填写字段（AcroForm），并同时检查两种表示形式：标准的`/AcroForm/Fields`树结构，以及每一页的`/Widget`注释（通过`/Parent`和`/Kids`进行追踪）。某个Widget可能在其外观流中绘制了值，而标准字段却缺失或保存的是过时的值，因此仅凭干净的渲染结果并不能证明表单已正确填写。

```python
from pypdf import PdfReader
reader = PdfReader("form.pdf")
fields = reader.get_fields()  # 标准的/AcroForm/Fields树结构
widgets = [
    annot
    for page in reader.pages
    for annot in (page.get("/Annots") or [])
    if annot.get_object().get("/Subtype") == "/Widget"
]
```

**可填写字段。** 默认情况下保持交互性；仅当用户明确要求生成已完成的静态表单时才进行扁平化处理，且绝不在未经明确同意的情况下对已签名的PDF进行扁平化。保留原始PDF，并在用户可能修改表单时保留未扁平化的副本。写入前请检查每个字段的类型及状态：文本字段接受字符串；复选框必须设置为其自身的选中导出值（查看字段的状态：`/Off`表示未选中，另一状态通常为`/Yes`或`/On`，表示已选中）；单选组则需填写其选项之一的导出值。

仅当 `get_fields()` 中确实缺少预期字段时才调用 `reattach_fields()`，切勿将其作为常规修复手段：其孤立检测条件是“携带 `/FT` 但不在顶级 `/Fields` 数组中的 widget”，它从不遍历 `/Kids`，因此在结构良好的表单中，它会将正确嵌套的子 widget（例如每个单选按钮的子 widget）附加到根 `/Fields` 下，从而生成一棵仅靠值检查无法发现的违反规范的树。如果某个 widget 和一个规范字段同名，但它们是彼此独立的对象且没有 `/Parent` 关系，则不要重新附加任何一个：报告这种歧义，或直接返回一个扁平化的静态结果。

pypdf 的两个已验证的扁平化边界情况：单选按钮组的子 widget 共享同一个扁平化后的外观名称 (`/Fm_<field>`)，因此扁平化后的单选按钮组会在每个位置绘制第一个子 widget 的外观，而选择状态则会被无声地丢失；此外，在预绘制过程中，如果遇到没有 `/AP` 的 `/Btn` widget（在表单发布时设置 `NeedAppearances` 是合法的），程序会崩溃。仅对那些所有按钮都带有 `/AP` 的无单选按钮组的表单进行扁平化；否则，提供交互式填写功能并说明原因。

```python
from pypdf import PdfReader, PdfWriter
from pypdf.generic import NameObject

writer = PdfWriter()
writer.clone_document_from_reader(PdfReader(input_pdf))
fields = writer.get_fields() or {}
if set(expected_values) - set(fields):
    writer.reattach_fields()  # 只恢复真正孤立的 widget
    fields = writer.get_fields() or {}
missing = set(expected_values) - set(fields)
if missing:
    raise ValueError(f"修复后仍找不到以下字段: {sorted(missing)}")

values = dict(expected_values)
if flatten:
    # 在移除下方 widget 之前，先为每个现有值绘制外观。
    values = {
        name: field.get("/V", "/Off" if field.get("/FT") == "/Btn" else "")
        for name, field in fields.items()
    } | expected_values

# auto_regenerate=None 会保留输入的 NeedAppearances 标志不变；
# True/False 都会覆盖该标志，而清除它可能会阻止未修改的
# 预填充值在严格显示外观的查看器中渲染。
writer.update_page_form_field_values(
    None, values, auto_regenerate=None, flatten=flatten
)
if flatten:
    # flatten=True 会绘制外观但保留小部件原位。
    writer.remove_annotations(subtypes="/Widget")
    writer.root_object.pop(NameObject("/AcroForm"), None)
with open(output_pdf, "wb") as stream:
    writer.write(stream)
```

**应在重新打开的输出文件上验证，而非在写入器上。** 对于交互式结果：在 `get_fields()` 中应存在每个预期字段，并带有预期的 `/V` 值；每个页面小部件的有效值（其自身的 `/V` 或继承的 `/Parent` 值）均一致；每个已更新的小部件都具有非空的 `/AP` `/N` 外观；且根级 `/Fields` 条目中不应包含 `/Parent`（被提升为根级的子小部件即为上述的重新附加损坏问题）。随后对每一页进行光栅化并检查是否有过时或裁剪的外观。无论是 `/NeedAppearances` 还是干净的 PNG 渲染都无法证明逻辑字段数据已被更新；真正能证明的是重新打开后的字段树。对于已展平的结果：应无 `/Widget` 注释且根级 `/AcroForm` 字段树也不再存在；所有值，尤其是单选按钮的选择，都应在渲染后的页面上可见。

**无可填写字段。** 表单只是页面内容，因此需在其上方叠加文本：

1. 定位每个空白区域。`pdftotext -bbox-layout` 可提供每个标签的确切坐标（原点为左上角，单位为 PDF 点），输入区域从标签结束处开始，延伸至下一个标签或分隔线。对于没有文本层的扫描 PDF，需以高分辨率进行光栅化并读取图像，通过精确裁剪区域（`gm convert page.png -crop WxH+X+Y out.png`）确定位置；再根据页面尺寸与图像尺寸之比将像素转换回 PDF 点。
2. 构建一个透明覆盖层：为每页生成一个 HTML 页面，尺寸与 PDF 页面完全一致，并将每个值绝对定位；然后按常规 PDF 工作流程进行渲染。
3. 使用 pypdf 打印：对每页执行 `page.merge_page(overlay_page)`，最后写入文件。
4. 通过光栅化输出并逐页读取来验证；若文本位于错误的行上或与标签重叠，则说明需要调整的是坐标而非方法。

请勿猜测表单中未知的答案：仅填写请求提供的内容，其余部分保持空白。

## 加密或损坏的输入文件

`PdfReader.is_encrypted` 结合 `reader.decrypt(password)` 可处理用户已提供密码的文件；切勿尝试绕过用户未掌握的密码。若 poppler 工具因文件损坏而拒绝打开，请明确告知并要求重新导出，而非手动修复二进制文件。

## 验证

任何生成或修改的 PDF 在交付前都必须通过 `/opt/hatch/skills/artifacts/testing/SKILL.md` 中的渲染检查：使用 pdftoppm 进行光栅化，并在提供下载链接前逐页读取图像。
