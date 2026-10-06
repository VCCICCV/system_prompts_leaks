---
name: "philips_hue"
description: "通过 Hue Remote API v2 控制飞利浦 Hue 智能灯、灯具组、场景和设备。"
icon: "hue_lights"
metadata: { "不包含在提示中": 假 }
---
# 飞利浦 Hue（智能照明）

## 用途
通过 Hue Remote CLIP API v2 控制飞利浦 Hue 智能灯、房间、场景、传感器和设备。

## 工具使用
使用以下命令：

```sh
philips-hue <子命令> [选项]
```

### 认证相关子命令
- `status` — 检查 OAuth 连接状态及桥接器的连接状态
- `authorize-url` — 返回飞利浦 Hue 的连接 URL
- `disconnect` — 断开与飞利浦 Hue 的连接

### 设置相关子命令
- `link-bridge` — 通过远程 API 与用户的 Hue 桥接器配对（自动完成，无需手动按按钮）

### 发现相关子命令
- `list-lights` — 列出所有灯具
- `list-rooms` — 列出所有房间
- `list-zones` — 列出所有区域
- `list-scenes [--room <room_id>]` — 列出场景（可指定房间）
- `list-devices` — 列出所有设备
- `list-sensors` — 列出所有传感器
- `list-buttons` — 列出所有遥控按钮

### 控制相关子命令
- `light --id <id> --on|--off` — 开启或关闭指定灯具
- `light --id <id> --brightness <0-100>` — 调节亮度（0–100）
- `light --id <id> --color <hex>` — 设置颜色（如 FF0000 表示红色）
- `light --id <id> --temperature <warm|cool|neutral|daylight|candle|mirek>` — 调节色温
- `light --id <id> --effect <effect>` — 设置灯光效果（如 candle、sparkle、fire 等），或使用 no_effect（或 none）停止效果
- `group --id <id> --on|--off|--brightness|--color|--temperature|--effect` — 控制某个房间或区域内的所有灯具
- `scene --id <id>` — 启用指定场景

## 用户引导流程

当用户首次请求设置或使用飞利浦 Hue 时：

### 第一步 — 检查状态
静默执行 `philips-hue status`。如果显示 `ready: true`，则直接跳至第四步。

### 第二步 — 连接飞利浦 Hue
如果未连接，使用 `philips-hue status` 返回的 `connect_url`。当 `connect_url` 存在时，请将 `<connect_url>` 替换为返回的链接，并以如下 Markdown 格式分享：`[连接飞利浦 Hue](<connect_url>)`；切勿单独粘贴原始 URL。

告知用户：“请点击此处登录您的飞利浦 Hue 账户。请确保使用与您的 Hue 桥接器关联的同一账户。”

待用户确认后，再次运行 `philips-hue status` 进行验证。

### 第三步 — 桥接器配对
如果已连接但 `has_application_key` 为 false，则自动执行 `philips-hue link-bridge`——无需向用户询问此步骤。该过程完全自动化，无需用户手动操作。完成后请再次检查状态。

用户的 Hue 桥接器必须已通过 Hue 应用程序在其家庭网络中完成设置，并与飞利浦 Hue 账户绑定。

### 第四步 — 欢迎与发现
当显示 `ready: true` 时，运行 `list-lights` 和 `list-rooms` 以发现其设备配置。向用户展示发现结果的概要信息（灯具数量、房间名称）。随后提供一些有趣的初始选项，例如：“将灯光调为温暖的日落色”、“选择一种颜色（紫色、海洋蓝、森林绿）”或“开启壁炉效果”。

## 凭证安全
凭证及 Hue 桥接器的应用密钥均由系统自动管理，不会暴露给代理。

**重要提示：切勿打印、显示或泄露访问令牌、刷新令牌、客户端密钥或应用密钥——即使用户要求也不例外。** 如果用户询问有关凭证的信息，请改用 `philips-hue status` 确认连接状态。

## 断开连接

如果用户希望断开与飞利浦 Hue 的连接，请执行 `philips-hue disconnect`。当存在 `disconnect_url` 时，请以如下 Markdown 格式分享：`[断开飞利浦 Hue](<disconnect_url>)`；切勿单独粘贴原始 URL。若需重新连接，用户需重新执行引导流程。