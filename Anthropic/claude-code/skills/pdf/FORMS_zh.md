**重要提示：您必须按顺序完成这些步骤，切勿跳过直接开始编写代码。**

如果需要填写 PDF 表单，请先检查该 PDF 是否包含可填写的表单字段。请在本文件所在目录下运行以下脚本：
`python scripts/check_fillable_fields <file.pdf>`，根据结果选择“可填写字段”或“不可填写字段”部分，并按照相应说明操作。

# 可填写字段
如果 PDF 包含可填写的表单字段：
- 在本文件所在目录下运行此脚本：`python scripts/extract_form_field_info.py <input.pdf> <field_info.json>`。该脚本将生成一个 JSON 文件，其中包含以如下格式列出的表单字段信息：
```
[
  {
    "field_id": （该字段的唯一 ID）,
    "page": （页码，从 1 开始计数）,
    "rect": （PDF 坐标系下的边界框，[左, 下, 右, 上]，y=0 为页面底部）,
    "type": （"text"、"checkbox"、"radio_group" 或 "choice"）,
  },
  // 复选框具有 "checked_value" 和 "unchecked_value" 属性：
  {
    "field_id": （该字段的唯一 ID）,
    "page": （页码，从 1 开始计数）,
    "type": "checkbox",
    "checked_value": （将字段设置为此值可选中该复选框）,
    "unchecked_value": （将字段设置为此值可取消选中该复选框）,
  },
  // 单选按钮组包含一个 "radio_options" 列表，列出所有可能的选项：
  {
    "field_id": （该字段的唯一 ID）,
    "page": （页码，从 1 开始计数）,
    "type": "radio_group",
    "radio_options": [
      {
        "value": （将字段设置为此值可选中该单选选项）,
        "rect": （该选项对应单选按钮的边界框）
      },
      // 其他单选选项
    ]
  },
  // 多选字段包含一个 "choice_options" 列表，列出所有可能的选项：
  {
    "field_id": （该字段的唯一 ID）,
    "page": （页码，从 1 开始计数）,
    "type": "choice",
    "choice_options": [
      {
        "value": （将字段设置为此值可选中该选项）,
        "text": （该选项的显示文本）
      },
      // 其他多选选项
    ],
  }
]
```
- 使用以下脚本将 PDF 转换为 PNG 图像（每页一张图像）：（请在本文件所在目录下运行）
`python scripts/convert_pdf_to_images.py <file.pdf> <output_directory>`
然后分析这些图像，确定每个表单字段的作用（请注意将 PDF 中的边界框坐标转换为图像坐标）。
- 创建一个 `field_values.json` 文件，按如下格式填写各字段的待填值：
```
[
  {
    "field_id": "last_name", // 必须与 `extract_form_field_info.py` 中的 field_id 一致，
    "description": "用户的姓氏",
    "page": 1, // 必须与 field_info.json 中的 "page" 值一致，
    "value": "Simpson"
  },
  {
    "field_id": "Checkbox12",
    "description": "若用户年满 18 岁则需勾选的复选框",
    "page": 1,
    "value": "/On" // 若为复选框，使用其 "checked_value" 值来选中；若为单选按钮组，则使用 "radio_options" 中的一个 "value" 值。
  },
  // 更多字段
]
```
- 在本文件所在目录下运行 `fill_fillable_fields.py` 脚本，生成已填写的 PDF：
`python scripts/fill_fillable_fields.py <input pdf> <field_values.json> <output pdf>`
该脚本会验证您提供的字段 ID 和值是否有效；若输出错误信息，请更正相应内容后重新尝试。

# 不可填写字段
如果 PDF 不包含可填写的表单字段，则需要添加文本注释。首先尝试从 PDF 结构中提取坐标（精度更高），必要时再采用视觉估算方法。

## 第一步：优先尝试结构提取
运行以下脚本，提取文本标签、线条和复选框及其精确的 PDF 坐标：
`python scripts/extract_form_structure.py <input.pdf> form_structure.json`这会生成一个包含以下内容的 JSON 文件：
- **labels**：每个文本元素及其精确坐标（x0、top、x1、bottom，单位为 PDF 点）
- **lines**：用于定义行边界的水平线
- **checkboxes**：作为复选框的小矩形（含中心坐标）
- **row_boundaries**：根据水平线计算出的各行顶部和底部位置

