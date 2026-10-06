---
name: threejs-shaders
description: Three.js 着色器——GLSL、ShaderMaterial、uniforms、自定义效果。适用于创建自定义视觉效果、修改顶点、编写片段着色器或扩展内置材质。
---
# Three.js 着色器

## 快速入门

```javascript
import * as THREE from "three";

const material = new THREE.ShaderMaterial({
  uniforms: {
    time: { value: 0 },
    color: { value: new THREE.Color(0xff0000) },
  },
  vertexShader: `
    void main() {
      gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
    }
  `,
  fragmentShader: `
    uniform vec3 color;

    void main() {
      gl_FragColor = vec4(color, 1.0);
    }
  `,
});

// 在动画循环中更新
material.uniforms.time.value = clock.getElapsedTime();
```

## ShaderMaterial 与 RawShaderMaterial

### ShaderMaterial

Three.js 提供了内置的 uniform 和 attribute。

```javascript
const material = new THREE.ShaderMaterial({
  vertexShader: `
    // 可用的内置 uniform：
    // uniform mat4 modelMatrix;
    // uniform mat4 modelViewMatrix;
    // uniform mat4 projectionMatrix;
    // uniform mat4 viewMatrix;
    // uniform mat3 normalMatrix;
    // uniform vec3 cameraPosition;

    // 可用的内置 attribute：
    // attribute vec3 position;
    // attribute vec3 normal;
    // attribute vec2 uv;

    void main() {
      gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
    }
  `,
  fragmentShader: `
    void main() {
      gl_FragColor = vec4(1.0, 0.0, 0.0, 1.0);
    }
  `,
});
```

### RawShaderMaterial

完全控制——一切由你定义。

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

## Uniforms

### Uniform 类型

```javascript
const material = new THREE.ShaderMaterial({
  uniforms: {
    // 数值
    floatValue: { value: 1.5 },
    intValue: { value: 1 },

    // 向量
    vec2Value: { value: new THREE.Vector2(1, 2) },
    vec3Value: { value: new THREE.Vector3(1, 2, 3) },
    vec4Value: { value: new THREE.Vector4(1, 2, 3, 4) },

    // 颜色（转换为 vec3）
    colorValue: { value: new THREE.Color(0xff0000) },

    // 矩阵
    mat3Value: { value: new THREE.Matrix3() },
    mat4Value: { value: new THREE.Matrix4() },

    // 纹理
    textureValue: { value: texture },
    cubeTextureValue: { value: cubeTexture },

    // 数组
    floatArray: { value: [1.0, 2.0, 3.0] },
    vec3Array: {
      value: [new THREE.Vector3(1, 0, 0), new THREE.Vector3(0, 1, 0)],
    },
  },
});
```

### GLSL 声明

```glsl
// 在着色器中
uniform float floatValue;
uniform int intValue;
uniform vec2 vec2Value;
uniform vec3 vec3Value;
uniform vec3 colorValue;    // 颜色会转换为 vec3
uniform vec4 vec4Value;
uniform mat3 mat3Value;
uniform mat4 mat4Value;
uniform sampler2D textureValue;
uniform samplerCube cubeTextureValue;
uniform float floatArray[3];
uniform vec3 vec3Array[2];
```

### 更新 Uniforms

```javascript
// 直接赋值
material.uniforms.time.value = clock.getElapsedTime();

// 向量/颜色更新
material.uniforms.position.value.set(x, y, z);
material.uniforms.color.value.setHSL(hue, 1, 0.5);

// 矩阵更新
material.uniforms.matrix.value.copy(mesh.matrixWorld);
```

## Varyings

在顶点着色器和片段着色器之间传递数据。

