---
name: threejs-geometry
description: Three.js 几何体创建——内置形状、BufferGeometry、自定义几何体、实例化。适用于创建 3D 形状、处理顶点、构建自定义网格，或通过实例化渲染进行优化。
---
# Three.js 几何体

## 快速入门

```javascript
import * as THREE from "three";

// 内置几何体
const box = new THREE.BoxGeometry(1, 1, 1);
const sphere = new THREE.SphereGeometry(0.5, 32, 32);
const plane = new THREE.PlaneGeometry(10, 10);

// 创建网格
const material = new THREE.MeshStandardMaterial({ color: 0x00ff00 });
const mesh = new THREE.Mesh(box, material);
scene.add(mesh);
```

## 内置几何体

### 基本形状

```javascript
// 立方体 - 宽度、高度、深度、宽度分段数、高度分段数、深度分段数
new THREE.BoxGeometry(1, 1, 1, 1, 1, 1);

// 球体 - 半径、经向分段数、纬向分段数、起始经度、经度范围、起始纬度、纬度范围
new THREE.SphereGeometry(1, 32, 32);
new THREE.SphereGeometry(1, 32, 32, 0, Math.PI * 2, 0, Math.PI); // 整球
new THREE.SphereGeometry(1, 32, 32, 0, Math.PI); // 半球

// 平面 - 宽度、高度、宽度分段数、高度分段数
new THREE.PlaneGeometry(10, 10, 1, 1);

// 圆 - 半径、分段数、起始角度、角度范围
new THREE.CircleGeometry(1, 32);
new THREE.CircleGeometry(1, 32, 0, Math.PI); // 半圆

// 圆柱 - 顶部半径、底部半径、高度、经向分段数、高度分段数、是否开放两端
new THREE.CylinderGeometry(1, 1, 2, 32, 1, false);
new THREE.CylinderGeometry(0, 1, 2, 32); // 锥体
new THREE.CylinderGeometry(1, 1, 2, 6); // 六棱柱

// 圆锥 - 半径、高度、经向分段数、高度分段数、是否开放顶端
new THREE.ConeGeometry(1, 2, 32, 1, false);

// 托里线环 - 大半径、管半径、经向分段数、管状分段数、弧度
new THREE.TorusGeometry(1, 0.4, 16, 100);

// 托里结环 - 大半径、管半径、管状分段数、经向分段数、p、q
new THREE.TorusKnotGeometry(1, 0.4, 100, 16, 2, 3);

// 环形 - 内半径、外半径、经向分段数、纬向分段数
new THREE.RingGeometry(0.5, 1, 32, 1);
```

### 高级形状

```javascript
// 胶囊体 - 半径、长度、端部分段数、经向分段数
new THREE.CapsuleGeometry(0.5, 1, 4, 8);

// 十二面体 - 半径、细节等级
new THREE.DodecahedronGeometry(1, 0);

// 二十面体 - 半径、细节等级（0 表示 20 个面，数值越高越平滑）
new THREE.IcosahedronGeometry(1, 0);

// 八面体 - 半径、细节等级
new THREE.OctahedronGeometry(1, 0);

// 四面体 - 半径、细节等级
new THREE.TetrahedronGeometry(1, 0);

// 多面体 - 顶点、索引、半径、细节等级
const vertices = [1, 1, 1, -1, -1, 1, -1, 1, -1, 1, -1, -1];
const indices = [2, 1, 0, 0, 3, 2, 1, 3, 0, 2, 3, 1];
new THREE.PolyhedronGeometry(vertices, indices, 1, 0);
```

### 基于路径的形状

```javascript
// Lathe - 点[], 段数, 起始角度, 扫描角度
const points = [
  new THREE.Vector2(0, 0),
  new THREE.Vector2(0.5, 0),
  new THREE.Vector2(0.5, 1),
  new THREE.Vector2(0, 1),
];
new THREE.LatheGeometry(points, 32);

// Extrude - 形状, 选项
const shape = new THREE.Shape();
shape.moveTo(0, 0);
shape.lineTo(1, 0);
shape.lineTo(1, 1);
shape.lineTo(0, 1);
shape.lineTo(0, 0);

const extrudeSettings = {
  段数: 2,
  拉伸深度: 1,
  是否启用倒角: true,
  倒角厚度: 0.1,
  倒角大小: 0.1,
  倒角分段数: 3,
};
new THREE.ExtrudeGeometry(shape, extrudeSettings);

// Tube - 路径, 管状分段数, 半径, 径向分段数, 是否闭合
const curve = new THREE.CatmullRomCurve3([
  new THREE.Vector3(-1, 0, 0),
  new THREE.Vector3(0, 1, 0),
  new THREE.Vector3(1, 0, 0),
]);
new THREE.TubeGeometry(curve, 64, 0.2, 8, false);
```

