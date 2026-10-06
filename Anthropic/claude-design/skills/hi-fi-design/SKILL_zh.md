---
name: hi-fi-design
description: "用于制作高保真、精良作品的设计流程——系统提示会指示 Claude 在开始任何设计之前调用该流程。"
user-invocable: false
---
# 高保真设计

创建一个高保真、精美的设计方案。

遵循以下通用设计流程（使用待办清单提醒自己）：
(1) 提出问题，(2) 寻找现有的 UI 套件并收集设计背景——复制所有相关组件并阅读所有相关示例；如果找不到，请询问用户，(3) 以假设 + 背景 + 设计思路开始你的文件（就像你是一名初级设计师，而用户是你的主管），为设计预留占位符，并尽早向用户展示，(4) 完成设计并尽快再次向用户展示；附上一些后续步骤，(5) 使用工具检查、验证并迭代设计。

优秀的高保真设计并非从零开始——它们植根于现有的设计背景。请用户导入他们的代码库，或寻找合适的 UI 套件/设计资源，或提供现有 UI 的截图。你必须花时间获取设计背景，包括组件。如果找不到，请向用户索要。在导入菜单中，用户可以链接本地代码库、提供截图或 Figma 链接；他们也可以链接另一个项目。从头开始模拟整个产品是最后的手段，会导致设计质量不佳。如果卡住了，尝试列出设计资产并列出设计系统文件——要主动！有些设计可能需要多个设计系统——全部获取。使用起始组件（设备框架等）免费获得高质量的脚手架。

当在一个页面上展示多个设计方案时，选择 (a) 一个带有调整面板的全尺寸响应式原型，或 (b) 一列垂直排列的锚定选项卡片。根据需求是更偏向设计还是原型、选项数量以及每个选项的大小来决定。对于 (b)：

将多个设计方案以垂直堆叠的形式呈现——每一轮选项作为一个独立的 `<section>`，最新一轮位于**顶部**，每个选项都有一个稳定的 `{turn}{letter}` ID（`1a`、`1b`、`2a`……），用户可以在聊天中引用，你也可以在各轮之间进行交叉链接。始终在 `<helmet>` 中加入 `<meta name="design_doc_mode" content="canvas">`——宿主提供平移和缩放功能，因此用户可以自由地将超出视口的设计区域缩小查看。

**编写方式**——在 `<helmet>` 中放置一个 `<style>` 块，然后在根元素的**直接子元素**中为每一回合添加一个 `<section class="dv-turn">`（紧接在 `</helmet>` 之后，无需外层包裹）。当用户要求再增加一轮时，**将新章节插入到现有章节的上方**，使最新成果位于顶部；切勿重新排序、重新编号或删除之前的回合。
```html
<helmet data-dc-atomics><meta name="design_doc_mode" content="canvas"><style>
body{margin:0;background:#f0eee9;font-family:system-ui,sans-serif}
.dv-turn{padding:40px 44px 32px;border-bottom:1px solid rgba(0,0,0,.08);scroll-margin-top:16px}
.dv-thd{display:flex;align-items:baseline;gap:10px;margin:0 0 20px}
.dv-tid{font:600 10px ui-monospace,Menlo,monospace;padding:3px 7px;background:#1a1a1a;color:#fff;border-radius:4px;text-decoration:none}
.dv-tname{font:600 13px/1.2 system-ui,sans-serif;color:#1a1a1a}
.dv-opts{display:flex;flex-wrap:wrap;gap:28px;align-items:flex-start}
.dv-opt{flex:none;display:flex;flex-direction:column;gap:9px;scroll-margin-top:16px}
.dv-oid{font:600 10.5px ui-monospace,Menlo,monospace;padding:3px 7px;background:rgba(0,0,0,.08);color:#1a1a1a;border-radius:5px;text-decoration:none}
.dv-olabel{display:flex;align-items:baseline;gap:8px;font:400 11px/1.3 system-ui,sans-serif;color:rgba(0,0,0,.55)}
.dv-card{max-width:100%;background:#fff;border:1px solid rgba(0,0,0,.08);border-radius:8px;box-shadow:0 1px 3px rgba(0,0,0,.06);overflow:hidden}
.dv-opt:target .dv-oid{background:#2a78d6;color:#fff}
.dv-next{margin:22px 0 0;font:12px/1.5 system-ui,sans-serif;color:rgba(0,0,0,.5)}
</style></helmet>
<section class="dv-turn" id="t2">
<div class="dv-thd"><a class="dv-tid" href="#t2">2</a><span class="dv-tname">对 <a class="dv-oid" href="#1b">1b</a> 的变奏</span></div>
<div class="dv-opts">
<div class="dv-opt" id="2a"><div class="dv-olabel"><a class="dv-oid" href="#2a">2a</a> 更紧凑的间距</div><div class="dv-card" style="width:360px">…设计…</div></div>
<div class="dv-opt" id="2b">…</div>
</div>
<p class="dv-next">试试下一个：“更像 <a class="dv-oid" href="#2a">2a</a>，但用 <a class="dv-oid" href="#1c">1c</a> 的衬线” · “让 <a class="dv-oid" href="#2b">2b</a> 满版显示” · “新的方向”</p>
</section>
<section class="dv-turn" id="t1">…第一轮，未改动…</section>
```

**规则：** 轮次的 section ID 是 `t1`、`t2`、`t3`……；选项的 ID 是 `1a`、`1b`、`2a`……，并且放在选项的**最外层**元素（`.dv-opt`）上，绝不在徽标上——因此 `#1b` 会将整个选项滚动到视图中。ID 永久不变，绝不重复使用或重新编号。一轮中的各个选项并排排列在一行中，不要手动实现平移和缩放——由宿主画布提供。文件中**所有**对选项 ID 的引用——包括轮次标题、选项标签、`.dv-next` 行以及任何文字——都必须是 `<a class="dv-oid" href="#1b">1b</a>` 链接，绝不能只写 `1b`；在聊天回复中，只需写 `1b` 即可。每一轮结束时，都要在 `.dv-next` 中写一句包含 2–3 个用户可以复制粘贴到聊天中的后续建议。

每个 `.dv-card` 的大小应根据内容自适应（明确指定宽度即可），不要使用 `height:100%`。

在设计时，提出大量好问题是至关重要的。
给出多个选项：尽量从不同维度提供 3 个以上的变体。将符合现有模式的经典设计与新颖的交互方式相结合，包括有趣的布局、隐喻和视觉风格。有些选项可以使用颜色或高级 CSS，有些则不使用；有些包含图标，有些则没有。从基础开始，逐步增加复杂度和创意！尝试以有趣的方式重新组合品牌资产和视觉基因——调整比例、填充、纹理、视觉节奏、图层叠加以及独特的布局和字体处理。目标不是找到完美的选项，而是探索用户可以自由组合的原子化变体。

CSS、HTML、JS 和 SVG 非常强大，用户往往不知道它们能做什么。给用户带来惊喜。
如果没有现成的图标、素材或组件，就画一个占位符：在高保真设计中，占位符总比拙劣的模仿要好。
