---
description: 基于 HTML 的 PDF 生成。构建 HTML 源文件，使用 render_audit.mjs 进行渲染，并验证输出的页面。
---
# PDF 文档

PDF 生成以 HTML 为先。构建一个自包含的 HTML 源文件，使用 `render_audit.mjs` 将其渲染为 PDF（这是一个由 Playwright 驱动的渲染工具，同时还会执行字体和溢出检查），然后在返回链接之前进行验证。所需的验证工具（`pdfinfo`、`pdftoppm`、`pdftotext`、`fc-list`）已预先安装。`render_audit.mjs` 依赖于与 Web Artifact 运行时资产捆绑在一起的 Playwright 运行时，并在存在时使用预装的 Chromium 浏览器（位于 `/opt/meta-chromium/chrome`；详见本文末尾的说明）。

**核心规则：**
- 在 `project_dir/.src/` 下编写构建源文件，并将最终的 PDF 保存到 `project_dir/<artifact-slug>.pdf`。渲染完成后，请保留 `.src/` 目录。
- 将图像资源以内联 `data:` URI 的形式嵌入，包括 CSS 中的 `background-image` URL。请勿使用外部的 `http://`、`https://` 或 `file://` 图像引用。
- 照片必须在打印分辨率下保持清晰。渲染阶段会对此进行检测：如果图像的像素数少于其渲染后的宽度，则视为不合格；若低于渲染后宽度的两倍，则会发出警告。请使用高分辨率的原始素材（封面或跨页大图需要高分辨率原图），并在嵌入前对图像进行压缩（照片使用 JPEG 格式），以确保 `data:` URI 的大小不会过大。
- 只要使用了 `media.generate_image` 来生成文档中的媒体内容，无论是 PDF 中的图片、装饰性或虚构的地图插图，还是其他任何插图类资产，都应将 `output_dir` 字段设置为 `artifact_media_dir`，以便生成的文件在嵌入前始终位于 `.src/media/` 目录下。
- 每个概念仅使用一张准确的图片。宁可省略图片，也不要重复使用不匹配的图片。
- 切勿为了使请求成功而禁用 TLS 证书验证。
- 当文档中涉及数据可视化时，请先阅读 `/opt/hatch/skills/artifacts/references/charts.md`：该文档规定了数据来源、如何为 PDF 渲染图表，以及适用于所有场景的编码规则。
- 对于真实地点或地理数据的地图，请先阅读 `/opt/hatch/skills/artifacts/references/maps.md`：该文档决定了 PDF 中地图的渲染方式及存储方式。`media.generate_image` 仅用于装饰性或虚构的地图艺术。
- 在渲染前检查内容的完整性：如果表格或时间表列出了 N 个项目，正文部分就应有 N 个对应的详细章节。
- 避免在 PDF 的 HTML 中使用表情符号；许多 PDF 字体组合会将其渲染为方框。
- 使用 ASCII 连字符，而非不常见的破折号字符（如 U+2011 不换行连字符等）：已安装的字体缺少这些字符的字形，会导致显示为方框。Noto 和 Liberation 字体支持范围内的 en dash 是可以接受的，但任何特殊的字符都必须在 PNG 渲染阶段确认能够正确显示，切勿默认其可用。
- 使用本地安装的字体。命名字体前请通过 `fc-list` 进行验证；可靠的选择包括 `Noto Serif`、`Noto Sans`、`Liberation Serif` 和 `Liberation Sans`。DejaVu 字体并未安装在虚拟机镜像中，因此若指定该字体，Chromium 会自动使用默认字体进行渲染。请勿依赖 Google Fonts 引入的字体，也不要用 Arial、Calibri、Georgia 或 Times New Roman 等 Microsoft 字体。
  
