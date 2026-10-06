# TypeScript Web 工件运行时

`@hatch/space-sdk` 包的源代码，供所有 TypeScript Web 工件使用。

当前的 TypeScript Web 工件由 Web 工件构建子代理通过 Rust 的 `web_artifacts.build` 工具构建，该工具会调用 `hatch_spaces::build_pipeline::build_space`。其规范的源码根目录为 `workspace/ts-spaces/<slug>`。

## 目录结构

```
ts-runtime/
├── build.mjs           # 生成本地 Bun 运行时工件
├── cloudflare/         # 显式 Worker 构建/类型检查工具
│   └── tsconfig.worker.base.json
├── sdk/                # @hatch/space-sdk 包的源码
│   ├── package.json
│   ├── tsconfig.json
│   └── src/
│       ├── index.ts    # 服务器端接口（defineAction、Ctx、ActionsModule 等）
│       └── client.ts   # 浏览器端接口（createActionClient）
└── dist/               # 被 .gitignore 忽略；由 build.mjs 生成
    └── space-sdk.tgz   # 通过 `file:` 依赖被 vendored 到每个 TypeScript Web 工件中
```

## 构建

生成本地 Bun 运行时工件：

```bash
cd skills/spaces/ts-runtime
bun build.mjs
```

输出位于 `dist/space-sdk.tgz`。在打包构建时，该文件会被打入部署包，因此运行时 `/opt/hatch/skills/spaces/ts-runtime/dist/space-sdk.tgz` 存在；每个脚手架生成的空间的 `package.json` 都通过 `"@hatch/space-sdk": "file:/opt/hatch/skills/spaces/ts-runtime/dist/space-sdk.tgz"` 引用它。

## Cloudflare Worker 导出

Cloudflare 导出是显式的，目前尚未由常规的 `web_artifacts.build` 执行：

```bash
cd skills/spaces/ts-runtime
bun cloudflare/build-cloudflare.mjs \
  --space-dir "$JARVIS_HOME/workspace/ts-spaces/<slug>" \
  --out-dir /tmp/<slug>-cloudflare
```

导出工具会生成：

- `worker.js`：打包后的 Worker 动作分发器。
- `deploy-manifest.json`：Worker 源码、来自 `client/dist` 的客户端文件，以及来自 `drizzle/*.sql` 的 SQL 迁移脚本。

### 初始 Cloudflare 数据库种子

当本地 VM Web 工件首次共享到 Cloudflare 时，Cloudflare 部署清单可能会包含 `initialDbSnapshot` 字段：

```json
{
  "runtime": "hatch-ts-cloudflare-v1",
  "slug": "my-space",
  "name": "My Web Artifact",
  "workerJs": "...",
  "migrations": [],
  "clientFiles": [],
  "initialDbSnapshot": {
    "runtime": "hatch-local-sqlite-snapshot-v1",
    "shortcode": "abc123",
    "slug": "my-space",
    "exportedAtMs": 1725000000000,
    "schemaWatermark": null,
    "payload": {
      "kind": "sqlite_sql",
      "sql": "<sqlite dump sql>"
    }
  }
}
```

`initialDbSnapshot` 在首次发布时会从一个隔离的本地 SQLite 快照中初始化 D1 数据库。`initialBlobSnapshot` 会将相应的对象传输到 R2。控制平面会在上传前验证其运行时、短码和 slug；绝不会用较新的本地种子覆盖正在使用的数据库或存储桶。在共享期间，写操作会读取远程快照，而部署则会对 D1 应用迁移。取消共享时会停止写入，并在重新启用本地动作之前将 D1 和 R2 恢复到本地状态。有关准入、失败与恢复语义，请参阅 [共享状态契约](../../../../docs/spaces-shared-state.md)。

构建过程会生成一个针对 Worker 的 tsconfig，并根据仅适用于 Cloudflare 的 `@hatch/space-sdk` 垫片对空间的 `server/src/actions.ts` 进行类型检查。该垫片暴露了由 D1 支持的可移植 `ctx.db` 接口，而省略了仅限本地的 API，如 `ctx.inference`、`ctx.agent` 和 `ctx.emit`，因此不支持的动作代码会在 TypeScript 类型检查阶段就被捕获，而不是等到源码扫描时才报错。`SpaceDb` 故意未暴露 `transaction` 接口。

## Blob 存储

本地 TypeScript Web 工件的动作可以使用 `ctx.blobs` 来存储不属于 `ctx.db` 的二进制或不透明对象数据：生成的图片、缩略图、导出文件、附件、缓存的 API 文件、快照以及大体积负载。对于结构化/可查询的状态，请使用 `ctx.db`；当 UI 需要跨基于 Blob 的记录进行查询时，可在 `ctx.db` 中存储 Blob 键或可搜索的元数据。

本地 VM 运行时会在首次使用时惰性地初始化 Blob 存储：
```text
<spaceDir>/blobs/objects/<id[0:2]>/<id>   id = sha256(key); 分片存储
<spaceDir>/blobs/index.sqlite
```

`index.sqlite` 记录了 `key`、`content_type`、`size_bytes`、`etag`、
`visibility`、`created_at_ms`、`updated_at_ms` 和 `object_id`。`etag` 目前是存储字节的 SHA-256 指纹。对象字节存放在经过哈希和分片处理的 `object_id` 路径下，而不是 `objects/<key>`，因此键可以很长或包含保留字符，而不会超出文件系统名称的限制；键绝不会用作文件系统路径。在此变更之前写入的 Blob 的 `object_id` 为 null，仍从旧版的 `objects/<key>` 路径读取（无论哪种情况，提供的 URL 都是一个不透明的 base64url 令牌）。

示例：

```ts
await ctx.blobs.put("images/avatar.png", bytes, {
  contentType: "image/png",
});
const meta = await ctx.blobs.head("images/avatar.png");
const url = await ctx.blobs.getUrl("images/avatar.png", {
  expiresInSeconds: 600,
});
```

`getUrl()` 会请求运行时生成一个可获取的 Blob URL。本地 Muse VM Web 资产返回的是一个基于文档的 `./blobs/<key>` URL，由守护进程在 nginx bearer 认证下提供——使用相对路径是为了使其能够解析到 `/spaces/v2/<slug>/` 文档的基础路径，无论该路径位于源站根目录（生产环境）还是位于 `/backend/<sid>/` 反向代理前缀之后（注释工具）。Cloudflare Web 资产则返回一个基于 Worker 的 `./blobs/public/<key>` 或 `./blobs/private/<key>?token=...` URL，后端由每个 Web 资产绑定的 R2 存储桶提供支持，并以 `BUCKET` 作为挂载点，私有 Blob 使用 HMAC 签名的令牌进行保护。需要身份验证的下载会设置 `Cache-Control: no-store`，包括应用内标记为公开的 Blob。无论是 Cloudflare 还是 VM 提供的 Blob 响应都会携带 CSP 的 `sandbox` 和 `nosniff` 属性；附件可以显示被动内容，但不能执行与应用同源或拥有相同权限的脚本。

运行本地 Cloudflare 导出器测试的命令如下：

```bash
bun test --timeout 30000 cloudflare/build-cloudflare.test.mjs cloudflare/shared-state.test.ts
```