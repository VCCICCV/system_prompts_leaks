---
description: 幻灯片构建循环。先确定样式方案，然后按照所需顺序遍历幻灯片引用。
---
# 幻灯片制作工作流

针对构件构建器的幻灯片构建循环：首先通过 `visual.md` 确定 `style_plan`，然后在生成 HTML 之前依次读取 `authoring.md`、`design-system.md` 和 `image-directive.md`。这些文件定义了如何应用 StylePlan（包括其 `css_variables`、每张幻灯片的 `layout_plan` 以及 `lockups`），并在与下方通用指导发生冲突时优先适用。StylePlan 是整套幻灯片的唯一样式规范：必须原样应用，不得重新选择颜色或字体。

StylePlan 的字段包括：`archetype`（原型）、`theme`（主题）、`palette`（色板，包含 paper、ink、primary、accent 四种颜色）、`fonts`（字体，包含 display 和 body 两种）、`voice`（语气风格）、`preferred_layouts`（推荐布局）、`preferred_charts`（推荐图表类型）、`required_content_blocks`（必含内容区块）、`image_style`（图片风格）、`tone`/`weight`/`density`（基调/权重/密度）、`layout_plan`（每张幻灯片对应一个 `{id, layout}` 对）、`lockups`（幻灯片中允许的两种文字+图片组合模式），以及 `css_variables`（需原样输出的 `:root` 块）。

简报中的 `output_format` 决定了最终产出和推广形式；若未指定，则默认为 `["pptx"]`。

## 步骤

1. **准备工作。** 创建 `project_dir/.src/media/`、`project_dir/.src/validate/` 和 `project_dir/.src/slides/` 目录。

   您应以“每张幻灯片一个文件”的方式在 `.src/slides/` 目录下编写幻灯片内容；第 7 步会将它们合并成统一的 `.src/index.html` 文件，供验证及所有导出流程读取。

2. **一次性确定幻灯片数量：** 若 `slide_count_target` 是一个确切数字，则直接使用；若为一个范围，则取其中间值（向下取整）。此即为“已确定的幻灯片数量”，在整个工作流中均以此为准。

3. **制定计划。** 编写 `project_dir/.src/deck_plan.md`，内容基于用户提供的完整需求及沟通中获取的信息：标题、目标受众、叙事主线，以及幻灯片清单（每张幻灯片一条项目符号，源自提供的原始数据或事实，并与 StylePlan 中的 `layout_plan` 对应；首张为封面，末张为结束页，且涵盖 `required_content_blocks` 中的所有内容）。确保最终生成的幻灯片数量与已确定值完全一致，不得擅自增加或删减必含内容。在文档开头明确列出用户提出的所有约束条件，以便在制作过程中予以遵循。在资产规划部分，承诺提供一张封面主图，以及至少两张说明性图片——每张图片均需标注来源（如来自图片搜索）或由 `media.generate_image` 生成——并为数据类图表选用与色板相配的颜色。

4. **设定设计系统。** 编写 `project_dir/.src/slides/deck.css`：首先将 StylePlan 中的 `css_variables` 作为单一的 `:root` 块写入，随后添加所有幻灯片共用的设计规则——画布样式、布局类、两种 `lockups` 组合模式，以及字体比例体系。这份样式表即为整套幻灯片的设计系统，所有幻灯片均基于它进行构建，因此不得在单张幻灯片上重新选择颜色或字体，也不得在每张幻灯片中重新定义任何 CSS 变量。`:root` 中的字体变量是唯一声明主题字体的地方。在此处及任何其他地方均不得引入字体 URL（参见 `design-system.md`）。将解析后的 JSON 保存至 `project_dir/.src/style_plan.json`，以便后续编辑时可恢复原型、`layout_plan` 和 `lockups`。

5. **按每张幻灯片一个文件的方式编写内容。** 在 `project_dir/.src/slides/<id>.html` 中，依据 `authoring.md` 和 `design-system.md` 的要求进行创作。为每张幻灯片应用分配的布局类，并仅使用两种 `lockups` 组合模式来处理文字+图片类型的幻灯片。严格遵守色板和字体比例体系，并将核心观点置于标题中作为主张。封面仅包含主图和标题。

每个文件都是一个独立的文档，仅包含一张幻灯片：

