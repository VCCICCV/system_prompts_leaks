---
name: "printify"
description: "使用 Printify 浏览商品目录数据、管理店铺和产品，并查看或创建订单。"
icon: "printify"
metadata: { "不包含在提示中": 假 }
---
# Printify（按需打印）

## 用途
使用 `printify` CLI 管理店铺、产品、上传文件、订单以及目录查询。

## 工具
直接通过系统 `PATH` 使用已安装的 CLI：

```sh
printify <子命令> [选项]
```

全局标志：
- `--timeout-secs <秒>`（可选，默认 30）：HTTP 超时时间。

核心命令：
- `status`
- `authorize-url`
- `verify`
- `set-token`（从标准输入读取令牌）
- `disconnect`
- `shops`
- `products --shop-id <id> [--product-id <id>]`
- `create-product --shop-id <id> --json '<json>'`
- `update-product --shop-id <id> --product-id <id> --json '<json>'`
- `delete-product --shop-id <id> --product-id <id>`
- `publish-product --shop-id <id> --product-id <id>`
- `orders --shop-id <id> [--order-id <id>]`
- `create-order --shop-id <id> --json '<json>'`
- `upload --file <路径>`
- `blueprints [--blueprint-id <id>]`
- `print-providers --blueprint-id <id>`
- `variants --blueprint-id <id> --provider-id <id>`
- `shipping --blueprint-id <id> --provider-id <id>`

JSON 输出格式：
- `status`：解析 `ok`、`status`、`connect_url`、`disconnect_url` 和 `reason`
- `authorize-url`：解析 `ok`、`authorize_url` 和 `connect_url`
- `verify`：解析 `ok`、`action` 和 `error`
- `set-token`：解析 `ok`、`status` 和 `reason`
- `disconnect`：解析 `ok`、`action`、`config_path` 和 `removed`
- 解析 `ok`、`status`、`body` 和 `error`
- 返回结果保留 Printify 的原始时间戳，并为记录的创建/更新时间以及订单的生产、履行、送达或取消时间添加语义化的 UTC 格式和用户本地时间格式。

## 认证
首次使用设置流程：
1. 运行 `printify status` 检查连接状态。
2. 如果未连接且存在 `connect_url`，请将 `<connect_url>` 替换为返回的 URL，并以如下 Markdown 格式分享链接：`[Connect Printify](<connect_url>)`；切勿单独粘贴原始 URL。
3. 如果缺少 `connect_url`，请运行 `printify authorize-url`，并以相同方式分享返回的 `connect_url`。
4. 用户可在 `https://printify.com/app/account/api` 生成令牌。
5. 如果用户通过 CLI 流程提供令牌，请仅通过 `printify set-token` 存储令牌，切勿直接写入认证文件。
6. 设置完成后，如需确认令牌有效，请运行 `printify verify`。
7. 切勿在聊天输出中打印访问令牌。

## 操作规范
1. 在调用 API 之前，务必先检查 `printify status`。如果未连接，请引导用户按照上述认证流程进行设置。
2. 执行与特定店铺相关的操作前，先列出所有店铺以获取正确的 `shop_id`。
3. 创建、更新或发布产品前，请与用户确认。删除产品时，若请求明确无误，可无需额外确认直接执行。
4. 创建订单前，请与用户确认（订单可能触发扣款）。
5. 浏览目录时，应先从 `blueprints` 入手，再逐步深入到 `print-providers` 和 `variants`。
6. 遇到 HTTP 401 错误时，请告知用户其令牌可能已过期，并建议其前往 `https://printify.com/app/account/api` 重新生成令牌。