**修订本工作区已生成的 PDF（简报中指定了现有 slug）：** PDF 是渲染结果，而非源文件。请恢复 `project_dir/.src/index.html`，在此处进行修改，然后按照相同的流程重新渲染。（用户提供的 PDF 没有 `.src/` 目录，属于另一项任务：参见 `existing-pdfs.md`。）
首先阅读现有的 HTML 文件，并将当前 PDF 中存在的每个部分都保留在重新渲染的过程中，除非简报要求删除某些部分。删减是正常的编辑操作：当简报要求缩短篇幅、删去附录或控制页数时，应明确移除整节内容，而保留的部分则按原样传递。非编辑导致的内容丢失——即因完全重写文档而非基于原稿修改而导致的内容缺失——是不可接受的。切勿从零开始撰写替代文档，也切勿在原有 PDF 之外再生成一份新的 PDF。如果该 slug 缺少 `.src/` 目录，请予以说明并寻求指示——直接根据已渲染的 PDF 重建会无意间丢失原有的所有内容。**HTML 设置：**
1. 创建 `project_dir/.src/media/` 目录，并在其中写入 `project_dir/.src/index.html` 文件。
2. 包含 DOCTYPE、`lang` 属性、字符集、视口设置、一个描述性的 `<title>` 标签，以及一个仅用于打印的 `<style>` 块。
3. 如果提供了用户在原话请求或上文对话中的样式要求，则按其指示进行设置。在 CSS 中使用之前，先将请求的字体映射到本地已安装的字体族。
4. 使用以下基础打印 CSS，然后仅根据需要进行调整：

   ```css
   @page { size: A4; margin: 0; }
   * { box-sizing: border-box; print-color-adjust: exact; -webkit-print-color-adjust: exact; }
   body { margin: 0; font-family: 'Liberation Sans', 'Noto Sans', sans-serif; }
   .page { min-height: 297mm; padding: 2cm; }  /* Letter: min-height: 11in */
   .card, section, figure, table { break-inside: avoid; page-break-inside: avoid; }
   img { display: block; width: 100%; max-width: 100%; height: auto; max-height: 8cm; object-fit: contain; }
   .card, section { display: flow-root; }
   ```

   对于图表、地图、示意图和截图，请始终使用 `object-fit: contain`。如果同时设置了 `max-height`，则 `cover` 会将图像裁剪以填满容器，并在边缘悄悄截掉部分内容：坐标轴、标签、图例和最外侧的数据点都会消失，而渲染报告和 `validate_pdf.sh` 脚本仍然会通过，因为被裁剪的图像并未溢出其容器。只有在您确实打算对装饰性照片进行裁剪时，才使用 `cover`。

   避免使用出血封面：将照片拉伸至页面边缘会放大小尺寸素材，裁剪内容，并留下意外的空白边距。默认采用非出血的页面内主图，仅在设计需要且素材像素能够覆盖整个页面时才使用出血效果。

**渲染 + 审计：**

```sh
SLUG="<artifact-slug>"
# 您的构建任务名称为 `project_dir`，请使用它。一个目标的文档会构建在其对应的 `files/` 目录下，因此硬编码的 `your_files` 路径会将 PDF 写入无人读取的位置。
# `project_dir` 来自您的构建任务，请将其开头的 `~/` 写成 `$JARVIS_HOME/`：因为在引号内，Shell 会保留波浪号字面量，所以 `"~/workspace/..."` 会被解析为字面意义的 `~` 目录。
DIR="$JARVIS_HOME/<来自构建任务的 project_dir，去掉开头的 ~/>"
bun run "/opt/hatch/skills/artifacts/scripts/render_audit.mjs" \
  --html "$DIR/.src/index.html" \
  --pdf "$DIR/$SLUG.pdf" \
  --page-selector ".page" \
  --fonts "Liberation Sans:400,Liberation Sans:700" \
  --require-geometry --require-text-floor --require-image-resolution --hermetic --gate
```

`render_audit.mjs` 在 Chromium 中加载 HTML，遵循 `@page { size: ... }` CSS 规则
（`preferCSSPageSize`），而非 Chromium 的默认页面尺寸，生成 PDF，并将一份 JSON 报告输出到标准输出。对于设计所依赖的每种网页字体/字形，请使用 `--fonts "Family:weight,..."` 参数；对于溢出检查，请使用与页面容器匹配的 `--page-selector`（默认为
`.slide-container, .page, section.slide`）。诊断信息输出到标准错误；若发生严重故障（缺少 Chromium、Playwright 无法解析、加载失败），进程将以非零状态退出。

JSON 报告的格式如下：

```json
{ "ok": true, "pdf": "<abs>", "pages": 0, "pngs": [],
  "fonts": { "missing": [], "unused": [], "used": ["..."], "expected": ["..."] },
  "overflow": [ {"index": 0, "overflowY_px": 42, "overflowX_px": 0} ] }
```

**报告判定条件：** 如果 `ok` 为假，或者 `fonts.missing` 不为空（该字体在指定策略下未能解析——`fonts.used` 列出了 Chromium 实际栅格化的字体，请修正字体族名称或选择已安装的本地字体），或者 `overflow` 不为空（内容溢出页面框——请减少内容或调整指定元素 `index` 的布局），则应视渲染失败，并重新编辑 `.src/index.html`。这种逐元素的溢出检测比仅凭肉眼查看 PNG 更加可靠。
通过裁剪内容或让其溢出到下一页来解决溢出问题，绝不能通过压缩设计来处理。正文字号保持在10pt及以上，任何地方不得低于8pt；内容与纸张边缘之间至少留出12mm的间距（即基准CSS中的页面边距，不要为此添加`@page`页边距）；各区块之间应有明显的间隔（3mm以上）。`--require-text-floor`强制执行字体方面的要求：小于8pt的文本将导致检查不通过。如果正文大部分低于10pt，只会发出警告，因此即使检查通过也不能视为合格——应将该警告视为需要重新编辑的信号，恢复字号并裁剪内容，只有当任务书本身要求文档紧凑时才允许保留这种设置。当任务书限定了页数时，内容的分配才是关键：删除最弱的内容或精简措辞，直到在完整字号下也能恰好满足页数要求。

