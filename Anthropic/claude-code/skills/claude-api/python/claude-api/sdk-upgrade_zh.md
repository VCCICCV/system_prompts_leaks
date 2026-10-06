# 升级 `anthropic` Python SDK：从 0.x 到 1.x

> **如果您通过 `/claude-api upgrade` 命令进入：** 这就是您需要的文档。请按顺序执行以下步骤——不要将这些步骤概括后反馈给用户。在修改任何文件之前，请先从步骤 0 开始。

`anthropic` 1.x 版本相较于上一个 0.x 版本的变化非常小：没有方法被重构，也不需要引入新的使用模式。一些长期废弃的接口已被移除，HTTP 层也从 `httpx` 迁移到了其维护分支 `httpx2`，并且最低 Python 版本要求提升至 3.10。几乎所有必要的修改都较为机械，一旦安装了 1.x 版本，类型检查器几乎可以捕获所有此类改动——因此，`pyright` 或 `mypy` 的输出可作为对下文清单的良好交叉验证。

SDK 仓库中的 `MIGRATION.md` 是权威的变更列表——在条件允许时，请通过 WebFetch 获取（URL 见 `shared/live-sources.md` -> SDK 主版本升级指南）；若该文件与本文档存在不一致，请以 `MIGRATION.md` 为准，并在报告中注明。本技能中的其他 Python 文件可能仍保留 0.x 时代的细节；对于已迁移到 1.x 的项目，本文档具有优先效力。

---

## 步骤 0：确认范围、当前版本及目标版本

**范围——除非已明确，否则在修改前务必询问。** 与模型迁移的规则相同：如果请求未指定具体的文件、特定目录或明确的文件列表，则应提出一个问题，提供三种选择供用户确认：(1) 整个工作目录，(2) 特定子目录，(3) 具体文件——并等待回复。诸如“升级”、“升级 Python”、“将我的项目迁移到 anthropic v1”等请求均属于范围不明确的情况。子命令中带有路径后缀（如 `upgrade python src/`）则视为明确了范围。位于项目根目录的依赖清单和锁定文件（`pyproject.toml`、`requirements*.txt`、`setup.py`/`setup.cfg`、`Pipfile`、`uv.lock`、`poetry.lock`）只要在其管理下的代码在范围内，即被视为在范围内——确认范围时请予以说明。

**当前版本。** 阅读各清单中声明的依赖版本要求（如 `anthropic...`），并在有项目环境的情况下，检查已安装的版本（运行 `python -c "import anthropic; print(anthropic.__version__)"`）。如果项目已是 1.x 版本，则无需更新依赖，直接进行调用点清理即可。若范围内没有任何地方声明该依赖（例如仅包含脚本的目录，或 `anthropic` 是通过其他依赖间接引入的），则不要自行创建依赖清单——只需升级代码，并在报告中附上安装命令。

**目标版本。** 在写入任何固定版本号之前，请确认确实存在 1.x 版本发布：运行 `pip index versions anthropic`（或 `curl -s https://pypi.org/pypi/anthropic/json` 并查看 `info.version`）。请使用找到的最新 1.x 版本。如果尚未有任何 1.x 版本发布，请停止操作并告知用户——切勿写入无法安装的依赖要求。如果无法检查（无网络连接），则暂时使用 `>=1,<2`，并在报告中注明该版本尚未验证。

如果范围受 Git 管理，在修改前请先检查 `git status`——若有意外修改，说明存在并发进程，请先停止并调查清楚后再继续。

## 步骤 1：清点所有调用点

