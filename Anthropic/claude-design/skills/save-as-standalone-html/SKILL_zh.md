---
name: save-as-standalone-html
description: "可离线使用的单个自包含文件"
user-invocable: true
---
# 另存为独立 HTML 文件

将当前设计导出为一个完全自包含的单文件 HTML，可在离线状态下运行——无需任何外部依赖。

### 工作原理

我们提供了一个确定性的打包工具（super_inline_html），它可以内联那些直接在 HTML 属性中引用的资源，包括：`img` 的 `src`/`srcset`、`source` 的 `src`/`srcset`、`video`/`audio`/`track` 的 `src`、`video` 的 `poster`、SVG 中的 `<image href>` 和 `<use href>`、`link` 的 `href`（样式表、favicon）、`script` 的 `src`、CSS 中的 `url()` 和 `@import`，以及内联的 `style` 属性。然而，该工具无法识别仅以字符串形式出现在 JavaScript 或 JSX 代码中的资源引用，例如：
- React 中设置的图片路径：`<img src={"./hero.png"} />`
- styled-components 中的背景 URL：`background: url('./pattern.svg')`
- 动态导入的脚本

您的任务是先对 HTML 文件进行预处理，确保打包工具能够捕获所有内容，然后再运行打包流程。

### 第一步：复制 HTML 文件并更新代码中引用的资源

复制当前的 HTML 文件，仔细阅读并提取其中的所有依赖项。检查所有代码部分（内联脚本、引入的 JSX 文件、styled-components 等），找出所有以字符串形式而非 HTML 属性引用的资源 URL。这包括：
- React/JSX 中的图片 URL（如 `<img src={...} />`、`style={{ backgroundImage: ... }}`）
- CSS-in-JS（styled-components、通过 JS 设置的内联样式）中的 URL
- 引入其他脚本的 `<script>` 标签，而这些脚本本身又引用了资源
- 通过 `fetch()` 或 `XMLHttpRequest` 加载资源的请求
- 以编程方式设置的音频/视频源

注意：如果项目中使用了 Anthropic API，则该功能在独立模式下将无法正常工作。若此功能为核心需求，请立即停止并告知用户！

### 第二步：添加 ext-resource-dependency 元标签

对于第一步中找到的每一个资源，在 `<head>` 部分添加一个 `<meta>` 标签：

```html
<meta name="ext-resource-dependency" content="<url>" data-resource-id="<id>" />
```

其中：
- `content` 是资源的 URL（相对于 HTML 文件的相对路径，或绝对路径）
- `data-resource-id` 是一个简短且唯一的标识符（如 "heroImage"、"patternSvg"）

然后，将代码中硬编码的 URL 替换为 `window.__resources[id]`。在打包后的文件运行时，`window.__resources[id]` 将包含指向内联资源数据的 Blob URL。

示例：
```html
<!-- 在 <head> 中： -->
<meta name="ext-resource-dependency" content="./hero.png" data-resource-id="heroImg" />
<meta name="ext-resource-dependency" content="./pattern.svg" data-resource-id="patternBg" />

<!-- 在代码中，将： -->
<!-- <img src={"./hero.png"} /> -->
<!-- 替换为： -->
<!-- <img src={window.__resources.heroImg} /> -->
```

重要提示：
- `content` 中的相对路径是相对于 HTML 文件本身的路径。
- 对于所有被引入且自身也引用资源的外部 `<script>` 标签，同样需要执行上述操作——这些脚本会被打包工具内联，但其内部的资源引用也需要一并处理。
- 请务必仔细检查！遗漏任何一个资源都会导致最终文件中的图片或资产显示异常。

### 第三步：创建缩略图（必填——缺少缩略图将导致打包失败）

创建一个轻量级的 SVG 缩略图，作为打包文件解压时的启动画面。该 SVG 应当是对设计的简化、具代表性的预览，例如关键形状、布局轮廓或品牌加载动画。它不需要像素级精确，只需能直观地传达设计的核心信息即可。由于缩略图会以极小尺寸显示，因此在一个鲜艳背景色上放置一个简单的图标就足够了。

将其作为 `<template>` 标签添加到源 HTML 中：

```html
<template id="__bundler_thumbnail" data-bg-color="#0a5e3e">
  <svg viewBox="0 0 1200 800" xmlns="http://www.w3.org/2000/svg">
    <!-- 简化的图标 -->
  </svg>
</template>
```

- 将 `data-bg-color` 设置为与页面背景色一致
- SVG 应使用 `viewBox` 属性以实现正确的等比例缩放
- 保持简洁——这只是一个加载占位符，无需完整还原设计
- 使用设计中的实际颜色，使过渡效果更加自然流畅

打包工具会提取该内容，并在解包资源时以全屏方式显示（按背景色进行等比例缩放），随后将其替换为真实的页面内容。此外，当 JavaScript 被禁用时，该内容也会作为永久的后备显示。

### 第4步：运行打包工具

如果您在第1至3步中进行了修改，请先保存修改后的HTML文件。然后（或者如果没有进行任何修改），调用以下命令：

```
super_inline_html({ input_path: "<html文件路径>", output_path: "My Deck.html" })
```

为输出文件指定一个便于识别的名称。

### 第5步：验证（仅限内部检查）

**请先阅读工具的输出结果**——如果存在无法解析的资源，super_inline_html 会在输出中直接列出这些资源（“N个资源未能打包：- 资源未找到：./foo.png”）。这是权威的缺失资源清单，请先修复这些引用并重新运行，然后再打开任何文件。

之后，使用 `show_html` 打开打包后的输出文件以确认其正常运行——这是一个仅供您内部验证的步骤，而非交付环节。请通过 `get_webview_logs` 检查是否存在运行时错误（如JS异常、解码失败等）。如有问题，请修复源文件并重新运行。

### 第6步：提供下载——强制要求

您必须使用 `present_fs_item_for_download` 直接指向内联后的HTML输出文件来交付最终结果。这是导出独立文件的唯一正确方式。

- 请勿使用 `show_html` 或 `show_to_user` 作为交付步骤——它们是预览工具，而非下载工具，用户无法通过它们保存文件。
- 请勿询问用户是否要下载，只需调用 `present_fs_item_for_download` 即可。
- 如果跳过此步骤，用户将无法获取该文件。此步骤不可省略。