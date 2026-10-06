---
name: threejs-postprocessing
description: Three.js 后期处理——EffectComposer、泛光、景深、屏幕特效。在添加视觉效果、颜色分级、模糊、发光，或创建自定义屏幕空间着色器时使用。
---
# Three.js 后处理

## 快速入门

```javascript
import * as THREE from "three";
import { EffectComposer } from "three/addons/postprocessing/EffectComposer.js";
import { RenderPass } from "three/addons/postprocessing/RenderPass.js";
import { UnrealBloomPass } from "three/addons/postprocessing/UnrealBloomPass.js";

// 设置合成器
const composer = new EffectComposer(renderer);

// 渲染场景
const renderPass = new RenderPass(scene, camera);
composer.addPass(renderPass);

// 添加泛光效果
const bloomPass = new UnrealBloomPass(
  new THREE.Vector2(window.innerWidth, window.innerHeight),
  1.5, // 强度
  0.4, // 半径
  0.85, // 阈值
);
composer.addPass(bloomPass);

// 动画循环——使用合成器而非渲染器
function animate() {
  requestAnimationFrame(animate);
  composer.render(); // 不是 renderer.render()
}
```

## EffectComposer 的设置

```javascript
import { EffectComposer } from "three/addons/postprocessing/EffectComposer.js";
import { RenderPass } from "three/addons/postprocessing/RenderPass.js";

const composer = new EffectComposer(renderer);

// 第一个通道：渲染场景
const renderPass = new RenderPass(scene, camera);
composer.addPass(renderPass);

// 添加更多通道……
composer.addPass(effectPass);

// 最后一个通道应直接渲染到屏幕上
effectPass.renderToScreen = true; // 默认为最后一个通道

// 处理窗口大小变化
function onResize() {
  const width = window.innerWidth;
  const height = window.innerHeight;

  camera.aspect = width / height;
  camera.updateProjectionMatrix();

  renderer.setSize(width, height);
  composer.setSize(width, height);
}
```

## 常见效果

### 泛光（发光）

```javascript
import { UnrealBloomPass } from "three/addons/postprocessing/UnrealBloomPass.js";

const bloomPass = new UnrealBloomPass(
  new THREE.Vector2(window.innerWidth, window.innerHeight),
  1.5, // 强度——发光的强度
  0.4, // 半径——发光的扩散范围
  0.85, // 阈值——亮度阈值
);

composer.addPass(bloomPass);

// 运行时调整参数
bloomPass.strength = 2.0;
bloomPass.threshold = 0.5;
bloomPass.radius = 0.8;
```

### 局部泛光

仅对特定对象应用泛光效果。

```javascript
import { UnrealBloomPass } from "three/addons/postprocessing/UnrealBloomPass.js";
import { ShaderPass } from "three/addons/postprocessing/ShaderPass.js";

// 层级设置
const BLOOM_LAYER = 1;
const bloomLayer = new THREE.Layers();
bloomLayer.set(BLOOM_LAYER);

// 标记需要泛光的对象
glowingMesh.layers.enable(BLOOM_LAYER);

// 用于非泛光对象的暗色材质
const darkMaterial = new THREE.MeshBasicMaterial({ color: 0x000000 });
const materials = {};

function darkenNonBloomed(obj) {
  if (obj.isMesh && !bloomLayer.test(obj.layers)) {
    materials[obj.uuid] = obj.material;
    obj.material = darkMaterial;
  }
}

function restoreMaterial(obj) {
  if (materials[obj.uuid]) {
    obj.material = materials[obj.uuid];
    delete materials[obj.uuid];
  }
}

// 自定义渲染循环
function render() {
  // 渲染泛光通道
  scene.traverse(darkenNonBloomed);
  composer.render();
  scene.traverse(restoreMaterial);

  // 在泛光之上渲染最终场景
  renderer.render(scene, camera);
}
```

### FXAA（抗锯齿）

```javascript
import { ShaderPass } from "three/addons/postprocessing/ShaderPass.js";
import { FXAAShader } from "three/addons/shaders/FXAAShader.js";

const fxaaPass = new ShaderPass(FXAAShader);
fxaaPass.material.uniforms["resolution"].value.set(
  1 / window.innerWidth,
  1 / window.innerHeight,
);

composer.addPass(fxaaPass);

// 窗口大小变化时更新
function onResize() {
  fxaaPass.material.uniforms["resolution"].value.set(
    1 / window.innerWidth,
    1 / window.innerHeight,
  );
}
```

