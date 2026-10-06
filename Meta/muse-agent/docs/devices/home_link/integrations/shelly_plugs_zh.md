# Shelly 插头（第 4 代）：开关与功率计量

**最后验证日期：** 2026-09-21，使用 Shelly Plug US 第 4 代（型号 S4PL-00116US，固件版本 2.0.0）

当新设备发现识别出一个 Shelly 智能插头，且用户要求对其进行开关操作或读取其功耗时，请参考本指南。

## 设备识别

- **mDNS** 是最可靠的识别信号。Shelly 插头通过 `_http._tcp` 进行服务通告，主机名由设备型号和 MAC 地址组成，例如 `ShellyPlugUSG4-<MAC>.local`，其中 `<MAC>` 是十二位十六进制数字，不含分隔符。设备自身报告的 MAC 地址也采用相同格式，不含冒号。
- **MAC 厂商前缀** 只是辅助证据，并非确凿证明。Shelly 设备曾使用过多个厂商前缀，因此匹配仅表示可能，不匹配也无法排除。请按照 `~/docs/devices/home_link.md` 中的要求解析当前的厂商前缀：查询制造商公开的厂商前缀列表，然后在本地与发现结果进行比对。向在线查询工具提交时，仅提供厂商前缀，切勿发送完整的 MAC 地址。
- **采取行动前务必确认。** 发送 `Shelly.GetDeviceInfo` 请求，检查 `model`、`app` 和 `gen` 字段。只有该响应结果，而非主机名，才能确定设备是否如您所想。

默认情况下，设备地址为 DHCP 分配。每次均应通过 MAC 或主机名识别插头，并从当前发现结果中获取其地址。切勿重复使用已记忆的地址。

## 首先确认设备代次

API 的实现依赖于设备代次，不同代次之间并不兼容。
- `gen` 2 及以上（包括第 4 代）使用下文所述的 RPC API。
- `gen` 1 在 RPC 出现之前推出，使用 `/relay/0?turn=on` 接口。如果发现的是第 1 代设备，则本指南其余内容不适用。

## 推荐流程

1. 重新发现插头，从最新结果中获取其 IP 地址和端口。
2. 通过 Home Link CONNECT 代理访问该设备。它提供标准化的 HTTP API，因此 `~/docs/devices/home_link.md` 中关于 HTTP 的部分适用，包括认证与重定向规则。
3. 调用 `Shelly.GetDeviceInfo` 确认型号与代次，并读取 `auth_en` 状态。
4. 在执行任何操作前，先调用 `Switch.GetStatus?id=0`，以了解当前状态，从而区分无操作与实际变更。每个插头只有一个开关，其 ID 为 0。
5. 使用 `Switch.Set?id=0&on=true` 或 `on=false` 进行开关操作。返回结果为 `{"was_on": <bool>}`，表示调用前的状态，而非操作结果。若将其误认为新状态，会导致逻辑错误。
6. 再次读取状态。调用 `Switch.GetStatus?id=0` 可获取继电器状态 `output` 以及实际功耗 `apower`。仅凭 `output` 只能判断继电器是否动作；而 `apower` 才能反映负载是否响应。在告知用户操作成功前，请同时确认这两项指标。

对于不应保持锁定状态的操作，`Switch.Set` 还支持 `toggle_after=<seconds>` 参数，可在指定时间后自动恢复初始状态，无需再次调用。

## API 端点

| 操作 | 路径 |
|---|---|
| 设备型号、代次、固件及 `auth_en` | `/rpc/Shelly.GetDeviceInfo` |
| 设备完整状态 | `/rpc/Shelly.GetStatus` |
| 单个开关状态 | `/rpc/Switch.GetStatus?id=0` |
| 开启或关闭 | `/rpc/Switch.Set?id=0&on=true` |
| 开启或关闭并定时恢复 | `/rpc/Switch.Set?id=0&on=true&toggle_after=60` |
| 切换状态 | `/rpc/Switch.Toggle?id=0` |
| 开关配置（含自动关闭功能）| `/rpc/Switch.SetConfig` |
| 定时任务 | `/rpc/Schedule.List`、`Schedule.Create`、`Schedule.Delete` |

## 认证

请从 `Shelly.GetDeviceInfo` 中读取 `auth_en`，不要假设。家庭网络中的插头通常未启用认证，直接使用纯 HTTP 协议即可。这取决于具体设备的设置，而非设备型号。

当 `auth_en` 为真时，设备需要 HTTP 摘要认证，用户名始终为 `admin`。请勿在请求中包含密码。如果尚未为该插头存储凭证，请遵循 `~/docs/devices/home_link.md` 中的凭证管理规则，并告知用户需完成相关设置。

## 其他功能值得了解，尽管大多数请求并不需要这些功能：

- 除了 `apower` 之外的功率计量：电压、电流、频率，以及以瓦时为单位的累计电能（在 `aenergy` 下显示），并提供最近的每分钟历史数据。
- 内置光传感器，其读数显示在 `illuminance:0` 下。
- 开关配置中可设置的过功率和过电流保护阈值。
- BLE、BTHome、MQTT、厂商云服务以及 Matter。这些都是可选的控制通道。建议优先使用本地 RPC API，而非该智能插座已支持的网络连接方式。