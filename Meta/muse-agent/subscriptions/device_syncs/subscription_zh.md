# 数据源事件

一个只读的日志，记录从已连接设备同步过来的各类事件：通知、位置、联系人等。

## 读取事件

通过守护进程沙盒 API 进行读取（无需访问数据库）。按 `source` 过滤列出事件：

```bash
curl -sS --unix-socket "$JARVIS_SANDBOX_API_SOCK" \
  "http://localhost/device-syncs?source=notifications&limit=20"
```

- `source`：任意数据流，可以是常见的设备数据流（如 `contacts | location | notifications`），也可以是客户端自定义的数据源。省略该参数则显示所有数据流；随着客户端注册更多数据源，可能会出现更多选项。
- `producer_id`：限定为某个设备。
- 按 `global_seq` 从新到旧排序。

列表返回的是元数据和简短的 `summary_preview`，而非原始负载。每个事件包含：
- `global_seq`：跨所有数据流的规范到达顺序。
- `producer_id`：稳定的设备 ID，例如 `phone-1`。
- `source`：负载所属类别（如上）。
- `received_at`：RFC3339 格式的接收时间。
- `status`：处理/结算状态。
- `summary_preview`：在收到通知时显示的工作消息预览。

## 获取负载

当 `summary_preview` 不足以满足需求时，可通过 `global_seq` 获取某条事件的存储负载（若未知则返回 404）：

```bash
curl -sS --unix-socket "$JARVIS_SANDBOX_API_SOCK" \
  "http://localhost/device-syncs/<global_seq>"
```

响应中会额外包含 `payload` 和 `payload_representation`（`raw`、`summary` 或 `redacted`）。对于 `notification`、`notifications`、`sms` 和 `imessage` 类型的新插入事件，其元数据会被脱敏，绝不会包含原始消息文本。

## 使用场景

此账本适用于查询近期已同步的设备活动，例如“我的手机最近是否同步了任何通知？” 若要获取规范的联系人、日历或健康数据，请使用 `contacts.search`/`calendar.search` 设备命令，或使用缓存的 `device-data` 技能以及类型化的健康数据存储。