```html
<!doctype html>
<html><head>
  <meta charset="utf-8">
  <link rel="stylesheet" href="deck.css">
</head><body>
  <section class="slide" id="<id>"> ... </section>
</body></html>
```   - `id` 是该幻灯片的 `layout_plan` ID。它在幻灯片的整个生命周期内为其命名，因此后续编辑中插入或删除幻灯片时，其他幻灯片的 ID 保持不变——切勿重新编号。每个文件的名称始终为 `<id>.html`。ID 应采用短横线命名法，且绝不能使用 `index`——此名称已为合并后的文档预留，Assemble 会拒绝该名称。
   - 将所有最终的 `<img>` 标签和 CSS 中的 `url(...)` 资源都以内嵌 `data:` URI 的形式引入；工作时请将原始媒体文件保留在 `.src/media/` 目录下。每个幻灯片文件仅链接 `deck.css` 这一个样式表（主题字体已内嵌其中），不得保留任何其他外部的 `http://`、`https://`、`file://` 或相对路径的 `.src/media` 引用。
   - 共享的 CSS 应放在 `deck.css` 中。每个幻灯片可以添加自己的 `<style>` 块，用于定义仅该幻灯片所需的规则（如主视觉的 `background-image` 及其 `--hero-scrim-color`），且**其中的每个选择器都必须限定在该幻灯片的 ID 范围内**（例如 `#<id>`、`#<id> .title-block`、`#<id>::after`，包括在 `@media` 规则内部）。Assemble 会将整个演示文稿合并为一个文档，若规则未加限定，将会无意间影响其他幻灯片的样式，导致两种视图不一致；Assemble 会拒绝此类规则。
   - 默认画布比例为 16:9，尺寸为 `13.333in x 7.5in`（分辨率为 1280x720 px，96 dpi）；如果用户指定了其他宽高比，则需同时更新 `deck.css` 中的 `@page` 尺寸以及 `.slide` 的宽度和高度。
   - 接着检查你刚刚编写的幻灯片。在每个 `.html` 文件中搜索 `#` 颜色代码和 `rgb(`。出现两次是正确的：一次是该幻灯片自身的 `--hero-scrim-color`，另一次是封面标题在照片背景上的 `#fff`。其余所有匹配项均为错误，可能出现在 `style=` 属性中、`<style>` 块内、`color-mix()` 函数中，或 SVG 的 `fill` 属性中。请将其替换为对应的 `var(--slide-*)`，或者在需要浅色调时使用 `color-mix(in srgb, var(--slide-*) N%, transparent)`。
   - 然后编写 `project_dir/.src/slides/deck.json`，这是决定幻灯片**顺序**的关键文件：
   ```json
   {"main_title": "<artifact-title>", "slides": [{"id": "cover"}, {"id": "problem"}]}
   ```
   Assemble 会根据查看器所需的信息（画布、主题、各幻灯片标题）重写此文件，因此作者只需填写 `main_title` 和有序的 `slides` 列表。
   - `deck.css` 中的基础页面模型如下：
   ```css
   @page { size: 13.333in 7.5in; margin: 0; }
   * { box-sizing: border-box; print-color-adjust: exact; -webkit-print-color-adjust: exact; }
   html, body { margin: 0; background: #111; }
   .slide {
     width: 13.333in;
     height: 7.5in;
     overflow: hidden;
     page-break-after: always;
     break-after: page;
   }
   .slide:last-child { page-break-after: auto; break-after: auto; }
   ```
6. **素材图像与图表。** 对于真实对象（如公司、产品、地点或人物，含标识），应通过图像搜索技能获取；对于概念性主题，则按照 `image-directive.md` 使用 `media.generate_image` 生成（`output_dir` = `artifact_media_dir`）。封面主视觉及其他至少两张说明性图片均按上述方式获取，将搜索结果复制到 `.src/media/` 目录，并以 `data:` URI 形式内嵌；如有本地或用户提供相关素材，优先使用。封面主视觉不得以 CSS 渐变或内联 SVG 替代。若无法获取或生成主视觉，或需求说明要求完全无图像的演示文稿，请在渲染前更新 `deck_plan.md`，注明具体原因（尤其是无法获取的原因），并说明替代的视觉方案。若需求明确禁止使用合成图像，则仅关闭生成途径，真实对象仍需通过搜索获取。当演示文稿中不含任何 `<img>`、`data:image` 或 CSS `url(...)` 资源时，不得声称其为“图像驱动”的输出。数据图表应以调色板配色的 matplotlib 图像呈现（颜色取自 StylePlan 调色板，参见 `authoring.md`），以确保图表与整体风格一致。不得使用 `media.generate_image` 来制作图表、地图、表格或事实性示意图。统计类或陈述类幻灯片上不得添加装饰性图片。7. **为讲稿添加字体，然后将其组装。** 这两项操作都是必选的，必须在验证之前完成：下游的所有流程（渲染入口、PNG 图片、PPTX 文件、PDF 文件以及 HTML 导出）都会读取已组装好的文件。