`--require-geometry --gate`会增加对版面几何的检查：从生成的PDF中读取页数和纸张尺寸，并与在打印媒体环境下DOM中测量的页面容器进行核对。由于PDF并不知道哪个容器对应哪一张纸，因此需要同时考虑两者才能识别页面拆分的情况。

必须修复所有“失败”情况：包括页面元素高度超过纸张，以及作者指定的页数与实际生成的页数不符。页面容器的高度必须与它所声明的纸张尺寸相符（A4为`min-height: 297mm`，Letter为`11in`），且使用`box-sizing: border-box`，这样内边距才会包含在高度之内，而不会额外增加高度。

“建议”项则不会导致检查不通过：例如无法测量的版面、作为可见文本出现在页面上的标记，以及正文大部分低于10pt的目标值。如果文档中特意引用代码或标记，则可忽略标记部分；至于正文部分，按上述最低段落高度的要求处理，仅在任务书明确要求文档紧凑时才允许保留。

**验证循环（必选）：**每次渲染后都需运行此流程。若某项检查未通过，请编辑`.src/index.html`，重新执行上述渲染与审计步骤，并再次验证。最多迭代三次；超过三次后，请报告具体失败原因并寻求指导。在验证通过之前，切勿提交最终成果链接。

```sh
PDF="$DIR/$SLUG.pdf"
HTML="$DIR/.src/index.html"
OUT="$DIR/.src/validate"
"/opt/hatch/skills/artifacts/scripts/validate_pdf.sh" "$PDF" "$HTML" "$OUT"
```

`validate_pdf.sh`会通过HTML解析器检查PDF的完整性、页面元数据，以及`<img>`标签和CSS `url(...)`中的data URI嵌入情况，随后使用`pdftoppm`将每一页PDF栅格化为`.src/validate/`目录下的`page-*.png`文件。这些PNG图将是视觉审查的依据，因为它们直接来自PDF的实际字节数据。

在渲染报告和`validate_pdf.sh`均通过之后，**在回复前务必使用`read`工具逐个查看`project_dir/.src/validate/`中的所有PNG文件**（构建任务中使用的目录名为`project_dir`，不一定位于`your_files`下），并逐一确认：图像显示正常，文字清晰且未被裁剪，文字留有足够的边距，跨页时的间距一致，各部分之间留有明显的空白而非紧贴边缘，章节、表格或图表不会在页间出现难看的断开，页眉、页脚及页码符合该类型成果的预期位置，字体正确且未退化为缺失字符方块，不存在意外的空白页或巨大空隙，封面仅在有意全出血时才全出血，图片或图表与相邻内容匹配，引用和参考文献以正常文本形式呈现，未残留任何工具标记或占位符字符串。

凡发现空白或白色图像框、文字与其他元素重叠、或文字溢出页面的PNG，一律拒绝——需修正源文件并重新渲染。缺少的图像属于构建失败，而非外观问题，而且渲染报告未必能捕捉到此类问题，因为一个空框并不会使其容器溢出。
那些 PNG 文件也带有文本内容，因此在这次同样的流程中，从
`/opt/hatch/skills/artifacts/references/prose.md` 读取并回显，并将渲染完美但文本可读性差的页面视为验证失败。文本修复会重新写入
`.src/index.html`，并像布局修复一样，进入相同的渲染与再验证循环。

常见修复：移除空白尾页的分页符，在图表或地图显示过小难以辨认时提高图像的 `max-height`（或直接去掉该属性），通过 `display: flow-root` 或外层容器来解决外边距折叠问题，将封面图片改为 CSS 背景，并用本地已安装的字体族替换缺失的字体。如果图片边缘内容缺失而非图片被截断，原因在于 `object-fit: cover` 导致的裁剪，而非 `max-height` 设置。

## 按产物类型进行的内容检查

仅使用与产物类型匹配的检查项：
- **报告与白皮书：** 包含执行摘要、页码、可读的图表，以及带有表头的表格。
- **数据导出：** 包括数据来源、提取时间、查询/筛选上下文及记录数。抽样核对源数据值与 PDF 中的一致性。
- **发票与收据：** 核实必填的发票字段、两位小数的货币格式，以及小计、税额和总计的计算是否正确。
- **旅游指南：** 包含完整的地址、营业时间、费用、最近的交通信息，以及可选中的电话号码。
- **简历：** 确保文本可选中且阅读顺序符合逻辑。针对 ATS 优化的简历应采用单栏布局。
- **手册：** 包含版本/日期、目录、图注，以及保留缩进的代码块。

> **运行时依赖说明（`render_audit.mjs`）：** 渲染与审计脚本会导入与 Web 产物运行时资产一同打包的 Playwright（位于
> `/opt/hatch/skills/spaces/ts-runtime/dist/node_modules/playwright`），并在存在时启动由 Muse 提供镜像的 Chromium 浏览器（路径为
> `/opt/meta-chromium/chrome`）。若该镜像路径缺失，脚本可能会使用已缓存的 Playwright 自带 Chromium，但不会在运行时安装 Chromium。若解析失败，脚本将以非零状态退出，并输出 JSON 格式的错误信息
> `{ok:false,error}`，指明缺失的部分。