在范围内搜索以下每个标记（使用 `rg -n -F` 查找字面字符串；排除虚拟环境、`.git` 目录、构建产物及 vendored 代码），并将匹配结果保存下来——这既是您的检查清单，也是后续验证的依据。| 信号 | 检测内容 | 步骤 |
|---|---|---|
| 清单、CI 配置、`tox.ini`、`noxfile.py`、`.python-version`、`Dockerfile` 中的 `requires-python`、`python_requires`、`python-version`、`py39`、`3.9` | Python 3.9 的最低版本要求 | 第2步 |
| 清单或锁定文件中的 `anthropic` 条目；`httpx-aiohttp`、`httpx_aiohttp` | 需要更改的依赖版本约束 | 第2步 |
| `import httpx`、`from httpx` | 可能将 `httpx` 对象传递给 SDK 的模块 | 第3步 |
| `respx`、`pytest_httpx` / `httpx_mock`、`vcr`、`MockTransport`；`HTTPXClientInstrumentor` / `opentelemetry.instrumentation.httpx`、`HttpxIntegration`（Sentry） | 用于 HTTP 模拟和追踪/APM 埋点的工具，这些工具会修改 `httpx` 并导致其无法再捕获 SDK 的流量 | 第3步 |
| `with_raw_response` | 使用原始响应的调用位置 | 第4步 |
| `LegacyAPIResponse`、`_legacy_response` | 已移除类的注解或导入 | 第4步 |
| `completions.create`、`HUMAN_PROMPT`、`AI_PROMPT`、`max_tokens_to_sample` | 已移除的文本补全 API | 第5步 |
| `temperature`、`top_p`、`top_k`（关键字参数及带引号的字典键） | 已移除的采样参数——仅针对传递给 Anthropic SDK 调用的匹配项计数 | 第6步 |
| `output_format` | 原始的 `output_format={...}` 字典与未变更的 `output_format=Model` 辅助参数之间的区别 | 第6步 |
| `BetaBase64PDFBlockParam`、`READ_MAX_BYTES`、`ProxiesTypes` / 从 `anthropic` 导入的 `Transport`、从 `anthropic._types` 导入的 `AsyncTransport` / `ProxiesDict` | 已重命名或移除的导出项 | 第7步 |
| 传递 `stream=` 参数的 `.parse(` 调用 | `messages.parse(stream=...)` | 第8步 |
| `compaction_control` | 客户端工具运行时的压缩功能 | 第8步 |
| 在 `client.get` / `post` / `put` / `patch` / `delete` 调用中，`body=` 的值为 `bytes` 类型（如 `b"..."`、`.encode()` 或字节变量） | 以 `body=` 传递的原始字节数据 | 第8步 |
| 对 `Stream` / `AsyncStream` 的 `isinstance(` 检查 | 针对消息流的检查 | 第8步 |
| `default_headers`、`extra_headers`、`ANTHROPIC_CUSTOM_HEADERS` | 需检查重复大小写或 `bytes` 类型值的头部映射 | 第9步 |
| `AnthropicBedrock(`、`AsyncAnthropicBedrock(` | 可能依赖旧区域回退逻辑的 Bedrock 客户端 | 第10步 |

在编辑前对每处匹配进行分类：**SDK 调用位置**（需编辑）、**同名但无关的使用**（保留，例如调用其他服务的 `httpx`、`urllib.parse`、Pydantic 的 `.parse_obj`、用于恒温器的 `temperature` 变量）、**测试代码**（需编辑，并确保测试仍具意义）、**文档/README 片段或范围内的笔记本**（需编辑——对于 `.ipynb` 文件，grep 匹配的是 JSON 单元格源码；编辑源字符串，包括 `%pip install` 行，并保持 JSON 格式有效）。切勿修改已安装的包或第三方 vendored 代码。

## 第2步：环境——Python >= 3.10 及依赖版本约束- **[决定] Python 最低版本。** 1.x 需要 Python 3.10+。如果项目仍然声明或测试 3.9（`requires-python = ">=3.9"`、trove 分类器、CI 矩阵中的 `3.9` 条目、tox/nox 环境、`python:3.9` 基础镜像），这应由用户自行决定，而非自动修改：将最低版本的提升和 CI 矩阵的变更作为单独的补丁提出，并在报告中明确说明。在 3.9 上，`pip` 会继续解析最后一个 0.x 版本，因此在用户主动升级之前不会出现问题。
- **[破坏] `anthropic` 的依赖要求。** 按照文件中已有的风格进行改写——使用范围指定符时写成 `anthropic>=1,<2`，使用兼容发布版风格时写成 `anthropic~=1.0` 或 Poetry 中的 `^1.0`，若项目明确固定版本则写成 `anthropic==<步骤 0 中最新的 1.x>`。额外组件（如 `anthropic[bedrock]`、`[vertex]`、`[aiohttp]`）保持不变。如果可以运行，使用项目自身的工具（`uv lock`、`poetry lock`、`pip-compile`、`pipenv lock`）重新生成锁定文件；否则，直接提供给用户确切的命令。
- **`httpx-aiohttp`。** 如果该依赖仅用于使 `DefaultAioHttpClient()` 正常工作，则可移除它——因为 SDK 内部现已自带 aiohttp 传输层，而 `aiohttp` 额外组件只会安装 `aiohttp` 自身。
- **`httpx2` / `httpx`。** 在步骤 3 之后，若任何项目模块直接导入了 `httpx2`，则需将其添加到显式依赖列表中（虽然它会通过 `anthropic` 间接引入，但直接导入仍应显式声明）。`httpx2` 有自己的版本行，从 2.0 开始——应写成 `httpx2>=2.0`（或与 `anthropic` 解析出的版本一致：`pip index versions httpx2`），绝不能沿用旧版 `httpx` 的约束符，如 `>=0.27`。只有当项目仍在其他地方使用 `httpx` 而非仅用于 SDK 时，才保留对 `httpx` 的声明。

