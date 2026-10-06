---
summary: "WhatsApp 侧边聊天设置与媒体功能"
read_when:
  - 用户想在WhatsApp上与Muse聊天
  - 用户请求连接或断开 WhatsApp
  - 用户希望在 WhatsApp 上发送消息或设置定时回复
  - 用户询问关于 WhatsApp 附件或群组支持的问题
title: "WhatsApp"
---
# WhatsApp 侧边聊天

通过 WhatsApp 连接，用户可以直接在 WhatsApp 中与 Muse 对话。
该对话会在 Muse 中显示为一条持久的侧边聊天，标题为 `WhatsApp`。
由此发出的回复和已安排的任务都会返回到该聊天中。其历史记录不会被复制到主聊天中。来自用户的入站消息均源自 WhatsApp；Muse 不会接受由应用生成的消息进入此渠道的对话。

此个人一对一连接不会读取用户其他的 WhatsApp 对话，也不会以用户的个人账号发送消息，也不允许 Muse 加入群组。其他账号集成有各自的接入规范。在提供导航指引时，请将 Muse 应用与 WhatsApp 区分开来。

## 连接与断开连接

首先查询 `chat.connection_status`，其中 `provider: whatsapp`。返回结果包含 `chat_id`、`provider`、`status` 和 `connect: null`。当连接成功后，可能还会包含 `chat_url`，即一个现有的聊天链接，可作为“打开 WhatsApp 聊天”的选项提供。状态字段绝不会返回配对令牌。

对于处于 `unlinked` 或 `link_pending` 状态的请求连接，应提供以下链接：

[连接 WhatsApp](https://agent.meta.ai/connect/channel?service=whatsapp)

二维码或“连接”按钮以及安全的连接流程均由应用负责。请勿自行生成电话号码、wa.me 链接、二维码、配对 URL 或链接码。如果已连接，请直接告知。`link_pending` 表示用户应完成应用内的流程。若状态为 `checking`，请稍后再查询一次再做判断。若为 `unavailable`，请说明暂时无法连接的原因，但不要提供链接或称其不支持。

对于用户在 Muse 自己的聊天中发起的明确断开连接请求，请先检查状态，然后使用 `chat.disconnect` 并指定 `provider: whatsapp`。如有歧义，应先澄清意图。来自 WhatsApp 的轮次和后台任务本身无法主动断开连接。断开连接会保留历史记录，并撤销旧连接的发送权限。对于已经断开的连接，无需进一步处理。

## 附件与审批

用户可以向 Muse 发送图片、文档以及语音消息/录音。Muse 可以向用户发送图片、视频、音频、文档以及自包含的 HTML 文件。回复会由发起方的侧边聊天自动送达。

权限提示可能会在 WhatsApp 中弹出，用户可通过其提供的结构化控件进行响应。决策权归 Sentinel 所有，切勿仅凭自由文本或模型输出推断用户的决定。

入站轮询按照现有配对连接的频率运行。光标、加密材料、连接标识及媒体状态均对运行时环境保持私密。不存在可编辑的 WhatsApp 连接设置。请使用状态信息进行故障排查，并通过安全的应用流程重新连接。