### SMAA（更优的抗锯齿）

```javascript
import { SMAAPass } from "three/addons/postprocessing/SMAAPass.js";

const smaaPass = new SMAAPass(
  window.innerWidth * renderer.getPixelRatio(),
  window.innerHeight * renderer.getPixelRatio(),
);

composer.addPass(smaaPass);
```### SSAO（环境光遮蔽）

```javascript
import { SSAOPass } from "three/addons/postprocessing/SSAOPass.js";

const ssaoPass = new SSAOPass(
  scene,
  camera,
  window.innerWidth,
  window.innerHeight,
);
ssaoPass.kernelRadius = 16;
ssaoPass.minDistance = 0.005;
ssaoPass.maxDistance = 0.1;

composer.addPass(ssaoPass);

// 输出模式
ssaoPass.output = SSAOPass.OUTPUT.Default;
// SSAOPass.OUTPUT.Default - 最终合成输出
// SSAOPass.OUTPUT.SSAO - 仅环境光遮蔽
// SSAOPass.OUTPUT.Blur - 模糊后的环境光遮蔽
// SSAOPass.OUTPUT.Depth - 深度缓冲
// SSAOPass.OUTPUT.Normal - 法线缓冲
```

### 景深（DOF）

```javascript
import { BokehPass } from "three/addons/postprocessing/BokehPass.js";

const bokehPass = new BokehPass(scene, camera, {
  focus: 10.0, // 对焦距离
  aperture: 0.025, // 光圈（越小景深越大）
  maxblur: 0.01, // 最大模糊量
});

composer.addPass(bokehPass);

// 动态更新对焦
bokehPass.uniforms["focus"].value = distanceToTarget;
```

### 底片颗粒

```javascript
import { FilmPass } from "three/addons/postprocessing/FilmPass.js";

const filmPass = new FilmPass(
  0.35, // 噪声强度
  0.5, // 扫描线强度
  648, // 扫描线条数
  false, // 灰度
);

composer.addPass(filmPass);
```

### 柔边效果

```javascript
import { ShaderPass } from "three/addons/postprocessing/ShaderPass.js";
import { VignetteShader } from "three/addons/shaders/VignetteShader.js";

const vignettePass = new ShaderPass(VignetteShader);
vignettePass.uniforms["offset"].value = 1.0; // 柔边大小
vignettePass.uniforms["darkness"].value = 1.0; // 柔边强度

composer.addPass(vignettePass);
```

### 色彩校正

```javascript
import { ShaderPass } from "three/addons/postprocessing/ShaderPass.js";
import { ColorCorrectionShader } from "three/addons/shaders/ColorCorrectionShader.js";

const colorPass = new ShaderPass(ColorCorrectionShader);
colorPass.uniforms["powRGB"].value = new THREE.Vector3(1.2, 1.2, 1.2); // 幂运算
colorPass.uniforms["mulRGB"].value = new THREE.Vector3(1.0, 1.0, 1.0); // 乘法

composer.addPass(colorPass);
```

### Gamma校正

```javascript
import { GammaCorrectionShader } from "three/addons/shaders/GammaCorrectionShader.js";

const gammaPass = new ShaderPass(GammaCorrectionShader);
composer.addPass(gammaPass);
```

### 像素化
```javascript
import { RenderPixelatedPass } from "three/addons/postprocessing/RenderPixelatedPass.js";

const pixelPass = new RenderPixelatedPass(6, scene, camera); // 6 = 像素大小

composer.addPass(pixelPass);
```

### 效果故障

```javascript
import { GlitchPass } from "three/addons/postprocessing/GlitchPass.js";

const glitchPass = new GlitchPass();
glitchPass.goWild = false; // 连续的故障效果

composer.addPass(glitchPass);
```

### 半色调