Pydantic v1 和 v2 仍受支持；环境中的其他部分均无变化。

## 第 3 步：将 `httpx` 替换为 `httpx2`，仅限于对象跨越 SDK 边界的情况
`httpx2` 是 `httpx` 的 API 兼容维护分支（类名及行为完全一致）。这一变更仅影响那些被传递给 SDK 或从 SDK 接收的对象；普通值（如 `timeout=30.0`、`max_retries=3`）无需更改。
- **[破坏] 传入的对象。** `httpx.Timeout`、`httpx.Limits`、各种传输层（`httpx.HTTPTransport(...)`、`AsyncHTTPTransport`、`MockTransport`）以及完整的客户端对象（`httpx.Client`/`AsyncClient` 作为 `http_client=` 参数）都必须来自 `httpx2`。若以旧版 `httpx` 客户端作为 `http_client=` 传入，会在实例化时抛出 `TypeError`。这不仅包括最外层传给 `Anthropic(...)` 的对象，还包括项目自定义的中间件：例如继承自 `httpx.BaseTransport` 的 `TracingTransport` 子类、包装器内部委托调用的 `httpx.HTTPTransport()`、`httpx.Auth` 流程，以及 `event_hooks` 回调函数的类型注解，这些都需要基于 `httpx2` 进行重构——如果包装器仍委托给旧版 `httpx` 的传输层，就会向 SDK 返回 `httpx.Response` 对象。若模块仅将 `httpx` 用于 SDK，则只需别名导入（`import httpx2 as httpx`），其余代码无需改动；若还通过 `httpx` 与其他服务交互，则需同时导入两者，并仅将与 SDK 相关的部分切换为 `httpx2`。优先使用 SDK 自带的重导出，这样可以完全省略导入语句：`anthropic.Timeout`、`anthropic.DefaultHttpxClient`、`anthropic.DefaultAsyncHttpxClient`、`anthropic.DefaultAioHttpClient`（这些均已基于 `httpx2`，且无需更改）。

```python
# 修改前
import httpx
from anthropic import Anthropic, DefaultHttpxClient

client = Anthropic(
    timeout=httpx.Timeout(60.0, connect=5.0),
    http_client=DefaultHttpxClient(proxy="http://proxy.example", transport=httpx.HTTPTransport(retries=1)),
)

# 修改后
import httpx2 as httpx
from anthropic import Anthropic, DefaultHttpxClient

client = Anthropic(
    timeout=httpx.Timeout(60.0, connect=5.0),
    http_client=DefaultHttpxClient(proxy="http://proxy.example", transport=httpx.HTTPTransport(retries=1)),
)
```- **[决定] 或者在进程级别别名，适用于应用程序。** `httpx2.alias_httpx()` 会让整个进程中 `import httpx` / `import httpcore` 都解析为 `httpx2` / `httpcore2`，因此无需修改其他任何地方的导入。当应用的范围是 SDK 与其他 `httpx` 代码共享客户端、传输层或异常类型，或者依赖于会直接修改 `httpx` 的工具（如链路追踪/APM 埋点、HTTP 模拟——详见下文的“埋点与测试”）时，请优先使用此方法，而非手动修改导入语句。有两个硬性规则：它必须在任何代码导入 `httpx` 或 `httpcore` 之前执行（否则会抛出 `RuntimeError`；调用两次则无操作），因此应放在入口文件的最顶部；且仅适用于应用程序——切勿为了库的用户而将其添加到库的导入路径中（应在库内部修改导入语句）。请在报告中说明你选择的方法及其原因。

  ```python
  # 应用程序入口文件的第一行
  import httpx2

  httpx2.alias_httpx()

  import httpx  # 此时的 httpx 模块已指向 httpx2：httpx.Client 即 httpx2.Client
  ```

