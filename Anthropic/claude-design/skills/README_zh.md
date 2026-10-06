# 技能

每个技能一个文件夹，每个文件夹内包含一个带有 YAML 前置元数据和完整提示文本的 `SKILL.md` 文件。这些内容均从实时环境中提取（于 2026 年 8月19日完成对齐）。

前置元数据遵循本项目自身 `SKILL.md` 所采用的 schema——包括 `name`、`description` 和 `user-invocable`，除此之外无其他字段。环境以纯提示文本的形式提供技能，不带任何自有元数据，因此 `name` 和 `description` 是根据系统提示中的技能列表重新构建的。

系统提示和原始工具 schema 存放在项目根目录下的 `claude-design.md` 中。启动组件源代码存放在 `starter-components/` 目录下。

## 内置技能

全部 19 项技能均可通过斜杠菜单调用，顺序如下：

**创建类**

| 技能 | 文件夹 |
|---|---|
| 制作演示文稿 | `skills/make-a-deck/` |
| 制作文档 | `skills/make-a-doc/` |
| 交互式原型 | `skills/interactive-prototype/` |
| 线框图 | `skills/wireframe/` |
| 动画视频 | `skills/animated-video/` |
| 创建设计系统 | `skills/create-design-system/` |
| 前端设计 | `skills/frontend-design/` |
| 地图与地理 | `skills/maps-geography/` |
| 3D 对象 | `skills/3d-object/` |
| HTML 邮件 | `skills/html-email/` |
| 海报 | `skills/flier/` |

**增强类**

| 技能 | 文件夹 |
|---|---|
| 可调节化 | `skills/make-tweakable/` |
| 在原型中使用 Claude API | `skills/claude-api-in-prototypes/` |

**研究与数据类**

| 技能 | 文件夹 |
|---|---|
| 网络调研 | `skills/web-research/` |

**导出与交付类**

| 技能 | 文件夹 |
|---|---|
| 另存为 PDF | `skills/save-as-pdf/` |
| 导出为可编辑的 PPTX | `skills/export-as-pptx-editable/` |
| 导出为截图版 PPTX | `skills/export-as-pptx-screenshots/` |
| 另存为独立 HTML | `skills/save-as-standalone-html/` |
| 交付给 Claude Code | `skills/handoff-to-claude-code/` |

## 内部技能

这些技能内置且可被调用，但未出现在系统提示的技能列表及斜杠菜单中——由 Claude 自行触发，用户无法直接选择。

| 技能 | 文件夹 |
|---|---|
| 高保真设计 | `skills/hi-fi-design/` |
| 选项 | `skills/options/` |

## 2026年8月19日对齐后变更

- **动画视频** 已针对 `animations_v3.jsx` 连续合成引擎进行重写（单棵元素树绑定到作者定义的时钟，具有 `OM_SCENES` / `OM_PLAYBACK` 回写契约，并使用 `<Shot>` 和 `<Captions>` 组件）。`copy_starter_component` 不再提供 `animations_v2.jsx`。
- **海报** 现在基于 `doc_page.js` 启动组件（显式分页的单个 `<section class="page">`）构建，不再使用手写的打印 CSS。
- **制作文档** 已围绕 `doc_page.js` 重新编写——采用流动页面而非固定纸张布局，并添加了 CSS-columns 打印规则；原有的 `<main class="doc">` 布局与排版指导已被移除。
- **制作演示文稿** 现在会通过 `ask_user` 提问（包括设计系统相关问题）；重复的幻灯片撰写与规划模块已被合并。
- **另存为 PDF** 新增了 `omelette-print-source` 出处标记、doc_page 的重建路径以及固定画布的页面决策。
- **questions_v2** 被 **ask_user** 取代：新增 13 种问题类型（包括芯片选择、分段选择、下拉选择、颜色选择、用户自定义问题、文件选项、设计系统、代码来源等），并增加了 `prompt` 小标题、`follow_up` 循环，以及“替我决定”和跳过语义。
- 系统提示新增了 **工具搜索**，扩展了 **GitHub** 部分（包含 `github.md` 收据文件、屏幕映射及一轮同步），增加了 **额外设计指导**，并注入了 **默认美学** 和 **系统信息** 块。
- 新增文档记录的工具：`github_compare` 和 `tool_search_tool_bm25`。`connect_github` 现已变为一条空操作横幅，由代码来源问题取代。
- 经确认未发生变化的有：交互式原型、3D 对象、网络调研、HTML 邮件、可调节化、在原型中使用 Claude API、前端设计、线框图、两种 PPTX 导出方式、创建设计系统、另存为独立 HTML、交付给 Claude Code、地图与地理、高保真设计、选项。

在早期的版本中已移除：**发送至 Canva**（已从内置列表中移除）、**Canvas**（已被“选项”替代——`design_canvas.jsx` 文件已不存在）、**读取 PDF**（`read_skill_prompt` 已不再提供该功能，尽管系统提示的工作流部分仍提及调用该功能）。
