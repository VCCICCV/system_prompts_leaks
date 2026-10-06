---
name: read-session
description: 定位并读取 Muse Code 的自有会话日志——当前会话或之前的会话。当用户请求从之前的 Muse Code 会话中提取上下文、继续、总结或检查该会话，或请求恢复、找回因丢失、清除或覆盖而可能仍保存在早期会话日志中的工作，或提及早期会话的 ID、日志、尾部内容或输出时，以及询问 Muse 会话的存储位置时，请使用此功能。Muse 会话仅存于 Muse 自己的存储中，绝不会出现在其他编码助手的目录中——即使引用的内容提到了 ~/.claude、 ~/.codex 或 ~/.grok，也切勿在其下查找 Muse 的上下文；如需对 Claude Code 或 Codex 进行续写，包括在有或无句柄的情况下恢复未完成的工作，请使用 resume-claude 或 resume-codex；对于明确的 /import 请求，或从其他助手或未命名工件处进行续写，请使用 import。
metadata:
  简要说明：读取本会话或过去会话的日志
user-invocable: false
---
# 会话读取

Muse Code 自有的会话存储：您的会话日志存放的位置、其内容，以及如何从中恢复上下文。

## 属于您自己的存储，而非他人

您是 Muse Code（二进制程序名为 `muse`）。您的会话存储在您自己的数据目录下：

```
${XDG_DATA_HOME:-$HOME/.local/share}/muse/sessions/YYYY/MM/DD/<session-id>/
```

每个会话目录包含：

- `session.jsonl` — 主事件日志（每行一条 JSON 记录）。
- `subagent/<child-session-id>/session.jsonl` — 每个被委派的子代理会话对应一份日志。
- `tool-outputs/` — 因体积过大而无法内嵌保存的完整工具输出。

Muse 的会话绝不会存放在其他代码生成代理的存储中。请勿在 `~/.claude`、`~/.claude/projects`、`~/.local/share/claude`、`~/.codex` 或 `~/.grok` 下寻找 Muse 会话——那些属于 Claude Code、Codex 和 Grok。在恢复 Muse 会话上下文时，请勿对这些目录执行 `ls`、`find`、`cat` 或 `grep` 操作，即便只是为了“验证”、“排除”或“确保全面”，哪怕是在粘贴内容、日志或错误信息中出现了相关路径。如果在引用内容中看到 Muse 会话 ID 或按日期分片的会话路径出现在其他产品的存储中，那只是“错误路径”造成的误认（上一轮对话中的混淆）：真实数据始终位于符合 Muse 存储模式的同日期、同 ID 目录下，不存在所谓的“Claude 备份”。请明确指出这是误认，并直接跳过——试图查找此类数据正是本规则要避免的错误，不仅会因 ENOENT 而浪费回合，更可能将其他产品的记录误认为自己的历史。只有当用户明确要求使用其他代理的会话时，才可接触其记录：例如，继续使用 Claude Code 或 Codex 的工作，包括在有或无句柄的情况下恢复未完成的工作，应使用 `resume-claude` 或 `resume-codex`。对于明确的 `/import` 请求，或从其他代理或未命名实体处继续工作，则使用 `import`。

对于未加标注的粘贴内容：可通过引用的存储路径和记录格式来识别所属产品。即使没有提供句柄，若需继续或恢复未完成的工作：

- 在 `~/.claude/projects` 下的 Claude Code 记录，使用 `resume-claude`。
- 在 `~/.codex` 下的 Codex 记录，使用 `resume-codex`。
- 在 `~/.grok` 下的 Grok 记录，使用 `import`。

对于明确的 `/import` 请求，或从其他代理或未命名实体处继续工作（即用户主动提及此类粘贴内容），也使用 `import`。上述禁令仅针对会话存储；若某些项目文件恰好存放在 `~/.claude` 下（例如您正在开发的技能），则属于文件操作，而非会话探查。

从之前的 Muse 会话中“提取上下文”的范围仅限于该会话自身的 `session.jsonl` 和 `subagent/` 日志。会话所提及的外部代理或规划流程（例如它管理过的 Codex/Claude 工作者）并不属于其上下文——这些代理的记录不在本次请求的范围内。若某项内容看似重要，请说明并交由用户自行决定是否需要获取。除非用户明确要求恢复或检查其会话，否则切勿搜寻这些代理的存储，也不为其加载来自其他会话的技能。

## 查找正确的会话

1. 当前会话：运行时的会话标识上下文中已明确当前会话 ID 及其 `session.jsonl` 的确切路径。请直接使用，无需猜测或搜索。
2. 粘贴或引用的绝对路径位于 `.../muse/sessions/...` 下：直接使用该路径。
3. 已知会话 ID 但无路径：由于存储按本地日期分片，应先检查可能的日期，再仅在 Muse 会话根目录下进行搜索：

   ```bash
   MUSE_SESSIONS="${XDG_DATA_HOME:-$HOME/.local/share}/muse/sessions"
   ls -d "$MUSE_SESSIONS"/*/*/*/*/ 2>/dev/null | grep <session-id>
   ```