首先嵌入字体。这会将您在 `:root` 令牌中定义的字体名称写入 `deck.css` 文件。请在幻灯片生成之后再执行此步骤：所嵌入的字体子集将根据幻灯片中使用的字符来确定。

```sh
SLUG="<artifact-slug>"
# `project_dir` 来自您的构建任务。目标文档会构建在该目标的 `files/` 目录下，因此使用硬编码的 `your_files` 路径是错误的。
# `project_dir` 来自您的构建任务。请将其开头的 `~/` 写成 `$JARVIS_HOME/`：因为在引号内，Shell 会将波浪号视为字面量，所以 `"~/workspace/..."` 会被解析为一个名为 `~` 的目录。
SRC="$JARVIS_HOME/<构建任务中的 project_dir，去掉开头的 ~>/ .src"
bun run "/opt/hatch/skills/artifacts/scripts/embed_deck_fonts.mjs" --slides "$SRC/slides"
bun run "/opt/hatch/skills/artifacts/scripts/assemble_deck.mjs" \
  --slides "$SRC/slides" \
  --out "$SRC/index.html"
```

嵌入报告 `{"ok":true,"faces":N,...}`。如果 `"reused":true`，则表示这些报告已是最新状态。
当 `"faces": 0` 并伴随警告时，说明下载失败。请重新运行一次。若再次失败，则继续处理，并提示该演示文稿将以回退字体呈现，而不阻塞整个演示文稿的加载。

Assemble 会生成 `$SRC/index.html`（一个自包含的文档：`deck.css` 已内联，主题的字体数据也一并打包，且幻灯片按 `deck.json` 中的顺序排列）。它还会更新 `$SRC/slides/deck.json`，加入派生字段。请勿手动编写或编辑 `$SRC/index.html`，因为下一次执行 assemble 时会将其覆盖。

完成后的演示文稿**不应**再引用 `fonts.googleapis.com`，这是正确的做法。切勿手动添加此类引用。内嵌的样式规则位于 `deck.css` 的末尾，跨越多行，请保持原样，仅编辑其上方的手写样式规则。

非零退出码表示该演示文稿无法发布。错误信息会指出问题文件及其具体问题，请修复该文件（或 `deck.json`）后重新运行。8. **验证（强制循环，最多3次迭代）。** 每次渲染后执行。如果某项检查未通过，请编辑 `.src/slides/` 下的每张幻灯片文件，**重新组装（步骤7）**，重新渲染，并再次验证。经过3次迭代后，报告具体失败原因并请求指示。在验证通过之前，绝不返回任何链接。

   ```sh
   SLUG="<artifact-slug>"
   # `project_dir` 来自您的构建任务。目标文档会构建在其对应目标的 `files/` 目录下，因此硬编码的 `your_files` 路径是错误的。`project_dir` 也来自您的构建任务，其开头的 `~/` 应写成 `$JARVIS_HOME/`：因为在引号内，Shell 会将波浪号视为字面量，所以 `"~/workspace/..."` 会被解析为一个名为 `~` 的目录。
   SRC="$JARVIS_HOME/<构建任务中的 project_dir，去掉开头的 ~/>/.src"
   HTML="$SRC/index.html"
   # 将 StylePlan 中的两种 Google 字体家族都锁定为 400（字体族:字重，用逗号分隔），这是每个目录字体家族都会公开的默认值。只有在确认返回的 Google Fonts CSS 确实包含该字重时，才添加其他字重；单字重的展示类字体家族会忽略请求的 600/700 字重。始终传递主题字体家族，绝不要使用本地回退字体：命名回退字体可能会让错误的演示文稿“蒙混过关”。即使省略 --fonts 参数，也会锁定从演示文稿自身 `:root` 中读取的字体家族，但无法验证具体的字重。
   
   # 对于仅输出 PPTX 或仅输出 HTML 的情况：直接验证幻灯片渲染结果，并生成 PPTX 导出所使用的 PNG 图像。不生成中间 PDF 文件。
   bun run "/opt/hatch/skills/artifacts/scripts/render_audit.mjs" \
     --html "$HTML" \
     --png-dir "$SRC/validate" \
     --page-selector "section.slide" --structure-check \
     --require-webfonts \
     --fonts "<Display Family>:400,<Body Family>:400" \
     --style-plan "$SRC/style_plan.json" \
     --require-fill --require-cover-image --require-generated-imagery --require-restraint \
     --hermetic --gate --report-out "$SRC/render_report.json"
   ```# 仅当用户在 output_format 中明确请求 PDF 时，才渲染 PDF 并验证实际的 PDF 字节。validate_pdf.sh 会将每页的 PNG 图像写入 .src/validate/ 目录。