```javascript
import { HalftonePass } from "three/addons/postprocessing/HalftonePass.js";

const halftonePass = new HalftonePass(window.innerWidth, window.innerHeight, {
  shape: 1, // 1 = 点状，2 = 椭圆，3 = 线条，4 = 方形
  radius: 4, // 点的大小
  rotateR: Math.PI / 12,
  rotateB: (Math.PI / 12) * 2,
  rotateG: (Math.PI / 12) * 3,
  scatter: 0,
  blending: 1,
  blendingMode: 1,
  greyscale: false,
});

composer.addPass(halftonePass);
```

### 轮廓线

```javascript
import { OutlinePass } from "three/addons/postprocessing/OutlinePass.js";

const outlinePass = new OutlinePass(
  new THREE.Vector2(window.innerWidth, window.innerHeight),
  scene,
  camera,
);

outlinePass.edgeStrength = 3;
outlinePass.edgeGlow = 0;
outlinePass.edgeThickness = 1;
outlinePass.pulsePeriod = 0;
outlinePass.visibleEdgeColor.set(0xffffff);
outlinePass.hiddenEdgeColor.set(0x190a05);

// 选择需要描边的对象
outlinePass.selectedObjects = [mesh1, mesh2];

composer.addPass(outlinePass);
```

## 自定义着色器通道

创建您自己的后处理效果。

```javascript
import { ShaderPass } from "three/addons/postprocessing/ShaderPass.js";

const CustomShader = {
  uniforms: {
    tDiffuse: { value: null }, // 必需：输入纹理
    time: { value: 0 },
    intensity: { value: 1.0 },
  },
  vertexShader: `
    varying vec2 vUv;

    void main() {
      vUv = uv;
      gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
    }
  `,
  fragmentShader: `
    uniform sampler2D tDiffuse;
    uniform float time;
    uniform float intensity;
    varying vec2 vUv;

    void main() {
      vec2 uv = vUv;

      // 波浪扭曲
      uv.x += sin(uv.y * 10.0 + time) * 0.01 * intensity;

      vec4 color = texture2D(tDiffuse, uv);
      gl_FragColor = color;
    }
  `,
};

const customPass = new ShaderPass(CustomShader);
composer.addPass(customPass);

// 在动画循环中更新
customPass.uniforms.time.value = clock.getElapsedTime();
```

### 反色着色器

```javascript
const InvertShader = {
  uniforms: {
    tDiffuse: { value: null },
  },
  vertexShader: `
    varying vec2 vUv;
    void main() {
      vUv = uv;
      gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
    }
  `,
  fragmentShader: `
    uniform sampler2D tDiffuse;
    varying vec2 vUv;

    void main() {
      vec4 color = texture2D(tDiffuse, vUv);
      gl_FragColor = vec4(1.0 - color.rgb, color.a);
    }
  `,
};
```

### 色差效果

```javascript
const ChromaticAberrationShader = {
  uniforms: {
    tDiffuse: { value: null },
    amount: { value: 0.005 },
  },
  vertexShader: `
    varying vec2 vUv;
    void main() {
      vUv = uv;
      gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
    }
  `,
  fragmentShader: `
    uniform sampler2D tDiffuse;
    uniform float amount;
    varying vec2 vUv;

    void main() {
      vec2 dir = vUv - 0.5;
      float dist = length(dir);

      float r = texture2D(tDiffuse, vUv - dir * amount * dist).r;
      float g = texture2D(tDiffuse, vUv).g;
      float b = texture2D(tDiffuse, vUv + dir * amount * dist).b;

      gl_FragColor = vec4(r, g, b, 1.0);
    }
  `,
};
```

## 组合多种效果

