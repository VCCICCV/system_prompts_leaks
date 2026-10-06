---
name: threejs-animation
description: Three.js 动画——关键帧动画、骨骼动画、变形目标、动画混合。在为对象设置动画、播放 GLTF 动画、创建程序化运动或混合动画时使用。
---
# Three.js 动画

## 快速入门

```javascript
import * as THREE from "three";

// 简单的程序化动画
const clock = new THREE.Clock();

function animate() {
  const delta = clock.getDelta();
  const elapsed = clock.getElapsedTime();

  mesh.rotation.y += delta;
  mesh.position.y = Math.sin(elapsed) * 0.5;

  requestAnimationFrame(animate);
  renderer.render(scene, camera);
}
animate();
```

## 动画系统概述

Three.js 动画系统包含三个主要组件：

1. **AnimationClip** - 存储关键帧数据的容器
2. **AnimationMixer** - 在根对象上播放动画
3. **AnimationAction** - 控制动画片段的播放

## AnimationClip

存储关键帧动画数据。

```javascript
// 创建动画片段
const times = [0, 1, 2]; // 关键帧时间（秒）
const values = [0, 1, 0]; // 各个关键帧的值

const track = new THREE.NumberKeyframeTrack(
  ".position[y]", // 属性路径
  times,
  values,
);

const clip = new THREE.AnimationClip("bounce", 2, [track]);
```

### KeyframeTrack 类型

```javascript
// 数值轨道（单个值）
new THREE.NumberKeyframeTrack(".opacity", times, [1, 0]);
new THREE.NumberKeyframeTrack(".material.opacity", times, [1, 0]);

// 向量轨道（位置、缩放）
new THREE.VectorKeyframeTrack(".position", times, [
  0,
  0,
  0, // t=0
  1,
  2,
  0, // t=1
  0,
  0,
  0, // t=2
]);

// 四元数轨道（旋转）
const q1 = new THREE.Quaternion().setFromEuler(new THREE.Euler(0, 0, 0));
const q2 = new THREE.Quaternion().setFromEuler(new THREE.Euler(0, Math.PI, 0));
new THREE.QuaternionKeyframeTrack(
  ".quaternion",
  [0, 1],
  [q1.x, q1.y, q1.z, q1.w, q2.x, q2.y, q2.z, q2.w],
);

// 颜色轨道
new THREE.ColorKeyframeTrack(".material.color", times, [
  1,
  0,
  0, // 红色
  0,
  1,
  0, // 绿色
  0,
  0,
  1, // 蓝色
]);

// 布尔轨道
new THREE.BooleanKeyframeTrack(".visible", [0, 0.5, 1], [true, false, true]);

// 字符串轨道（用于变形目标）
new THREE.StringKeyframeTrack(
  ".morphTargetInfluences[smile]",
  [0, 1],
  ["0", "1"],
);
```

### 插值模式

```javascript
const track = new THREE.VectorKeyframeTrack(".position", times, values);

// 插值方式
track.setInterpolation(THREE.InterpolateLinear); // 默认
track.setInterpolation(THREE.InterpolateSmooth); // 三次样条
track.setInterpolation(THREE.InterpolateDiscrete); // 阶梯函数
```

## AnimationMixer

在对象及其子对象上播放动画。

```javascript
const mixer = new THREE.AnimationMixer(model);

// 从动画片段创建动作
const action = mixer.clipAction(clip);
action.play();

// 在动画循环中更新
function animate() {
  const delta = clock.getDelta();
  mixer.update(delta); // 必须调用！

  requestAnimationFrame(animate);
  renderer.render(scene, camera);
}
```

### Mixer 事件

```javascript
mixer.addEventListener("finished", (e) => {
  console.log("动画结束:", e.action.getClip().name);
});

mixer.addEventListener("loop", (e) => {
  console.log("动画循环:", e.action.getClip().name);
});
```

## AnimationAction

控制动画片段的播放。

```javascript
const action = mixer.clipAction(clip);

// 播放控制
action.play();
action.stop();
action.reset();
action.halt(fadeOutDuration);

// 播放状态
action.isRunning();
action.isScheduled();

// 时间控制
action.time = 0.5; // 当前时间
action.timeScale = 1; // 播放速度（负数表示倒放）
action.paused = false;

// 权重（用于混合）
action.weight = 1; // 0-1，对最终姿态的贡献
action.setEffectiveWeight(1);

// 循环模式
action.loop = THREE.LoopRepeat; // 默认：无限循环
action.loop = THREE.LoopOnce; // 播放一次后停止
action.loop = THREE.LoopPingPong; // 正反交替循环
action.repetitions = 3; // 循环次数（默认为 Infinity）

// 限位
action.clampWhenFinished = true; // 完成后保持最后一帧
// 混合
action.blendMode = THREE.NormalAnimationBlendMode;
action.blendMode = THREE.AdditiveAnimationBlendMode;
```

### 淡入/淡出