```javascript
const material = new THREE.ShaderMaterial({
  vertexShader: `
    varying vec2 vUv;
    varying vec3 vNormal;
    varying vec3 vPosition;

    void main() {
      vUv = uv;
      vNormal = normalize(normalMatrix * normal);
      vPosition = (modelViewMatrix * vec4(position, 1.0)).xyz;

      gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
    }
  `,
  fragmentShader: `
    varying vec2 vUv;
    varying vec3 vNormal;
    varying vec3 vPosition;

    void main() {
      // 使用插值后的值
      gl_FragColor = vec4(vNormal * 0.5 + 0.5, 1.0);
    }
  `,
});
```

## 常见着色器模式

### 纹理采样

```javascript
const material = new THREE.ShaderMaterial({
  uniforms: {
    map: { value: texture },
  },
  vertexShader: `
    varying vec2 vUv;

    void main() {
      vUv = uv;
      gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
    }
  `,
  fragmentShader: `
    uniform sampler2D map;
    varying vec2 vUv;

    void main() {
      vec4 texColor = texture2D(map, vUv);
      gl_FragColor = texColor;
    }
  `,
});
```

### 顶点位移

```javascript
const material = new THREE.ShaderMaterial({
  uniforms: {
    time: { value: 0 },
    amplitude: { value: 0.5 },
  },
  vertexShader: `
    uniform float time;
    uniform float amplitude;

    void main() {
      vec3 pos = position;

      // 波浪位移
      pos.z += sin(pos.x * 5.0 + time) * amplitude;
      pos.z += sin(pos.y * 5.0 + time) * amplitude;

      gl_Position = projectionMatrix * modelViewMatrix * vec4(pos, 1.0);
    }
  `,
  fragmentShader: `
    void main() {
      gl_FragColor = vec4(0.5, 0.8, 1.0, 1.0);
    }
  `,
});
```

### 菲涅尔效应

```javascript
const material = new THREE.ShaderMaterial({
  vertexShader: `
    varying vec3 vNormal;
    varying vec3 vWorldPosition;

    void main() {
      vNormal = normalize(normalMatrix * normal);
      vWorldPosition = (modelMatrix * vec4(position, 1.0)).xyz;
      gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
    }
  `,
  fragmentShader: `
    varying vec3 vNormal;
    varying vec3 vWorldPosition;

    void main() {
      // cameraPosition 由 ShaderMaterial 自动提供
      vec3 viewDirection = normalize(cameraPosition - vWorldPosition);
      float fresnel = pow(1.0 - dot(viewDirection, vNormal), 3.0);

      vec3 baseColor = vec3(0.0, 0.0, 0.5);
      vec3 fresnelColor = vec3(0.5, 0.8, 1.0);

      gl_FragColor = vec4(mix(baseColor, fresnelColor, fresnel), 1.0);
    }
  `,
});
```

### 基于噪声的效果

```glsl
// 简单的噪声函数
float random(vec2 st) {
  return fract(sin(dot(st.xy, vec2(12.9898, 78.233))) * 43758.5453);
}

// 值噪声
float noise(vec2 st) {
  vec2 i = floor(st);
  vec2 f = fract(st);

  float a = random(i);
  float b = random(i + vec2(1.0, 0.0));
  float c = random(i + vec2(0.0, 1.0));
  float d = random(i + vec2(1.0, 1.0));

  vec2 u = f * f * (3.0 - 2.0 * f);

  return mix(a, b, u.x) + (c - a) * u.y * (1.0 - u.x) + (d - b) * u.x * u.y;
}

// 使用示例
float n = noise(vUv * 10.0 + time);
```

### 渐变

```glsl
// 线性渐变
vec3 color = mix(colorA, colorB, vUv.y);

// 径向渐变
float dist = distance(vUv, vec2(0.5));
vec3 color = mix(centerColor, edgeColor, dist * 2.0);

// 带自定义曲线的平滑渐变
float t = smoothstep(0.0, 1.0, vUv.y);
vec3 color = mix(colorA, colorB, t);
```

### 边缘光

