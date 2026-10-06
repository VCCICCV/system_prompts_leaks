---
name: threejs-materials
description: Three.js 材质——PBR、基础材质、Phong 材质、着色器材质以及材质属性。适用于为网格设置样式、处理纹理、创建自定义着色器或优化材质性能。
---
# Three.js 材质

## 快速入门

```javascript
import * as THREE from "three";

// PBR材质（推荐用于真实感渲染）
const material = new THREE.MeshStandardMaterial({
  color: 0x00ff00,
  roughness: 0.5,
  metalness: 0.5,
});

const mesh = new THREE.Mesh(geometry, material);
```

## 材质类型概览

| 材质                 | 使用场景                              | 光照支持           |
| -------------------- | ------------------------------------- | ------------------ |
| MeshBasicMaterial    | 无光照、纯色、线框                    | 不支持             |
| MeshLambertMaterial  | 哑光表面、性能优先                    | 支持（仅漫反射）   |
| MeshPhongMaterial    | 亮面、高光效果                        | 支持               |
| MeshStandardMaterial | PBR、真实感材质                       | 支持（PBR）        |
| MeshPhysicalMaterial | 高级PBR、清漆层、透射效果              | 支持（PBR+）       |
| MeshToonMaterial     | 卡通风格、赛璐珞着色                  | 支持（卡通着色）   |
| MeshNormalMaterial   | 法线调试                              | 不支持             |
| MeshDepthMaterial    | 深度可视化                            | 不支持             |
| ShaderMaterial       | 自定义GLSL着色器                      | 自定义             |
| RawShaderMaterial    | 完全自定义着色器控制                   | 自定义             |

## MeshBasicMaterial

不进行任何光照计算。速度快，始终可见。

```javascript
const material = new THREE.MeshBasicMaterial({
  color: 0xff0000,
  transparent: true,
  opacity: 0.5,
  side: THREE.DoubleSide, // FrontSide, BackSide, DoubleSide
  wireframe: false,
  map: texture, // 颜色/漫反射贴图
  alphaMap: alphaTexture, // 透明度贴图
  envMap: envTexture, // 反射贴图
  reflectivity: 1, // 环境贴图强度
  fog: true, // 受场景雾效影响
});
```

## MeshLambertMaterial

仅支持漫反射光照。速度快，无高光。

```javascript
const material = new THREE.MeshLambertMaterial({
  color: 0x00ff00,
  emissive: 0x111111, // 自发光颜色
  emissiveIntensity: 1,
  map: texture,
  emissiveMap: emissiveTexture,
  envMap: envTexture,
  reflectivity: 0.5,
});
```

## MeshPhongMaterial

支持高光效果。适用于有光泽的塑料类表面。

```javascript
const material = new THREE.MeshPhongMaterial({
  color: 0x0000ff,
  specular: 0xffffff, // 高光颜色
  shininess: 100, // 高光锐度（0-1000）
  emissive: 0x000000,
  flatShading: false, // 平面着色 vs 平滑着色
  map: texture,
  specularMap: specTexture, // 每像素的光泽度
  normalMap: normalTexture,
  normalScale: new THREE.Vector2(1, 1),
  bumpMap: bumpTexture,
  bumpScale: 1,
  displacementMap: dispTexture,
  displacementScale: 1,
});
```

## MeshStandardMaterial（PBR）

基于物理的渲染。推荐用于实现真实感效果。