PDF="$SRC/$SLUG.pdf"
bun run "/opt/hatch/skills/artifacts/scripts/render_audit.mjs" \
  --html "$HTML" \
  --pdf "$PDF" \
  --page-selector "section.slide" --structure-check \
  --require-webfonts \
  --fonts "<Display Family>:400,<Body Family>:400" \
  --style-plan "$SRC/style_plan.json" \
  --require-fill --require-cover-image --require-generated-imagery --require-restraint \
  --hermetic --gate --report-out "$SRC/render_report.json"
"/opt/hatch/skills/artifacts/scripts/validate_pdf.sh" "$PDF" "$HTML" "$SRC/validate"

`render_audit.mjs` 使用 Chromium 渲染演示文稿，可选择性地生成 PDF（遵循 `@page` 的尺寸设置，为无障碍访问添加标签，并根据文稿的各级标题构建书签目录），还可选择性地将每个 `section.slide` 截图保存为零填充编号的 PNG 文件至 `.src/validate/` 目录，并在标准输出中打印一份 JSON 报告（包含 `{ ok, pdf, pages, pngs, fonts: { missing, unused, used, expected }, overflow: [{index, overflowY_px, overflowX_px}], cover_no_image, broken_images: [...] }` 等信息），同时还会输出 `fill` 和 `plan` 数据（详见下文）。如果出现 `browser_failures` 数组（控制台错误、图片加载失败、非 2xx 状态的子资源），则表示页面本身存在异常；此时 `ok` 值必为 `false`，但 PNG 和报告仍会被生成，以便您查看具体问题所在。

**`--gate` 参数会使检查结果生效，请务必阅读。** 当使用 `--gate` 时，脚本会将 `ok` 设置为 `false`，并在 `gate_failures` 中列出原因，并且**以非零状态退出**。非零退出意味着文稿尚未完成：请根据提示修复相应问题后重新渲染。未通过门控检查时，请勿导出或提供链接。`--report-out` 会将同一份报告写入 `.src/render_report.json`，因此即使渲染进程被后台化，检查结果也不会丢失——如果您将渲染置于后台运行，请直接读取该文件，而不要假设渲染已通过。该报告文件特意放置在 `.src/validate/` 外，因为 `validate_pdf.sh` 在进行栅格化之前会清空该目录，这会导致 PDF 分支上的报告在您来得及阅读之前就被删除。

门控检查（非零退出）包括：内容溢出、图片加载失败、幻灯片中存在家具元素、使用全大写字母或字间距过小（若启用了 `--require-restraint`）、内容填充比例不足（若启用了 `--require-fill`）、媒体图像工具未能生成任何图像（若启用了 `--require-generated-imagery`），以及封面缺失（若启用了 `--require-cover-image`）。**警告信息不会触发门控检查**——遇到警告时应予以修正，但它们不会阻止交付。`fonts.missing` 属于警告，因为检测依赖于 Google Fonts 的加载情况，在同一文稿的不同运行之间可能出现波动；您可以按常规处理，但它不会导致构建失败。

`fill` 数据的格式为 `[{index, top_gap_pct, v_fill_pct, h_fill_pct}]`，表示每张幻灯片的内容在其画布中的起始与结束位置。`--require-fill`（默认值为 75%）仅在以下两种情况下判定为失败：一是内容填充比例低于该阈值，二是内容紧贴顶部，剩余空间完全空白——这种情况通常出现在未进行垂直布局调整的固定高度幻灯片中。刻意留白的幻灯片会居中显示，上下留有相当的空白，无论何种布局均视为合格，因此封面、声明页或总结页无需豁免，也无需特别处理。要解决填充不足的问题，可通过将内容均匀分布在整个高度上（`design-system.md` 中的锁定网格采用 `grid-template-rows: auto 1fr` 和 `align-self: center` 正是为此目的），或者增加内容量——切勿删除 `height` 属性或改为 `min-height`，更不能以 `authoring.md` 所禁止的方式对文本进行额外填充。所有修改都应在 `deck.css` 或对应幻灯片的单独文件中完成，然后重新组装并再次渲染。