### 文字几何体

```javascript
import { FontLoader } from "three/examples/jsm/loaders/FontLoader.js";
import { TextGeometry } from "three/examples/jsm/geometries/TextGeometry.js";

const loader = new FontLoader();
loader.load("fonts/helvetiker_regular.typeface.json", (font) => {
  const geometry = new TextGeometry("Hello", {
    字体: font,
    大小: 1,
    深度: 0.2, // 在旧版本中称为“高度”
    曲线分段数: 12,
    是否启用倒角: true,
    倒角厚度: 0.03,
    倒角大小: 0.02,
    倒角分段数: 5,
  });

  // 居中文字
  geometry.computeBoundingBox();
  geometry.center();

const mesh = new THREE.Mesh(geometry, material);
scene.add(mesh);
});
```

## BufferGeometry

所有几何体的基类。为提高 GPU 效率，数据以类型化数组的形式存储。

### 自定义 BufferGeometry

```javascript
const geometry = new THREE.BufferGeometry();

// 顶点（每个顶点包含 3 个浮点数：x、y、z）
const vertices = new Float32Array([
  -1,
  -1,
  0, // 顶点 0
  1,
  -1,
  0, // 顶点 1
  1,
  1,
  0, // 顶点 2
  -1,
  1,
  0, // 顶点 3
]);
geometry.setAttribute("position", new THREE.BufferAttribute(vertices, 3));
// 索引（用于索引几何体——重用顶点）
const indices = new Uint16Array([
  0,
  1,
  2, // 三角形1
  0,
  2,
  3, // 三角形2
]);
geometry.setIndex(new THREE.BufferAttribute(indices, 1));

// 法线（光照所需）
const normals = new Float32Array([0, 0, 1, 0, 0, 1, 0, 0, 1, 0, 0, 1]);
geometry.setAttribute("normal", new THREE.BufferAttribute(normals, 3));

// UV坐标（用于贴图）
const uvs = new Float32Array([0, 0, 1, 0, 1, 1, 0, 1]);
geometry.setAttribute("uv", new THREE.BufferAttribute(uvs, 2));

// 颜色（顶点颜色）
const colors = new Float32Array([
  1,
  0,
  0, // 红色
  0,
  1,
  0, // 绿色
  0,
  0,
  1, // 蓝色
  1,
  1,
  0, // 黄色
]);
geometry.setAttribute("color", new THREE.BufferAttribute(colors, 3));
// 使用时：material.vertexColors = true
```

### BufferAttribute 类型

```javascript
// 常用属性类型
new THREE.BufferAttribute(array, itemSize);

// 各种 TypedArray 选项
new Float32Array(count * itemSize); // 位置、法线、UV
new Uint16Array(count); // 索引（最多 65535 个顶点）
new Uint32Array(count); // 索引（更大规模的网格）
new Uint8Array(count * itemSize); // 颜色（范围 0-255）

// 每项大小
// 位置：3（x, y, z）
// 法线：3（x, y, z）
// UV：2（u, v）
// 颜色：3（r, g, b）或 4（r, g, b, a）
// 索引：1
```

### 修改 BufferGeometry

```javascript
const positions = geometry.attributes.position;

// 修改顶点
positions.setXYZ(index, x, y, z);

// 访问顶点
const x = positions.getX(index);
const y = positions.getY(index);
const z = positions.getZ(index);

// GPU更新标志
positions.needsUpdate = true;

// 在位置变化后重新计算法线
geometry.computeVertexNormals();

// 在变化后重新计算包围盒和包围球
geometry.computeBoundingBox();
geometry.computeBoundingSphere();
```

### 交错缓冲（高级）