```javascript
const material = new THREE.MeshStandardMaterial({
  color: 0xffffff,
  roughness: 0.5, // 0=镜面，1=漫反射
  metalness: 0.0, // 0=绝缘体，1=金属

  // 贴图
  map: colorTexture, // 反照率/基础颜色
  roughnessMap: roughTexture, // 每像素的粗糙度
  metalnessMap: metalTexture, // 每像素的金属度
  normalMap: normalTexture, // 表面细节
  normalScale: new THREE.Vector2(1, 1),
  aoMap: aoTexture, // 环境光遮蔽（使用uv2！）
  aoMapIntensity: 1,
  displacementMap: dispTexture, // 顶点位移
  displacementScale: 0.1,
  displacementBias: 0,

  // 自发光
  emissive: 0x000000,
  emissiveIntensity: 1,
  emissiveMap: emissiveTexture,

  // 环境贴图
  envMap: envTexture,
  envMapIntensity: 1,

  // 其他
  flatShading: false,
  wireframe: false,
  fog: true,
});

// 注意：环境光遮蔽需要第二套UV坐标
geometry.setAttribute("uv2", geometry.attributes.uv);
```

## MeshPhysicalMaterial（高级PBR）

在MeshStandardMaterial基础上增加了更多高级特性。

```javascript
const material = new THREE.MeshPhysicalMaterial({
  // 包含所有MeshStandardMaterial的属性外：

  // 清漆层（汽车漆、清漆）
  clearcoat: 1.0, // 清漆层强度（0-1）
  clearcoatRoughness: 0.1,
  clearcoatMap: ccTexture,
  clearcoatRoughnessMap: ccrTexture,
  clearcoatNormalMap: ccnTexture,
  clearcoatNormalScale: new THREE.Vector2(1, 1),

  // 透射（玻璃、水）
  transmission: 1.0, // 0=完全不透明，1=完全透明
  transmissionMap: transTexture,
  thickness: 0.5, // 体积厚度，用于折射
  thicknessMap: thickTexture,
  attenuationDistance: 1, // 吸收距离
  attenuationColor: new THREE.Color(0xffffff),

  // 折射
  ior: 1.5, // 折射率（1-2.333）

  // 绒面（织物、天鹅绒）
  sheen: 1.0,
  sheenRoughness: 0.5,
  sheenColor: new THREE.Color(0xffffff),
  sheenColorMap: sheenTexture,
  sheenRoughnessMap: sheenRoughTexture,

  // 纹彩（肥皂泡、油膜）
  iridescence: 1.0,
  iridescenceIOR: 1.3,
  iridescenceThicknessRange: [100, 400],
  iridescenceMap: iridTexture,
  iridescenceThicknessMap: iridThickTexture,

  // 各向异性（拉丝金属）
  anisotropy: 1.0,
  anisotropyRotation: 0,
  anisotropyMap: anisoTexture,

  // 高光
  specularIntensity: 1,
  specularColor: new THREE.Color(0xffffff),
  specularIntensityMap: specIntTexture,
  specularColorMap: specColorTexture,
});
```

### 玻璃材质示例

```javascript
const glass = new THREE.MeshPhysicalMaterial({
  color: 0xffffff,
  metalness: 0,
  roughness: 0,
  transmission: 1,
  thickness: 0.5,
  ior: 1.5,
  envMapIntensity: 1,
});
```

### 汽车漆面示例

```javascript
const carPaint = new THREE.MeshPhysicalMaterial({
  color: 0xff0000,
  metalness: 0.9,
  roughness: 0.5,
  clearcoat: 1,
  clearcoatRoughness: 0.1,
});
```

## MeshToonMaterial

卡通渲染效果。

```javascript
const material = new THREE.MeshToonMaterial({
  color: 0x00ff00,
  gradientMap: gradientTexture, // 可选：自定义着色渐变
});

// 创建分段渐变纹理
const colors = new Uint8Array([0, 128, 255]);
const gradientMap = new THREE.DataTexture(colors, 3, 1, THREE.RedFormat);
gradientMap.minFilter = THREE.NearestFilter;
gradientMap.magFilter = THREE.NearestFilter;
gradientMap.needsUpdate = true;
```

## MeshNormalMaterial

可视化表面法线，可用于调试。

```javascript
const material = new THREE.MeshNormalMaterial({
  flatShading: false,
  wireframe: false,
});
```

## MeshDepthMaterial