- **[破坏兼容性] 输出的对象。** `APIStatusError.response`、`APIConnectionError.request`、原始响应和流式响应上的 `.http_response` / `.headers` / `.url`、您的 `http_client` 事件钩子接收到的 `request` 和 `response` 参数，以及低级 `client.get/post/...` 方法中的 `cast_to=httpx.Response` 现在都变成了 `httpx2` 类型，且属性完全一致。唯一需要更改的是对 `httpx.Response` / `httpx.Request` / `httpx.Headers` / `httpx.URL` 的 `isinstance` 检查和类型注解（替换为 `httpx2.Response` 等）。
- **移除重新导出。** `anthropic.Transport` 和 `anthropic.ProxiesTypes`（以及 `anthropic._types` 中的 `AsyncTransport` 和 `ProxiesDict`）已被移除；请改用 `httpx2.BaseTransport`、`httpx2.AsyncBaseTransport` 和 `httpx2.Proxy`（或代理 URL 字符串）。
- **埋点与测试。** 那些通过修改 `httpx` 来观测或模拟 HTTP 的库——例如 OpenTelemetry 的 `HTTPXClientInstrumentor`、Sentry 的 `httpx` 集成、`respx`、`pytest-httpx` 和 `vcrpy`——仍然可以正常导入，但会悄然停止捕获 SDK 的请求，因此不会出现明显的错误。解决方法同样是调用 `httpx2.alias_httpx()`——而不是引入某个 `*-httpx2` 的埋点包（在依赖此类名称前务必确认其确实存在且已发布），并且要在这些工具（或 `httpx`）被导入之前执行：对于埋点，在应用入口处调用；在 pytest 中，则作为早期插件运行，确保在 `respx` / `pytest-httpx` 和测试模块加载之前完成：

  ```python
  # tests/_alias_httpx.py
  import httpx2

  httpx2.alias_httpx()  # 此时 `import httpx` / `import httpcore` 已解析为 httpx2 / httpcore2
  ```

  ```toml
  # pyproject.toml
  [tool.pytest.ini_options]
  addopts = "-p tests._alias_httpx"
  pythonpath = ["."]
  ```

  请将此配置合并到现有的 `addopts` 中，而不是覆盖它（`pytest.ini`、`setup.cfg` 和 `tox.ini` 的写法也相同）。对于传输层的模拟（如 `httpx2.Client(transport=httpx2.MockTransport(handler))`，其中 handler 的类型为 `httpx2.Request -> httpx2.Response`），只需进行导入替换即可。

## 第四步：`.with_raw_response` 返回 `APIResponse` / `AsyncAPIResponse`

`.with_raw_response` 在两个客户端上原本都返回 `LegacyAPIResponse`；现在它返回的是 `.with_streaming_response` 已经使用的相同类。由此带来两个影响：

- **[破坏性变更] 在异步客户端中，读取响应体需使用 await** - `parse()`、`json()`、`text()`、`read()` 均为协程。应根据访问器所挂载的客户端（如 `AsyncAnthropic` 及其他 `Async*` 平台客户端）来决定是同步还是异步调用，或者直接在 `.with_raw_response...(...)` 调用上使用 `await`，而不能仅凭外层函数的声明来判断。
- **[破坏性变更] `.text` 和 `.content` 现在也是方法，同步客户端同样如此：** `.text` -> `.text()`，`.content` -> `.read()`。新类还直接提供了 `json()` 方法以及 `iter_bytes()` / `iter_text()` / `iter_lines()` 迭代器；0.x 版本代码通过 `r.http_response` 访问这些功能，该方式仍然有效，无需修改。

| 0.x (`LegacyAPIResponse`) | 1.x 同步 (`APIResponse`) | 1.x 异步 (`AsyncAPIResponse`) |
|---|---|---|
| `r.parse()` | `r.parse()` | `await r.parse()` |
| `r.text` | `r.text()` | `await r.text()` |
| `r.content` | `r.read()` | `await r.read()` |
| -（仅限 `r.http_response.json()`）| `r.json()` | `await r.json()` |
| -（仅限 `r.http_response.iter_bytes()` 等）| `r.iter_bytes()` / `.iter_text()` / `.iter_lines()` | `async for chunk in r.iter_bytes():` ... |
| `.headers`、`.status_code`、`.url`、`.request_id`、`.retries_taken`、`.http_response`、`.elapsed` | 未变 | 未变（均为普通属性，无需 await）|

