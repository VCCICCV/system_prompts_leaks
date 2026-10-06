---
name: threejs-lighting
description: Three.js 灯光——灯光类型、阴影、环境光。在添加灯光、配置阴影、设置 IBL 或优化光照性能时使用。
---
# Three.js 灯光

## 快速入门

```javascript
import * as THREE from "three";

// 基础灯光设置
const ambientLight = new THREE.AmbientLight(0xffffff, 0.5);
scene.add(ambientLight);

const directionalLight = new THREE.DirectionalLight(0xffffff, 1);
directionalLight.position.set(5, 5, 5);
scene.add(directionalLight);
```

## 光源类型概览

| 光源类型         | 描述                     | 是否支持阴影 | 性能开销 |
| ---------------- | ------------------------ | ------------ | -------- |
| AmbientLight     | 各向同性均匀照明         | 不支持       | 非常低   |
| HemisphereLight  | 天空/地面渐变光照        | 不支持       | 非常低   |
| DirectionalLight | 平行光（模拟太阳）       | 支持         | 低       |
| PointLight       | 全方位点光源（灯泡）     | 支持         | 中等     |
| SpotLight        | 锥形聚光灯               | 支持         | 中等     |
| RectAreaLight    | 区域光源（窗户）         | 不支持*      | 高       |

\*RectAreaLight 的阴影需要自定义实现方案

## AmbientLight

对所有物体进行均匀照明，无方向性，不投射阴影。

```javascript
// AmbientLight(颜色, 强度)
const ambient = new THREE.AmbientLight(0xffffff, 0.5);
scene.add(ambient);

// 运行时修改
ambient.color.set(0xffffcc);
ambient.intensity = 0.3;
```

## HemisphereLight

从天空色到地面色的渐变光照，适用于户外场景。

```javascript
// HemisphereLight(天空色, 地面色, 强度)
const hemi = new THREE.HemisphereLight(0x87ceeb, 0x8b4513, 0.6);
hemi.position.set(0, 50, 0);
scene.add(hemi);

// 属性
hemi.color; // 天空色
hemi.groundColor; // 地面色
hemi.intensity;
```

## DirectionalLight

平行光束，用于模拟远处的光源（如太阳）。

```javascript
// DirectionalLight(颜色, 强度)
const dirLight = new THREE.DirectionalLight(0xffffff, 1);
dirLight.position.set(5, 10, 5);

// 光线指向目标点（默认为原点）
dirLight.target.position.set(0, 0, 0);
scene.add(dirLight.target);

scene.add(dirLight);
```

### 平行光阴影

```javascript
dirLight.castShadow = true;

// 阴影贴图尺寸（越大越清晰，但性能开销也越大）
dirLight.shadow.mapSize.width = 2048;
dirLight.shadow.mapSize.height = 2048;

// 阴影相机（正交相机）
dirLight.shadow.camera.near = 0.5;
dirLight.shadow.camera.far = 50;
dirLight.shadow.camera.left = -10;
dirLight.shadow.camera.right = 10;
dirLight.shadow.camera.top = 10;
dirLight.shadow.camera.bottom = -10;

// 阴影柔化程度
dirLight.shadow.radius = 4; // 模糊半径（仅适用于 PCFSoftShadowMap）

// 阴影偏移（用于消除阴影锯齿）
dirLight.shadow.bias = -0.0001;
dirLight.shadow.normalBias = 0.02;

// 可视化阴影相机的辅助工具
const helper = new THREE.CameraHelper(dirLight.shadow.camera);
scene.add(helper);
```

## 点光源

从一个点向所有方向发射光线，类似于灯泡。

```javascript
// PointLight(颜色, 强度, 距离, 衰减)
const pointLight = new THREE.PointLight(0xffffff, 1, 100, 2);
pointLight.position.set(0, 5, 0);
scene.add(pointLight);

// 属性
pointLight.distance; // 最大作用范围（0 表示无限远）
pointLight.decay; // 光线衰减系数（物理上正确的值为 2）
```

### 点光源阴影

```javascript
pointLight.castShadow = true;
pointLight.shadow.mapSize.width = 1024;
pointLight.shadow.mapSize.height = 1024;

// 阴影相机（透视投影，用于立方体贴图的六个方向）
pointLight.shadow.camera.near = 0.5;
pointLight.shadow.camera.far = 50;

pointLight.shadow.bias = -0.005;
```

## 聚光灯

锥形光源。类似于手电筒或舞台灯光。

```javascript
// SpotLight(颜色, 强度, 距离, 角度, 半影, 衰减)
const spotLight = new THREE.SpotLight(0xffffff, 1, 100, Math.PI / 6, 0.5, 2);
spotLight.position.set(0, 10, 0);

// 目标点（灯光指向此处）
spotLight.target.position.set(0, 0, 0);
scene.add(spotLight.target);

scene.add(spotLight);

// 属性
spotLight.angle; // 锥形角度（弧度，最大值为 Math.PI/2）
spotLight.penumbra; // 软边缘（0-1）
spotLight.distance; // 范围
spotLight.decay; // 衰减

### 聚光灯阴影

```javascript
spotLight.castShadow = true;
spotLight.shadow.mapSize.width = 1024;
spotLight.shadow.mapSize.height = 1024;

