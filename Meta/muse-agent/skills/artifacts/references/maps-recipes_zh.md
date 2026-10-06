---
description: 使用随附的地图辅助工具生成 URL、创建交互式地图以及叠加数据。
---
# 地图辅助函数

请参阅 [地图与地点](maps.md)，了解渲染和数据获取的相关要求。  
实现位于 `/opt/hatch/skills/artifacts/scripts/map_helpers.mjs`。请直接使用该脚本，避免复制或重写 URL 及地图初始化代码。

## 地点查询

请使用现有的命令行工具 `/opt/hatch/bin/local-search` 和 `/opt/hatch/bin/places`。  
将任务的 `project_dir` 设置为绝对路径，并将其中的 `~/` 替换为 `$JARVIS_HOME/`。  
将查询结果保存至 `.src/research/places.json`（用于文件类成果）或 `client/src/research/places.json`（用于 Web 类成果）。  
按照 [解析地点](maps.md#resolve-places) 中的要求传递相应参数，将标准输出重定向到目标文件，并在读取前等待命令执行完成。  
建议设置明确的 `yield_ms: 60000`；若命令执行时间超过该值，则应收集其后台返回结果。

## 位置数据

`PlaceRef` 对象包含必填字段 `label`，以及可选字段 `venue`、`address`、`locality`、`region`、`country`、`lat`、`lng` 和 `source`。坐标为数值类型，地址字段为字符串类型。请使用相关辅助函数存储信息并生成服务提供商的 URL。

## 静态地图 URL

通过 JSON 格式的选项文件运行 URL 生成脚本：

```bash
bun /opt/hatch/skills/artifacts/scripts/map_urls.mjs static map-options.json
```

必填项：`center: [lat, lng]` 和 `zoom`。  
可选项：`width`、`height`、`scale`、`format`、`theme`、任务指定的 `language` 和 `region`，以及  
`markers: [{position: [lat, lng], scale: 1 | 2 | 3}]`。  
脚本会自动添加调用方及归属信息参数，并输出最终 URL；请在构建时下载图片并缓存副本以供后续渲染。

对于圆形和路径，请导入 `buildStaticMapUrl` 函数，并按形状分别追加 `circles` 或 `paths[]` 参数。  
管道分隔的语法格式为：可选的 `color:0xRRGGBBAA`、`fillcolor:0xRRGGBBAA`、`weight:N`、`dash:N` 或 `gap:N`，后接坐标。  
圆形需在最后指定半径，如 `500m`、`1k`、`1mi` 或 `2000ft`；路径则列出各顶点坐标。填充颜色应使用半透明的 Alpha 值。

## 交互式地图

将 `/opt/hatch/skills/artifacts/map-runtime/dist/` 目录下的三个文件全部复制到当前成果的 assets 目录中，并以传统 script 标签加载它们。静态页面可能无法加载模块，因此请在此处使用生成的 `map_helpers.js` 文件，而非 `.mjs` 文件：

```html
<link rel="stylesheet" href="assets/hatch-maps.css">
<script src="assets/hatch-maps.js"></script>
<script src="assets/map_helpers.js"></script>
```

```js
const map = MapHelpers.mountMap(el, { places, baseStyle: 'light' }, showPlaceList);
```

在 TypeScript 空间中，请将 `map_helpers.mjs` 复制到 `client/src/` 目录下，并从该目录导入。无论哪种方式，以下方法名均相同：在静态页面中通过 `MapHelpers` 调用，在空间中直接导入。

`mountMap` 会检测浏览器功能，并将不可用情况的错误回调交给用户提供的 `onUnavailable` 回调函数（第三个参数）。该函数返回地图实例句柄或 `null`。请自行实现回调逻辑：显示地点列表，或将数据以表格或图表形式呈现。该辅助函数不会创建页面 UI。

若地点列表较短，可调用 `mountPlaceListMap(el, {places, rows}, showPlaceList)`。  
`rows` 是一个 DOM 元素数组，顺序与 `places` 一致。该辅助函数会为标记编号、同步点击与高亮、滚动至选中行，并清除选中状态。请为 `.pin`、`.pin--on` 和 `.map-selected` 定义样式；可通过 `selectedClass` 更改最后一个类名。每个容器和列表只需挂载一次。

## 数据叠加层

`heatmapOverlay(id, geoJSON, colorStops, radius)` 返回一个可用于 `mountMap` 的热力图叠加层。  
`colorStops` 是密度阈值与对应颜色的交替序列；应在零密度处加入透明度，并根据页面主题推导颜色。默认半径为 20。将其作为 `overlays` 传入，并选择 `baseStyle: 'grayscale'`，同时使用图表或表格作为回调展示数据。

其他编码方式可使用控件的常规 `overlays` 和 `tooltip` 选项。  
叠加层始终位于底图标签之下；控件的标记始终位于其之上。

### 区域分级统计图

调用 `mountChoropleth(el, options, showValueTable)`。  
必填选项：

| 选项 | 值 |
|---|---|
| `boundaryUrl` | 下方的已发布几何 URL |
| `rows` | 包含区域标识符和数值的源记录 |
| `featureKey`、`rowKey`、`valueKey` | 几何关联键、行关联键和数值字段 |
| `colorStops` | 来自页面调色板的交替数值阈值与颜色 |
| `outlineColor` | 来自同一调色板的轮廓颜色 |

该辅助函数会检查功能支持情况，获取几何数据，进行数值关联，并完成组件挂载，同时提供灰度渲染、数值悬浮提示，以及用于处理获取或地图加载失败的统一回调。缺失值将保持为 `null` 并不参与填充；零值则会保持可见。请添加图例以说明缺失值及其比例尺。

悬浮文本采用页面语言编写。默认显示格式为 `name: value`；可通过传递 `missingLabel` 参数来自定义无数值区域的显示文字，或通过 `tooltip` 参数传入一个以要素为输入的函数，以完全替换默认的悬浮文本内容。

| 几何类型 | URL | 要素键 |
|---|---|---|
| 国家 | `https://external.xx.fbcdn.net/maps/static/boundaries/v1/countries.json` | `iso_2`、`iso_3` |
| 美国各州 | `https://external.xx.fbcdn.net/maps/static/boundaries/v1/us_states.json` | `postal_code`、`iso_3166` |

## 导航链接

在页面中调用 `mapsSearchUrl(place)` 或 `mapsDirectionsUrl(destination, options)`。`options` 接受实际的 `origin: {lat, lng}` 和 `travelmode`，取值可为 `driving`、`walking`、`bicycling` 或 `transit`。当无可用位置时，这些辅助函数会返回 URL 或 `null`。请在新标签页中打开链接。

如需在构建时生成，请运行 `map_urls.mjs search place.json`，或使用 `{"destination": PlaceRef, "options": ...}` 运行 `map_urls.mjs directions route.json`。有关 CLI 使用方法，请执行 `bun /opt/hatch/skills/artifacts/scripts/map_urls.mjs --help`。