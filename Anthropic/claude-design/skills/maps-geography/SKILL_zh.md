---
name: maps-geography
description: "基于真实地理数据的精准地图——可用于任何地图场景，或在需要以地理信息作为交付成果可视化元素时使用。"
user-invocable: true
---
# 地图与地理
地理地图是数据问题，而非手绘作品：切勿使用手绘的国家轮廓、海岸线或街道布局——手绘的地理信息往往不准确，用户很容易察觉。应加载真实的几何数据并进行渲染。

每个地图页面都应以纯 HTML 构建——即一个普通的 .html 文件，搭配标准的 <script> 标签，绝不能使用 .dc.html 设计组件，即使项目中的其他所有设计都采用这种组件也不行：因为 DC 会将脚本限制在 <helmet> 中，而其挂载时机与地图容器存在竞争关系——这与数据可视化和 3D 技术所面临的问题相同。

对于展示面板、文档、图形和动画——凡是静态或需导出的内容——请使用 d3-geo 渲染 TopoJSON 几何数据：从 https://cdn.jsdelivr.net/npm/world-atlas@2.0.2/countries-110m.json 获取（Natural Earth 数据，公共领域；该 URL 已固定版本，请原样使用），通过 topojson.feature(topology, topology.objects.countries) 进行转换，并在根据具体需求选择的投影下，使用 d3.geoPath() 进行绘制（例如，显示全球时可选用 d3.geoNaturalEarth1；若要缩放某个区域，则可使用 d3.geoMercator().fitSize(...)）。d3-geo 已包含在下方的 d3 库包中。请仅通过以下这些精确且经过哈希校验的标签，在 <head> 中加载相关库。这些标签一旦被篡改便会失效；您后续添加的任何其他脚本都将未经验证地加载——因此请勿更改版本、URL 或哈希值，也切勿从 CDN 引入其他资源：

<script src="https://unpkg.com/d3@7.9.0/dist/d3.min.js" integrity="sha384-CjloA8y00+1SDAUkjs099PVfnY2KmDC2BZnws9kh8D/lX1s46w6EPhpXdqMfjK6i" crossorigin="anonymous"></script>
<script src="https://unpkg.com/topojson-client@3.1.0/dist/topojson-client.min.js" integrity="sha384-Ukv1p/xTma6P4/2bY5KzWBw+ydSpXmhCMtyciIQVDJ1RmOxtCYNMF1uXT9T63H67" crossorigin="anonymous"></script>

由 d3 生成的内嵌 SVG 也能干净地导出为 PNG 和 PDF，而动态地图瓦片则无法做到这一点——因此，所有导出成果一律使用 d3 几何数据，绝不嵌入地图瓦片。

对于街景级别的交互式地图——包括原型、网站以及任何需要用户平移和缩放的场景——请使用 Leaflet 结合 OpenStreetMap 瓦片，并且仅通过以下这些确切的标签加载（样式表是必需的：缺少 leaflet.css 会导致瓦片显示错乱）：

<link rel="stylesheet" href="https://unpkg.com/leaflet@1.9.4/dist/leaflet.css" integrity="sha384-sHL9NAb7lN7rfvG5lfHpm643Xkcjzp4jFvuavGOndn6pjVqS6ny56CAt3nsEVT4H" crossorigin="anonymous">
<script src="https://unpkg.com/leaflet@1.9.4/dist/leaflet.js" integrity="sha384-cxOPjt7s7Iz04uaHJceBmS+qpjv2JkIHNVcuOrM+YHwZOmJGBXI00mdUXEq65HTH" crossorigin="anonymous"></script>

创建地图时，请使用 L.map(...) 和 L.tileLayer('https://tile.openstreetmap.org/{z}/{x}/{y}.png', { attribution: '© OpenStreetMap contributors' })。其中的署名字符串是 OpenStreetMap 的许可要求，切勿省略。