// 阴影相机（透视）
spotLight.shadow.camera.near = 0.5;
spotLight.shadow.camera.far = 50;
spotLight.shadow.camera.fov = 30;

spotLight.shadow.bias = -0.0001;

// 聚焦（影响阴影投射）
spotLight.shadow.focus = 1;
```

## 矩形面光源

矩形面光源，非常适合用于柔和、逼真的光照效果。

```javascript
import { RectAreaLightHelper } from "three/examples/jsm/helpers/RectAreaLightHelper.js";
import { RectAreaLightUniformsLib } from "three/examples/jsm/lights/RectAreaLightUniformsLib.js";

// 必须先初始化 uniform
RectAreaLightUniformsLib.init();

// RectAreaLight(颜色, 强度, 宽度, 高度)
const rectLight = new THREE.RectAreaLight(0xffffff, 5, 4, 2);
rectLight.position.set(0, 5, 0);
rectLight.lookAt(0, 0, 0);
scene.add(rectLight);

// 辅助工具
const helper = new RectAreaLightHelper(rectLight);
rectLight.add(helper);

// 注意：仅适用于 MeshStandardMaterial 和 MeshPhysicalMaterial
// 默认不投射阴影
```

## 阴影设置

### 启用阴影

```javascript
// 1. 在渲染器上启用
renderer.shadowMap.enabled = true;
renderer.shadowMap.type = THREE.PCFSoftShadowMap;

// 阴影贴图类型：
// THREE.BasicShadowMap - 最快，质量较低
// THREE.PCFShadowMap - 默认，带过滤
// THREE.PCFSoftShadowMap - 边缘更柔和
// THREE.VSMShadowMap - 方差阴影贴图

// 2. 在光源上启用
light.castShadow = true;

// 3. 在物体上启用
mesh.castShadow = true;
mesh.receiveShadow = true;

// 地面平面
floor.receiveShadow = true;
floor.castShadow = false; // 地面通常不投射阴影
```

### 优化阴影

```javascript
// 缩小阴影相机的视锥体
const d = 10;
dirLight.shadow.camera.left = -d;
dirLight.shadow.camera.right = d;
dirLight.shadow.camera.top = d;
dirLight.shadow.camera.bottom = -d;
dirLight.shadow.camera.near = 0.5;
dirLight.shadow.camera.far = 30;

// 解决阴影“痤疮”问题
dirLight.shadow.bias = -0.0001; // 深度偏移
dirLight.shadow.normalBias = 0.02; // 沿法线的偏移

// 阴影贴图尺寸（在质量和性能之间权衡）
// 512 - 低质量
// 1024 - 中等质量
// 2048 - 高质量
// 4096 - 极高质量（代价较高）
```

### 接触阴影（伪阴影，快速）

```javascript
import { ContactShadows } from "three/examples/jsm/objects/ContactShadows.js";

const contactShadows = new ContactShadows({
  resolution: 512,
  blur: 2,
  opacity: 0.5,
  scale: 10,
  position: [0, 0, 0],
});
scene.add(contactShadows);
```

## 光源辅助工具

```javascript
import { RectAreaLightHelper } from "three/examples/jsm/helpers/RectAreaLightHelper.js";

// 平行光辅助工具
const dirHelper = new THREE.DirectionalLightHelper(dirLight, 5);
scene.add(dirHelper);

// 点光源辅助工具
const pointHelper = new THREE.PointLightHelper(pointLight, 1);
scene.add(pointHelper);

// 聚光灯辅助工具
const spotHelper = new THREE.SpotLightHelper(spotLight);
scene.add(spotHelper);

// 半球光辅助工具
const hemiHelper = new THREE.HemisphereLightHelper(hemiLight, 5);
scene.add(hemiHelper);

// 矩形面光源辅助工具
const rectHelper = new RectAreaLightHelper(rectLight);
rectLight.add(rectHelper);

// 当光源发生变化时更新辅助工具
dirHelper.update();
spotHelper.update();
```

## 环境光照（IBL）

使用 HDR 环境贴图进行基于图像的光照。

```javascript
import { RGBELoader } from "three/examples/jsm/loaders/RGBELoader.js";