渲染深度值，常用于阴影贴图和景深效果。

```javascript
const material = new THREE.MeshDepthMaterial({
  depthPacking: THREE.RGBADepthPacking,
});
```

## PointsMaterial

适用于点云。

```javascript
const material = new THREE.PointsMaterial({
  color: 0xffffff,
  size: 0.1,
  sizeAttenuation: true, // 随距离缩放
  map: pointTexture,
  alphaMap: alphaTexture,
  transparent: true,
  alphaTest: 0.5, // 丢弃低于阈值的像素
  vertexColors: true, // 使用顶点颜色
});

const points = new THREE.Points(geometry, material);
```

## LineBasicMaterial 和 LineDashedMaterial

```javascript
// 实线
const lineMaterial = new THREE.LineBasicMaterial({
  color: 0xffffff,
  linewidth: 1, // 注意：>1 仅在部分系统上生效
  linecap: "round",
  linejoin: "round",
});

// 虚线
const dashedMaterial = new THREE.LineDashedMaterial({
  color: 0xffffff,
  dashSize: 0.5,
  gapSize: 0.25,
  scale: 1,
});

// 虚线需要额外设置
const line = new THREE.Line(geometry, dashedMaterial);
line.computeLineDistances();
```

## ShaderMaterial

使用 Three.js 内置 uniform 的自定义 GLSL 着色器。

```javascript
const material = new THREE.ShaderMaterial({
  uniforms: {
    time: { value: 0 },
    color: { value: new THREE.Color(0xff0000) },
    texture1: { value: texture },
  },
  vertexShader: `
    varying vec2 vUv;
    uniform float time;

    void main() {
      vUv = uv;
      vec3 pos = position;
      pos.z += sin(pos.x * 10.0 + time) * 0.1;
      gl_Position = projectionMatrix * modelViewMatrix * vec4(pos, 1.0);
    }
  `,
  fragmentShader: `
    varying vec2 vUv;
    uniform vec3 color;
    uniform sampler2D texture1;

    void main() {
      // GLSL 1.0 使用 texture2D()，GLSL 3.0 使用 texture()（glslVersion: THREE.GLSL3）
      vec4 texColor = texture2D(texture1, vUv);
      gl_FragColor = vec4(color * texColor.rgb, 1.0);
    }
  `,
  transparent: true,
  side: THREE.DoubleSide,
});

// 在动画循环中更新 uniform
material.uniforms.time.value = clock.getElapsedTime();
```

### 内置 Uniform（自动提供）

```glsl
// 顶点着色器
uniform mat4 modelMatrix;         // 对象到世界
uniform mat4 modelViewMatrix;     // 对象到相机
uniform mat4 projectionMatrix;    // 相机投影
uniform mat4 viewMatrix;          // 世界到相机
uniform mat3 normalMatrix;        // 法线变换矩阵
uniform vec3 cameraPosition;      // 相机的世界位置

// 属性
attribute vec3 position;
attribute vec3 normal;
attribute vec2 uv;
```

## RawShaderMaterial

完全控制，无内置 uniform/attribute。
```javascript
const material = new THREE.RawShaderMaterial({
  uniforms: {
    projectionMatrix: { value: camera.projectionMatrix },
    modelViewMatrix: { value: new THREE.Matrix4() },
  },
  vertexShader: `
    precision highp float;
    attribute vec3 position;
    uniform mat4 projectionMatrix;
    uniform mat4 modelViewMatrix;

    void main() {
      gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
    }
  `,
  fragmentShader: `
    precision highp float;

    void main() {
      gl_FragColor = vec4(1.0, 0.0, 0.0, 1.0);
    }
  `,
});
```

## 常见材质属性

所有材质都共享以下基础属性：