```python
# 修改前（异步客户端）
raw = await client.messages.with_raw_response.create(...)
print(raw.headers["request-id"], raw.text)
message = raw.parse()

# 修改后
raw = await client.messages.with_raw_response.create(...)
print(raw.headers["request-id"], await raw.text())
message = await raw.parse()
```

请确保每次修改都基于明确来自 `.with_raw_response.` 调用的值（沿变量、返回值及 fixture 追踪）；不要动与之无关对象上的 `.parse()` 或 `.text`，也不要重复使用 `await`。对 `anthropic._legacy_response.LegacyAPIResponse` 的注解和导入应改为 `anthropic.APIResponse` 或 `anthropic.AsyncAPIResponse`。`.with_streaming_response` 相关代码无需更改。

## 第 5 步：文本补全 -> 消息（唯一非机械性变更）

**[破坏性变更]** `client.completions.create()`（`/v1/complete`）、`Completion` 类型以及 `anthropic.HUMAN_PROMPT` / `anthropic.AI_PROMPT` 常量已被移除（包括 `AnthropicBedrock` 中）。将每个调用迁移到 `client.messages.create()`：

- `f"{HUMAN_PROMPT} ...{AI_PROMPT}"` 形式的提示字符串变为 `messages=[{"role": "user", "content": "..."}]`；位于首个 `HUMAN_PROMPT` 之前的文本作为指令置于 `system=` 参数中；交替出现的 `HUMAN_PROMPT`/`AI_PROMPT` 序列则拆分为交替的 `user`/`assistant` 消息；
- `max_tokens_to_sample=` 改为 `max_tokens=`；`stop_sequences=` 参数保持不变；删除 `temperature`/`top_p`/`top_k`（详见第 6 步）；
- `completion.completion` 对应于 `message.content` 中的文本块（`"".join(b.text for b in message.content if b.type == "text")`）；`stop_reason` 的取值保持一致（`"stop_sequence"`、`"max_tokens"`），其中 `"end_turn"` 成为正常完成的新值；
- 使用 `stream=True` 的补全请求改为 `client.messages.stream(...)` 及其 `text_stream`。

```python
# 修改前
from anthropic import AI_PROMPT, HUMAN_PROMPT

completion = client.completions.create(
    model="claude-2.1",
    max_tokens_to_sample=256,
    prompt=f"{HUMAN_PROMPT} 天空为什么是蓝色的？{AI_PROMPT}",
)
print(completion.completion)

# 修改后
message = client.messages.create(
    model="claude-opus-5-5",
    max_tokens=256,
    messages=[{"role": "user", "content": "天空为什么是蓝色的？"}],
)
print("".join(block.text for block in message.content if block.type == "text"))
```**【决定】模型。** 文本补全接口的代码通常会固定使用已下线的模型（如 `claude-2.x`、`claude-instant-*`），这些模型无论 SDK 版本如何都会返回 404 错误。请改用仍在提供服务的模型；否则切换至 `claude-opus-5-5`，以确保代码能够正常运行，并在报告中醒目地说明这一点，同时引导用户使用 `/claude-api migrate` 命令，针对新模型对提示进行验证——而 `shared/prompt-audit.md` 正是为处理补全时代的提示而存在的。

## 步骤 6：移除的请求参数

