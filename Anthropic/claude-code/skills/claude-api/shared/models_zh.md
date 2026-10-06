# Claude 模型目录

**请仅使用本文件中列出的精确模型 ID。** 切勿猜测或自行构造模型 ID，错误的 ID 会导致 API 报错。在可用的情况下，请使用别名。如需获取最新信息，请通过 WebFetch 获取 `shared/live-sources.md` 中的“模型概览”URL，或直接查询 Models API（详见下方的程序化模型发现）。

## 程序化模型发现

对于**实时**的能力数据——上下文窗口、最大输出 token 数、功能支持情况（思考、视觉、努力等级、结构化输出等）——请直接查询 Models API，而不要依赖下方的缓存表格。当用户询问“X 的上下文窗口是多少”、“模型 X 是否支持视觉/思考/努力等级”、“哪些模型支持 Y 功能”，或希望在运行时按能力筛选模型时，应使用此方法。

```python
m = client.models.retrieve("claude-opus-4-8")
m.id                 # "claude-opus-4-8"
m.display_name       # "Claude Opus 4.8"
m.max_input_tokens   # 上下文窗口（整数）
m.max_tokens         # 最大输出 token 数（整数）

# capabilities 是一个无类型的嵌套字典——使用方括号访问，在叶节点处检查 ["supported"]
caps = m.capabilities
caps["image_input"]["supported"]                       # 视觉功能
caps["thinking"]["types"]["adaptive"]["supported"]     # 自适应思考
caps["effort"]["max"]["supported"]                     # 努力等级：最大值（也包括低/中/高）
caps["structured_outputs"]["supported"]
caps["context_management"]["compact_20260112"]["supported"]

# 在所有模型中进行筛选——直接遍历分页对象（自动分页）；切勿使用 .data
[m for m in client.models.list()
 if m.capabilities["thinking"]["types"]["adaptive"]["supported"]
 and m.max_input_tokens >= 200_000]
```

顶级字段（`id`、`display_name`、`max_input_tokens`、`max_tokens`）是类型化的属性。`capabilities` 是一个字典——请使用方括号访问，而非属性访问。API 会为每个模型返回完整的功能树，并在每个叶节点上标注 `supported: true/false`，因此可以直接使用链式方括号访问，无需 `.get()` 防护。TypeScript SDK：方法名相同，且在迭代时也会自动分页。

### 原始 HTTP 请求

```bash
curl https://api.anthropic.com/v1/models/claude-opus-4-8 \
  -H "x-api-key: $ANTHROPIC_API_KEY" \
  -H "anthropic-version: 2023-06-01"
```

```js
{
  "id": "claude-opus-4-8",
  "display_name": "Claude Opus 4.8",
  "max_input_tokens": 1000000,
  "max_tokens": 128000,
  "capabilities": {
    "image_input": {"supported": true},
    "structured_outputs": {"supported": true},
    "thinking": {"supported": true, "types": {"enabled": {"supported": false}, "adaptive": {"supported": true}}},
    "effort": {"supported": true, "low": {"supported": true}, ..., "max": {"supported": true}},
    ...
  }
}
```

## 当前推荐模型| 友好名称     | 别名（请使用此别名）    | 完整ID                       | 上下文        | 最大输出 | 状态 |
|-------------------|---------------------|-------------------------------|----------------|------------|--------|
| Claude Fable 5.1    | `claude-fable-5-1`      | -                             | 1M             | 128K       | 已启用 |
| Claude Mythos 5.1   | `claude-mythos-5-1`     | -                             | 1M             | 128K       | 已启用（仅限Project Glasswing） |
| Claude Fable 5 | `claude-fable-5` | -                             | 1M             | 128K       | 已启用 |
| Claude Mythos 5 | `claude-mythos-5` | -                          | 1M             | 128K       | 已启用（仅限Project Glasswing） |
| Claude Opus 5.5 | `claude-opus-5-5` | -                             | 1M             | 128K       | 已启用 |
| Claude Opus 5     | `claude-opus-5`       | -                             | 1M             | 128K       | 已启用 |
| Claude Opus 4.8   | `claude-opus-4-8`   | -                             | 1M             | 128K       | 已启用 |
| Claude Opus 4.7   | `claude-opus-4-7`   | -                             | 1M             | 128K       | 已启用 |
| Claude Opus 4.6   | `claude-opus-4-6`   | -                             | 1M             | 128K       | 已启用 |
| Claude Sonnet 5.5 | `claude-sonnet-5-5` | -                     | 1M             | 128K       | 已启用 |
| Claude Sonnet 5 | `claude-sonnet-5` | -                         | 1M             | 128K       | 已启用 |
| Claude Sonnet 4.6 | `claude-sonnet-4-6` | -                             | 1M             | 128K       | 已启用 |
| Claude Haiku 4.5  | `claude-haiku-4-5`  | `claude-haiku-4-5-20251001`   | 200K           | 64K        | 已启用 |

