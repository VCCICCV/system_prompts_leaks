---
name: library
description: 当用户提及“库”、请求查找或处理由库支持的文件、站点，或库中可能存在的命名文件，或者希望对库中的文件夹进行整理、恢复先前版本，或共享原生的库文件或文件夹时，请使用 ChatGPT 库。此外，即使用户未明确提到“库”，但希望创建或更新面向用户的文件或可复用的工件时，也应使用该功能。ChatGPT 库能够搜索并读取库中的内容，将文件引入本地工作流，保存新的交付成果，并在保留其标识和版本历史的前提下更新现有的库文件。
---
# ChatGPT 资料库

将此用作持久化 ChatGPT 资料库文件的顶层路由。以“资料库”为锚点，选择一条内容访问路径，并在每次本地编辑与回写时保持资料库的身份一致性。

资料库的第一方应用 `connector_openai_library` 负责执行已认证的资料库操作：

- `list`、`search`、`read` 和 `find` 用于查看资料库内容。
- `prepare_materialize` 使解析后的资料库文件可供本地工具使用。
- `create_library_file`、`replace_library_file` 和 `manage_library` 用于写入或整理资料库内容。
- `share` 用于授予或撤销访问权限；请遵循其当前由应用提供的说明与模式。
- 当被调用时，`prepare_uploads` 和 `finalize_uploads` 处理已准备好的上传任务。
- 仅限应用使用的文件 `@` 提及搜索可解析并定位选定的资料库条目。

运行时环境提供模式定义，并决定可用的工具。本技能不暴露任何工具，也不提供本地 MCP 服务器。

## 工作流程

1. 锚定目标：
   - 解析并复用选定资料库 `@` 提及中的标识符。
   - 使用 `search` 按文件名、标题、描述或内容查询。
   - 使用 `list` 获取近期文件、文件夹或库存信息。
   - 如果用户提供了本地路径，或明确表示文件为本地文件，则继续使用本地工具。
2. 选择一条访问路径：
   - 当资料库能够提供任务所需内容时，使用 `read`。
   - 对于已知文件内的精确匹配或正则表达式匹配，使用 `find`。
   - 在编辑、脚本处理、可视化检查、生成、字节比对或其他需要文件字节的本地工具场景下，使用 `materialize` 进行材料化。
3. 写入时保持身份一致性：
   - 仅在不存在资料库身份或用户希望创建副本时进行新建。
   - 编辑现有条目时，请使用相同的 `library_file_id` 进行替换。
   - 对于文件夹、节点变更、删除及恢复等操作，请使用 `manage_library`。

## 使用当前辅助程序

1. 下载前，请从当前作用域的工作区目录中移除之前运行时下载的所有辅助程序副本。
2. 从当前资料库技能源获取该辅助程序及其所有配套文件，并将其存放在一个全新的私有目录中。
3. 在本次请求或运行期间的每次调用与重试中，均重复使用已下载的辅助程序。

## 路由规则