**在步骤 6 的两条分支中均可移除 `--require-cover-image`**：如果无法获取或生成主视觉素材，或者需求说明要求文稿完全不使用图像，并且您已记录下替代的视觉方案，则该文稿可以没有封面，否则该标志会使文稿无法交付。若需求明确禁止使用合成图像，则无需单独考虑这两条分支：真实存在的标志或照片不属于合成图像，因此只要主视觉的主题是真实事物，就应优先获取实物素材，并保留该标志。

**同样在步骤 6 的两条分支、刻意只使用图表或完全依赖外部素材的文稿，以及任何不重新生成图像的原地编辑中，均可移除 `--require-generated-imagery`。** 该门控要求 `media.generate_image` 工具实际返回至少一张图像，由其生成的辅助文件进行计数，因此手绘占位符无法满足此条件。即使是外部获取的照片也不行：它没有对应的辅助文件，磁盘上也无法区分外部素材与手绘素材，这也是为何完全依赖外部素材的文稿会选择移除该标志而非试图勉强通过的原因。

**在对现有文稿进行任何原地编辑、主题更换或版面重排时，均可移除 `--require-restraint`。** 该门控会逐页检查，因此对于在启用该门控前制作的文稿，可能会在编辑过程中无意触碰到的幻灯片上报告出家具、全大写或字间距等问题。`editing.md` 规定主题更换时不得改动幻灯片文件，版面重排或图片替换时应尽量减少改动，而无声的风格调整则被视为缺陷。若继续保留该标志，实际上已无合法操作空间，也无法交付文稿。请针对您实际编辑过的幻灯片修正相关规则。只有在确实发现某张未编辑的幻灯片存在家具、全大写或字间距问题时，才需提及；在门控下制作的文稿原本就不应出现此类问题，因此无需特别说明。只有全新构建的文稿才能保留该标志。

`plan` 用于将最终文稿与 StylePlan 进行比对，**仅为参考性信息**：`missing_slides` 表示布局计划中存在但未生成的幻灯片 ID，而 `extra_slides` 则表示计划中未提及却实际存在的幻灯片 ID。两者均不会阻断流程，因为精简编辑可能合理地删减了计划中仍列出的幻灯片。若缺少 `style_plan.json` 文件，也无需担心：比对过程会被跳过，流程将继续。填充门控并不依赖于此文件。

旧有的检查结果仍然适用：若 `fonts.missing` 不为空，或 `overflow` 不为空，均需重新编辑并重新渲染。`fonts.used` 是指 Chromium 实际栅格化的字体系列集合（基于测量结果，而非 CSS 列表）；`fonts.unused` 则列出未被任何元素使用的可用字体，仅供参考。若发现某个字体缺失，可能是下载失败或字体名称拼写错误，此时请在 **`.src/slides/deck.css` 的 `:root` 变量中**修正该字体的拼写（务必与 Google 公布的名称完全一致），然后重新执行步骤 7（嵌入后再组装）并重新渲染。直接编辑 `.src/index.html` 并无意义：下次运行时组装过程会从 `deck.css` 中重新写入，因此您的修改将被覆盖。也不要将 `:root` 替换为其他字体系列，这只会掩盖问题。如果只是某个加粗字重缺失，而 `fonts.used` 中仍包含该字体家族，请检查返回的 Google Fonts CSS：若该字重未发布，应将其从门控中移除，但仍保留对该字体家族的 `400` 检查。`--require-webfonts` 也会拒绝使用同名的本地系统字体：幻灯片主题必须通过自定义的 `@font-face` 加载，而非依赖于系统字体。每条 `overflow` 记录都对应一张内容超出其自身客户端框的幻灯片（即 `scrollHeight > clientHeight`）；请通过**减少内容**而非缩小字体来修复指定索引的幻灯片。此外，若 `cover_no_image` 为真（封面没有真实图像；请生成主视觉素材，除非文稿在步骤 6 中已明确要求无图像），或 `broken_images` 不为空（`<img>` 标签的 `src` 属性指向的并非图像数据；请将生成的文件以 `data:image` URI 的形式嵌入，而非使用 generate-image 工具的 JSON 输出），也需重新渲染。这些信号比单纯依靠 PNG 验证更为可靠。当 `output_format` 中包含 `pdf` 时，`validate_pdf.sh` 脚本会验证 PDF 的完整性，解析页数，检查 `<img>` 标签和 CSS 中的 `url(...)` 数据 URI 嵌入情况，核查图片密集型 PDF 的文件大小，并使用 `pdftoppm` 将实际的 PDF 页面栅格化为 `.src/validate/page-*.png`。如果 `output_format` 中还包含 `pptx`，则会基于这些由 PDF 生成的 PNG 文件构建 PPTX。