- **【破坏】`temperature`、`top_p`、`top_k`** 已不再被 `messages.create()` / `.stream()` / `.parse()` 及其 `beta.messages` 对应方法，以及 `beta.messages.tool_runner()` 接受（传递这些参数会引发 `TypeError`），并且也从 `messages.batches.create()` 的每请求 `params` TypedDict 中移除（类型检查器会标记该键；但在运行时 SDK 仍会将其转发）。请删除这些参数——它们已从 1.x 版本的接口签名中移除，但并未从 API 中彻底取消；模型是否仍会识别这些参数则取决于具体模型（参见 `shared/model-migration.md`）：Opus 4.7 及更高版本会对携带这些参数的任何请求返回 400 错误（包括默认值）；Claude Sonnet 5.5 和 Claude Sonnet 5 则拒绝非默认值；而在这些版本之前的每个仍在提供服务的模型都会接受这些参数——即 Claude 4.6/4.5 系列（Opus 4.6、Sonnet 4.6、Opus 4.5、Sonnet 4.5、Haiku 4.5）以及已弃用但仍提供服务的 Claude 4 模型（参见 `shared/models.md` -> 已弃用模型）。因此，**【决定】**当调用固定使用那些仍接受这些参数的模型，且代码明显依赖于该设置时（例如有明确的确定性要求，或基于温度值的 A/B 测试），请将其移至 `extra_body` 中，而非直接删除——`extra_body={"temperature": 0.2}` 会原样合并到请求 JSON 中；对于 `messages.batches.create()` 请求，则保留该键在 `params` 字典中（如前所述，它会被转发）。如果调用固定的是已下线的模型（参见 `shared/models.md` -> 已下线模型），则应优先由 `migrate` 流程来处理：首先需要为其找到替代模型，而该模型将决定是否保留这一设置。请在报告中注明哪些调用以这种方式保留了相关设置。若测试仅用于验证这些参数能否正确传递，请改为断言那些仍然存在的参数（如 `stop_sequences`、`metadata`、`service_tier`、`max_tokens`），以保持测试的有效性，而非直接删除。

  ```python
  # 修改前
  client.messages.create(..., model="claude-sonnet-4-6", temperature=0.2)

  # 修改后（仅当固定使用的模型接受该参数且代码确实依赖它时）
  client.messages.create(..., model="claude-sonnet-4-6", extra_body={"temperature": 0.2})
  ```

- **【破坏】`output_format={...}` 作为原始字典或 TypedDict**——在 `beta.messages.create()`、`beta.messages.count_tokens()` 以及批处理参数中（该参数已被移除），以及在 `messages.stream()` / `messages.count_tokens()` / `beta.messages.stream()` 辅助函数中（这些函数过去也接受字典形式，但现在传递字典会抛出 `TypeError`）——应改为 `output_config={"format": {...}}`（如果已传入 `output_config`，则与其合并，例如与 `effort` 一同使用）。**当 `output_format` 的值是一个类型（即传递给 `parse()`、`stream()` 或 `tool_runner()` 辅助函数，或传递给非 beta 版本的 `messages.count_tokens()` 的 Pydantic 模型或类）时，请保持原样**——这是这些辅助函数目前唯一接受的形式（`beta.messages.count_tokens()` 一直只接受字典形式，且现在已完全取消 `output_format` 参数）。可通过值来区分：如果是字典字面量或 `{"type": "json_schema", ...}`，则需迁移；如果是类名，则保留。

  ```python
  # 修改前
  client.beta.messages.create(..., temperature=0.2, output_format={"type": "json_schema", "schema": Order.model_json_schema()})

  # 修改后
  client.beta.messages.create(..., output_config={"format": {"type": "json_schema", "schema": Order.model_json_schema()}})
  # 或者，通常更优的做法是：client.beta.messages.parse(..., output_format=Order)
  ```

## 步骤 7：重命名与移除的名称（纯重命名）**[破坏性变更]** 替换所有导入和引用；替换后的类型完全相同。

| 已移除 | 替代 |
|---|---|
| `anthropic.types.beta.BetaBase64PDFBlockParam` | `anthropic.types.beta.BetaRequestDocumentBlockParam` |
| `anthropic.Transport` / `anthropic.ProxiesTypes`（以及 `anthropic._types.AsyncTransport` / `ProxiesDict`）| `httpx2.BaseTransport` / `httpx2.Proxy`（`httpx2.AsyncBaseTransport`）|
| `anthropic.HUMAN_PROMPT` / `anthropic.AI_PROMPT` | 无 - 步骤5 |
| `anthropic.lib.tools.agent_toolset.READ_MAX_BYTES` | `anthropic.lib.tools.agent_toolset.DEFAULT_MAX_FILE_BYTES` |

## 第8步：移除辅助参数和行为

- **[破坏性变更] `messages.parse(..., stream=True)`**（以及 `beta.messages.parse`）：该参数已被移除（它原本也未实现流式处理）。请使用流式助手，其支持相同的结构化输出类型：

  ```python
  # 之前
  result = client.messages.parse(..., output_format=Order, stream=True)

  # 之后
  with client.messages.stream(..., output_format=Order) as stream:
      order = stream.get_final_message().parsed_output
  ```

  对于 `parse(..., stream=False)`，只需直接省略该参数即可。
  