| 用户需求 | 必需路由 |
| --- | --- |
| 选定的库引用 `@` 提及 | 解码其 `oai-library://...` URI，并使用返回的标识符。仅在缺少标识符、元数据需刷新或目标不明确时才进行搜索。 |
| 确切文件名或标题 | 使用 `search`；引用实际名称并设置 `search_title_only=true`。 |
| 按用途或内容描述的文件 | 使用常规的库内 `search`。不要将描述性短语视为确切文件名。 |
| 最近的文件、文件夹或库存 | 使用 `list`；将返回的 `next_cursor` 作为 `cursor` 传递以继续获取更多结果。 |
| 来自指定文件的事实、摘要或比较 | 先解析每个文件，然后对选定的每个文件使用 `read`。仅靠搜索片段是不够的。 |
| 已知文件中的字面匹配或正则表达式匹配 | 使用 `find`；仅当需要更多上下文时再执行 `read`。 |
| 共享写作块（`library_artifact_type: writing_block`） | 使用 `read` 重建其完整的本地文本；当 `has_more` 为真时，继续调用 `next_read`。保留行边界，验证 `size_bytes`，并保留权威的 `version_id`；无需物化。 |
| 来自 `list` 或 `search` 结果的本地字节 | 当文件标识符和路径仍有效时，复用先前的完整结果；仅在元数据缺失、过时或存在歧义时才重新执行 `list` 或 `search`。直接将其传递给内置的仅标准输入下载辅助工具。不要先调用库的 `read` 或 `prepare_materialize`。 |
| 来自已解析但非由 `list` 或 `search` 获取的引用的本地字节 | 使用 `prepare_materialize`；参阅 [materialization.md](references/materialization.md)。 |
| 显式本地路径 | 使用本地工具。不要将本地路径发送至库的 `read` 或 `find`。 |
| 新的本地交付物 | 选择下方的创建路由之一。 |
| 编辑由库支持的文件 | 对于 `library_artifact_type: site`，使用 Sites；切勿对其投影进行物化、替换或恢复。其他情况则视需要进行物化，在本地编辑并验证后，再替换相同的 `library_file_id`。 |
| 创建、移动、重命名或删除库节点 | 使用 `manage_library`；在进行变更前请阅读 [library-management.md](references/library-management.md)。 |
| 恢复早期版本 | 使用 `manage_library` 并配合 `restore_version`；请阅读 [library-management.md](references/library-management.md)。 |

对于 `list`，将 `limit` 设置为最多 `200`。对于 `search`，请使用规范的请求格式：`{"search_query":[{"q":"quarterly revenue"}],"top_k":5}`。`search_query` 必须是一个包含一到五个对象的数组，即使只进行一次搜索也是如此。`search_title_only` 参数只能置于 `search_query` 对象内部。切勿在顶层发送 `query`、`queries`、`q`、`search_title_only` 或 `limit`；应在顶层使用 `top_k`，取值范围为 1 至 100。

就库内意图而言，除非用户提供了本地路径或明确说明文件为本地文件，否则应优先在库内搜索，而非本地工作区。一旦路由至库内，便不应再在工作区或之前的对话中寻找同一目标。未能解析库内项目并不意味着该文件为本地文件。

## 读取库内内容

优先使用 `structuredContent`；仅在不可用时才解析 JSON 文本块。务必严格按照返回的标识符、文件名、版本和路径操作。后续的 `read` 或 `find` 应优先使用返回的 `library_file_id`，其次才是 `file_id` 或 `id`。切勿将搜索的 `result_id` 用作文件引用。对于 `read`，请将该标识符置于 `read[i].ref_id` 中：`{"read":[{"ref_id":"<returned library_file_id>"}]}`。顶层的 `read` 数组必须包含 1 至 5 个条目；切勿在顶层直接发送标识符。

尽可能将独立的 `read` 和 `find` 请求批量处理。即使已有搜索片段，针对内容主张也应在搜索后使用 `read`；只有在候选目标解析完成后才使用 `find`。

有关选定的 `@` 提及、图像搜索元数据、变更后的搜索限制、结果处理以及引用，请参阅 [evidence-and-citations.md](references/evidence-and-citations.md)。

## 分类新文件

对于新文件，在支持的情况下设置 `library_artifact_type`：- `image_gen`: 仅由 imagegen 生成的图片。
- `image`: 其他生成的图片。
- `report`、`sheet`、`slides`: 生成的文档/报告、电子表格或演示文稿。
- `other`: 用户导入的文件、未知的生成记录或其他情况。

按生成历史分类，而非按文件名、扩展名或 MIME 类型。
`create_library_file` 每次调用只指定一种类型；预准备上传则每文件指定一种类型。
对于替换操作，或当上传工具或辅助程序不支持时，可省略该字段。

## 写入单个本地文件

对于大小在约 50 MiB 以下的已确认本地文件，可使用此快速路径。复用在生成或编辑该工件时已完成的验证结果，不要仅因文件将被保存至 Library 而再次进行内容检查。

