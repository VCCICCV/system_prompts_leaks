---
name: 3d-object
description: "three.js 模型，可下载为 OBJ 或 GLB 格式"
user-invocable: true
---
# 3D 对象

使用 three.js 构建一个用户可以从各个角度查看并下载的 3D 对象模型，并将其展示在 three_d_stage 入门项目中。

首先，调用 copy_starter_component，指定 kind 为 "three_d_stage.js" —— 这是一个完整的查看器：包含工作室灯光、地面阴影、轨道控制器、自动调整视角的相机，以及一个用于将对象导出为 OBJ + MTL 或 GLB 格式的工具栏。请仔细阅读复制后的文件中的使用说明，并严格按照其页面结构进行编写。你只需编写用于构建模型的模块脚本。

页面应以纯 HTML 的形式构建——即使用普通的 <script> 标签的 .html 文件——即使项目的其他设计采用的是 .dc.html 设计组件：因为 DC 会将脚本限制在 <helmet> 中，这会导致舞台挂载过早执行，无法承载此页面骨架。

仅通过以下精确的固定导入映射，在 <head> 中加载 three.js，且必须在任何模块脚本之前完成。请勿更改版本、URL 或哈希值，不要引入其他 three.js 的副本，也不要在列出的三个库之外再导入任何附加模块——该映射被刻意设置为封闭集合，因此任何其他未列出的内容都将导致解析失败，而不会加载未经验证的代码：

<script type="importmap">
{
  "imports": {
    "three": "https://unpkg.com/three@0.184.0/build/three.module.js",
    "three/addons/controls/OrbitControls.js": "https://unpkg.com/three@0.184.0/examples/jsm/controls/OrbitControls.js",
    "three/addons/exporters/OBJExporter.js": "https://unpkg.com/three@0.184.0/examples/jsm/exporters/OBJExporter.js",
    "three/addons/exporters/GLTFExporter.js": "https://unpkg.com/three@0.184.0/examples/jsm/exporters/GLTFExporter.js"
  },
  "integrity": {
    "https://unpkg.com/three@0.184.0/build/three.module.js": "sha384-8FCZ1eVO6it4+pbec2aDtnTrwjWXZLJRC+MAGCIPDgsYnUrl/E0A2YlF8ioMKI/J",
    "https://unpkg.com/three@0.184.0/build/three.core.js": "sha384-dw2ooPewaEIrAgl6oFDBmmBWCE9oW9LxRGcfwZ0hLvEprzo202wXl7vCYHRlSnOT",
    "https://unpkg.com/three@0.184.0/examples/jsm/controls/OrbitControls.js": "sha384-4rziNxOBZKQ69i+w+f89KJ55TCYquwchVbByQwmaOeIOXdOU2PLDn3kOfXHwIJC9",
    "https://unpkg.com/three@0.184.0/examples/jsm/exporters/OBJExporter.js": "sha384-nbwtoZENJD3Vq+ACK0CuGQdPMuDWHkamC2KJD70EV5nfg6jQjfppKOea07YJN+N3",
    "https://unpkg.com/three@0.184.0/examples/jsm/exporters/GLTFExporter.js": "sha384-VofkvpG6HERhFCYbsUOHeNXBCqID2nfqkQqnVzE1jc/oPcz+qJ13ADdXH08hE+cQ"
  }
}
</script>

以编程方式构建模型，将其组织为由命名部件组成的 THREE.Group：
- 在使用原始的 BufferGeometry 之前，优先组合基本几何体（BoxGeometry、CylinderGeometry、SphereGeometry、TorusGeometry、LatheGeometry、带 Shape 的 ExtrudeGeometry）；现实中的物体往往由比你想象中更多的基本几何体构成。
- 为每个网格和每种材质命名（如“hull”、“walnut”、“brass”）——这些名称会成为导出的 OBJ 文件中的 o / usemtl 条目，以及 GLB 文件中的节点名称，从而使下载的文件能在 Blender 中正常使用。
- 使用 MeshStandardMaterial，并选用一个精简的材质调色板（3–5 种材质，在各部件间共享）；有意识地设置粗糙度和金属度。纹理在 OBJ 导出时会丢失——应优先通过几何形状和材质颜色来表现细节，而非依赖纹理。
- 按真实世界单位（米）、Y 向上、以原点为中心进行建模，并使基座位于最低的 Y 值处。将共面的面片之间故意偏移约 0.001，以避免深度冲突。
- 曲面需有足够的细分段数，以便在全屏下呈现光滑效果（特征曲面上的径向细分段数不少于 32），但不要对那些无人可见的部分过度细分。

舞台工具栏为用户提供 OBJ + MTL（通用格式，包含几何体和各材质的颜色）以及 GLB（现代交换格式——保留部件层级和 PBR 材质，可无缝导入 Blender、Maya、Cinema 4D、Unity 和 Unreal）。这两种就是可供导出的格式——当用户请求其他格式（如 FBX、USDZ、STEP）时，请明确告知：本平台仅支持导出 OBJ + MTL 和 GLB。通过截图进行迭代：舞台会保持最后一帧的可读性，因此普通的截图工具可以直接捕捉实时画布——无需额外操作。编辑完模块文件后，在截图前使用 show_html 重新加载（iframe 会缓存已加载的模块）。从默认的构图角度观察对象，并调整其轮廓、比例以及材质的区分——轮廓是承载对象的关键。