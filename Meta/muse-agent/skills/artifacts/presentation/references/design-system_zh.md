---
description: CSS 合同文档已据此编写，内容涵盖主题变量、画布以及布局框架。
---
# 幻灯片设计系统

每张幻灯片都依据以下 CSS 约定进行开发：主题令牌集、画布，以及英雄区/标志区/内容布局的结构框架。交付给你的 `StylePlan` 提供了令牌值（即 CSS 变量），并为每张幻灯片选择布局，同时为整个演示文稿选定标志配对。所有这些都位于演示文稿自身的 `.src/slides/deck.css` 文件中，每张幻灯片文件都会引用它；无需外部基础 CSS，而演示文稿唯一引入的外部资源是主题的 Google Fonts 样式表，该样式表在 `deck.css` 的顶部被导入（见下文）。

## 主题令牌（`:root`）

将 `StylePlan.css_variables` 原封不动地写入一个 `:root` 块中，然后在各处引用这些变量。切勿硬编码主题色值或字体族名；也绝不可重新定义它们。（页面背景、英雄区遮罩的回退色，以及英雄区叠加文字是允许使用具体颜色值的例外情况。）

```css
:root {
  --slide-bg: <paper>;           /* 页面背景 */
  --slide-fg: <ink>;             /* 主要文本 */
  --slide-primary: <primary>;    /* 强调色：标题、关键数据、边框、图标（慎用） */
  --slide-accent: <accent>;      /* 辅助元素 */
  --slide-muted: color-mix(in srgb, var(--slide-fg) 80%, var(--slide-bg));  /* 次要强调；绝不使用透明度 */
  --slide-font-display: "<display font>";  /* 标题；引用多词名称时使用 */
  --slide-font-body: "<body font>";        /* 正文 */
}
```

`--hero-scrim-color` 仅在全幅英雄区的情况下按每张幻灯片单独设置（从图片中采样的深色调），绝不在 `:root` 中定义；`var(--hero-scrim-color, #1a1a2e)` 的回退值可确保未设置时的安全性。

**在 `:root` 中命名主题字体，且任何地方都不写字体 URL。** `StylePlan` 中的两种字体均来自 Google Fonts 字体库，但演示文稿本身绝不直接引用 Google。在 `deck.css` 或任何幻灯片文件中，**禁止使用 `@import`、`<link>`，以及 `fonts.googleapis.com` 或 `fonts.gstatic.com`**。你只需在 `--slide-font-display` 和 `--slide-font-body` 中声明每种字体家族。随后，`embed_deck_fonts.mjs` 脚本（workflow.md 第 7 步）会下载这些字体文件并将其嵌入到 `deck.css` 中。渲染浏览器根本无法从网络获取字体，因此真正导致字体加载失败的是外部引用，而非满足加载条件。

这些 `:root` 中的字体名称是下载过程的唯一输入。务必按照 Google 官方发布的名称准确拼写每个字体家族。字重由你编写的 CSS 决定，例如 `font-weight: 900` 才会加载真正的 900 字重。Google 仅返回该字体家族实际提供的字重；目前所有 StylePlan 字体家族都提供 `400` 字重，而像 `Bebas Neue`、`Anton`、`DM Serif Display` 和 `Archivo Black` 这样的单字重展示字体则不提供 `600` 或 `700` 字重。切勿手动编写 `@font-face` 规则或其 Base64 编码。单个字体文件通常超过 100 KB，只有脚本才应负责写入这些内容。

**在 `:root` 的字体堆栈中始终将 StylePlan 的原始字体家族置于首位，切勿用本地字体替代它。** 在离线打开 HTML 版本时，可以添加一个本地或通用字体作为备选，但这并不能通过幻灯片渲染检查。报告会区分“缺失的主题字体”（`fonts.missing`）与“已解析但未被任何元素使用”的字体（`fonts.unused`），而 `fonts.used` 则列出 Chromium 实际栅格化的字体家族。用 `Liberation` 或 `Noto` 替换主字体并不能解决加载失败问题，反而会使演示文稿永久以系统默认字体呈现。由于幻灯片渲染使用 `--require-webfonts` 设置，即使已安装的同名本地字体也无法满足 Web 字体政策。如果检查提示某字体存在问题，请核对 `:root` 中的字体拼写，重新运行 `embed_deck_fonts.mjs`，并重新渲染。两个字体家族均需以 `400` 字重通过检查；只有当某个字体家族确实发布了该字重时，才可将其加入检查要求。

## 画布

每张幻灯片都是一个独立的 `section.slide`，各自存于单独的文件中；这些幻灯片再组合成演示文稿的一个自包含文档。相关规则均位于 `deck.css` 中。
```css
* { box-sizing: border-box; print-color-adjust: exact; -webkit-print-color-adjust: exact; }
html, body { margin: 0; background: #111; }
section.slide {
  width: 1280px; height: 720px;   /* 13.333英寸 x 7.5英寸 @96dpi；切勿使用 min-height */
  overflow: hidden;               /* 隐藏溢出内容，确保可用区域约620px */
  position: relative;
  background: var(--slide-bg);    /* 不透明背景；未设置背景时会显示页面灰色，视为加载失败 */
  color: var(--slide-fg);
  font-family: var(--slide-font-body);
  page-break-after: always; break-after: page;
}
section.slide:last-child { page-break-after: auto; break-after: auto; }
```

将布局类添加到 section 元素：`<section class="slide slide-hero-stats">`。

## 内容布局类

共有八种内容布局（第九、第十种分别为 `cover` 和 `closing`）。StylePlan 为每张幻灯片分配一种布局；将其视为默认选项，最终选择由内容决定——较浅色的幻灯片可选用更简单的布局。每种布局均按 `authoring.md` 中的 CSS Grid 规则编写样式；尺寸依据该文件中的排版规范确定。