- 对于新项目，调用 `create_library_file(file=...)` 并传入本地绝对路径。成功后，使用本技能的 [scripts/library_file_transfer.py](scripts/library_file_transfer.py)，以 `python3` 执行 `apply-xattrs` 子命令，传入原始本地路径和返回的 `library_file_id`。将创建结果中的完整 `xattrs` 数组（或 `[]`）以 JSON 格式通过标准输入传递，如下所示。若使用代码模式（例如 `functions.exec`），可在同一调用中先后执行 `create_library_file` 和元数据辅助脚本，无需在两者之间返回模型。
- 对于已有项目，先确定其 Library 身份，并在本地工作文件已包含预期结果时直接使用该文件，切勿覆盖原文件。如需重新生成，则应用缺失的修改并进行一次验证。对于自有文件的替换，使用 `replace_library_file(file=...)`，并传入相同的 `library_file_id`。共享存储的文件始终使用 `library_upload.py`，即使没有预准备工具亦然。共享写入块也使用 `library_upload.py` 及其内置的预准备上传辅助脚本；将其十进制的 `version_id` 转换为整数形式的 `expected_current_version`。
- 编辑器的新输出路径仍视为对同一 Library 项目的替换。仅在尚无 Library 身份或用户希望获得独立副本时才新建。
- 当已保留具体版本号时，请传入 `expected_current_version`。切勿自行编造版本号，亦不得为解决冲突而移除版本保护机制。

创建或替换完成后，请检查返回结果。以其中的文件名和 Library 路径为准，并同时保存精确的 `library_file_id`、`file_id`、版本号及原始本地路径。除非确有必要进行精确验证，否则无需仅为了确认写入成功而重新读取文件。

在一次辅助脚本调用中，私下持久化返回的 `xattrs` 和 Library 身份，无需额外添加进度提示。辅助脚本的调用格式为：`apply-xattrs PATH LIBRARY_FILE_ID`，两个位置参数均为必填：

```bash
skill_md_path="<本 SKILL.md 的绝对路径>"
transfer_helper_path="$(dirname "$skill_md_path")/scripts/library_file_transfer.py"
python3 "$transfer_helper_path" \
  apply-xattrs "$local_path" "$library_file_id" <<'JSON'
<完整的返回 xattrs 数组，或 []>
JSON
```

完成前请检查辅助脚本的执行结果；若失败，则不应声称本地身份已成功持久化。有关批量创建、版本冲突或详细结果关联的信息，请参阅 [writeback-and-conflicts.md](references/writeback-and-conflicts.md)。

## 处理列表或搜索结果的实体化对于由 `list` 或 `search` 返回的文件，请保留完整的结构化结果，并使用打包的下载辅助工具。请在希望下载树存放的工作区中运行该工具。当处于会话作用域的工作区激活时，应使用该工作区，以便符合条件的字节能够直接放置到位。通过标准输入传递完整的、未经修改的 `list` 或 `search` JSON、一个 `ALL` 选项或拼接的三位索引选择（例如 `000002` 表示选择第一个和第三个文件），以及一个相对目标路径。切勿将返回的字段插值到 shell 参数中。将您读取的 Library `SKILL.md` 的绝对路径复制到 `skill_md_path` 变量中，然后使用下面的带引号的 here document。请勿从插件缓存根目录重新构造或缩短辅助工具的路径：

```bash
skill_md_path="<此 SKILL.md 的绝对路径>"; \
python3 "$(dirname "$skill_md_path")/scripts/library_download.py" <<'JSON'
{"result": <完整的 list 或 search JSON>,
 "selection": "ALL|NNN[NNN...]",
 "destination": "<相对目录>"}
JSON
```

目标路径是指在其下根据所选文件在 Library 中的规范相对路径重新创建这些文件的父目录。请勿在其中重复已选的 Library 根目录：对于 `/fruits/apple.md`，应使用 `downloads` 而非 `downloads/fruits`，除非用户明确要求额外的嵌套层级。

对于搜索结果，索引首先指向 `results`，然后再指向 `retrieval_title_results`。后者是补充性的模糊匹配候选，因此当它们并非全部相关时，请使用显式索引而非 `ALL`。

该辅助工具会创建或复用目标目录，覆盖每个被选中的文件，并保持无关内容不变。它以每批最多 20 个为单位进行经过身份验证的 `prepare_materialize` 调用，同时支持工作区传输和签名 URL 传输，并应用 Library 的身份标识与扩展属性。无论是 `list` 还是 `search` 结果，均保留每个文件在 Library 中的规范相对路径。请使用返回的 `directory` 和权威的 `files` 路径。

