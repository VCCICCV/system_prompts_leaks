---
name: options
description: "以垂直堆叠的锚点转角形式呈现多个设计选项。"
user-invocable: false
---
# 选项

将多个设计方案以垂直堆叠的回合形式呈现——每个方案回合作为一个独立的 `<section>`，最新回合位于**最上方**，且每个选项都拥有稳定的 `{turn}{letter}` 格式的 ID（如 `1a`、`1b`、`2a` 等），用户可在聊天中引用这些 ID，您也可在各回合之间建立互链。务必在 `<helmet>` 中加入 `<meta name="design_doc_mode" content="canvas">`——托管环境会提供平移和缩放功能，因此用户可以自由地将视图缩小，查看超出视口宽度的设计。

**编写方式**——在 `<helmet>` 中放置一个 `<style>` 块，然后为每个回合在根元素的**直接子级**中添加一个 `<section class="dv-turn">`（紧接在 `</helmet>` 后，无需额外的包裹容器）。当用户请求新一轮时，**将新回合的 `<section>` 插入到现有回合的上方**，使最新作品位于最顶端；切勿重新排序、重新编号或删除之前的回合。

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
<div class="dv-thd"><a class="dv-tid" href="#t2">2</a><span class="dv-tname">对 <a class="dv-oid" href="#1b">1b</a> 的变体</span></div>
<div class="dv-opts">
<div class="dv-opt" id="2a"><div class="dv-olabel"><a class="dv-oid" href="#2a">2a</a> 更紧凑的字距</div><div class="dv-card" style="width:360px">…设计…</div></div>
<div class="dv-opt" id="2b">…</div>
</div>
<p class="dv-next">下一步尝试：“更像 <a class="dv-oid" href="#2a">2a</a>，但使用 <a class="dv-oid" href="#1c">1c</a> 的衬线” · “让 <a class="dv-oid" href="#2b">2b</a> 满版出血” · “新的方向”</p>
</section>
<section class="dv-turn" id="t1">…第一轮，未改动…</section>
```

**规则：** 轮次 ID 为 `t1`、`t2`、`t3`……；选项 ID 为 `1a`、`1b`、`2a`……，且应加在选项的**最外层**元素（`.dv-opt`）上，绝不能加在徽章上——因此，`#1b` 会将整个选项滚动到视图中。ID 永久固定，不会被重复使用或重新编号。同一轮次内的选项以换行方式并排显示；请勿自行实现平移或缩放功能——这些由宿主画布提供。文件中**所有**对选项 ID 的引用——包括轮次标题、选项标签、`.dv-next` 行以及任何正文——都必须是 `<a class="dv-oid" href="#1b">1b</a>` 格式的链接，而不能直接写成 `1b`；在聊天回复中则只需写 `1b` 即可。每轮结束时，请添加一行 `.dv-next`，内容为 2–3 条用户可直接粘贴到聊天中的简单英文后续选项。请根据内容调整每个 `.dv-card` 的大小（明确设置宽度即可），切勿使用 `height:100%`。