**检查结果**：如果 `form_structure.json` 中存在有意义的标签（即与表单字段对应的文本元素），请使用 **方法 A：基于结构的坐标法**。如果 PDF 是扫描件或图片格式，且几乎没有或没有标签，则使用 **方法 B：视觉估算法**。

---

## 方法 A：基于结构的坐标法（首选）

当 `extract_form_structure.py` 在 PDF 中识别出文本标签时，请使用此方法。

### A.1：分析结构

读取 `form_structure.json` 并识别：
1. **标签组**：构成单个标签的相邻文本元素（如“Last”+“Name”）
2. **行结构**：具有相似 `top` 值的标签位于同一行
3. **字段列**：输入区域起始于标签结束之后（x0 = 标签.x1 + 间隙）
4. **复选框**：直接使用结构中提供的复选框坐标

**坐标系**：PDF 坐标系中，y=0 位于页面顶部，y 值向下递增。

### A.2：检查缺失元素

结构提取可能无法检测到所有表单元素。常见情况包括：
- **圆形复选框**：仅方形矩形会被识别为复选框
- **复杂图形**：装饰性元素或非标准表单控件
- **褪色或浅色元素**：可能无法被提取

如果在 PDF 图像中看到 `form_structure.json` 中未包含的表单字段，则需要对这些特定字段采用 **视觉分析法**（详见下文“混合方法”）。

### A.3：创建包含 PDF 坐标的 fields.json

针对每个字段，根据提取的结构计算输入区域的坐标：

**文本字段：**
- 输入区域 x0 = 标签 x1 + 5（标签后留有少量间距）
- 输入区域 x1 = 下一个标签的 x0，或行边界
- 输入区域 top = 与标签 top 相同
- 输入区域 bottom = 下方的行边界线，或标签 bottom + 行高

**复选框：**
- 直接使用 `form_structure.json` 中提供的复选框矩形坐标
- 输入区域 bounding_box = [复选框.x0, 复选框.top, 复选框.x1, 复选框.bottom]

使用 `pdf_width` 和 `pdf_height`（表示 PDF 坐标）创建 `fields.json`：
```json
{
  "pages": [
    {"page_number": 1, "pdf_width": 612, "pdf_height": 792}
  ],
  "form_fields": [
    {
      "page_number": 1,
      "description": "姓氏输入字段",
      "field_label": "Last Name",
      "label_bounding_box": [43, 63, 87, 73],
      "entry_bounding_box": [92, 63, 260, 79],
      "entry_text": {"text": "Smith", "font_size": 10}
    },
    {
      "page_number": 1,
      "description": "美国公民‘是’选项复选框",
      "field_label": "Yes",
      "label_bounding_box": [260, 200, 280, 210],
      "entry_bounding_box": [285, 197, 292, 205],
      "entry_text": {"text": "X"}
    }
  ]
}
```

**重要提示**：请直接使用 `pdf_width`/`pdf_height` 和 `form_structure.json` 中的坐标。

### A.4：验证边界框

在填写前，请检查您的边界框是否存在错误：
`python scripts/check_bounding_boxes.py fields.json`

该脚本会检查是否存在边界框相交的情况，以及输入框是否过小而无法容纳指定字号的文本。请在填写前修正所有报告的错误。

---

## 方法 B：视觉估算法（备用方案）

当 PDF 为扫描件或图片格式，且结构提取未发现可用的文本标签时（例如所有文本均显示为“(cid:X)”模式），请使用此方法。

### B.1：将 PDF 转换为图像

`python scripts/convert_pdf_to_images.py <input.pdf> <images_dir/>`

### B.2：初步识别字段

逐页查看图像，识别表单各部分，并对字段位置进行 **粗略估算**：
- 表单字段标签及其大致位置
- 输入区域（用于文本输入的线条、方框或空白区域）
- 复选框及其大致位置对于每个字段，记录大致的像素坐标（目前无需非常精确）。

### B.3：缩放精修（对精度至关重要）

对于每个字段，在估计位置周围裁剪一个区域，以精确调整坐标。