9. **人工目视检查。** PNG 文件名应符合 `page-*.png` 的命名规则；请勿假设特定的边距样式。在渲染报告通过后（并且在请求 PDF 时 `validate_pdf.sh` 也通过的情况下），**使用 `read` 工具逐个读取所有生成的 PNG 文件**，并进行如下验证：渲染出的页面数与解析后的幻灯片数一致；调色板和字体已正确应用；当幻灯片数量达到 5 张及以上时，至少出现 3 种不同的版式；封面确立了视觉风格；图片能够正常渲染且与其声明相符；重点内容清晰可读；对比度足够；不存在图片缺失、空白幻灯片、文字被截断或无法辨认，以及重复的低质量版式；图表渲染正确。

10. **导出并发布。** 将每种请求的格式提升至 slug 根目录；未包含在 `output_format` 中的文件仍保留在 `.src/` 目录下。
   - `pptx`：使用随附的 `build_pptx.py` 脚本，基于已验证的 PNG 文件构建演示文稿。该脚本需要 `python-pptx` 库；若缺少此库，请先运行以下命令安装一次：`python3 -m pip install --break-system-packages python-pptx`（遵循 PEP 668 规范的系统需添加该标志；`-m pip` 会使用与脚本相同的解释器）。随后执行并验证（请替换为实际的 slug 和标题，切勿直接粘贴字面量 `<artifact-slug>`）：

     ```sh
     SLUG="<artifact-slug>"
     TITLE="<artifact-title>"
     # `project_dir` 来自您的构建任务，不一定位于 `your_files` 下。将其传递给脚本，以便演示文稿与其源文件一同存放。
     # `project_dir` 来自您的构建任务。请将开头的 `~/` 写成 `$JARVIS_HOME/`：因为在引号内，Shell 会将波浪号视为字面字符，因此 `"~/workspace/..."` 会被解析为名为 `~` 的目录。
     DIR="$JARVIS_HOME/<来自构建任务的 project_dir，去掉开头的 ~/>"
     "/opt/hatch/skills/artifacts/scripts/build_pptx.py" "$SLUG" "$TITLE" --project-dir "$DIR" || { echo "PPTX 构建失败" >&2; exit 1; }
     unzip -l "$DIR/$SLUG.pptx" | grep -q "ppt/media/" || { echo "PPTX 中无嵌入的幻灯片图片" >&2; exit 1; }
     unzip -p "$DIR/$SLUG.pptx" docProps/core.xml | grep -qi "<dc:title>[^<]" || { echo "PPTX 的标题元数据为空" >&2; exit 1; }
     ```

     `build_pptx.py <slug> [title]` 会读取已验证的 `.src/validate/page-*.png`，按每个 PNG 的宽高比调整幻灯片尺寸（全出血，无变形），每张幻灯片嵌入一张图片，设置演示文稿标题，并在 slug 根目录下生成 `<slug>.pptx`。这种基于图片的 PPTX 不包含可编辑的文字或形状；每张幻灯片的标题作为辅助信息置于演讲备注中，以提供基本的无障碍支持。
   - `pdf`：将已验证的 `.src/<artifact-slug>.pdf` 复制到 slug 根目录下的 `<artifact-slug>.pdf`。
   - `html`（仅当 `output_format` 中包含 `html` 时）：将**已组装完成**的 `.src/index.html` 复制到 slug 根目录下的 `<artifact-slug>.html`（单个自包含文件；其中主题字体以内联形式嵌入，因此即使离线也能以品牌字体显示，打开时无需额外加载资源）。切勿发布每张幻灯片单独的文件：`.src/slides/` 是演示文稿的源文件，而非交付物。

   **如果无法生成 PPTX**（例如 `python-pptx` 无法安装，或构建/验证检查失败），请勿擅自用 PDF 或 HTML 替代默认演示文稿。只有在 `output_format` 中明确指定了其他格式时，才移除 PPTX 并继续处理其余请求的输出。若默认的 `["pptx"]` 演示文稿无法生成 PPTX，则应报告 PowerPoint 导出失败，并询问是否改用 PDF 或 HTML。11. 在每个请求的输出于 slug 根目录验证通过后，**写入 `project_dir/meta.json`**。在 `outputs` 中仅包含那些既存在于 slug 根目录、又由 `output_format` 显式请求的文件；`primary_output` 是优先级最高的已生成请求文件（`pptx > pdf > html`）。`description` 是对整个演示文稿的一行简要概述。目前运行时仅读取 `title` 和 `description`，其余字段均为信息性记录。