```javascript
const material = new THREE.ShaderMaterial({
  vertexShader: `
    varying vec3 vNormal;
    varying vec3 vViewPosition;

    void main() {
      vNormal = normalize(normalMatrix * normal);
      vec4 mvPosition = modelViewMatrix * vec4(position, 1.0);
      vViewPosition = mvPosition.xyz;
      gl_Position = projectionMatrix * mvPosition;
    }
  `,
  fragmentShader: `
    varying vec3 vNormal;
    varying vec3 vViewPosition;

    void main() {
      vec3 viewDir = normalize(-vViewPosition);
      float rim = 1.0 - max(0.0, dot(viewDir, vNormal));
      rim = pow(rim, 4.0);

      vec3 baseColor = vec3(0.2, 0.2, 0.8);
      vec3 rimColor = vec3(1.0, 0.5, 0.0);

      gl_FragColor = vec4(baseColor + rimColor * rim, 1.0);
    }
  `,
});
```

### 溶解效果

```glsl
uniform float progress;
uniform sampler2D noiseMap;

void main() {
  float noise = texture2D(noiseMap, vUv).r;

  if (noise < progress) {
    discard;
  }

  // 边缘光效
  float edge = smoothstep(progress, progress + 0.1, noise);
  vec3 edgeColor = vec3(1.0, 0.5, 0.0);
  vec3 baseColor = vec3(0.5);

  gl_FragColor = vec4(mix(edgeColor, baseColor, edge), 1.0);
}
```

## 扩展内置材质

### onBeforeCompile

修改现有材质的着色器。

```javascript
const material = new THREE.MeshStandardMaterial({ color: 0x00ff00 });

material.onBeforeCompile = (shader) => {
  // 添加自定义uniform
  shader.uniforms.time = { value: 0 };

  // 存储引用以便更新
  material.userData.shader = shader;

  // 修改顶点着色器
  shader.vertexShader = shader.vertexShader.replace(
    "#include <begin_vertex>",
    `
    #include <begin_vertex>
    transformed.y += sin(position.x * 10.0 + time) * 0.1;
    `,
  );

  // 添加uniform声明
  shader.vertexShader = "uniform float time;\n" + shader.vertexShader;
};

// 在动画循环中更新
if (material.userData.shader) {
  material.userData.shader.uniforms.time.value = clock.getElapsedTime();
}
```

### 常见注入点

```javascript
// 顶点着色器片段
"#include <begin_vertex>"; // 在位置计算之后
"#include <project_vertex>"; // 在gl_Position之后
"#include <beginnormal_vertex>"; // 法线计算开始

// 片段着色器片段
"#include <color_fragment>"; // 在漫反射颜色之后
"#include <output_fragment>"; // 最终输出
"#include <fog_fragment>"; // 在雾效应用之后
```

## GLSL内置函数

### 数学函数

```glsl
// 基本函数
abs(x), sign(x), floor(x), ceil(x), fract(x)
mod(x, y), min(x, y), max(x, y), clamp(x, min, max)
mix(a, b, t), step(edge, x), smoothstep(edge0, edge1, x)

// 三角函数
sin(x), cos(x), tan(x)
asin(x), acos(x), atan(y, x), atan(x)
radians(degrees), degrees(radians)

// 指数函数
pow(x, y), exp(x), log(x), exp2(x), log2(x)
sqrt(x), inversesqrt(x)
```

### 向量函数

```glsl
// 长度与距离
length(v), distance(p0, p1), dot(x, y), cross(x, y)

// 归一化
normalize(v)

// 反射与折射
reflect(I, N), refract(I, N, eta)

// 分量操作
lessThan(x, y), lessThanEqual(x, y)
greaterThan(x, y), greaterThanEqual(x, y)
equal(x, y), notEqual(x, y)
any(bvec), all(bvec)
```

### 纹理函数

```glsl
// GLSL 1.0（默认）——使用texture2D/textureCube
texture2D(sampler, coord)
texture2D(sampler, coord, bias)
textureCube(sampler, coord)

// GLSL 3.0（glslVersion: THREE.GLSL3）——使用texture()
// texture(sampler, coord)替代texture2D/textureCube
// 此外，使用：out vec4 fragColor代替gl_FragColor

// 纹理尺寸（GLSL 1.30及以上）
textureSize(sampler, lod)
```