**使用 ImageMagick 创建缩放后的裁剪图像：**
```bash
magick <page_image> -crop <width>x<height>+<x>+<y> +repage <crop_output.png>
```

其中：
- `<x>, <y>` = 裁剪区域的左上角坐标（使用您的粗略估计值减去边距）
- `<width>, <height>` = 裁剪区域的尺寸（字段区域加上每边约50像素的边距）

**示例：** 对“姓名”字段进行精修，其估计位置为 (100, 150)：
```bash
magick images_dir/page_1.png -crop 300x80+50+120 +repage crops/name_field.png
```

（注意：如果 `magick` 命令不可用，可尝试使用 `convert` 并传入相同的参数。）

**检查裁剪后的图像**以确定精确坐标：
1. 确定输入区域在标签之后的确切起始像素位置；
2. 确定输入区域在下一个字段或边界之前的结束位置；
3. 确定输入行或输入框的顶部和底部。

**将裁剪坐标转换回全图坐标：**
- full_x = crop_x + crop_offset_x
- full_y = crop_y + crop_offset_y

例如：如果裁剪区域的起始点为 (50, 120)，而输入框在裁剪区域内从 (52, 18) 开始：
- entry_x0 = 52 + 50 = 102
- entry_top = 18 + 120 = 138

**对每个字段重复上述步骤**，并尽可能将相邻字段合并到同一张裁剪图中。

### B.4：创建带有精修坐标信息的 fields.json

使用 `image_width` 和 `image_height`（表示图像坐标）创建 fields.json：
```json
{
  "pages": [
    {"page_number": 1, "image_width": 1700, "image_height": 2200}
  ],
  "form_fields": [
    {
      "page_number": 1,
      "description": "姓氏输入字段",
      "field_label": "Last Name",
      "label_bounding_box": [120, 175, 242, 198],
      "entry_bounding_box": [255, 175, 720, 218],
      "entry_text": {"text": "Smith", "font_size": 10}
    }
  ]
}
```

**重要提示**：请使用 `image_width`/`image_height` 以及通过缩放分析得到的精修像素坐标。

### B.5：验证边界框

在填写之前，请检查您的边界框是否存在错误：
`python scripts/check_bounding_boxes.py fields.json`

该脚本会检查是否有边界框相互重叠，以及输入框是否过小、无法容纳指定字号的文字。在填写前请修复所有报告的错误。

---

## 混合方法：结构与视觉结合

当结构提取能够识别大部分字段，但遗漏了某些元素（如圆形复选框、特殊表单控件）时，可采用此方法。

1. 对于已在 form_structure.json 中检测到的字段，使用方法 A；
2. 将 PDF 转换为图像，以便对缺失字段进行视觉分析；
3. 对缺失字段使用缩放精修方法（见方法 B）；
4. 合并坐标：对于通过结构提取获得的字段，使用 `pdf_width`/`pdf_height`；对于通过视觉估算的字段，需将图像坐标转换为 PDF 坐标：
   - pdf_x = image_x * (pdf_width / image_width)
   - pdf_y = image_y * (pdf_height / image_height)
5. 在 fields.json 中统一使用一种坐标系——将所有坐标都转换为基于 `pdf_width`/`pdf_height` 的 PDF 坐标。

---

## 第二步：填写前验证

**在填写前务必验证边界框：**
`python scripts/check_bounding_boxes.py fields.json`

该脚本会检查以下问题：
- 边界框是否相交（会导致文本重叠）；
- 输入框是否过小，无法容纳指定字号的文字。

在继续操作之前，请先修复 fields.json 中报告的所有错误。

## 第三步：填写表单

填写脚本会自动检测坐标系并完成转换：
`python scripts/fill_pdf_form_with_annotations.py <input.pdf> fields.json <output.pdf>`

## 第四步：验证输出

将填写后的 PDF 转换为图像，并检查文本排版是否正确：
`python scripts/convert_pdf_to_images.py <output.pdf> <verify_images/>`如果文本位置偏移：
- **方法 A**：检查是否使用了来自 form_structure.json 的 PDF 坐标，并已乘以 `pdf_width`/`pdf_height` 进行缩放。
- **方法 B**：检查图像尺寸是否匹配，且坐标是否为准确的像素值。
- **混合方法**：对于通过目测估算的字段，确保坐标转换正确。