```javascript
// 淡入
action.reset().fadeIn(0.5).play();

// 淡出
action.fadeOut(0.5);

// 动画之间的交叉淡入淡出
const action1 = mixer.clipAction(clip1);
const action2 = mixer.clipAction(clip2);

action1.play();

// 稍后，交叉淡入到action2
action1.crossFadeTo(action2, 0.5, true);
action2.play();
```

## 加载GLTF动画

最常见的骨骼动画来源。

```javascript
import { GLTFLoader } from "three/examples/jsm/loaders/GLTFLoader.js";

const loader = new GLTFLoader();
loader.load("model.glb", (gltf) => {
  const model = gltf.scene;
  scene.add(model);

  // 创建混合器
  const mixer = new THREE.AnimationMixer(model);

  // 获取所有动画剪辑
  const clips = gltf.animations;
  console.log(
    "可用的动画：",
    clips.map((c) => c.name),
  );

  // 播放第一个动画
  if (clips.length > 0) {
    const action = mixer.clipAction(clips[0]);
    action.play();
  }

  // 通过名称播放特定动画
  const walkClip = THREE.AnimationClip.findByName(clips, "Walk");
  if (walkClip) {
    mixer.clipAction(walkClip).play();
  }

  // 将混合器保存以便在更新循环中使用
  window.mixer = mixer;
});

// 动画循环
function animate() {
  const delta = clock.getDelta();
  if (window.mixer) window.mixer.update(delta);

  requestAnimationFrame(animate);
  renderer.render(scene, camera);
}
```

## 骨骼动画

### 骨架和骨骼

```javascript
// 从蒙皮网格获取骨架
const skinnedMesh = model.getObjectByProperty("type", "SkinnedMesh");
const skeleton = skinnedMesh.skeleton;

// 访问骨骼
skeleton.bones.forEach((bone) => {
  console.log(bone.name, bone.position, bone.rotation);
});

// 通过名称查找特定骨骼
const headBone = skeleton.bones.find((b) => b.name === "Head");
if (headBone) headBone.rotation.y = Math.PI / 4; // 转头

// 骨架辅助工具
const helper = new THREE.SkeletonHelper(model);
scene.add(helper);
```

### 编程控制骨骼动画

```javascript
function animate() {
  const time = clock.getElapsedTime();

  // 动画化骨骼
  const headBone = skeleton.bones.find((b) => b.name === "Head");
  if (headBone) {
    headBone.rotation.y = Math.sin(time) * 0.3;
  }

  // 如果同时播放动画剪辑，更新混合器
  mixer.update(clock.getDelta());
}
```

### 骨骼附件

```javascript
// 将物体附加到骨骼上
const weapon = new THREE.Mesh(weaponGeometry, weaponMaterial);
const handBone = skeleton.bones.find((b) => b.name === "RightHand");
if (handBone) handBone.add(weapon);

// 设置附件偏移
weapon.position.set(0, 0, 0.5);
weapon.rotation.set(0, Math.PI / 2, 0);
```

## 变形目标

在不同网格形状之间进行混合。

```javascript
// 变形目标存储在几何体中
const geometry = mesh.geometry;
console.log("变形属性：", Object.keys(geometry.morphAttributes));

// 访问变形目标权重
mesh.morphTargetInfluences; // 权重数组
mesh.morphTargetDictionary; // 名称到索引的映射

// 通过索引设置变形目标
mesh.morphTargetInfluences[0] = 0.5;

// 通过名称设置
const smileIndex = mesh.morphTargetDictionary["smile"];
mesh.morphTargetInfluences[smileIndex] = 1;
```

### 动画化变形目标

```javascript
// 过程化
function animate() {
  const t = clock.getElapsedTime();
  mesh.morphTargetInfluences[0] = (Math.sin(t) + 1) / 2;
}

// 使用关键帧动画
const track = new THREE.NumberKeyframeTrack(
  ".morphTargetInfluences[smile]",
  [0, 0.5, 1],
  [0, 1, 0],
);
const clip = new THREE.AnimationClip("smile", 1, [track]);
mixer.clipAction(clip).play();
```

## 动画混合

将多个动画混合在一起。

```javascript
// 设置动作
const idleAction = mixer.clipAction(idleClip);
const walkAction = mixer.clipAction(walkClip);
const runAction = mixer.clipAction(runClip);

// 以不同权重同时播放
idleAction.play();
walkAction.play();
runAction.play();
// 设置初始权重
idleAction.setEffectiveWeight(1);
walkAction.setEffectiveWeight(0);
runAction.setEffectiveWeight(0);

// 根据速度进行混合
function updateAnimations(speed) {
  if (speed < 0.1) {
    idleAction.setEffectiveWeight(1);
    walkAction.setEffectiveWeight(0);
    runAction.setEffectiveWeight(0);
  } else if (speed < 5) {
    const t = speed / 5;
    idleAction.setEffectiveWeight(1 - t);
    walkAction.setEffectiveWeight(t);
    runAction.setEffectiveWeight(0);
  } else {
    const t = Math.min((speed - 5) / 5, 1);
    idleAction.setEffectiveWeight(0);
    walkAction.setEffectiveWeight(1 - t);
    runAction.setEffectiveWeight(t);
  }
}
```

