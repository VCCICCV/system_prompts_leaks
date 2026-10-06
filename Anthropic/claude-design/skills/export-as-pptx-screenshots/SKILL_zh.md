---
name: export-as-pptx-screenshots
description: "平面图像——像素完美但不可编辑"
user-invocable: true
---
# 导出为 PPTX（截图模式）

将 HTML 幻灯片演示文稿导出为 `.pptx` 文件，其中包含全幅 PNG 图像。像素级精确，不可编辑。只需一次 `gen_pptx` 工具调用。

### 步骤

1. 使用 `show_to_user` 向用户展示演示文稿。
2. 调用 `gen_pptx`：

```jsonc
{
  "mode": "screenshots",
  "width": 1920, "height": 1080,
  "slides": [
    { "showJs": "goToSlide(0)", "selector": "body" },  // 截图模式下该选择器虽未使用，但为必填项
    { "showJs": "goToSlide(1)", "selector": "body" }
  ],
  "hideSelectors": [".nav", ".progress"],
  // 如果演示文稿的幻灯片被包裹在带有 transform: scale() 的容器中，请在此处指定该容器的选择器，
  // 以确保演示文稿在锁定的 iframe 中按指定的宽度和高度显示。
  "resetTransformSelector": ".slide-container",
  "filename": "my-deck"
}
```

仅当用户请求导出为 Google Slides 时，还需传入 `"offer_google_slides": true`：导出对话框会新增“发送到 Google Slides”按钮，且仅在用户点击该按钮时才会执行上传操作。

`slides[].delay` 的默认值为 600 毫秒——如果过渡效果较慢，可适当调大。

#### 如果演示文稿使用 `<deck-stage>` 启动组件

- `resetTransformSelector: "deck-stage"` — 与可编辑模式相同；该组件会移除其影子 DOM 中的 `transform: scale()` 样式，使幻灯片能够充满锁定的 iframe。
- `slides[N].showJs`: `"document.querySelector('deck-stage').goTo(N)"` — 索引从 0 开始，因此第 1 张幻灯片应写为 `goTo(0)`。
- `hideSelectors` 不再需要——因为覆盖层和点击区域位于影子 DOM 中，不会被截取。

### 验证

验证标志与可编辑模式相同。需留意 `duplicate_adjacent`（`showJs` 未成功导航）以及 `reset_selector_miss` / `slide_size_mismatch`（`resetTransformSelector` 未匹配到任何元素，或未正确调整至指定宽高）等错误。

演讲者备注会自动附加到 `#speaker-notes` 元素，并在页面重新加载后生效。