在此流程中，切勿单独调用 `read` 或 `prepare_materialize`，也勿在插件缓存中搜索辅助工具、检查辅助工具源码、直接调用 `library_file_transfer.py`、自行传输返回的 URL，或对返回的传输结果再次处理。

对于并非来自 `list` 或 `search` 的已解析引用，请使用 [materialization.md](references/materialization.md) 中的低层流程。

## 处理较大或多个写入操作

将单个用户任务写入的每个本地文件视为一个有序的上传批次。请保留其原始的变更顺序，并使用绝对本地路径。

| 条件 | 必选流程 |
| --- | --- |
| 同时具备准备好的工具，且任务写入多个文件，或写入一个约 50 MiB 或更大的文件 | 使用下方打包的预准备上传辅助工具。 |
| 准备好的工具不可用，且所有操作均为创建 | 当 `files` 参数可用时，按顺序使用 `create_library_file(files=[...])` 批次，每次调用不超过 500 MB；否则按顺序逐个创建。 |
| 准备好的工具不可用，且任务包含替换操作 | 按照原始顺序，对共享文件使用上传辅助工具，对自有文件则直接操作。 |

经准备的应用程序调用最多包含 20 个文件。Library 写入相关的应用程序调用必须按序执行：对于创建、替换、删除或最终确认操作，切勿使用 `Promise.all(...)`。只有经过准备的字节传输可以并行进行。在使用预准备流程之前，请阅读 [prepared-uploads.md](references/prepared-uploads.md)。该文档定义了辅助工具的一次性输入，并负责准备、传输、最终确认、结果关联及扩展属性的回写。在直接批次或预准备辅助工具之后，请按请求顺序检查每个项目的处理结果。顶层操作成功并不意味着所有项目都已成功。

## 站点支持的 Library 项目`library_artifact_type: site` 项会投影出规范的 `site_metadata.project_id`。`list`、`search`、`read` 和 `find` 操作仍然允许。`manage_library` 权限可以移动它；重命名会更改站点标题，删除则会删除该站点。切勿对其进行物化、下载、打补丁、替换、更新、覆盖或恢复；站点自行管理其内容、版本及发布历史。若所有权或路由不明确，请立即停止。

## 整理、恢复与保护文件

在写入之前，应先明确可能的变更目标。若仍有多个候选对象，应请用户进行选择。有关确切的文件夹规则、移动、重命名、删除、恢复、受保护的深度研究报告以及各操作的结果规则，请参阅 [library-management.md](references/library-management.md)。

## 隐私与安全

除非用户明确要求获取实现细节，否则应在 Library 层面呈现用户可见的逻辑说明、进度、错误信息及最终响应。不得暴露连接器或工具名称、辅助命令、原始 URL、存储或提供商详情、清单、扩展属性、传输输出或索引内部信息。

隐私设置仅影响叙述方式，不影响路由流程。切勿仅因预准备流程具有更严格的可见性规则，就将其替换为直接上传。对于预准备上传，在调用辅助程序之前，应提供一条简要的“正在将文件保存至 Library”或“正在将文件保存至 Library”的进度提示，随后再返回保存结果。不要将准备、传输、最终处理或本地元数据分别作为独立阶段进行叙述。

不得虚构 Library ID、文件 ID、版本、文件名、路径、操作或工具可用性。确保签名 URL 不出现在响应中。在每次变更过程中，均应保持 Library 的身份及其无关内容的一致性。

## 参考资料

- [evidence-and-citations.md](references/evidence-and-citations.md)：提及、搜索详情、结果处理及引用。
- [materialization.md](references/materialization.md)：低层级已解析引用的物化及传输处理。
- [prepared-uploads.md](references/prepared-uploads.md)：预准备字节传输、最终处理、扩展属性回写及清理。
- [writeback-and-conflicts.md](references/writeback-and-conflicts.md)：详细的创建、替换、编辑及冲突处理机制。
- [library-management.md](references/library-management.md)：文件夹、节点变更、恢复及受保护报告。