## 常见材质属性

```javascript
const material = new THREE.ShaderMaterial({
  uniforms: {
    /* ... */
  },
  vertexShader: "/* ... */",
  fragmentShader: "/* ... */",

  // 渲染相关
  transparent: true,
  opacity: 1.0,
  side: THREE.DoubleSide,
  depthTest: true,
  depthWrite: true,

  // 混合模式
  blending: THREE.NormalBlending,
  // AdditiveBlending、SubtractiveBlending、MultiplyBlending

  // 线框模式
  wireframe: false,
  wireframeLinewidth: 1, // 注意：>1在大多数平台上无效（WebGL限制）

  // 扩展功能
  extensions: {
    derivatives: true, // 支持fwidth、dFdx、dFdy
    fragDepth: true, // gl_FragDepth
    drawBuffers: true, // 多渲染目标
    shaderTextureLOD: true, // texture2DLod
  },

  // GLSL版本
  glslVersion: THREE.GLSL3, // 用于WebGL2特性
});
```

## 着色器包含

### 使用Three.js着色器片段

```javascript
import { ShaderChunk } from "three";

const fragmentShader = `
  ${ShaderChunk.common}
  ${ShaderChunk.packing}

  uniform sampler2D depthTexture;
  varying vec2 vUv;

  void main() {
    float depth = texture2D(depthTexture, vUv).r;
    float linearDepth = perspectiveDepthToViewZ(depth, 0.1, 1000.0);
    gl_FragColor = vec4(vec3(-linearDepth / 100.0), 1.0);
  }
`;
```

### 外部着色器文件

```javascript
// 使用 Vite/Webpack
import vertexShader from "./shaders/vertex.glsl";
import fragmentShader from "./shaders/fragment.glsl";

const material = new THREE.ShaderMaterial({
  vertexShader,
  fragmentShader,
});
```

## 实例化着色器

```javascript
// 实例化属性
const offsets = new Float32Array(instanceCount * 3);
// 填充偏移值...
geometry.setAttribute("offset", new THREE.InstancedBufferAttribute(offsets, 3));

const material = new THREE.ShaderMaterial({
  vertexShader: `
    attribute vec3 offset;

    void main() {
      vec3 pos = position + offset;
      gl_Position = projectionMatrix * modelViewMatrix * vec4(pos, 1.0);
    }
  `,
  fragmentShader: `
    void main() {
      gl_FragColor = vec4(1.0, 0.0, 0.0, 1.0);
    }
  `,
});
```

## 调试着色器

```javascript
// 检查编译错误
material.onBeforeCompile = (shader) => {
  console.log("顶点着色器:", shader.vertexShader);
  console.log("片段着色器:", shader.fragmentShader);
};

// 可视化调试
fragmentShader: `
  void main() {
    // 调试 UV
    gl_FragColor = vec4(vUv, 0.0, 1.0);

    // 调试法线
    gl_FragColor = vec4(vNormal * 0.5 + 0.5, 1.0);

    // 调试位置
    gl_FragColor = vec4(vPosition * 0.1 + 0.5, 1.0);
  }
`;

// 检查 WebGL 错误
renderer.debug.checkShaderErrors = true;
```

## 性能优化建议

1. **减少 uniform 的数量**：将相关数值打包成向量
2. **避免条件语句**：用 mix/step 替代 if/else
3. **提前计算**：尽可能将计算移到 JavaScript 中
4. **使用纹理**：对于复杂函数，可采用查找表
5. **减少过度绘制**：尽量避免使用透明物体

```glsl
// 不推荐：
if (value > 0.5) {
  color = colorA;
} else {
  color = colorB;
}

// 推荐：
color = mix(colorB, colorA, step(0.5, value));
```

## 参考资料

- `threejs-materials` - 内置材质类型
- `threejs-postprocessing` - 全屏着色器效果
- `threejs-textures` - 着色器中的纹理采样