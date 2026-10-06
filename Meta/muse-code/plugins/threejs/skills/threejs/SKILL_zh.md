---
name: threejs
description: Three.js 3D 参考中心。在使用 Three.js 时，请选择下方的主题并阅读相应文档，以获取模式和 API 使用指南。
source: https://github.com/CloudAI-X/threejs-skills
license: MIT
---
# Three.js

此技能是一个枢纽：一条目录项指向十个主题引用。
当任务涉及某个主题时，只需一步即可读取该主题的文件：将主题的 `references/threejs-<topic>.md` 路径与本主体包装头中的 `skill-dir` 值拼接，得到绝对路径后传递给 `read_file` 函数。

- `references/threejs-fundamentals.md` — 场景设置、相机、渲染器、Object3D 层次结构、坐标系。
- `references/threejs-geometry.md` — 内置几何体、BufferGeometry、自定义几何体、实例化。
- `references/threejs-materials.md` — PBR 材质、基础材质、Phong 材质、着色器材质、材质属性。
- `references/threejs-lighting.md` — 光源类型、阴影、环境光。
- `references/threejs-textures.md` — 纹理类型、UV 映射、环境贴图、纹理设置。
- `references/threejs-animation.md` — 关键帧动画与骨骼动画、变形目标、动画混合。
- `references/threejs-loaders.md` — GLTF 加载、纹理加载、图片加载、模型加载、异步加载模式。
- `references/threejs-shaders.md` — GLSL、ShaderMaterial、uniform 变量、自定义效果。
- `references/threejs-postprocessing.md` — EffectComposer、泛光效果、景深、屏幕后处理效果。
- `references/threejs-interaction.md` — 射线投射、控件、鼠标/触控输入、对象选择。