```javascript
// 针对大型网格更高效的内存布局
const interleavedBuffer = new THREE.InterleavedBuffer(
  new Float32Array([
    // pos.x, pos.y, pos.z, uv.u, uv.v（每个顶点重复）
    -1, -1, 0, 0, 0, 1, -1, 0, 1, 0, 1, 1, 0, 1, 1, -1, 1, 0, 0, 1,
  ]),
  5, // 步长（每个顶点的浮点数个数）
);

geometry.setAttribute(
  "position",
  new THREE.InterleavedBufferAttribute(interleavedBuffer, 3, 0),
); // 大小为3，偏移量为0
geometry.setAttribute(
  "uv",
  new THREE.InterleavedBufferAttribute(interleavedBuffer, 2, 3),
); // 大小为2，偏移量为3
```

## EdgesGeometry 和 WireframeGeometry

```javascript
// 边线（仅硬边）
const edges = new THREE.EdgesGeometry(boxGeometry, 15); // 15为临界角度
const edgeMesh = new THREE.LineSegments(
  edges,
  new THREE.LineBasicMaterial({ color: 0xffffff }),
);

// 线框（所有三角形）
const wireframe = new THREE.WireframeGeometry(boxGeometry);
const wireMesh = new THREE.LineSegments(
  wireframe,
  new THREE.LineBasicMaterial({ color: 0xffffff }),
);
```

## 点

```javascript
// 创建点云
const geometry = new THREE.BufferGeometry();
const positions = new Float32Array(1000 * 3);

for (let i = 0; i < 1000; i++) {
  positions[i * 3] = (Math.random() - 0.5) * 10;
  positions[i * 3 + 1] = (Math.random() - 0.5) * 10;
  positions[i * 3 + 2] = (Math.random() - 0.5) * 10;
}

geometry.setAttribute("position", new THREE.BufferAttribute(positions, 3));

const material = new THREE.PointsMaterial({
  size: 0.1,
  sizeAttenuation: true, // 大小随距离衰减
  color: 0xffffff,
});

const points = new THREE.Points(geometry, material);
scene.add(points);
```

## 线
```javascript
// 线（连接的点）
const points = [
  new THREE.Vector3(-1, 0, 0),
  new THREE.Vector3(0, 1, 0),
  new THREE.Vector3(1, 0, 0),
];
const geometry = new THREE.BufferGeometry().setFromPoints(points);
const line = new THREE.Line(
  geometry,
  new THREE.LineBasicMaterial({ color: 0xff0000 }),
);

// 线环（闭合环）
const loop = new THREE.LineLoop(geometry, material);

// 线段（点对）
const segmentsGeometry = new THREE.BufferGeometry();
segmentsGeometry.setAttribute(
  "position",
  new THREE.BufferAttribute(
    new Float32Array([
      -1,
      0,
      0,
      0,
      1,
      0, // 段1
      0,
      1,
      0,
      1,
      0,
      0, // 段2
    ]),
    3,
  ),
);
const segments = new THREE.LineSegments(segmentsGeometry, material);
```

## InstancedMesh

高效渲染同一几何体的多个实例。

```javascript
const geometry = new THREE.BoxGeometry(1, 1, 1);
const material = new THREE.MeshStandardMaterial({ color: 0x00ff00 });
const count = 1000;

const instancedMesh = new THREE.InstancedMesh(geometry, material, count);

// 设置每个实例的变换
const dummy = new THREE.Object3D();
const matrix = new THREE.Matrix4();

for (let i = 0; i < count; i++) {
  dummy.position.set(
    (Math.random() - 0.5) * 20,
    (Math.random() - 0.5) * 20,
    (Math.random() - 0.5) * 20,
  );
  dummy.rotation.set(Math.random() * Math.PI, Math.random() * Math.PI, 0);
  dummy.scale.setScalar(0.5 + Math.random());
  dummy.updateMatrix();

  instancedMesh.setMatrixAt(i, dummy.matrix);
}

// 标记 GPU 需要更新
instancedMesh.instanceMatrix.needsUpdate = true;

// 可选：为每个实例设置颜色
instancedMesh.instanceColor = new THREE.InstancedBufferAttribute(
  new Float32Array(count * 3),
  3,
);
for (let i = 0; i < count; i++) {
  instancedMesh.setColorAt(
    i,
    new THREE.Color(Math.random(), Math.random(), Math.random()),
  );
}
instancedMesh.instanceColor.needsUpdate = true;

scene.add(instancedMesh);
```

