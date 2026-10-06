# 资产地图运行时

将 `@meta/maps` 打包成一个独立的脚本，生成的 Web 资产可以通过两个普通的 `<script>` 和 `<link>` 标签加载。由于资产中不包含打包工具和 React，因此无法直接使用组件 API。

构建输出会被内置于 bundle 中，路径为 `/opt/hatch/skills/artifacts/map-runtime/dist/`：

| 文件 | 用途 |
|---|---|
| `hatch-maps.js` | IIFE 暴露 `window.HatchMaps`——提供 `mountMap` 和 `isMetaMapSupported` 方法 |
| `hatch-maps.css` | MapLibre 控件及画布的样式 |

**两者缺一不可。** 如果只加载脚本而不加载样式表，瓦片会相互叠加，导致地图显示异常而非完全缺失。

## 在资产中使用

首先将这两个文件复制到资产自身的资源目录中。bundle 路径是构建虚拟机上的文件系统位置；而发布后的资产是从其自身源站提供服务的，因此在页面中引用 `/opt/hatch/...` 会导致 404 错误，并使 `HatchMaps` 未定义。

```bash
cp /opt/hatch/skills/artifacts/map-runtime/dist/hatch-maps.{js,css} <assets>/
```

```html
<link rel="stylesheet" href="assets/hatch-maps.css">
<script src="assets/hatch-maps.js"></script>
<div id="map" style="height: 420px"></div>
<script>
  HatchMaps.mountMap(document.getElementById('map'), {
    clientId: 'artifact_web',
    places: [
      {label: 'Chaotic Coffee', lat: 47.6588, lng: -117.426},
      {label: 'Mobius Discovery Center', lat: 47.6575, lng: -117.4231},
    ],
    onFatalError: () => {
      document.getElementById('map').textContent = '地图不可用';
    },
  });
</script>
```

容器必须具有明确的高度。如果地图位于高度为零的元素中，将不会渲染任何内容，也不会抛出错误。

`onFatalError` 在实际使用中并非可选项。它会在地图控件自身发生致命错误时触发，也会捕获地图初始化过程中抛出的任何异常——渲染阶段的异常会在 `mountMap` 内部被捕获，而不会冒泡到全局作用域。不同环境对地图的支持情况可能不同，同一个 bundle 在某些环境中能正常渲染，而在另一些环境中则不能。因此务必为回调函数提供一个可靠的回退方案——例如显示地点列表，或显示一段简短的“地图不可用”提示——因为构建时无法预先确定运行时所处的具体环境。

`_nc_client_caller` 固定为 `Muse_Artifact`；请将资产类型作为 `clientId` 传递，该值会映射为 `_nc_client_id`。这两者都是用于后端仪表盘分组的标识符，切勿为每个资产单独定义不同的值。

`locale` 和 `politicalView` 是可选的覆盖配置，示例中特意将其省略。样式、字形和瓦片请求均从用户的浏览器发出，因此地图后端会根据访问者的实际情况解析这些值；若手动设置，则会强制所有用户都采用同一套有争议边界的显示方式。只有在发现默认解析结果有误时，才应进行手动设置。

`rtlTextPluginUrl` 默认为 `false`，这意味着 MapLibre 的 RTL 文本插件未被注册，阿拉伯语、希伯来语和波斯语的标签会显示错位且反向。该库的默认实现通过 `import.meta.url` 加载内嵌资源，而本构建无法输出这种依赖（esbuild 在 `--format=iife` 模式下不支持资源加载），因此保持默认关闭是更合理的做法，而不是静默失败。如果页面需要展示使用 RTL 脚本的区域，并且能够从自身源站提供相关资源，则可以传入该插件的 URL。

地图控件会自动添加版权信息。这是强制性的，请勿隐藏、覆盖或重新实现。详情参见：`hatch-skills/skills/artifacts/references/maps.md`。

## 并非所有资产运行环境都支持地图渲染

矢量地图需要 WebGL 和 Web Worker，而并非所有资产打开的环境都能提供这些能力。在挂载前应先检查：

```js
const {supported, reason} = HatchMaps.isMetaMapSupported();
if (supported) {
  HatchMaps.mountMap(el, {
    clientId: 'artifact_web',
    places,
    onFatalError: () => renderPlaceListOnly(),
  });
} else {
  renderPlaceListOnly(); // reason 可能是 'webgl' 或 'worker'
}
```

**探针并非全部。** 即使表面通过了探针检测，一旦地图完成挂载，它仍然可能拒绝样式、字形和瓦片请求；这些请求是通过 `onFatalError` 而非探针传递的。`onFatalError` 也是 GL 上下文丢失以及构造过程中抛出异常时的兜底处理机制，而 `mountMap` 会捕获这些异常，避免它们传播到窗口层。将两者都指向同一个回退逻辑，无论哪个触发，页面降级的表现都是一致的。关于表面的具体实现细节，请参阅 `references/maps.md` 中的“挂载前先探查表面”部分。

在多个表面上测试地图变更——仅在一个地方渲染成功的构建，并不能说明其他地方的表现。

## 构建

```bash
bun build.mjs
```

`@meta/maps` 从 Metaccio 解析。`.npmrc` 和 `bunfig.toml` 文件仅对 `@meta` 前缀进行了作用域限定，因此本仓库中的其他所有 JS 依赖仍会从公共注册表解析。

Metaccio 通过网络位置而非令牌来认证企业主机。Buildkite 代理符合这一条件，且 `bundle-setup/scripts/build.sh` 脚本会在 `fwdproxy` 解析成功时为构建容器配置主机网络。运行 `x2pagentd` 的企业笔记本电脑则可通过 `127.0.0.1:10054` 经由 HTTP 访问同一注册表；请勿将这些代理配置写入已提交的配置文件中，因为 CI 环境中没有代理守护进程在监听。

补充说明：此包不携带任何运行时依赖，也不应引入此类依赖。`@meta/maps` 将 `next` 和 `@opentelemetry/api` 声明为可选的 peer 依赖，且在模块作用域内并未导入它们——当其他模块已将 OpenTelemetry 注册到全局注册表（`Symbol.for('opentelemetry.js.api.1')`）时，本包会直接从全局注册表读取遥测数据。唯一的引用仅出现在 `mapTelemetry.d.ts` 中的一个仅用于类型的导入语句，该语句会被 `skipLibCheck` 忽略；如果某次变更导致该类型在 `src/` 目录中变为可访问状态，则它应归入 `devDependencies`，而绝不能放入 `dependencies`。