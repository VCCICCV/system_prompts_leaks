---
name: export-as-pptx-editable
description: "原生文本与形状——可在 PowerPoint 中编辑"
user-invocable: true
---
# 导出为 PPTX（可编辑）

将 HTML 幻灯片文稿导出为包含原生 PowerPoint 对象的 `.pptx` 文件（文本、形状、图片均可编辑）。一次调用 `gen_pptx` 工具即可完成：捕获、字体处理、生成和下载。

### 操作步骤

1. **熟悉文稿。** 你很可能就是作者。如果不是，使用 `read_file` 读取 HTML 文件，确认：幻灯片的选择器、导航方式（函数名？类切换？）、使用的字体，以及是否存在缩放容器。
2. **通过 `show_to_user` 将文稿展示给用户**，使其出现在用户的预览中。
3. **调用 `gen_pptx`**，并传入以下参数。
4. **查看结果中的校验标志**，判断是否需要重试。

### `gen_pptx` 输入参数

```jsonc
{
  "width": 1920, "height": 1080,   // CSS 像素 — 与文稿的幻灯片尺寸一致
  "slides": [                      // 每张幻灯片一条记录，按顺序排列
    { "showJs": "goToSlide(0)", "selector": ".slide.active" },
    { "showJs": "goToSlide(1)", "selector": ".slide.active" }
    // 对于所有幻灯片同时存在于 DOM 中且无需导航的文稿：
    //   { "selector": ".slide:nth-child(1)" }, { "selector": ".slide:nth-child(2)" }
  ],
  "hideSelectors": [".nav", ".progress", "[data-omelette-chrome]", "[data-noncommentable]"],
  // 如果文稿的幻灯片被包裹在带有 transform:scale() 的容器中，请在此处指定该容器的选择器。
  // `gen_pptx` 会清除该容器的变换，并强制设置其宽度和高度。
  "resetTransformSelector": ".slide-container",
  // 字体处理 — 根据文稿底部的指示选择一种策略。
  // 替换操作在捕获之前进行，以确保布局正确重排。
  "googleFontImports": ["Poppins", "Lora"],
  "fontSwaps": [{ "from": "BrandSans", "to": "Poppins" }],
  // 或者使用 `fontSwaps: [{from:"BrandSans", to:"Arial"}]` 来替换为网页安全字体。
  // 或者两者都省略，以保留品牌字体不变。
  "filename": "my-deck"
}
```

如果用户明确要求导出为 Google 幻灯片，且仅在这种情况下，还需传入 `"offer_google_slides": true`：导出对话框将新增“发送到 Google 幻灯片”按钮，只有用户点击该按钮时才会执行上传操作。

`slides[].showJs` 会在 iframe 内以同步表达式的形式运行——请勿使用 `await`。如果你的文稿导航函数是异步的，也请直接调用，不要加 `await`；每张幻灯片的默认延迟时间为 600 毫秒，足以覆盖过渡效果。对于 CSS 过渡时间较长的文稿，可适当增加延迟。

#### 如果文稿使用了 `<deck-stage>` 启动组件

- `resetTransformSelector: "deck-stage"` — 导出工具会为其设置 `noscale` 属性，组件会监听该属性的变化，并移除其影子 DOM 中的 `transform: scale()` 样式。无法通过其他方式访问缩放后的画布。
- `slides[N].showJs`: `"document.querySelector('deck-stage').goTo(N)"` — 索引从 0 开始，因此第 1 张幻灯片应写为 `goTo(0)`。
- `slides[N].selector`: `"deck-stage > [data-deck-active]"`。
- `hideSelectors` 不再需要——因为遮罩层和点击区域位于影子 DOM 中，不会被捕获。

### 演讲备注

演讲备注会自动从 `<script type="application/json" id="speaker-notes">` 中读取，并按索引关联。无需手动传递。

### 校验标志

结果中会列出一些标志。**这些只是警告，而非错误**——请逐条阅读消息，并判断对于当前文稿而言是否属于正常情况：

- `duplicate_adjacent` / `duplicate_majority` — 滑动条被完全相同地捕获。几乎总是意味着 `showJs` 没有正确导航。请检查函数名，尝试增加 `delay` 时间，或确认幻灯片是使用 0 索引还是 1 索引。
- `slide_size_mismatch` — 捕获的矩形区域宽高与实际不符。选择器可能匹配到了外层容器，或者需要指定一个 `resetTransformSelector`。
- `notes_uniform_nonempty` — 所有的演讲者备注内容都是同一个字符串。很可能是占位符，如果是有意为之则无需处理。
- `notes_count_mismatch` — #speaker-notes 的数量与幻灯片数量不一致。备注是按索引绑定的，因此末尾部分会出错。
- `no_speaker_notes` — 幻灯片中没有 #speaker-notes 标签。如果没有备注，则属于正常情况。
- `fonts_timeout` — fonts.ready 超时超过 8 秒。字体 URL 可能无法访问。
- `font_swap_failed` — 一个或多个 `fontSwaps` 目标未加载成功（字体族名称拼写错误，或 Google Fonts 未提供该字体），导致在文件名仍显示替换字体的情况下，页面使用了回退字体进行排版。请使用正确的或不同的字体族重试，或回退到网页安全字体。无论后续如何处理，都应明确告知用户哪些字体未能应用——例如：“请注意：导出时 Poppins 字体未能加载，因此使用了替代字体，文本换行可能会有所不同。是否要尝试其他字体？”
- `images_failed` — 图像在捕获前未能解码。通常是 404 错误或跨域问题。
- `reset_selector_miss` — 您指定的 `resetTransformSelector` 没有匹配到任何元素。

如果这些标志提示确实存在问题，请修正输入并重试；如果属于预期情况（如确实没有备注、两张幻灯片完全相同等），则只需告知用户下载已触发，并继续下一步操作。

**与用户沟通时关于标志的说明：** 这些名称和信息仅供内部诊断之用，切勿原样转述给用户。如果一切正常，无需提及校验过程，直接确认下载完成即可。如果确实存在问题，应以通俗易懂的语言描述，避免使用标志名称或技术细节——例如，不要说“我收到了 no_speaker_notes 标志”，而应说“演讲者备注可能未正确导出”；也不要引用 `duplicate_adjacent`，而应说“可能有几张幻灯片被重复捕获了，我将修复导航后重试”。

捕获完成后页面会自动刷新，DOM 的变更（如隐藏浏览器控件、字体替换等）都会被还原。

### 字体策略

请阅读本提示末尾的指令，并将其转化为相应的输入参数：

| 指令 | 输入 |
|---|---|
| 使用品牌自有字体 | 忽略 `googleFontImports` 和 `fontSwaps` |
| 替代为网页安全字体 | `fontSwaps: [{from:"EachCustomFont", to:"Arial"}]`（衬线字体可替为 Georgia，等宽字体可替为 Courier New） |
| 替代为 Google Fonts 字体 | `googleFontImports: ["Poppins","Lora"]` + `fontSwaps: [{from:"EachCustomFont", to:"Poppins"}]` |

系统字体（Arial、Helvetica、Georgia、Times、Courier、sans-serif 等）无需更改。