### 模型说明
- **Claude Fable 5.1** - Anthropic 最强大、面向公众发布的模型，适用于最复杂的推理任务和长周期的代理式工作。它是 Claude Fable 5 的后继版本，同属同一能力层级，按 token 计费价格相同（10 美元/50 美元每百万 token；缓存读取 0.25 美元/百万 token，仅为 Claude Fable 5 的四分之一；批量调用 5 美元/25 美元）。在长时间运行的代理式编程、文档/表格/幻灯片等知识型工作、多步骤研究、视觉理解、长上下文检索以及计算机操作等方面表现更优。API 接口与 Claude Fable 5 相同（思考始终开启，无预填充，无采样参数，“refusal”为停止原因，缓存最小长度为 512 token），但有三项重大变更：强制工具使用时（`tool_choice` 为 `any` 或 `tool`）将返回 400 错误；思考块仅限于生成它的模型访问（只有 Claude Mythos 5.1 可以读取，其他模型会丢弃）；编辑历史记录中的早期轮次会导致思考块失效（“保留思考”功能；自 2026 年 8 月 31 日起创建的新账户，在所有平台上对已编辑的历史都会收到 400 错误，具体执行范围由各模型自行决定；该可选控制功能已在 Claude API、AWS 上的 Claude Platform、Bedrock 和 Vertex 中启用，Foundry 尚未确认，详见 `shared/platform-availability.md`）。新增每条消息的 `effort` 参数、作用于单轮的 `clear_at` 系统消息、“thinking.display: 'updates'”进度更新，以及内容溯源信息。采用与 Claude Fable 5 相同的分词器；默认上下文窗口 100 万 token，最大输出 12.8 万 token。适用覆盖模型：需遵守 30 天数据保留要求（仅经 Anthropic 明确授权方可豁免 ZDR），否则 ZDR 组织将收到 400 invalid_request_error 错误，与 Claude Fable 5 相同。无优先级层级，共享 Fable 5.x 的速率限制池。详情参见 `shared/model-migration.md` -> 从 Claude Fable 5 迁移到 Claude Fable 5.1。
- **Claude Fable 5** / **Claude Mythos 5** (`claude-fable-5` / `claude-mythos-5`) - 前一代 Fable / Mythos 版本：与 Claude Fable 5.1 同属同一层级，具有相同的限制和按 token 定价，后者在此基础上增加了三项重大 API 变更（详见上文；此处缓存读取费用为 1 美元/百万 token，而 Claude Fable 5.1 为 0.25 美元）。仍可通过模型 ID 调用并选择。Claude Mythos 5 不运行安全分类器，因此不会出现 `stop_reason: "refusal"`。新项目应优先选择 claude-fable-5-1。
- **Claude Mythos 5.1** - 与 Claude Fable 5.1 为同一模型（能力、限制、按 token 定价及 API 行为均相同，唯独不执行历史编辑检查），仅向获批准的 Project Glasswing 客户提供；是 Claude Mythos 5 的后继版本（后者又接替了仅限邀请使用的 `claude-mythos-preview`）。与 Claude Mythos 5 不同，它会根据准入计划启用相应的安全防护机制，因此需处理 `stop_reason: "refusal"`。不在 AWS 上的 Claude Platform 提供。仅当组织参与 Project Glasswing 时才使用此模型，否则请使用 `claude-fable-5-1`。
- **Claude Opus 5.5** - 是 Opus 系列中用于长时间代理式编程和知识工作的后继版本，定价更低（4 美元/20 美元每百万 token；缓存读取 0.20 美元）。与 Claude Opus 5 具有相同的 100 万 token 上下文窗口、12.8 万 token 输出、分词器和功能集，但有四项重大变更：思考不可关闭（仅可通过 `effort` 控制，默认为 `medium`），强制工具选择时会返回 400 错误，思考块与模型及对话绑定，且使用计算机功能需配备 `computer_toolset_20260801` 工具集。安全分类器范围扩大（新增 `bio` 和 `reasoning_extraction` 分类器）。目前为默认模型及当前主流 Opus 版本；详情参见 `shared/model-migration.md` -> 迁移到 Claude Opus 5.5。
- **Claude Opus 5** - 适用于复杂代理式编程和企业级工作；相比 Claude Opus 4.8 实现了质的飞跃，在深度推理、代理式及长周期任务，以及测试时的计算规模扩展方面表现最强，成本仅为 Claude Fable 5.1 的一半（Claude Fable 5.1 仍为最高能力层级）。安全分类器可能返回 `stop_reason: "refusal"`，读取 `content` 前需妥善处理。可在 Opus 4.8 的定价水平下直接升级（5 美元/25 美元每百万 token），功能集保持不变。思考默认开启（省略 `thinking` 参数则自动启用自适应模式，`{type: "adaptive"}` 等效），`thinking: {type: "disabled"}` 仅在 `effort` 为 `high` 或更低时可用，若与 `xhigh`/`max` 搭配则会返回 400 错误。原始思考 token 永远不会被返回。支持完整的 `effort` 阶梯直至 `max`；提示缓存最小长度为 512 token（较 Opus 4.8 的 1024 token 缩小），仅在 Claude API 上提供快速模式。网络安全防护进一步加强。与合并后的 Opus 4.x 速率限制池分开。上下文窗口默认及最大均为 100 万 token，最大输出 12.8 万 token。详情参见 `shared/model-migration.md` -> 迁移到 Claude Opus 5。
- **Claude Opus 4.8** - Opus 4 系列中能力最强的模型——高度自主，擅长长周期代理式工作、知识型任务和记忆；写作风格更清晰、更富亲和力。API 接口与 Opus 4.7 相同（仅支持自适应思考，已移除采样参数和 `budget_tokens`）。标准 API 定价，无长上下文附加费。详情参见 `shared/model-migration.md` -> 迁移到 Opus 4.8——从 4.7 升级至 4.8 仅需更换模型 ID 并重新调整提示，无新增重大变更。
- **Claude Opus 4.7** - 上一代 Opus。高度自主，擅长长周期代理式工作、知识型任务、视觉理解和记忆。仅支持自适应思考，已移除采样参数和 `budget_tokens`。上下文窗口 100 万 token。详情参见 `shared/model-migration.md` -> 迁移到 Opus 4.7。
- **Claude Opus 4.6** - 更早的 Opus 版本。支持自适应思考（推荐使用），最大输出 12.8 万 token（大输出需流式传输）。上下文窗口 100 万 token。
- **Claude Sonnet 5** - 上一代 Sonnet，在编程和代理式任务上的表现接近 Opus 水平。默认启用自适应思考（省略 `thinking` 则自动启用自适应模式）；已移除手动 `budget_tokens`；拒绝非默认采样参数。`effort` 支持 `low`/`medium`/`high`/`xhigh`/`max`。采用全新分词器（与 Sonnet 4.6 相比，同等文本所需 token 数量约增加 30%）。高分辨率视觉输入（2576 像素）。上下文窗口 100 万 token，最大输出 12.8 万 token。详情参见 `shared/model-migration.md` -> 迁移到 Claude Sonnet 5。
- **Claude Sonnet 5.5** - Sonnet 系列中 Claude Sonnet 5 的后继版本，定价相同（2 美元/10 美元每百万 token；缓存读取 0.20 美元）。采用与 Claude Sonnet 5 相同的分词器；上下文窗口 100 万 token，最大输出 12.8 万 token。默认启用自适应思考，`effort` 默认值为 `high`，且各级别已重新校准。有五项重大变更：`thinking: {type: "disabled"}` 返回 400 错误（如需关闭思考，请发送 `{type: "between_tools"}` 且 `effort` 为 `high` 或更低）；强制工具选择时会返回 400 错误；思考块与模型及对话绑定；在 Claude API 和 Google Cloud 上使用计算机功能需配备 `computer_toolset_20260801` 工具集；顾问工具拒绝接受来自 Claude Opus 4.8、Claude Opus 4.7 和 Claude Sonnet 5 的顾问。详情参见 `shared/model-migration.md` -> 迁移到 Claude Sonnet 5.5。
- **Claude Sonnet 4.6** - 上一代 Sonnet。支持自适应思考（推荐使用）。上下文窗口 100 万 token。最大输出 12.8 万 token。
- **Claude Haiku 4.5** - 速度最快、性价比最高的简单任务专用模型。

