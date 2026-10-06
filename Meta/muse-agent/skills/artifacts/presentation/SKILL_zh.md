---
name: artifact_presentation
metadata: { "不包含在提示中": 假 }
description: 构建或修订幻灯片演示文稿（默认为 PPTX 格式；可根据需求导出为 PDF 或 HTML）。当构建任务的产物类型为“演示文稿”，或用户请求生成文稿、幻灯片或演示时使用。涵盖每张幻灯片的 HTML 内容创作、StylePlan 主题系统、字体嵌入、文稿组装、渲染门控机制，以及 PPTX 导出功能。
---
# 幻灯片文稿的构建产物

一个文稿由每张幻灯片对应的一个独立 HTML 文件、一份共享的 `deck.css` 样式表以及一个 `deck.json` 清单文件构成，经过确定性地组装与校验后导出。位于 `.src/slides/` 目录下的各张幻灯片源文件是未来所有修订版本的唯一可编辑依据；幻灯片 ID 永远不会重新编号。

在阅读任何内容之前，每张幻灯片都必须遵守以下三条规则：**全文采用首字母大写格式**（包括统计标签）；不得对任何文本使用 `text-transform: uppercase` 或 `letter-spacing` 属性；严禁添加任何形式的“幻灯片装饰元素”：如标签、副标题、摘要行、页眉、引语、徽章、芯片、标签及说明文字等，均不得出现在幻灯片上。封面仅包含文稿标题及其主图，除此之外不应有任何其他内容。

| 任务 | 首先阅读 |
|---|---|
| 创建新文稿 | `/opt/hatch/skills/artifacts/presentation/references/workflow.md` 及其引用的资料 |
| 编辑现有文稿 | `/opt/hatch/skills/artifacts/presentation/references/editing.md` 及其引用的资料 |
| 设计与布局 | `/opt/hatch/skills/artifacts/presentation/references/visual.md` |
| 主题生成或应用 | `/opt/hatch/skills/artifacts/presentation/references/theme.md` |
| 展示数据图表的幻灯片 | 共享文档 `/opt/hatch/skills/artifacts/references/charts.md` |
| 展示地点或地图的幻灯片 | 共享文档 `/opt/hatch/skills/artifacts/references/maps.md` |

## 脚本

| 脚本 | 功能 |
|---|---|
| `/opt/hatch/bin/hatch-slide-style` | 编译样式规划；文稿绝不基于临时编写的替代方案进行构建 |
| `/opt/hatch/skills/artifacts/scripts/embed_deck_fonts.mjs` | 对网页字体进行子集化并嵌入到 `deck.css` 中；无法下载的字体会发出警告，但绝不会阻断加载 |
| `/opt/hatch/skills/artifacts/scripts/assemble_deck.mjs` | 根据清单校验幻灯片文件，强制实施 CSS 作用域，并输出合并后的文档；非零退出码表示不可交付 |
| `/opt/hatch/skills/artifacts/scripts/render_audit.mjs`（共享）| 渲染幻灯片并执行文稿的各项校验；`workflow.md` 文档中列出了相关标志位 |
| `/opt/hatch/skills/artifacts/scripts/build_pptx.py` | 将通过校验的 PNG 图片导出为 PPTX 格式，并尽可能将每张幻灯片的标题作为辅助性文本写入演讲者备注；守护进程侧的重建路径可在 UI 编辑后保留这些备注 |

## 验证流程

请遵循 `/opt/hatch/skills/artifacts/testing/SKILL.md` 的步骤：先运行各项校验，然后在提供链接前逐张重新查看每张幻灯片的 PNG 图片。