### 运行时更新实例

```javascript
// 更新单个实例
const matrix = new THREE.Matrix4();
instancedMesh.getMatrixAt(index, matrix);
// 修改矩阵...
instancedMesh.setMatrixAt(index, matrix);
instancedMesh.instanceMatrix.needsUpdate = true;

// 使用射线检测与实例化网格
const intersects = raycaster.intersectObject(instancedMesh);
if (intersects.length > 0) {
  const instanceId = intersects[0].instanceId;
}
```

## InstancedBufferGeometry（进阶）

用于自定义每个实例的属性，超出变换和颜色之外。

```javascript
const geometry = new THREE.InstancedBufferGeometry();
geometry.copy(new THREE.BoxGeometry(1, 1, 1));

// 添加每个实例的属性
const offsets = new Float32Array(count * 3);
for (let i = 0; i < count; i++) {
  offsets[i * 3] = Math.random() * 10;
  offsets[i * 3 + 1] = Math.random() * 10;
  offsets[i * 3 + 2] = Math.random() * 10;
}
geometry.setAttribute("offset", new THREE.InstancedBufferAttribute(offsets, 3));

// 在着色器中使用
// attribute vec3 offset;
// vec3 transformed = position + offset;
```

## 几何工具

```javascript
import * as BufferGeometryUtils from "three/examples/jsm/utils/BufferGeometryUtils.js";

// 合并几何体（必须具有相同的属性）
const merged = BufferGeometryUtils.mergeGeometries([geo1, geo2, geo3]);

// 带分组的合并（适用于多材质）
const merged = BufferGeometryUtils.mergeGeometries([geo1, geo2], true);

// 计算切线（法线贴图所需）
BufferGeometryUtils.computeTangents(geometry);

// 交错存储属性以提升性能
const interleaved = BufferGeometryUtils.interleaveAttributes([
  geometry.attributes.position,
  geometry.attributes.normal,
  geometry.attributes.uv,
]);
```

## 常见模式

### 将几何体居中

```javascript
geometry.computeBoundingBox();
geometry.center(); // 将顶点移动到原点
```

### 缩放以适应
```javascript
geometry.computeBoundingBox();
const size = new THREE.Vector3();
geometry.boundingBox.getSize(size);
const maxDim = Math.max(size.x, size.y, size.z);
geometry.scale(1 / maxDim, 1 / maxDim, 1 / maxDim);
```

### 克隆与变换

```javascript
const clone = geometry.clone();
clone.rotateX(Math.PI / 2);
clone.translate(0, 1, 0);
clone.scale(2, 2, 2);
```

### 形变目标

```javascript
// 基础几何体
const geometry = new THREE.BoxGeometry(1, 1, 1, 4, 4, 4);

// 创建形变目标
const morphPositions = geometry.attributes.position.array.slice();
for (let i = 0; i < morphPositions.length; i += 3) {
  morphPositions[i] *= 2; // X轴缩放
  morphPositions[i + 1] *= 0.5; // Y轴压缩
}

geometry.morphAttributes.position = [
  new THREE.BufferAttribute(new Float32Array(morphPositions), 3),
];

const mesh = new THREE.Mesh(geometry, material);
mesh.morphTargetInfluences[0] = 0.5; // 50%混合
```

## 性能优化建议

1. **使用索引化几何体**：通过索引重用顶点
2. **合并静态网格**：使用 `mergeGeometries` 减少绘制调用
3. **使用 InstancedMesh**：适用于大量相同对象
4. **选择合适的细分段数**：段数越多越平滑，但性能越低
5. **释放未使用的几何体**：`geometry.dispose()`

```javascript
// 常见用途的合理细分段数
new THREE.SphereGeometry(1, 32, 32); // 良好质量
new THREE.SphereGeometry(1, 64, 64); // 高质量
new THREE.SphereGeometry(1, 16, 16); // 性能优先

// 使用完毕后释放
geometry.dispose();
```

## 参考资料

- `threejs-fundamentals` - 场景搭建与 Object3D
- `threejs-materials` - 网格的材质类型
- `threejs-shaders` - 自定义顶点处理