```javascript
import { EffectComposer } from "three/addons/postprocessing/EffectComposer.js";
import { RenderPass } from "three/addons/postprocessing/RenderPass.js";
import { UnrealBloomPass } from "three/addons/postprocessing/UnrealBloomPass.js";
import { ShaderPass } from "three/addons/postprocessing/ShaderPass.js";
import { FXAAShader } from "three/addons/shaders/FXAAShader.js";
import { VignetteShader } from "three/addons/shaders/VignetteShader.js";
import { GammaCorrectionShader } from "three/addons/shaders/GammaCorrectionShader.js";

const composer = new EffectComposer(renderer);

// 1. 渲染场景
composer.addPass(new RenderPass(scene, camera));

// 2. 泛光效果
const bloomPass = new UnrealBloomPass(
  new THREE.Vector2(window.innerWidth, window.innerHeight),
  0.5,
  0.4,
  0.85,
);
composer.addPass(bloomPass);

// 3. 暗角效果
const vignettePass = new ShaderPass(VignetteShader);
vignettePass.uniforms["offset"].value = 0.95;
vignettePass.uniforms["darkness"].value = 1.0;
composer.addPass(vignettePass);

// 4. 伽马校正
composer.addPass(new ShaderPass(GammaCorrectionShader));

// 5. 抗锯齿（始终在输出前作为最后一道工序）
const fxaaPass = new ShaderPass(FXAAShader);
fxaaPass.uniforms["resolution"].value.set(
  1 / window.innerWidth,
  1 / window.innerHeight,
);
composer.addPass(fxaaPass);
```

## 渲染到纹理

```javascript
// 创建渲染目标
const renderTarget = new THREE.WebGLRenderTarget(512, 512);

// 将场景渲染到目标
renderer.setRenderTarget(renderTarget);
renderer.render(scene, camera);
renderer.setRenderTarget(null);

// 使用纹理
const texture = renderTarget.texture;
otherMaterial.map = texture;
```

## 多通道渲染

```javascript
// 为不同场景或图层使用多个合成器
const bgComposer = new EffectComposer(renderer);
bgComposer.addPass(new RenderPass(bgScene, camera));

const fgComposer = new EffectComposer(renderer);
fgComposer.addPass(new RenderPass(fgScene, camera));
fgComposer.addPass(bloomPass);

// 在渲染循环中合并
function animate() {
  // 渲染背景且不清除颜色缓冲区
  renderer.autoClear = false;
  renderer.clear();

  bgComposer.render();

  // 在其上渲染前景
  renderer.clearDepth();
  fgComposer.render();
}
```

## WebGPU 后处理（Three.js r150+）

```javascript
import { postProcessing } from "three/addons/nodes/Nodes.js";
import { pass, bloom, dof } from "three/addons/nodes/Nodes.js";

// 使用节点系统
const scenePass = pass(scene, camera);
const bloomNode = bloom(scenePass, 0.5, 0.4, 0.85);

const postProcessing = new THREE.PostProcessing(renderer);
postProcessing.outputNode = bloomNode;

// 渲染
function animate() {
  postProcessing.render();
}
```

## 性能优化建议

1. **限制后处理通道数**：每个通道都会进行一次全屏渲染。
2. **降低分辨率**：对于模糊等通道，可使用较小的渲染目标。
3. **禁用未使用的特效**：根据需要开启或关闭相应的通道。
4. **优先使用 FXAA 而非 MSAA**：FXAA 的抗锯齿开销更低。
5. **使用 DevTools 进行性能分析**：检查 GPU 使用情况。

```javascript
// 禁用泛光效果
bloomPass.enabled = false;

// 降低泛光分辨率
const bloomPass = new UnrealBloomPass(
  new THREE.Vector2(window.innerWidth / 2, window.innerHeight / 2),
  strength,
  radius,
  threshold,
);

// 仅在高性能场景下应用特效
const isMobile = /iPhone|iPad|Android/i.test(navigator.userAgent);
if (!isMobile) {
  composer.addPass(expensivePass);
}
```

## 处理窗口大小变化

```javascript
function onWindowResize() {
  const width = window.innerWidth;
  const height = window.innerHeight;
  const pixelRatio = renderer.getPixelRatio();

  camera.aspect = width / height;
  camera.updateProjectionMatrix();

  renderer.setSize(width, height);
  composer.setSize(width, height);

  // 更新通道相关的分辨率
  if (fxaaPass) {
    fxaaPass.material.uniforms["resolution"].value.set(
      1 / (width * pixelRatio),
      1 / (height * pixelRatio),
    );
  }

  if (bloomPass) {
    bloomPass.resolution.set(width, height);
  }
}

window.addEventListener("resize", onWindowResize);
```

## 参考

- `threejs-shaders` - 自定义着色器开发
- `threejs-textures` - 渲染目标
- `threejs-fundamentals` - 渲染器设置