## 旧版模型（仍在使用）

| 通用名称         | 别名（请使用此别名）    | 完整ID                       | 状态     |
|-------------------|---------------------|-------------------------------|--------|
| Claude Opus 4.5   | `claude-opus-4-5`   | `claude-opus-4-5-20251101`    | 使用中 |
| Claude Opus 4.1   | `claude-opus-4-1`   | `claude-opus-4-1-20250805`    | 已弃用（将于2026年8月5日停用——请迁移到`claude-opus-5-5`） |
| Claude Sonnet 4.5 | `claude-sonnet-4-5` | `claude-sonnet-4-5-20250929`  | 使用中 |

## 已弃用模型（即将停用）

| 通用名称         | 别名（请使用此别名）    | 完整ID                       | 状态     | 停用日期      |
|-------------------|---------------------|-------------------------------|------------|--------------|
| Claude Sonnet 4   | `claude-sonnet-4-0` | `claude-sonnet-4-20250514`    | 已弃用 | 待定          |
| Claude Opus 4     | `claude-opus-4-0`   | `claude-opus-4-20250514`      | 已弃用 | 待定          |
| Claude Haiku 3    | -                   | `claude-3-haiku-20240307`     | 已弃用 | 2026年4月19日 |

## 已停用模型（已不可用）