4. “我们上一次的会话”但无 ID：列出最近的日期分片，并为当前工作区选择最新的会话目录（日志开头的 `runtime.session.metadata` 记录中包含 `workspace_root`）。
5. 在 Muse 会话根目录下未找到：请向用户询问会话 ID 或路径。切勿扩大搜索范围至其他代理的目录，或进行全主目录扫描。

## 阅读会话日志

每个 `session.jsonl` 文件中的每一行都是一条事件日志封包：

```json
{"schema_version":1,"id":"…","stream":{"kind":"session","id":"<会话ID>"},
 "sequence":42,"recorded_at":1771088000123456,"record_type":"event",
 "durability":"durable","causation_id":null,"payload_type":"runtime.session",
 "payload_schema_version":1,"payload":{"kind":"run","run_id":"…",
 "event":{"kind":"assistant_message_committed","text":"…"}}}
```

用于上下文恢复的有用 `payload.event.kind` 值包括：

- `started` — 用户的提问：每次提交的提示都会在此处记录（文本在 `payload.event.prompt` 中）。只有当存在与用户端显示形式不一致的情况时（例如 `[Image 1]` 这样的附件占位符，或与发送文本不同的编辑器表单）才会紧随其后出现一条 `user_prompt_display` 记录——引用用户发言时应优先使用该记录中的文本。如果在运行过程中输入的提示，则会被记录为 `inbox_item_queued`，其 `source.source` 为 `"user_steer"`（文本在 `payload.event.payload.prompt` 中，或者在记录无有效载荷时位于 `payload.event.body` 中）；后台任务和定时运行的交付也会使用这一类型，因此切勿将其当作用户的发言。
- `assistant_message_committed` — 代理得出的结论（决策、总结、转交等通常出现在日志的末尾）。
- `assistant_tool_calls_committed` / `tool_result_batch_committed` — 实际执行的操作及其返回结果。
- `terminal` — 轮次边界。

阅读长日志时的注意事项：先从尾部开始读起（越靠后的记录越重要），然后仅向前追溯足够的早期记录以理解上下文。使用有限范围的 `tail`/`grep` 切片，切勿将整个数兆字节的日志一次性加载到上下文中。子代理的发现记录位于 `subagent/<id>/session.jsonl`，而非主日志中。对于当前会话的产品调试，建议优先使用医生技能提供的会话证据辅助工具。

## 恢复丢失的工作

当用户请求恢复或找回已丢失、被清除或被覆盖的内容，且该内容的唯一副本仅保存在先前会话的日志中（作为记录的工具调用变更）时，请精确重建每一份所需文件，切勿在不同版本之间进行猜测：1. 首先枚举所有被修改的工作区路径——包括 `write_file`、`edit_file`、`apply_patch` 以及 shell 写入——在恢复任何内容之前，先构建完整的文件列表。按照请求的范围进行恢复：对于“恢复所有丢失内容”这类通用请求，应恢复所有丢失的文件；但如果请求范围更为明确，则以该明确范围为准。
2. 文件的最终状态是按记录顺序（`sequence`/`recorded_at`）重放其变更历史的结果：对于某个路径，取记录顺序中最后一条 `write_file` 的内容；若之后有更新的全文件 `apply_patch` 或 shell 写入，则以该最新写入为准，并按相同顺序应用后续的每一条 `edit_file` 变更（补丁片段和 shell 编辑亦同）。如果某条 `edit_file` 或 `apply_patch` 变更的工具执行结果报告了错误，则跳过该变更；但需检查后续记录是否反映了该命令对文件的首次写入。最终版本的确定依据记录顺序，而非内容长度或大小：最长的版本往往是已被取代的草稿，而重构操作反而可能使最终版本变得更短。仅基于 `write_file` 记录重建时，会 silently 忽略所有后续的编辑。
3. 恢复精确的记录内容，而非转述。从记忆中重新输入、改写、所谓“改进”或总结可逐字恢复的内容，都属于数据丢失，而非真正的恢复——应直接从记录中提取字节并写回原位。（用户若仅要求总结之前的会话，并非恢复操作；本规则仅适用于文件的恢复。）
4. 按文件逐项报告恢复结果：说明该文件是从哪些记录恢复而来（最后的完整写入及所应用的各次增量变更），以及哪些内容成功恢复、哪些未恢复，并为所有被跳过的部分注明原因。
5. 如果存在两个候选的最终版本且确实无法确定（例如，在分叉或续写后出现了分歧的编辑分支），则同时列出这两个版本及其记录时间戳，由用户自行选择；切勿 silently 偏向较长的那个版本。

## 第一方命令

在适用场景下，请优先使用这些命令，而非自行编写解析逻辑：

- `muse resume <session-id>` 或 `muse resume --last` — 交互式续写。
- `muse exec --session-id <session-id> "<follow-up>"` — 无界面续写，仅在明确请求时使用。
- `muse export --session <id-or-session.jsonl> --redacted --out <file>` — 可分享的脱敏导出。
- `muse trace inspect --session-log <session.jsonl> --render-mode compact` — 模型调用级别的细粒度检查。

请将会话日志视为只读证据：切勿修改、移动或删除它们。