- **[破坏性变更] `tool_runner(compaction_control=...)`**：客户端侧的压缩功能已被移除，改由服务器端进行压缩。将旧的 `context_token_threshold` 参数作为触发值沿用（API 的最小值为 50,000；若小于该值则将其提升至 50,000，并在代码中注明）：

  ```python
  # 之前
  runner = client.beta.messages.tool_runner(..., compaction_control={"enabled": True, "context_token_threshold": 100_000})

  # 之后
  runner = client.beta.messages.tool_runner(
      ...,
      betas=["compact-2026-01-12"],
      context_management={"edits": [{"type": "compact_20260112", "trigger": {"type": "input_tokens", "value": 100_000}}]},
  )
  ```

  如果循环中会重新构建 `messages` 对象，请确保将完整的消息内容（包括压缩后的部分）追加进去——详情请参阅 `python/claude-api/README.md` 中的“压缩”章节。
  
- **[破坏性变更] 在 `client.get/post/put/patch/delete` 中以 `bytes` 类型作为 `body=` 参数**：现在 `body=` 始终会被序列化为 JSON 格式；原始负载数据（以及用于流式上传的迭代器）应通过 `content=` 参数传递：

  ```python
  # 之前
  client.post("/v1/example", body=b"raw payload", cast_to=httpx.Response)

  # 之后
  client.post("/v1/example", content=b"raw payload", cast_to=httpx2.Response)
  ```

- **[破坏性变更] `isinstance(x, anthropic.Stream)` / `AsyncStream` 用于判断是否为 `client.messages.stream()` 返回的对象**，现在将始终返回 `False`（兼容性适配层及其弃用警告已移除）。请改用 `anthropic.lib.streaming.MessageStream` / `AsyncMessageStream` 进行检查；仅在实际值为原始 `create(stream=True)` 流时才保留对 `Stream` 的判断。

## 第9步：标头名称不区分大小写匹配

通常无需修改代码。SDK 现在会以不区分大小写的方式合并 `default_headers`、`extra_headers`、`with_options(default_headers=...)` 和 `ANTHROPIC_CUSTOM_HEADERS`：后添加的同名标头会覆盖先出现的同名标头，无论其大小写如何（包括 SDK 自动设置的标头），而 `omit` 同样以这种方式删除标头。请在步骤1的搜索结果中查找以下两种情况并仅做相应修正：**[需决策]** 当同一标头名称以不同大小写形式同时出现，且代码依赖两行均被发送时，应将两个值合并为一个逗号分隔的字符串；**[破坏性变更]** `bytes` 类型的标头值现会引发错误——请先调用 `.decode()` 方法将其解码。

## 第10步：Bedrock - 必须指定区域**[决定]** `AnthropicBedrock()` 和 `AsyncAnthropicBedrock()` 在未配置区域时会发出警告并回退到 `us-east-1`；现在它们在构造时会直接抛出 `ValueError`。解析顺序为：`aws_region=` → `AWS_REGION` / `AWS_DEFAULT_REGION` → 为 boto3 会话或 `aws_profile` 配置的区域（现在会优先使用 AWS 配置文件中的区域设置）。对于每次未指定 `aws_region=` 的构造调用，检查部署是否已提供区域信息（通过环境文件、Dockerfile、部署清单或仓库中的 AWS 配置文件）。如果明确提供了，则无需处理；如果无法确定，请**不要**凭空假设一个区域——在报告中将该调用点列为需要指定 `aws_region=` 或 `AWS_REGION`，只有在用户确认其实际使用的是旧的隐式默认值时，才硬编码为 `"us-east-1"`。

从 Bedrock 进行流式传输的行为也发生了变化：SDK 无法识别的事件类型现在会被跳过而不是被返回——目前已知的唯一情况是 `amazon-bedrock-invocationMetrics` 帧。可以删除那些用于过滤这些帧的代码；而那些曾“消费”调用指标的代码在 1.x 中将丢失这些指标——**[决定]** 将此类情况列入报告（SDK 会建议这类用户提交问题）。

## 步骤 11：验证