| 通用名称         | 完整ID                       | 停用日期     |
|-------------------|-------------------------------|-------------|
| Claude Sonnet 3.7 | `claude-3-7-sonnet-20250219`  | 2026年2月19日 |
| Claude Haiku 3.5  | `claude-3-5-haiku-20241022`   | 2026年2月19日 |
| Claude Opus 3     | `claude-3-opus-20240229`      | 2026年1月5日 |
| Claude Sonnet 3.5 | `claude-3-5-sonnet-20241022`  | 2025年10月28日 |
| Claude Sonnet 3.5 | `claude-3-5-sonnet-20240620`  | 2025年10月28日 |
| Claude Sonnet 3   | `claude-3-sonnet-20240229`    | 2025年7月21日 |
| Claude 2.1        | `claude-2.1`                  | 2025年7月21日 |
| Claude 2.0        | `claude-2.0`                  | 2025年7月21日 |

## 处理用户请求

当用户按名称请求某个模型时，请使用下表查找正确的模型ID：

| 用户说法…                              | 使用此模型 ID              |
|-------------------------------------------|--------------------------------|
| “fable”、“最强大模型”             | `claude-fable-5-1`                 |
| “最强大”                           | `claude-fable-5-1`                 |
| “mythos”、“mythos 5.1”                    | `claude-mythos-5-1`（仅限 Project Glasswing 参与者；否则使用 `claude-fable-5-1`） |
| “fable 5”、“mythos 5”（旧版本） | `claude-fable-5` / `claude-mythos-5`（仍在提供服务；新任务优先使用 `claude-fable-5-1`） |
| “mythos 预览版”                          | `claude-mythos-5-1`（`claude-mythos-preview` 的后续版本——请参阅迁移指南） |
| “opus”                                    | `claude-opus-5-5`                   |
| “opus 5”                                  | `claude-opus-5`             |
| “opus 5.5”                                | `claude-opus-5-5` |
| “opus 4.8”                                | `claude-opus-4-8`              |
| “opus 4.7”                                | `claude-opus-4-7`              |
| “opus 4.6”                                | `claude-opus-4-6`              |
| “opus 4.5”                                | `claude-opus-4-5`              |
| “opus 4.1”                                | `claude-opus-4-1`（已弃用，将于 2026 年 8 月 5 日停止服务——建议使用 `claude-opus-5-5`） |
| “opus 4”、“opus 4.0”                      | `claude-opus-4-0`（已弃用——建议使用 `claude-opus-5-5`） |
| “sonnet”、“平衡型”                      | `claude-sonnet-5-5`           |
| “sonnet 5”                                | `claude-sonnet-5`           |
| “sonnet 5.5”                              | `claude-sonnet-5-5` |
| “最便宜的 sonnet”、“最新的 sonnet”、“最晚发布的 sonnet”（任何表述方式） | `claude-sonnet-5-5` |
| “sonnet 4.6”                              | `claude-sonnet-4-6`            |
| “sonnet 4.5”                              | `claude-sonnet-4-5`            |
| “sonnet 4”、“sonnet 4.0”                  | `claude-sonnet-4-0`（已弃用——建议使用 `claude-sonnet-5-5`） |
| “sonnet 3.7”                              | 已停用——建议使用 `claude-sonnet-5-5` |
| “sonnet 3.5”                              | 已停用——建议使用 `claude-sonnet-5-5` |
| “haiku”、“快速”、“廉价”                  | `claude-haiku-4-5`             |
| “haiku 4.5”                               | `claude-haiku-4-5`             |
| “haiku 3.5”                               | 已停用——建议使用 `claude-haiku-4-5` |
| “haiku 3”                                 | 已弃用——建议使用 `claude-haiku-4-5` |