```javascript
// 可见性
material.visible = true;
material.transparent = false;
material.opacity = 1.0;
material.alphaTest = 0; // 丢弃 alpha 小于该值的像素

// 渲染
material.side = THREE.FrontSide; // FrontSide、BackSide、DoubleSide
material.depthTest = true;
material.depthWrite = true;
material.colorWrite = true;

// 混合
material.blending = THREE.NormalBlending;
// NormalBlending、AdditiveBlending、SubtractiveBlending、MultiplyBlending、CustomBlending

// 遮罩
material.stencilWrite = false;
material.stencilFunc = THREE.AlwaysStencilFunc;
material.stencilRef = 0;
material.stencilMask = 0xff;

// 多边形偏移（解决 z-fighting 问题）
material.polygonOffset = false;
material.polygonOffsetFactor = 0;
material.polygonOffsetUnits = 0;

// 其他
material.dithering = false;
material.toneMapped = true;
```

## 多种材质

```javascript
// 为几何体的不同分组指定不同材质
const geometry = new THREE.BoxGeometry(1, 1, 1);
const materials = [
  new THREE.MeshBasicMaterial({ color: 0xff0000 }), // 右侧
  new THREE.MeshBasicMaterial({ color: 0x00ff00 }), // 左侧
  new THREE.MeshBasicMaterial({ color: 0x0000ff }), // 顶部
  new THREE.MeshBasicMaterial({ color: 0xffff00 }), // 底部
  new THREE.MeshBasicMaterial({ color: 0xff00ff }), // 前侧
  new THREE.MeshBasicMaterial({ color: 0x00ffff }), // 后侧
];
const mesh = new THREE.Mesh(geometry, materials);

// 自定义分组
geometry.clearGroups();
geometry.addGroup(0, 6, 0); // 起始索引、数量、材质索引
geometry.addGroup(6, 6, 1);
```

## 环境贴图

```javascript
// 加载立方体贴图
const cubeLoader = new THREE.CubeTextureLoader();
const envMap = cubeLoader.load([
  "px.jpg",
  "nx.jpg", // X 轴正/负方向
  "py.jpg",
  "ny.jpg", // Y 轴正/负方向
  "pz.jpg",
  "nz.jpg", // Z 轴正/负方向
]);

// 应用到材质
material.envMap = envMap;
material.envMapIntensity = 1;

// 或者设置为场景环境（影响所有 PBR 材质）
scene.environment = envMap;

// HDR 环境（推荐）
import { RGBELoader } from "three/examples/jsm/loaders/RGBELoader.js";
const rgbeLoader = new RGBELoader();
rgbeLoader.load("environment.hdr", (texture) => {
  texture.mapping = THREE.EquirectangularReflectionMapping;
  scene.environment = texture;
  scene.background = texture;
});
```

## 材质克隆与修改

```javascript
// 克隆材质
const clone = material.clone();
clone.color.set(0x00ff00);

// 运行时修改
material.color.set(0xff0000);
material.needsUpdate = true; // 只有部分属性变更时才需要

// 需要调用 needsUpdate 的情况：
// - 切换平面着色模式
// - 更换纹理
// - 修改透明度
// - 自定义着色器代码发生变化
```

## 性能优化建议

1. **复用材质**：相同的材质可以合并绘制调用
2. **尽量避免使用透明材质**：透明材质需要进行排序
3. **在适用时使用 alphaTest 代替透明度**：速度更快
4. **选择更简单的材质**：基础 > 朗伯 > 冯氏 > 标准 > 物理
5. **限制活动光源数量**：每个光源都会增加着色器的复杂度

```javascript
// 材质池化
const materialCache = new Map();
function getMaterial(color) {
  const key = color.toString(16);
  if (!materialCache.has(key)) {
    materialCache.set(key, new THREE.MeshStandardMaterial({ color }));
  }
  return materialCache.get(key);
}

// 使用完毕后释放资源
material.dispose();
```

## 另请参阅

- `threejs-textures` - 纹理加载与配置
- `threejs-shaders` - 自定义着色器开发
- `threejs-lighting` - 光照与材质的交互