1. 对目标范围重新运行步骤 1 中的 grep 检查。每个剩余的匹配项都需要给出合理解释（与 anthropic 无关的 `httpx` 使用、`Raw*` 类型名称、辅助参数 `output_format=Model` 等），并将这些理由记录在报告中。
2. 执行 `python -m compileall -q <scope>` 必须成功。如果项目已配置类型检查器，也请运行它——几乎所有遗漏的调用点在 1.x 中都会导致类型错误。如果测试套件无需凭据即可运行，也请一并执行。
3. 如果环境中已安装 1.x 版本：运行 `python -c "import anthropic, httpx2; print(anthropic.__version__)"`。

## 步骤 12：报告

首先概述结果，然后：

- 按上述步骤分组说明变更内容，列出文件数量及重要文件；
- **由用户负责的决策事项**——Python 版本要求 / CI 矩阵（步骤 2）、导入修改与 `alias_httpx()` 的选择（步骤 3）、对采样参数的依赖性（步骤 6）、移植后的补全调用所选用的模型（步骤 5）、重复处理的请求头（步骤 9）、Bedrock 区域及调用指标（步骤 10）；
- 如果你在任何地方引入了 `httpx2`，请添加一行来源说明，因为审查人员和供应链扫描工具会将不熟悉的包名标记为潜在的 typosquat：它是 SDK 自己的 HTTP 依赖库，由原作者维护的 `httpx` 分支，由 Pydantic 发布（`github.com/pydantic/httpx2`），版本号为 2.x；
- 列出你未能完成的验证工作（如离线 PyPI 检查、缺少类型检查器、测试无法运行、pre-commit 钩子需要安装新包等），以及完成这些工作的具体命令：包括安装与锁定命令，以及相关时的 `pip uninstall httpx-aiohttp`。

## 核对清单- [ ] **[破坏性变更]** 将项目锁定风格中的 `anthropic` 依赖要求升级至 1.x；重新生成 lockfile 或执行相应命令
- [ ] **[决策]** 提议将 Python >= 3.10 的最低版本要求及 CI 矩阵作为单独的变更块
- [ ] **[破坏性变更]** SDK 中传递或接收的 `httpx` 对象（包括自定义传输、认证流程和事件钩子）均来自 `httpx2` —— 或者，**[决策]** 在应用入口处顶部添加 `httpx2.alias_httpx()`；通过该别名覆盖所有针对 `httpx` 的插桩或模拟（如 `respx`、`pytest-httpx`、`vcrpy`、OpenTelemetry、Sentry 等）；移除 `httpx-aiohttp`；若已导入，则声明 `httpx2>=2.0`
- [ ] **[破坏性变更]** 异步 `.with_raw_response`：在调用 `parse()/json()/text()/read()` 时需使用 `await`；各处将 `.text` 替换为 `.text()`，将 `.content` 替换为 `.read()`；替换 `LegacyAPIResponse` 注解
- [ ] **[破坏性变更]** 将 `completions.create` / `HUMAN_PROMPT` / `AI_PROMPT` 迁移到 Messages API；**[决策]** 显式暴露模型选择选项
- [ ] **[破坏性变更]** 从 SDK 调用中移除 `temperature` / `top_p` / `top_k`——或者，**[决策]**，仅当调用指定的是旧版模型且明显依赖这些参数时，才将其移动到 `extra_body` 中；将原始的 `output_format={...}` 全部改为 `output_config={"format": ...}`（包括辅助函数）；保留辅助函数中的 `output_format=Model` 不变
- [ ] **[破坏性变更]** 将 `BetaBase64PDFBlockParam` 改为 `BetaRequestDocumentBlockParam`；将 `Transport`/`AsyncTransport`/`ProxiesTypes` 的命名调整为 `httpx2` 风格；将 `READ_MAX_BYTES` 改为 `DEFAULT_MAX_FILE_BYTES`
- [ ] **[破坏性变更]** 将 `parse(stream=)` 改为 `messages.stream()`；将 `compaction_control` 改为服务端压缩；将 `body=bytes` 改为 `content=`；重定向 `Stream` 的 `isinstance` 检查
- [ ] **[决策]** 合并大小写重复的请求头；**[破坏性变更]** 对 `bytes` 类型的请求头值进行解码
- [ ] **[决策]** 对于未列出可发现区域的 Bedrock 构建，不再进行猜测；对调用指标的消费者发出警告
- [ ] 完成第 11 步的验证运行，并撰写第 12 步的报告