| 类名 | 适用场景 |
|---|---|
| `slide-two-column` | 两列并排的正文/图片布局 |
| `slide-bento` | 混合单元格的均衡网格布局（统计、文本、图片） |
| `slide-hero-stats` | 最多三个大型统计信息组合（数字在上，标签在下） |
| `slide-headline-bullets` | 标题 + ≤5条项目符号列表 |
| `slide-steps` | 左对齐的有序垂直序列 |
| `slide-comparison` | 对比展示（前后对比或 A vs B 块状布局） |
| `slide-statement` | 居中的一条编辑性陈述 |
| `slide-gallery` | 一行排列的2–3张占满版面的图片 |

`slide-statement`、`slide-hero-stats` 以及封面和结束页采用编辑级的尺寸比例（让标题或数字显得更大）；其余布局则遵循 `authoring.md` 中的内容尺寸规范。

## 封面英雄模板

封面仅包含英雄图片和标题（参见 `authoring.md`）。请根据图片构图选择合适的模板。图片应铺满整个版面，绝不使用黑边；英雄图片不得添加 `border-radius`、边框、阴影或内边距。

**全幅出血**（适用于16:9比例的图片，通常首选；标题位于图片的留白区域）。此模板为共享样式，置于 `deck.css` 中：
```css
section.slide.cover {
  background-color: var(--slide-bg);
  background-size: cover; background-position: center;
}
section.slide.cover::after {              /* 仅底部遮罩，绝非 inset:0 */
  content: ''; position: absolute; left: 0; right: 0; bottom: 0; height: 55%;
  background: linear-gradient(transparent,
    color-mix(in srgb, var(--hero-scrim-color, #1a1a2e) 90%, transparent));
  z-index: 1;
}
.cover .title-block { position: absolute; bottom: 40px; left: 56px; right: 56px; z-index: 2; color: #fff; }
```

嵌入式英雄及其采样的遮罩色调属于该张幻灯片，因此需在该幻灯片文件的 `<style>` 中定义，并以该幻灯片的 ID 限定作用域（参见 `workflow.md` 第5步）：
```html
<style>
  #cover {
    background-image: url('data:image/...');   /* 嵌入式英雄 */
    --hero-scrim-color: #1a1a2e;               /* 从图片中采样的深色系 */
  }
</style>
```

**分栏布局**（适用于高长形/竖版图片，图片占比≥50%，边缘出血，文字单独一栏）。共享模板，置于 `deck.css` 中，图片部分留空：
```css
section.slide.cover { display: flex; padding: 0; }
.cover .image-panel {
  flex: 0 0 50%; position: relative;
  background-size: cover; background-position: center;
}
.cover .text-panel {
  flex: 0 0 50%; background: var(--slide-bg);
  display: flex; flex-direction: column; justify-content: center;
  padding: 64px 56px; color: var(--slide-fg);
}
```
英雄图片属于该张幻灯片，因此需在该幻灯片文件的 `<style>` 中定义，并以该幻灯片的 ID 限定作用域：
```html
<style>
  #cover .image-panel { background-image: url('data:image/...'); }
</style>
```

请将 `.text-panel` 放在 DOM 结构的前面，以实现图片右移的效果。无需单独的装饰条 div。**对角分割**（一条贯穿全高的干净对角线；文字显示在纯色的 `var(--slide-bg)` 背景上）。
共享的骨架结构位于 `deck.css` 中，图片部分暂不包含：  
```css
section.slide.cover { position: relative; padding: 0; background: var(--slide-bg); }
.cover .image-panel {
  position: absolute; inset: 0 0 0 0; width: 100%;
  background-size: cover; background-position: center;
  clip-path: polygon(0 0, 58% 0, 42% 100%, 0 100%);
}
.cover .text-panel {
  position: absolute; top: 0; right: 0; bottom: 0; width: 42%;
  display: flex; flex-direction: column; justify-content: center;
  padding: 56px 60px 56px 40px; z-index: 2; color: var(--slide-fg);
}
```
幻灯片文件中通过同作用域的 `<style>` 标签定义主视觉：  
```html
<style>
  #cover .image-panel { background-image: url('data:image/...'); }
</style>
```
不要为 `.text-panel` 设置额外的背景或边框（那样会多出一条直线边缘）。

每张封面的布局都采用相同的分割方式：结构部分由 `deck.css` 共享，而嵌入的图片则在幻灯片自身的局部作用域 `<style>` 中定义。如果在 `deck.css` 中使用 `data:` 格式的主视觉，会导致所有幻灯片都加载同一张图片；而在幻灯片文件中使用全局选择器则会被构建工具拒绝。

## 锁版布局（内容页上的文字与图片）

仅使用设计规范中指定的两种锁版布局（`StylePlan.lockups`，源自 L1/L2/L4）。图片单元格应铺满整个区域（`width:100%; height:100%; object-fit:cover; border-radius:20px`），以避免出现空白条带。

- **L1 左重**：`grid-template-columns:55fr 45fr; grid-template-rows:auto 1fr;` 图片单元格设置为 `grid-row:2; grid-column:1`，文字单元格设置为 `grid-column:2`，并在垂直方向居中（`align-self:center`）。
- **L2 右重**：与 L1 对称，文字列在左，图片列在右。
- **L4 非对称**（最强烈的效果，优先选用）：采用 60/40 的网格布局，图片填满其单元格，文字内嵌并留有 56px 的内边距。

## 结束页

文字居中，置于统一的背景之上（完整规则参见 `authoring.md`）。除非文字覆盖在背景照片之上，否则无需添加遮罩层。