const rgbeLoader = new RGBELoader();
rgbeLoader.load("environment.hdr", (texture) => {
  texture.mapping = THREE.EquirectangularReflectionMapping;

  // 设置为场景环境（影响所有 PBR 材质）
  scene.environment = texture;
});
```  // 可选：同时用作背景
  scene.background = texture;
  scene.backgroundBlurriness = 0; // 0-1，用于模糊背景
  scene.backgroundIntensity = 1;
});

// PMREMGenerator 用于更优的反射效果
const pmremGenerator = new THREE.PMREMGenerator(renderer);
pmremGenerator.compileEquirectangularShader();

rgbeLoader.load("environment.hdr", (texture) => {
  const envMap = pmremGenerator.fromEquirectangular(texture).texture;
  scene.environment = envMap;
  texture.dispose();
  pmremGenerator.dispose();
});
```

### 立方体贴图环境

```javascript
const cubeLoader = new THREE.CubeTextureLoader();
const envMap = cubeLoader.load([
  "px.jpg",
  "nx.jpg",
  "py.jpg",
  "ny.jpg",
  "pz.jpg",
  "nz.jpg",
]);

scene.environment = envMap;
scene.background = envMap;
```

## 光探针（高级）

从空间中的某一点捕捉光照信息，用于环境光。

```javascript
import { LightProbeGenerator } from "three/examples/jsm/lights/LightProbeGenerator.js";

// 从立方体贴图生成
const lightProbe = new THREE.LightProbe();
scene.add(lightProbe);

lightProbe.copy(LightProbeGenerator.fromCubeTexture(cubeTexture));

// 或者从渲染目标生成
const cubeCamera = new THREE.CubeCamera(
  0.1,
  100,
  new THREE.WebGLCubeRenderTarget(256),
);
cubeCamera.update(renderer, scene);
lightProbe.copy(
  LightProbeGenerator.fromCubeRenderTarget(renderer, cubeCamera.renderTarget),
);
```

## 常见布光方案

### 三点布光

```javascript
// 主光（Key Light）
const keyLight = new THREE.DirectionalLight(0xffffff, 1);
keyLight.position.set(5, 5, 5);
scene.add(keyLight);

// 补光（Fill Light，较柔和，位于主光的对侧）
const fillLight = new THREE.DirectionalLight(0xffffff, 0.5);
fillLight.position.set(-5, 3, 5);
scene.add(fillLight);

// 背光（Rim Light）
const backLight = new THREE.DirectionalLight(0xffffff, 0.3);
backLight.position.set(0, 5, -5);
scene.add(backLight);

// 环境光填充
const ambient = new THREE.AmbientLight(0x404040, 0.3);
scene.add(ambient);
```

### 室外日光

```javascript
// 太阳光
const sun = new THREE.DirectionalLight(0xffffcc, 1.5);
sun.position.set(50, 100, 50);
sun.castShadow = true;
scene.add(sun);

// 天空环境光
const hemi = new THREE.HemisphereLight(0x87ceeb, 0x8b4513, 0.6);
scene.add(hemi);
```

### 室内影棚光

```javascript
// 多个矩形面光源
RectAreaLightUniformsLib.init();

const light1 = new THREE.RectAreaLight(0xffffff, 5, 2, 2);
light1.position.set(3, 3, 3);
light1.lookAt(0, 0, 0);
scene.add(light1);

const light2 = new THREE.RectAreaLight(0xffffff, 3, 2, 2);
light2.position.set(-3, 3, 3);
light2.lookAt(0, 0, 0);
scene.add(light2);

// 环境光填充
const ambient = new THREE.AmbientLight(0x404040, 0.2);
scene.add(ambient);
```

## 光源动画

```javascript
const clock = new THREE.Clock();

function animate() {
  const time = clock.getElapsedTime();

  // 让光源绕场景轨道运动
  light.position.x = Math.cos(time) * 5;
  light.position.z = Math.sin(time) * 5;

  // 强度脉动
  light.intensity = 1 + Math.sin(time * 2) * 0.5;

  // 颜色循环变化
  light.color.setHSL((time * 0.1) % 1, 1, 0.5);

  // 如果使用辅助工具，需更新它们
  lightHelper.update();
}
```

## 性能优化建议

1. **限制光源数量**：每个光源都会增加着色器的复杂度。
2. **使用烘焙光照**：对于静态场景，可将光照烘焙到纹理中。
3. **缩小阴影贴图尺寸**：通常 512-1024 就足够了。
4. **调整阴影视锥范围**：仅覆盖所需区域。
5. **禁用不必要的阴影**：并非所有光源都需要投射阴影。
6. **使用光源层**：控制哪些物体受特定光源影响。

```javascript
// 光源层
light.layers.set(1); // 该光源只影响第1层
mesh.layers.enable(1); // 该网格属于第1层
otherMesh.layers.disable(1); // 其他网格不受此光源影响

// 选择性地启用或禁用阴影
mesh.castShadow = true;
mesh.receiveShadow = true;
decorMesh.castShadow = false; // 较小的物体通常无需投射阴影
```

## 参考资料

- `threejs-materials` - 材质对光照的响应
- `threejs-textures` - 光照贴图与环境贴图
- `threejs-postprocessing` - 泛光等光照特效