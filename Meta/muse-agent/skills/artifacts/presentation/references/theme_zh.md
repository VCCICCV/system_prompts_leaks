---
description: 位于 ~/workspace/themes/ 下的主题 JSON 设计系统、其模式，以及如何生成或应用主题。
---
# 主题模式与生成

主题是一个小型的 JSON 设计系统，存储在 `~/workspace/themes/` 目录下。构件生成器的主题决策位于 `/opt/hatch/skills/artifacts/pdf/references/visual.md`；在生成或应用主题时，请阅读该文件。

## 主题文件模式

每个主题的 JSON 文件包含两个部分：`ui`（文档样式）和 `visual`（图像生成指导）。示例如下：

```json
{
  "name": "纸与墨", "mode": "light",
  "ui": {
    "colors": { "bg": "#FAF9F6", "surface": "#F5F0EB", "text": "#1A1A1A", "accent": "#3D3229", "textMuted": "#8B7355", "danger": "#9B2C2C", "success": "#16A34A", "warning": "#D97706" },
    "font": "Georgia, 'Times New Roman', serif", "fontSans": "system-ui, sans-serif",
    "radius": { "card": "6px", "button": "4px" },
    "shadows": { "md": "0 1px 3px rgba(61,50,41,.08)" },
    "notes": "编辑风格，书卷气息。大写标题、细分隔线、充裕的留白。"
  },
  "visual": {
    "style": "20世纪70年代复古胶片摄影", "mood": "温暖、怀旧",
    "lighting": "黄金时段", "palette": "暖金色系、日晒褪色的柔和色彩",
    "texture": "35毫米胶片颗粒感", "references": "斯利姆·亚伦斯",
    "orientation": "横版", "filter": "contrast(1.05) saturate(0.85) sepia(0.1)"
  }
}
```

### `ui` 部分字段说明

| 字段 | 用途 |
|-------|---------|
| `colors` | 完整配色方案：背景色、表面色、边框色、文字色、强调色、灰度文字色，以及语义色（危险、成功、警告） |
| `font` / `fontSans` / `fontMono` | 正文、界面和代码的字体堆栈 |
| `fontWeight` | 各级标题、正文和标签的字体权重映射 |
| `radius` | 各类元素的圆角半径值 |
| `shadows` | 盒阴影值（或“none”表示扁平化主题） |
| `notes` | 主题个性及特殊组件样式 |

请确保文本与背景之间的对比度不低于 4.5:1。

### `visual` 部分字段说明

| 字段 | 用途 | 示例 |
|-------|---------|---------|
| `style` | 艺术方向 | “3D 皮克斯渲染”、“水彩画”、“复古胶片” |
| `mood` | 情绪基调 | “温馨舒适”、“阴郁神秘” |
| `lighting` | 光线方向 | “黄金时段”、“戏剧化的明暗对比” |
| `palette` | 图像色彩指引 | “宝石色调”、“黑色背景上的霓虹” |
| `texture` | 表面质感 | “干净的数字感”、“胶片颗粒感” |
| `references` | 风格参考 | “韦斯·安德森”、“吉卜力工作室” |
| `orientation` | 默认图像方向 | “横版”、“正方形”、“竖版” |
| `filter` | 应用于 `<img>` 标签的 CSS 滤镜 | “contrast(1.05) sepia(0.1)” 或 “none” |

## 主题的生成流程

1. **推断 `ui` 部分**：根据整体氛围确定颜色、字体和形状，确保文本与背景的对比度不低于 4.5:1，并选择浅色或深色模式。
2. **推断 `visual` 部分**：确定艺术风格、情绪基调、参考元素及 CSS 滤镜。
3. **保存**至 `~/workspace/themes/<kebab-case-name>.json`。
4. **确认**：“我创建了‘韦斯·安德森’主题：粉彩色调、Futura 字体、电影剧照风格。是否使用此主题？”

重点捕捉整体氛围（色温、密度、字体风格），而非精确复制布局。

## 主题的应用（构建端）

- **字体需离线渲染**。在将主题字体用于 CSS 前，需将其映射到本地已安装的字体族（可通过 `fc-list` 确认）。切勿依赖 Google Fonts 或 Microsoft 字体；可靠的本地字体族请参阅 `/opt/hatch/skills/artifacts/pdf/references/workflow.md`。
- **图像处理**。使用 `media.generate_image` 生成主图或插图时，应基于 `visual` 部分的内容（风格、情绪、光线、配色、参考）构建提示词，将 `output_dir` 设置为 `artifact_media_dir`，并在嵌入的图片上应用主题指定的滤镜。