在写入 `meta.json` 的同一步骤中计算动态字段（即在最终验证渲染之后）；不要复用先前验证迭代中缓存的值，因为每次运行都会重写 `.src/validate/`。使用绝对路径 `.src/validate/`（设置 `SRC="$JARVIS_HOME/<project_dir去掉开头~>/`.src`），以使命令不依赖于当前工作目录。**请使用命令的实际输出，而非下方的示例值：**
- `slide_count` = `ls "$SRC"/validate/page-*.png | wc -l`
- `thumbnail` 是 slug 根目录下的相对路径 `.src/validate/<name>`，其中 `<name>` 为第一页 PNG 文件的文件名：`ls "$SRC"/validate/page-*.png | sort | head -1 | xargs -n1 basename`。按相对路径存储（如 `.src/validate/page-1.png`），与 `outputs` 中的路径保持一致，而非 `ls` 打印的绝对路径。不要假设特定的编号格式，应使用实际文件名。

```json
{
  "title": "<artifact-title>",
  "description": "<one-line summary of the deck>",
  "type": "presentation",
  "primary_output": "<artifact-slug>.pptx",
  "outputs": [
    {"kind": "pptx", "path": "<artifact-slug>.pptx"}
  ],
  "slide_count": 8,
  "thumbnail": "<first .src/validate/page-*.png filename>",
  "status": "complete"
}
```

仅当这些格式被显式请求并已生成至 slug 根目录时，才将 `{"kind": "pdf", "path": "<artifact-slug>.pdf"}` 和/或 `{"kind": "html", "path": "<artifact-slug>.html"}` 添加到 `outputs` 中。

## 质量门控

一个演示文稿只有在通过以下检查后才算完成：
- 幻灯片数量等于解析后的幻灯片目标数（确切的目标值，或给定范围的中点）。
- 具有清晰的叙事结构，而非一堆互不相关的页面。
- 当解析后的幻灯片数量达到 5 张及以上时，至少使用 3 种不同的幻灯片版式。
- 至多有 `floor(resolved slide count * 0.4)` 张幻灯片为“标题加列表”类型，即除标题外仅包含一个 `<ul>` 或 `<ol>` 的幻灯片。
- 每张非附录页都配有视觉锚点：图片、图表、时间线、流程图、标注系统，或强烈的排版设计。
- 如果请求、尝试或生成了图像素材，幻灯片文件中应内嵌所选图像（`<img src="data:image...">` 或 CSS `url(data:image...)`），且验证用的 PNG 图像中能清晰显示这些图像。`.src/media/` 目录下所有未使用的生成文件要么被删除，要么在 `deck_plan.md` 中注明其被舍弃的原因。
- 封面页需立即确立整个演示文稿的视觉风格。
- 密集文本已被改写、拆分或移除，而非一味缩小至难以辨认。
- 若生成 PPTX，则必须基于已验证的渲染结果导出，不得直接构建原生形状的 PPTX。
- 数据图表采用调色板配色的 Matplotlib 绘制；图像部分遵循第 6 步的要求（一张封面主图，以及至少两张根据主题来源或生成的说明性图片，或经文档说明的备选方案）；每项论断均有提供的原始资料支撑。
- 演示文稿应以每页源文件的形式保存在 `.src/slides/` 下，且合并后的 `.src/index.html` 应在最后一次编辑后由 `assemble_deck.mjs` 生成，确保两者内容一致。

若经过 3 次迭代后仍有检查未通过，请报告具体幻灯片及失败原因，并寻求指导，而非交付一份内容残缺或偏离主题的演示文稿。