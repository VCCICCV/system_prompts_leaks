# save-to-spotify CLI 版本固定

此目录记录了 `save-to-spotify` 包装器所使用的官方 `save-to-spotify` 发布版本。Spotify 会按操作系统和架构发布 ZIP 格式的发布资产；Jarvis（par-msl/hatch-image）会引入其中的 linux/amd64 资产，提取 `save-to-spotify` 可执行文件，并将其作为可信载荷安装到 `/opt/hatch-image/vendor/save-to-spotify-cli/save-to-spotify`。

Rust 包装器（`skills/crates/save-to-spotify-cli`）是策略、权限分离与认证的边界：它通过一个不透明的 Sentinel 代理重用 Spotify 共享的 authd 所拥有的 PKCE 授权，对涉及内容的操作命令实施人工审核（HITL），清除调用方的认证环境变量，禁止 `update` 命令，将转发的参数限制在各子命令的白名单内（由 `validate_args` 进行校验），并委托给固定的可执行文件路径。该包装器不会对上游二进制文件进行 fork 或打补丁。完整的设计方案请参阅该 crate 的 `DESIGN.md` 文件。

## SOURCE.toml 字段说明
- `repo` / `rev` / `tag` / `version` — 上游仓库及确切的发布版本标识。
- `package` — 该版本标识所代表的软件包名称。
- `artifact` — 安装后的可信载荷名称。
- `release_url` — 官方发布页面。
- `release_asset` — 官方提供的 linux/amd64 资产（`.zip` 文件）。
- `release_asset_sha256` — 该资产的预期校验值（来自发布附带的 `.sha256` 文件）。
- `release_asset_archive_member` — 从 ZIP 文件中提取的可执行文件名。

## 更新流程
1. 确定新的官方发布版本。
2. 更新 `SOURCE.toml` 中的 `tag`、`rev`、`version`、`release_url`、`release_asset` 和 `release_asset_sha256` 字段。
3. 同步更新 hatch-image 引入的可执行文件及其校验值。
4. 对比上游的 `auth/` 和 `config/` 目录，检查 OAuth 及常量配置的变化（如 client_id、scope、认证与令牌 URL、重定向 URI/端口、后端 URL 等）；若相关常量发生变更，则同步更新包装器中的对应常量。
5. 重新审核 CLI 的命令行选项界面（`--help`），并与包装器的 `validate_args()` 白名单进行比对：为包装器新增或重命名的必要选项添加至白名单，并拒绝任何新增的读写/执行类、或用于凭据替换的“工具型”选项。
6. 在 hatch-extensions 中运行 `cargo fmt --all` 和 `cargo test -p save-to-spotify-cli`。
7. 执行一次线上冒烟测试：授权 → 上传 → 检查 `episodes status` 是否显示 READY。
8. 执行镜像载荷/打包/部署相关的验证流程，并将 Jarvis 的 `extensions.toml` 升级至最新的 hatch-extensions 版本。

回滚步骤：恢复 `SOURCE.toml` 中的版本标识及 hatch-image 可执行文件与校验值，并将 `extensions.toml` 回退至已知稳定的版本。

## 运维提示
后端可通过 `X-Min-CLI-Version` 响应头强制弃用过时的 CLI 版本。请持续关注发布动态，保持版本标识为最新，以免上传功能出现故障。