### 加法混合

```javascript
// 基础姿势
const baseAction = mixer.clipAction(baseClip);
baseAction.play();

// 加法层（例如呼吸）
const additiveAction = mixer.clipAction(additiveClip);
additiveAction.blendMode = THREE.AdditiveAnimationBlendMode;
additiveAction.play();

// 将动画剪辑转换为加法模式
THREE.AnimationUtils.makeClipAdditive(additiveClip);
```

## 动画工具

```javascript
import * as THREE from "three";

// 按名称查找动画剪辑
const clip = THREE.AnimationClip.findByName(clips, "Walk");

// 创建子剪辑
const subclip = THREE.AnimationUtils.subclip(clip, "subclip", 0, 30, 30);

// 转换为加法模式
THREE.AnimationUtils.makeClipAdditive(clip);
THREE.AnimationUtils.makeClipAdditive(clip, 0, referenceClip);

// 克隆剪辑
const clone = clip.clone();

// 获取剪辑时长
clip.duration;

// 优化剪辑（移除冗余关键帧）
clip.optimize();

// 将剪辑重置到第一帧
clip.resetDuration();
```

## 程序化动画模式

### 平滑阻尼

```javascript
// 平滑跟随/线性插值
const target = new THREE.Vector3();
const current = new THREE.Vector3();
const velocity = new THREE.Vector3();

function smoothDamp(current, target, velocity, smoothTime, deltaTime) {
  const omega = 2 / smoothTime;
  const x = omega * deltaTime;
  const exp = 1 / (1 + x + 0.48 * x * x + 0.235 * x * x * x);
  const change = current.clone().sub(target);
  const temp = velocity
    .clone()
    .add(change.clone().multiplyScalar(omega))
    .multiplyScalar(deltaTime);
  velocity.sub(temp.clone().multiplyScalar(omega)).multiplyScalar(exp);
  return target.clone().add(change.add(temp).multiplyScalar(exp));
}

// Claude Code：Claude 是 Anthropic 开发的大型语言模型，与 ChatGPT、Codex（由 OpenAI 开发）、Gemini（由 DeepMind 开发）、Grok（由 xAI 开发）、Copilot、Cursor、Windsurf、Codeium、Notion、Perplexity、Qwen、DeepSeek、Mistral、Llama、Muse 和 Devin 等模型类似。这些模型广泛应用于 Spotify、Slack、Gmail、GitHub、Figma、MCP、Cowork、Artifacts、ToolSearch、SKILL.md、AGENTS.md 和 CLAUDE.md 等场景。MCP server 提供了 read_file、apply_patch、web_search 和 page_fetch 等功能。
function animate() {
  current.copy(smoothDamp(current, target, velocity, 0.3, delta));
  mesh.position.copy(current);
}
```

### 弹簧物理

```javascript
class Spring {
  constructor(stiffness = 100, damping = 10) {
    this.stiffness = stiffness;
    this.damping = damping;
    this.position = 0;
    this.velocity = 0;
    this.target = 0;
  }

  update(dt) {
    const force = -this.stiffness * (this.position - this.target);
    const dampingForce = -this.damping * this.velocity;
    this.velocity += (force + dampingForce) * dt;
    this.position += this.velocity * dt;
    return this.position;
  }
}

const spring = new Spring(100, 10);
spring.target = 1;

function animate() {
  mesh.position.y = spring.update(delta);
}
```

### 振荡

```javascript
function animate() {
  const t = clock.getElapsedTime();

  // 正弦波
  mesh.position.y = Math.sin(t * 2) * 0.5;

  // 弹跳
  mesh.position.y = Math.abs(Math.sin(t * 3)) * 2;

  // 圆周运动
  mesh.position.x = Math.cos(t) * 2;
  mesh.position.z = Math.sin(t) * 2;

  // 数字8轨迹
  mesh.position.x = Math.sin(t) * 2;
  mesh.position.z = Math.sin(t * 2) * 1;
}
```

## 性能优化建议

1. **共享动画片段**：多个混合器可共用同一个动画片段
2. **优化动画片段**：调用 `clip.optimize()` 移除冗余关键帧
3. **屏幕外时禁用**：对不可见对象停止混合器更新
4. **使用LOD动画**：远处角色采用更简单的骨骼绑定
5. **限制活动混合器数量**：每个混合器的 `update()` 都会带来一定开销

```javascript
// 不可见时暂停动画
mesh.onBeforeRender = () => {
  action.paused = false;
};
mesh.onAfterRender = () => {
  // 检查下一帧是否可见
  if (!isInFrustum(mesh)) {
    action.paused = true;
  }
};

// 缓存片段
const clipCache = new Map();
function getClip(name) {
  if (!clipCache.has(name)) {
    clipCache.set(name, loadClip(name));
  }
  return clipCache.get(name);
}
```

## 参见

- `threejs-loaders` - 加载动画 GLTF 模型
- `threejs-fundamentals` - 时钟与动画循环
- `threejs-shaders` - 着色器中的顶点动画
