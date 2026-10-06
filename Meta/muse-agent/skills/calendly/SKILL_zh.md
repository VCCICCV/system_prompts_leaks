---
name: "calendly"
description: "使用 Calendly CLI 查看日程安排事件和事件类型，并管理日程数据。"
icon: "calendly"
metadata: { "不包含在提示中": 假 }
---
# Calendly

## 用途
使用 `calendly` CLI 获取 Calendly 连接器的状态信息，并读取日程安排数据。

## 工具
直接通过系统 `PATH` 使用已安装的 CLI。

核心命令：
- `calendly status`
- `calendly disconnect`
- `calendly authorize-url`
- `calendly me`
- `calendly list-events --user-uri <uri> [--status active] [--min-start-time <iso>] [--max-start-time <iso>] [--count 20] [--page-token <token>]`
- `calendly event-types --user-uri <uri> [--active true] [--count 20] [--page-token <token>]`
- `calendly request --method GET --path '/scheduled_events' --query 'user=<uri>' [--query 'status=active'] [--json-body '<json>']`

输出约定：
- 每个命令均返回包含顶层字段 `ok` 的 JSON 数据。
- 对于 `disconnect` 命令，解析 `action` 和 `removed` 字段。
- 对于 `status` 命令，解析报告的连接状态，以及可能返回的授权 URL 或恢复详情。
- 对于 `me` 命令，在列出用户范围的资源之前，先解析当前用户的 URI 和组织的 URI。
- 对于 `list-events` 和 `event-types` 命令，解析返回的集合数据，并提取用于后续分页的令牌。对于有时效的日程事件，会额外提供 UTC 时间和用户本地时间形式的 `event_starts_at` 和 `event_ends_at`；输出中还会添加由运行时生成的 `retrieved_at` 字段。

## 认证
Calendly 的连接流程由 `calendly` 负责管理。

认证约定：
- 首先运行 `calendly status`。
- 如果用户希望断开连接，则运行 `calendly disconnect`。
- 如果未连接，则运行 `calendly authorize-url`。当返回 `connect_url` 时，将 `<connect_url>` 替换为该 URL，并按原样分享以下 Markdown 格式的链接：`[Connect Calendly](<connect_url>)`；请勿单独粘贴原始 URL。
- 认证完成后，运行 `calendly me` 进行验证。

凭据存储：
- 本技能不会直接读取客户端应用凭据或用户令牌状态。
- CLI 仅通过共享的 `hatch-tool-sdk` 连接器辅助工具访问连接状态；不存在代理可见的配置文件路径。
- CLI 会自动解析并刷新服务令牌；切勿输出访问令牌或刷新令牌。

## 操作规则
1. 在调用任何 Calendly API 之前，必须先通过 `calendly status` 确认连接状态已建立。
2. 当需要获取当前用户的 URI 或组织的 URI 时，应首先运行 `calendly me`。
3. 遇到令牌过期错误时，令牌刷新会自动进行。如果自动刷新后 API 调用仍失败，请再次检查 `calendly status`，必要时重新绑定连接。
4. 在执行任何具有破坏性或对用户可见的变更操作之前（包括可能取消事件或删除订阅的原始 `request` 调用），务必先确认。
5. 在摘要中优先使用 `event_starts_at.user_local` 和 `event_ends_at.user_local`。