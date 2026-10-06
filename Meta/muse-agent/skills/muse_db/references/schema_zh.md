# `muse.db` 的 PostgreSQL 模式

本指南描述了守护进程原生工具 `muse.db` 可用的数据库关系。在 SQL 查询中请使用模式限定名。该工具仅接受一个受限制的只读 `SELECT` 语句；它用于诊断和跨表追踪，而非替代专门构建的动态、想法、聊天、目标、工件或连接器工具。

迁移集指纹：`9b8575628206745e9350b3b7df759d9bcfbe1ebb0cdc33596255d6edcd65980c`。

以下列出的所有 Muse 应用基础表均可查询。PostgreSQL 系统目录、迁移记录、备份表、凭据、Sentinel 的独立审批存储以及每个工件对应的 `app.db` 文件均不在本接口范围内。被描述为“外部”或“不透明”的标识符没有可本地关联的所有者表。非递归 CTE 的名称必须以 `hatch_cte_` 开头；递归 CTE 将被拒绝。

此处，“凭据”指 OAuth 访问令牌或刷新令牌、密码、API 密钥，以及支付工具的机密信息（如卡号或 CVV）；这些信息始终由 authd 或其所属的保险库保管。非机密的生命周期元数据（包括 Stripe Link 的支出请求行）则在下文列出时仍可查询。

模型的私有推理过程在此处完全不可读。思考条目及由提供方加密的已脱敏思考条目与普通转录记录一同存储，因此承载这些条目的每张表均通过脱敏的安全屏障视图提供服务；评论文本仍可读取。每张受影响表的注释中会说明其是否过滤掉推理行、隐藏嵌入推理的列，或两者兼有。被隐藏的列不属于本工具所服务的表的一部分，因此查询中引用此类列将报未知列错误；被过滤的行则直接不显示，不会报错。在将缺失或未知列错误视为记录中的空缺之前，请先查看表的注释：若注释未提及行过滤，则缺失的行确为真实空缺；若未提及列隐藏，则未知列是查询中的错误。此外，该工具以最低权限数据库角色身份运行，仅对本指南中列出的关系具有 SELECT 权限，因此本指南之外的内容均无法访问。

## SQL 能力

仅允许使用下述内置函数和类型转换写法。由于 PostgreSQL 允许在 `SELECT` 中使用带有副作用的函数和转换，`muse.db` 会拒绝所有未列入此审查清单的内容。若错误提示使用了不支持的函数或转换，请改用列表中的操作重写查询，而不要据此推断记录缺失。

函数：`abs`、`age`、`array_length`、`array_position`、`array_to_string`、`avg`、`bit_and`、`bit_or`、`bool_and`、`bool_or`、`btrim`、`cardinality`、`ceil`、`ceiling`、`char_length`、`coalesce`、`concat`、`concat_ws`、`count`、`date_bin`、`date_part`、`date_trunc`、`dense_rank`、`encode`、`every`、`extract`、`first_value`、`floor`、`greatest`、`json_array_length`、`json_build_array`、`json_build_object`、`json_extract_path`、`json_extract_path_text`、`json_object_keys`、`json_typeof`、`jsonb_array_length`、`jsonb_build_array`、`jsonb_build_object`、`jsonb_extract_path`、`jsonb_extract_path_text`、`jsonb_object_keys`、`jsonb_typeof`、`lag`、`last_value`、`lead`、`least`、`left`、`length`、`lower`、`ltrim`、`make_interval`、`max`、`md5`、`min`、`mod`、`now`、`nth_value`、`ntile`、`nullif`、`octet_length`、`position`、`power`、`rank`、`regexp_match`、`regexp_replace`、`replace`、`reverse`、`right`、`round`、`row_number`、`rtrim`、`split_part`、`sqrt`、`strpos`、`substr`、`substring`、`sum`、`time_bucket`、`timezone`、`to_char`、`to_json`、`to_jsonb`、`trim`、`upper`。

类型转换写法：`bigint`、`bool`、`boolean`、`bytea`、`char`、`character`、`character varying`、`date`、`decimal`、`double precision`、`float4`、`float8`、`int`、`int2`、`int4`、`int8`、`integer`、`interval`、`json`、`jsonb`、`numeric`、`real`、`smallint`、`text`、`time`、`timestamp`、`timestamp with time zone`、`timestamp without time zone`、`timestamptz`、`timetz`、`uuid`、`varchar`。

诸如 `json_agg`、`jsonb_agg`、`array_agg` 和 `string_agg` 等集合聚合函数被有意禁用，因为它们可能会在应用外层行数和字节限制之前构建出一个无界的值。请直接选择受限的行。

在连接表时，请为投影列使用唯一的别名。包含重复输出列名的查询会在执行前被拒绝，因为 JSON 对象无法同时保留这两个值。

## 标识符解析器索引

常见的软引用，并不总是被声明为 PostgreSQL 的外键：

| 标识符族 | 解析于 |
|---|---|
| `action_argument_id` | `spaces.action_arguments.action_argument_id` |
| `action_id` | `activity.activity_monitor_thread_actions.action_id<br>goals.actions.action_id` |
| `activation_id` | `runtime.idea_execution_pending.activation_id` |
| `active_root_id` | `agent.agents.id` |
| `activity_key` | `activity.feed_entries.activity_key` |
| `activity_thread_id` | `activity.activity_monitor_thread_sections.activity_thread_id<br>activity.activity_monitor_threads.activity_thread_id` |
| `agent_id` | `agent.agents.agent_id` |
| `agent_identity` | `runtime.agent_todo_snapshots.agent_identity` |
| `aggregate_id` | `health.aggregates.aggregate_id` |
| `ancestor_id` | `agent.agent_ancestors.ancestor_id` |
| `anchor_id` | `ideas.idea_anchors.anchor_id` |
| `anchor_kind` | `ideas.idea_anchors.anchor_kind` |
| `anchored_idea_ids` | `ideas.ideas.idea_id` |
| `asset_id` | `ideas.idea_install_assets.asset_id` |
| `assigned_browser_task_id` | `runtime.browser_tasks.task_id` |
| `association_id` | `goals.associations.association_id` |
| `attachment_id` | `runtime.message_attachments.attachment_id` |
| `attempt` | `feed.run_steps.attempt` |
| `audit_id` | `self_improvement.connector_read_audit.audit_id` |
| `backfill_day_run_id` | `self_improvement.backfill_day_runs.backfill_day_run_id` |
| `batch_id` | `device.media_upload_batches.batch_id` |
| `binding_id` | `chat.message_bindings.binding_id` |
| `brief_id` | `self_improvement.relationship_briefs.brief_id` |
| `briefing_id` | `goals.briefings.briefing_id` |
| `calibration_id` | `self_improvement.calibration_records.calibration_id` |
| `call_id` | `runtime.workflow_agent_calls.call_id` |
| `call_log_id` | `device.call_log.call_log_id` |
| `cancellation_confirmation_response_message_id` | `runtime.messages.message_id` |
| `canonical_idea_id` | `ideas.ideas.idea_id` |
| `carrier_message_id` | `runtime.messages.message_id` |
| `chat_id` | `chat.chats.chat_id` |
| `checkpoint_key` | `runtime.checkout_spend_checkpoints.checkpoint_key` |
| `child_agent_id` | `agent.agents.agent_id` |
| `child_message_id` | `runtime.messages.message_id` |
| `chunk_index` | `device.upload_chunks.chunk_index` |
| `claim_id` | `memory.claims.claim_id` |
| `collection_id` | `runtime.raw_signal_collections.collection_id` |
| `compaction_id` | `agent.compactions.compaction_id` |
| `contact_address_id` | `device.contact_addresses.contact_address_id` |
| `contact_email_id` | `device.contact_emails.contact_email_id` |
| `contact_id` | `device.contacts.contact_id` |
| `contact_phone_id` | `device.contact_phones.contact_phone_id` |
| `content_hash` | `self_improvement.handoff_dedupe.content_hash` |
| `context_item_field_id` | `agent.context_item_fields.context_item_field_id` |
| `context_item_id` | `agent.context_item_derived_write_backlog.context_item_id<br>agent.context_items.context_item_id` |
| `context_text_segment_id` | `agent.context_text_segments.context_text_segment_id` |
| `continuation_root_task_id` | `runtime.browser_tasks.task_id` |
| `contribution_id` | `feed.fleet_engagement_contributions.contribution_id<br>feed.fleet_engagement_outbox.contribution_id` |
| `conversation_id` | `runtime.agent_todo_snapshots.conversation_id` |
| `coordinator_agent_id` | `agent.agents.agent_id` |
| `created_from_proposal_id` | `spaces.proposals.proposal_id` |
| `current_full_sync_id` | `device.upload_sessions.upload_session_id` |
| `current_full_sync_requester_root_session_id` | `agent.sessions.session_id` |
| `data_source` | `device.data_sync_state.data_source` |
| `decision_id` | `agent.subagent_monitor_decisions.decision_id` |
| `delivery_key` | `scheduler.delivery_outbox.delivery_key` |
| `delivery_submission_id` | `agent.message_mailbox.submission_id` |
| `descendant_id` | `agent.agent_ancestors.descendant_id` |
| `descriptions_digest` | `ideas.icon_embeddings.descriptions_digest` |
| `device_node_id` | `device.nodes.node_id` |
| `domain` | `ideas.bandit_arm_state.domain` |
| `embedding_model_id` | `memory.embedding_models.embedding_model_id` |
| `entry_id` | `runtime.raw_signal_entries.entry_id` |
| `event_id` | `agent.subagent_progress_message_events.event_id<br>agent.subagent_progress_tool_events.event_id<br>goals.engagement_events.event_id<br>ideas.bandit_folded_events.event_id<br>ideas.idea_events.event_id<br>ingest.data_source_events.event_id` |
| `event_payload_field_id` | `runtime.event_payload_fields.event_payload_field_id` |
| `event_seq` | `chat.event_transports.event_seq<br>runtime.chat_event_derived_write_backlog.event_seq<br>runtime.events.event_seq` |
| `event_type` | `agent.recovery_owner_terminal_events.event_type` |
| `execute_message_id` | `runtime.messages.message_id` |
| `exif_value_id` | `media.exif_values.exif_value_id` |
| `external_event_id` | `device.calendar_events.external_event_id` |
| `feed_id` | `ideas.idea_card_feeds.feed_id<br>podcasts.feeds.feed_id` |
| `folded_event_id` | `ideas.bandit_folded_events.event_id` |
| `generation` | `agent.recovery_owner_terminal_events.generation` |
| `goal_id` | `goals.goals.goal_id` |
| `handoff_message_id` | `runtime.messages.message_id` |
| `health_event_id` | `health.events.health_event_id` |
| `history_id` | `goals.momentum_history.history_id` |
| `history_source_agent_id` | `agent.agents.agent_id` |
| `hook_id` | `runtime.event_hook_space_owners.hook_id` |
| `icon_key` | `ideas.icon_embeddings.icon_key` |
| `id` | `agent.agent_compactions.id<br>agent.chat_preferences.id<br>runtime.summaries.id<br>self_improvement.learning_adoption_events.id` |
| `idea_card_id` | `ideas.ideas.idea_id` |
| `idea_id` | `ideas.ideas.idea_id` |
| `idea_item_id` | `ideas.idea_items.idea_item_id` |
| `ingest_id` | `ingest.data_source_events.ingest_id` |
| `input_message_ids` | `runtime.messages.message_id` |
| `interaction_id` | `feed.interactions.interaction_id` |
| `invocation_id` | `spaces.action_invocations.invocation_id<br>spaces.action_results.invocation_id` |
| `item_key` | `shell.user_state.item_key` |
| `jarvis_message_id` | `runtime.messages.message_id` |
| `job_definition_id` | `scheduler.job_definitions.job_definition_id` |
| `job_id` | `scheduler.jobs.job_id` |
| `key` | `ideas.discovery_pool_meta.key<br>memory.metadata.key` |
| `lane` | `device.media_upload_batches.lane<br>ideas.bandit_arm_state.lane` |
| `launch_occurrence_agent_id` | `runtime.workflow_launch_occurrence_aliases.launch_occurrence_agent_id<br>runtime.workflow_legacy_launch_occurrence_blocks.launch_occurrence_agent_id` |
| `launch_occurrence_message_id` | `runtime.workflow_launch_occurrence_aliases.launch_occurrence_message_id<br>runtime.workflow_legacy_launch_occurrence_blocks.launch_occurrence_message_id` |
| `launch_occurrence_tool_call_id` | `runtime.workflow_launch_occurrence_aliases.launch_occurrence_tool_call_id<br>runtime.workflow_legacy_launch_occurrence_blocks.launch_occurrence_tool_call_id` |
| `launching_agent_id` | `agent.agents.agent_id` |
| `launching_message_id` | `runtime.messages.message_id` |
| `launching_tool_call_id` | `runtime.tool_calls.tool_call_id` |
| `lease_key` | `self_improvement.leases.lease_key` |
| `link_seq` | `activity.activity_monitor_carrier_user_messages.link_seq` |
| `marker` | `self_improvement.objective_markers.marker<br>spaces.backfill_markers.marker` |
| `marker_key` | `runtime.maintenance_markers.marker_key` |
| `media_id` | `media.descriptions.media_id<br>media.items.media_id<br>media.locations.media_id` |
| `media_upload_event_id` | `device.media_upload_events.media_upload_event_id` |
| `memory_embedding_id` | `memory.embeddings.memory_embedding_id` |
| `memory_entry_attribute_id` | `memory.entry_attributes.memory_entry_attribute_id` |
| `memory_entry_id` | `memory.entries.memory_entry_id` |
| `message_id` | `runtime.messages.message_id` |
| `model_name` | `ideas.icon_embeddings.model_name` |
| `mutation_seq` | `scheduler.cron_mutations.mutation_seq` |
| `node_id` | `device.nodes.node_id` |
| `objective_id` | `self_improvement.handoff_ded`upe.objective_id<br>self_improvement.objective_markers.objective_id<br>self_improvement.objective_state.objective_id` |
| `occurrence` | `self_improvement.conversation_follow_up_attempts.occurrence` |
| `operation_id` | `runtime.checkout_spend_checkpoints.operation_id<br>runtime.checkout_spend_operations.operation_id` |
| `owner_agent_id` | `agent.agents.agent_id` |
| `owner_browser_task_id` | `runtime.browser_tasks.task_id` |
| `owner_id` | `agent.recovery_owner_terminal_events.owner_id<br>agent.recovery_owners.owner_id` |
| `owner_key` | `runtime.context_snapshots.owner_key` |
| `parent_agent_id` | `agent.agents.agent_id` |
| `parent_goal_id` | `goals.goals.goal_id` |
| `parent_id` | `agent.agents.id` |
| `parent_message_id` | `runtime.messages.message_id` |
| `parent_request_id` | `runtime.requests.request_id` |
| `parent_work_id` | `runtime.work_items.work_id` |
| `payload_hash` | `feed.fleet_fetch_receipts.payload_hash` |
| `phase_run_id` | `runtime.workflow_phase_runs.phase_run_id` |
| `policy_id` | `ideas.bandit_arm_state.policy_id<br>ideas.bandit_fold_state.policy_id<br>ideas.bandit_folded_events.policy_id<br>ideas.explore_policy.policy_id` |
| `position` | `ideas.feed_snapshot_cards.position<br>ideas.idea_tags.position` |
| `presentation_root_session_id` | `agent.sessions.session_id` |
| `producer_agent_id` | `agent.agents.agent_id` |
| `progress_id` | `agent.subagent_progress.progress_id` |
| `prompt_id` | `feed.promptless_unit_orders.prompt_id<br>feed.prompts.prompt_id` |
| `proposal_id` | `spaces.proposals.proposal_id` |
| `provider` | `chat.event_transports.provider` |
| `record_value_id` | `health.record_values.record_value_id` |
| `recovery_class` | `agent.recovery_owner_terminal_events.recovery_class<br>agent.recovery_owners.recovery_class` |
| `recovery_owner_id` | `agent.recovery_owners.owner_id` |
| `replay_id` | `self_improvement.calculation_records.replay_id` |
| `reply_target_message_id` | `runtime.messages.message_id` |
| `reply_to_message_id` | `runtime.messages.message_id` |
| `request_id` | `runtime.requests.request_id` |
| `resolve_message_id` | `runtime.messages.message_id` |
| `resource_id` | `runtime.resources.resource_id` |
| `resume_at_utc` | `scheduler.scheduled_resume_registrations.resume_at_utc` |
| `root_agent_id` | `agent.agents.agent_id` |
| `root_goal_id` | `goals.goals.goal_id` |
| `root_message_execution_id` | `runtime.messages.message_id` |
| `root_message_id` | `runtime.messages.message_id` |
| `root_request_id` | `runtime.requests.request_id` |
| `root_session_id` | `agent.sessions.session_id` |
| `root_submission_message_id` | `runtime.messages.message_id` |
| `root_work_id` | `runtime.work_items.work_id` |
| `run_id` | `feed.run_steps.run_id<br>feed.runs.run_id<br>runtime.execute_resolve_runs.run_id<br>runtime.workflow_runs.run_id<br>scheduler.doctor_run_plans.run_id<br>scheduler.job_runs.run_id<br>scheduler.terminal_signals.run_id<br>self_improvement.runs.run_id` |
| `sample_id` | `health.samples.sample_id` |
| `sample_value_id` | `health.sample_values.sample_value_id` |
| `scheduler_event_id` | `scheduler.events.scheduler_event_id` |
| `search_document_id` | `runtime.search_documents.search_document_id` |
| `section_id` | `ideas.feed_snapshot_cards.section_id<br>ideas.feed_snapshot_sections.section_id` |
| `section_key` | `activity.activity_monitor_thread_sections.section_key` |
| `selected_item_ids` | `ideas.idea_items.idea_item_id` |
| `seq` | `agent.context_item_resume_projection.seq` |
| `session_id` | `agent.sessions.session_id` |
| `singleton` | `feed.fleet_publish_state.singleton<br>feed.null_state_seed.singleton<br>feed.prompt_scope_verdict.singleton<br>feed.prompt_seed.singleton<br>feed.surface_state.singleton<br>podcasts.feed.singleton<br>runtime.avatar_state.singleton<br>runtime.client_rendering_capabilities.singleton<br>runtime.dev_notice_watermark.singleton<br>runtime.invite_badge_seen_state.singleton<br>runtime.product_improvements_preference.singleton<br>runtime.writer_epoch.singleton<br>scheduler.scheduled_resume_state.singleton` |
| `singleton_id` | `feed.preferences_projection.singleton_id` |
| `skill_name` | `runtime.skill_invalidation_state.skill_name` |
| `sleep_session_id` | `health.sleep_sessions.sleep_session_id` |
| `slug` | `podcasts.episodes.slug<br>spaces.file_artifact_identities.slug` |
| `snapshot_id` | `ideas.discovery_pool_history.snapshot_id<br>ideas.feed_snapshot_cards.snapshot_id<br>ideas.feed_snapshot_sections.snapshot_id<br>ideas.feed_snapshots.snapshot_id` |
| `snapshot_kind` | `runtime.context_snapshots.snapshot_kind` |
| `source` | `shell.user_state.source` |
| `source_id` | `ideas.idea_build_status.source_id<br>ideas.idea_sources.source_id` |
| `source_idea_id` | `ideas.ideas.idea_id` |
| `source_kind` | `ideas.idea_build_status.source_kind<br>ideas.idea_sources.source_kind` |
| `source_media_id` | `media.items.media_id` |
| `source_namespace` | `ideas.idea_build_status.source_namespace<br>ideas.idea_sources.source_namespace` |
| `source_root_agent_id` | `agent.agents.agent_id` |
| `source_session_id` | `agent.sessions.session_id` |
| `space_id` | `spaces.spaces.space_id` |
| `space_slug` | `spaces.shares.space_slug<br>spaces.spaces.space_slug<br>spaces.user_state.space_slug` |
| `spawn_call_id` | `runtime.tool_calls.tool_call_id` |
| `spawn_id` | `agent.subagent_spawns.spawn_id` |
| `step_id` | `feed.run_steps.step_id` |
| `stream_owner_message_id` | `runtime.messages.message_id` |
| `submission_id` | `agent.message_mailbox.submission_id` |
| `suggestion_id` | `goals.suggestions.suggestion_id` |
| `synced_range_id` | `health.synced_ranges.synced_range_id` |
| `task_id` | `runtime.browser_tasks.task_id` |
| `task_name` | `scheduler.doctor_task_state.task_name` |
| `terminal_handoff_message_id` | `runtime.messages.message_id` |
| `thread_action_id` | `goals.thread_actions.thread_action_id` |
| `thread_id` | `goals.threads.thread_id` |
| `token_usage_id` | `agent.token_usage.token_usage_id` |
| `tool_call_id` | `runtime.tool_calls.tool_call_id` |
| `tool_output_id` | `runtime.tool_outputs.tool_output_id` |
| `unit_id` | `feed.fleet_reaction_outbox.unit_id<br>feed.promptless_unit_orders.unit_id<br>feed.units.unit_id` |
| `update_id` | `goals.updates.update_id` |
| `upload_session_id` | `device.upload_chunks.upload_session_id<br>device.upload_sessions.upload_session_id` |
| `user_message_id` | `runtime.messages.message_id` |
| `variant_hash` | `agent.volatile_context_pins.variant_hash` |
| `widget_id` | `runtime.widgets.widget_id` |
| `work_id` | `runtime.work_items.work_id` |
| `worker_agent_id` | `agent.agents.agent_id` |
| `worker_history_agent_id` | `agent.agents.agent_id` |
| `workout_id` | `health.workouts.workout_id` |
| `approval_id` | Sentinel 审批标识符；Stripe Link 消费记录行在 `runtime.stripe_link_spend_requests` 中与其对应。 |
| `browser_session_id` | 浏览器运行时标识符；Muse 数据库中无对应的 PostgreSQL 所有者表。 |
| `connection_id` | 客户端连接标识符；无持久化的所有者表。 |
| `model_id` | 模型或设备提供商标识符；Muse 数据库中无单一的 PostgreSQL 所有者表。 |
| `spotify_show_id` | Spotify 提供商标识符；Muse 数据库中无对应的 PostgreSQL 所有者表。 |
| `stripe_spend_request_id` | Stripe Link 提供商标识符；Muse 数据库中无对应的 PostgreSQL 所有者表。

## 表格

### `activity`

#### `activity.activity_monitor_agent_threads`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `agent_id` | `text` | 否 |  | 本地行标识（主键）。 |
| `activity_thread_id` | `text` | 否 |  | 潜在的本地引用；可通过领域上下文在以下表中解析：`activity.activity_monitor_thread_sections.activity_thread_id`、`activity.activity_monitor_threads.activity_thread_id`。 |
| `created_at` | `text` | 否 |  |  |
| `updated_at` | `text` | 否 |  |  |

键与关系：

- 主键 `activity_monitor_agent_threads_pkey`：`agent_id`

#### `activity.activity_monitor_carrier_user_messages`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `link_seq` | `bigint` | 否 | `nextval('activity.activity_monitor_carrier_user_messages_link_seq_seq'::regclass)` | 本地行标识（主键）。 |
| `carrier_message_id` | `text` | 否 |  | 软本地引用 → `runtime.messages.message_id`。 |
| `user_message_id` | `text` | 否 |  | 软本地引用 → `runtime.messages.message_id`。 |
| `root_agent_id` | `text` | 是 |  | 软本地引用 → `agent.agents.agent_id`。 |
| `created_at` | `timestamp with time zone` | 否 | `now()` |  |

键与关系：

- 主键 `activity_monitor_carrier_user_messages_pkey`：`link_seq`
- 唯一约束 `activity_monitor_carrier_user_carrier_message_id_user_messa_key`：`carrier_message_id`、`user_message_id`

#### `activity.activity_monitor_message_threads`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `message_id` | `text` | 否 |  | 本地行标识（主键）。 |
| `activity_thread_id` | `text` | 否 |  | 潜在的本地引用；可通过领域上下文在以下表中解析：`activity.activity_monitor_thread_sections.activity_thread_id`、`activity.activity_monitor_threads.activity_thread_id`。 |
| `created_at` | `text` | 否 |  |  |
| `updated_at` | `text` | 否 |  |  |

键与关系：

- 主键 `activity_monitor_message_threads_pkey`：`message_id`

#### `activity.activity_monitor_runtime_work_threads`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `work_id` | `text` | 否 |  | 本地行标识（主键）。 |
| `activity_thread_id` | `text` | 否 |  | 外键 → `activity.activity_monitor_threads.activity_thread_id`。 |
| `created_at` | `text` | 否 |  |  |
| `updated_at` | `text` | 否 |  |  |

键与关系：

- 外键 `activity_monitor_runtime_work_threads_goal_id_fkey`：`activity_thread_id` → `activity.activity_monitor_threads`（`activity_thread_id`）。
- 主键 `activity_monitor_runtime_work_threads_pkey`：`work_id`。

#### `activity.activity_monitor_thread_actions`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `action_seq` | `bigint` | 否 | `nextval('activity.activity_monitor_thread_actions_action_seq_seq'::regclass)` |  |
| `action_id` | `text` | 否 |  | 本地行标识（主键）。 |
| `activity_thread_id` | `text` | 否 |  | 潜在的本地引用；可通过领域上下文在以下表中解析：`activity.activity_monitor_thread_sections.activity_thread_id`、`activity.activity_monitor_threads.activity_thread_id`。 |
| `action_index` | `bigint` | 否 |  |  |
| `agent_id` | `text` | 是 |  | 软本地引用 → `agent.agents.agent_id`。 |
| `agent_depth` | `integer` | 是 |  |  |
| `icon` | `text` | 否 |  |  |
| `title` | `text` | 否 |  |  |
| `subtitle` | `text` | 是 |  |  |
| `report` | `text` | 否 |  |  |
| `status` | `text` | 否 |  |  |
| `created_at` | `text` | 否 |  |  |
| `section_key` | `text` | 是 |  |  |
| `section_title` | `text` | 是 |  |  |
| `section_order` | `integer` | 是 |  |  |
| `section_depth` | `integer` | 是 |  |  |
| `parent_section_key` | `text` | 是 |  |  |
| `ordinal_label` | `text` | 是 |  |  |

键与关系：

- 主键 `activity_monitor_thread_actions_pkey`：`action_id`
- 唯一键 `activity_monitor_thread_actions_action_seq_key`：`action_seq`

#### `activity.activity_monitor_thread_sections`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `activity_thread_id` | `text` | 否 |  | 外键 → `activity.activity_monitor_threads.activity_thread_id` |
| `section_key` | `text` | 否 |  | 本行唯一标识（主键）。 |
| `title` | `text` | 否 |  |  |
| `status` | `text` | 否 |  |  |
| `section_order` | `integer` | 是 |  |  |
| `section_depth` | `integer` | 是 |  |  |
| `parent_section_key` | `text` | 是 |  |  |
| `ordinal_label` | `text` | 是 |  |  |
| `created_at` | `text` | 否 |  |  |
| `updated_at` | `text` | 否 |  |  |

键与关系：

- 外键 `activity_monitor_thread_sections_goal_id_fkey`：`activity_thread_id` → `activity.activity_monitor_threads`（`activity_thread_id`）
- 主键 `activity_monitor_thread_sections_pkey`：`activity_thread_id`, `section_key`

#### `activity.activity_monitor_threads`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `activity_thread_id` | `text` | 否 |  | 本行唯一标识（主键）。 |
| `name` | `text` | 否 |  |  |
| `subtitle` | `text` | 否 |  |  |
| `status_title` | `text` | 是 |  |  |
| `icon` | `text` | 否 |  |  |
| `emoji` | `text` | 是 |  |  |
| `activity_thread_kind` | `text` | 是 |  |  |
| `space_slug` | `text` | 是 |  |  |
| `expected_finish_description` | `text` | 是 |  |  |
| `status` | `text` | 否 |  |  |
| `finish_status` | `text` | 是 |  |  |
| `finish_status_reason` | `text` | 是 |  |  |
| `created_at` | `text` | 否 |  |  |
| `finished_at` | `text` | 是 |  |  |
| `finish_message` | `text` | 是 |  |  |
| `artifact_slug` | `text` | 是 |  |  |

键与关系：

- 主键 `activity_monitor_threads_pkey`：`activity_thread_id`

#### `activity.feed_entries`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `activity_key` | `text` | 否 |  | 本行唯一标识（主键）。 |
| `activity_type` | `text` | 否 |  |  |
| `is_goal` | `boolean` | 否 | `false` |  |
| `message_id` | `text` | 是 |  | 软本地引用 → `runtime.messages.message_id`。 |
| `title` | `text` | 是 |  |  |
| `status_title` | `text` | 是 |  |  |
| `subtitle` | `text` | 是 |  |  |
| `details_json` | `text` | 是 |  |  |
| `status` | `text` | 否 |  |  |
| `task_label` | `text` | 是 |  |  |
| `created_at_ms` | `bigint` | 否 |  |  |
| `updated_at_ms` | `bigint` | 否 |  |  |
| `finished_at_ms` | `bigint` | 是 |  |  |
| `created_at` | `timestamp with time zone` | 是 |  |  |
| `updated_at` | `timestamp with time zone` | 是 |  |  |
| `finished_at` | `timestamp with time zone` | 是 |  |  |

键与关系：

- 主键 `feed_entries_pkey`：`activity_key`

### `agent`

#### `agent.agent_ancestors`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `descendant_id` | `text` | 否 |  | 本行唯一标识（主键）。 |
| `ancestor_id` | `text` | 否 |  | 本行唯一标识（主键）。 |
| `distance` | `integer` | 否 |  |  |

键与关系：

- 主键 `agent_ancestors_pkey`：`descendant_id`, `ancestor_id`

#### `agent.agent_compactions`

已脱敏。压缩详情载荷未予公开，因其包含检查点间保留的模型推理信息。对该表的查询均通过安全屏障视图 `inspection.agent_compactions` 进行，该视图仅展示以下列出的列。被隐藏的列：`details_json`。| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `id` | `bigint` | 否 | `nextval('agent.agent_compactions_id_seq'::regclass)` | 本地行标识（主键）。 |
| `agent_id` | `text` | 否 |  | 软本地引用 → `agent.agents.agent_id`。 |
| `summary` | `text` | 否 |  |  |
| `first_kept_seq` | `bigint` | 是 |  |  |
| `checkpoint_seq` | `bigint` | 是 |  |  |
| `tokens_before` | `bigint` | 是 |  |  |
| `tokens_after` | `bigint` | 是 |  |  |
| `trigger` | `text` | 否 |  |  |
| `will_retry` | `boolean` | 否 | `false` |  |
| `created_at` | `bigint` | 否 |  |  |
| `has_replacement_history` | `boolean` | 否 | `false` |  |

键与关系：

- 主键 `agent_compactions_pkey`: `id`

#### `agent.agent_message_token_usage`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `agent_id` | `text` | 否 |  | 本地行标识（主键）。 |
| `message_id` | `text` | 否 |  | 本地行标识（主键）。 |
| `input_tokens` | `bigint` | 否 |  |  |
| `output_tokens` | `bigint` | 否 |  |  |
| `created_at` | `bigint` | 否 |  |  |

键与关系：

- 主键 `agent_message_token_usage_pkey`: `agent_id`, `message_id`

#### `agent.agents`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `agent_id` | `text` | 否 |  | 本地行标识（主键）。 |
| `id` | `text` | 是 |  | 本行拥有的旧版本地代理标识；`agent.agents.parent_id` 指向此处。 |
| `session_id` | `text` | 是 |  | 软本地引用 → `agent.sessions.session_id`。 |
| `kind` | `text` | 否 | `'root'::text` |  |
| `parent_id` | `text` | 是 |  | 软本地引用 → `agent.agents.id`。 |
| `parent_agent_id` | `text` | 是 |  | 软本地引用 → `agent.agents.agent_id`。 |
| `root_session_id` | `text` | 是 |  | 软本地引用 → `agent.sessions.session_id`。 |
| `request_id` | `text` | 是 |  | 外键 → `runtime.requests.request_id`。 |
| `model` | `text` | 否 | `''::text` |  |
| `status` | `text` | 否 |  |  |
| `depth` | `integer` | 否 | `0` |  |
| `created_at` | `bigint` | 否 | `(EXTRACT(epoch FROM now()))::bigint` |  |
| `updated_at` | `bigint` | 否 | `(EXTRACT(epoch FROM now()))::bigint` |  |
| `last_assistant_message` | `text` | 是 |  |  |
| `seen_at_ms` | `bigint` | 是 |  |  |
| `prompt_floor_seq` | `bigint` | 是 |  |  |
| `ephemeral` | `boolean` | 否 | `false` |  |
| `last_assistant_resources` | `text` | 是 |  |  |
| `agent_type` | `text` | 是 |  |  |
| `presentation_root_session_id` | `text` | 是 |  | 软本地引用 → `agent.sessions.session_id`。 |
| `spawn_metadata_json` | `text` | 是 |  |  |
| `originating_location_context_json` | `text` | 是 |  |  |

键与关系：

- 外键 `agents_request_id_fkey`: `request_id` → `runtime.requests` (`request_id`)。
- 主键 `agents_pkey`: `agent_id`。
- 唯一键 `agents_id_key`: `id`。

#### `agent.chat_preferences`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `id` | `integer` | 否 |  | 本地行标识（主键）。 |
| `active_root_id` | `text` | 否 |  | 软本地引用 → `agent.agents.id`。 |
| `verbose` | `bigint` | 否 | `0` |  |
| `usage_mode` | `text` | 否 | `'off'::text` |  |
| `active_root_cleared_at_ms` | `bigint` | 是 |  |  |

键与关系：

- 主键 `chat_preferences_pkey`: `id`

#### `agent.compactions`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `compaction_id` | `bigint` | 否 | `nextval('agent.compactions_compaction_id_seq'::regclass)` | 本地行标识（主键）。 |
| `agent_id` | `text` | 否 |  | 软本地引用 → `agent.agents.agent_id`。 |
| `checkpoint_context_item_id` | `bigint` | 是 |  | 外键 → `agent.context_items.context_item_id` |
| `replacement_history_first_seq` | `integer` | 是 |  |  |
| `replacement_history_last_seq` | `integer` | 是 |  |  |
| `created_at` | `timestamp with time zone` | 否 | `now()` |  |
| `summary_text` | `text` | 否 |  |  |

键与关系：

- 外键 `compactions_checkpoint_context_item_id_fkey`: `checkpoint_context_item_id` → `agent.context_items` (`context_item_id`)
- 主键 `compactions_pkey`: `compaction_id`

#### `agent.context_item_derived_write_backlog`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `context_item_id` | `bigint` | 否 |  | 外键 → `agent.context_items.context_item_id` |
| `attempts` | `integer` | 否 | `0` |  |
| `last_error` | `text` | 是 |  |  |
| `created_at` | `timestamp with time zone` | 否 | `now()` |  |
| `updated_at` | `timestamp with time zone` | 否 | `now()` |  |
| `deadlettered_at` | `timestamp with time zone` | 是 |  |  |

键与关系：

- 外键 `context_item_derived_write_backlog_context_item_id_fkey`: `context_item_id` → `agent.context_items` (`context_item_id`)
- 主键 `context_item_derived_write_backlog_pkey`: `context_item_id`

#### `agent.context_item_fields`

已脱敏。其所属项目为模型推理（思考或已脱敏的思考）的旧版字段行已被屏蔽。对该表的查询均通过安全屏障视图 `inspection.context_item_fields` 进行，该视图仅展示以下列出的列。

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `context_item_field_id` | `bigint` | 否 | `nextval('agent.context_item_fields_context_item_field_id_seq'::regclass)` | 本地行标识（主键）。 |
| `context_item_id` | `bigint` | 否 |  | 外键 → `agent.context_items.context_item_id` |
| `field_path` | `text` | 否 |  |  |
| `scalar_type` | `text` | 否 |  |  |
| `scalar_value` | `text` | 是 |  |  |
| `field_text_content` | `text` | 是 |  |  |

键与关系：

- 外键 `context_item_fields_context_item_id_fkey`: `context_item_id` → `agent.context_items` (`context_item_id`)
- 主键 `context_item_fields_pkey`: `context_item_field_id`
- 唯一键 `context_item_fields_context_item_id_field_path_key`: `context_item_id`, `field_path`

#### `agent.context_item_resume_projection`

已脱敏。来自模型推理项（思考或已脱敏的思考）的投影行已被屏蔽，且旧版项目副本也被屏蔽，因为其中可能包含模型推理内容。对该表的查询均通过安全屏障视图 `inspection.context_item_resume_projection` 进行，该视图仅展示以下列出的列。被屏蔽的列：`item_json`。

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `agent_id` | `text` | 否 |  | 本地行标识（主键）。 |
| `seq` | `bigint` | 否 |  | 本地行标识（主键）。 |
| `context_item_id` | `bigint` | 是 |  | 外键 → `agent.context_items.context_item_id` |
| `created_at` | `bigint` | 是 |  |  |
| `message_source` | `text` | 是 |  |  |
| `client_context_json` | `text` | 是 |  |  |
| `reply_prefix` | `text` | 是 |  |  |
| `provenance_json` | `text` | 是 |  |  |
| `client_context_projected` | `boolean` | 否 | `false` |  |
| `reply_prefix_projected` | `boolean` | 否 | `false` |  |
| `provenance_projected` | `boolean` | 否 | `false` |  |
| `updated_at` | `timestamp with time zone` | 否 | `now()` |  |

键与关系：- 外键 `context_item_resume_projection_context_item_id_fkey`：`context_item_id` → `agent.context_items`（`context_item_id`）
- 主键 `context_item_resume_projection_pkey`：`agent_id`，`seq`

#### `agent.context_items`

已脱敏。包含模型推理内容（思考及脱敏后的思考项）的行已被屏蔽；评论文本仍可读。对该表的查询均通过安全屏障视图 `inspection.context_items` 进行，该视图仅展示以下列出的列。

| 列名 | 类型 | 是否可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `context_item_id` | `bigint` | 否 | `nextval('agent.context_items_context_item_id_seq'::regclass)` | 本地行标识符（主键）。 |
| `agent_id` | `text` | 否 |  | 软本地引用 → `agent.agents.agent_id`。 |
| `seq` | `bigint` | 否 |  |  |
| `item_kind` | `agent.context_item_kind` | 否 |  |  |
| `message_id` | `text` | 是 |  | 外键 → `runtime.messages.message_id` |
| `tool_call_id` | `bigint` | 是 |  | 外键 → `runtime.tool_calls.tool_call_id` |
| `tool_output_id` | `bigint` | 是 |  | 外键 → `runtime.tool_outputs.tool_output_id` |
| `role` | `runtime.message_role` | 是 |  |  |
| `call_id` | `text` | 是 |  | 对 `runtime.workflow_agent_calls.call_id` 的软本地引用。 |
| `tool_name` | `text` | 是 |  |  |
| `success` | `boolean` | 是 |  |  |
| `message_source` | `text` | 是 |  |  |
| `created_at` | `timestamp with time zone` | 否 | `now()` |  |
| `data_fbids` | `bigint[]` | 是 |  |  |
| `data_message_ids` | `text[]` | 是 |  | 为保留清理而携带的源系统消息标识符；应根据上下文项的来源解析，而非假定为运行时消息 ID。 |
| `data_cleanup_checked_at` | `timestamp with time zone` | 是 |  |  |
| `data_summarized_at` | `timestamp with time zone` | 是 |  |  |
| `data_thread_ids` | `text[]` | 是 |  | 为保留清理而携带的源系统线程标识符；无单一的 Muse PostgreSQL 所有者表。 |
| `data_expires_at` | `timestamp with time zone` | 是 |  |  |
| `text_content` | `text` | 是 |  |  |
| `item_json` | `text` | 是 |  |  |

键与关系：

- 外键 `context_items_message_id_fkey`：`message_id` → `runtime.messages`（`message_id`）
- 外键 `context_items_tool_call_id_fkey`：`tool_call_id` → `runtime.tool_calls`（`tool_call_id`）
- 外键 `context_items_tool_output_id_fkey`：`tool_output_id` → `runtime.tool_outputs`（`tool_output_id`）
- 主键 `context_items_pkey`：`context_item_id`
- 唯一键 `context_items_agent_id_seq_key`：`agent_id`，`seq`

#### `agent.context_text_segments`

| 列名 | 类型 | 是否可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `context_text_segment_id` | `bigint` | 否 | `nextval('agent.context_text_segments_context_text_segment_id_seq'::regclass)` | 本地行标识符（主键）。 |
| `context_item_id` | `bigint` | 否 |  | 外键 → `agent.context_items.context_item_id` |
| `ordinal` | `integer` | 否 |  |  |
| `text_content` | `text` | 否 |  |  |

键与关系：

- 外键 `context_text_segments_context_item_id_fkey`：`context_item_id` → `agent.context_items`（`context_item_id`）
- 主键 `context_text_segments_pkey`：`context_text_segment_id`
- 唯一键 `context_text_segments_context_item_id_ordinal_key`：`context_item_id`，`ordinal`

#### `agent.message_mailbox`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `submission_id` | `text` | 否 |  | 本地行标识符（主键）。 |
| `accepted_ordinal` | `bigint` | 否 |  |  |
| `agent_id` | `text` | 否 |  | 外键 → `agent.agents.agent_id` |
| `conversation_epoch` | `bigint` | 否 |  |  |
| `submission_kind` | `text` | 否 |  |  |
| `routing_scope` | `text` | 否 |  |  |
| `state` | `text` | 否 |  |  |
| `stream_owner_message_id` | `text` | 是 |  | 软本地引用 → `runtime.messages.message_id`。 |
| `submission_json` | `text` | 否 |  |  |
| `transcript_event_seq` | `bigint` | 是 |  | 外键 → `runtime.events.event_seq` |
| `accepted_at` | `timestamp with time zone` | 否 | `now()` |  |
| `attached_at` | `timestamp with time zone` | 是 |  |  |
| `terminalized_at` | `timestamp with time zone` | 是 |  |  |
| `checkpoint_custody` | `text` | 是 |  |  |

键与关系：

- 外键 `message_mailbox_agent_id_fkey`: `agent_id` → `agent.agents` (`agent_id`)
- 外键 `message_mailbox_transcript_event_seq_fkey`: `transcript_event_seq` → `runtime.events` (`event_seq`)
- 主键 `message_mailbox_pkey`: `submission_id`
- 唯一键 `message_mailbox_agent_id_conversation_epoch_accepted_ordina_key`: `agent_id`, `conversation_epoch`, `accepted_ordinal`

#### `agent.recovery_owner_terminal_events`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `recovery_class` | `text` | 否 |  | 外键 → `agent.recovery_owners.recovery_class` |
| `owner_id` | `text` | 否 |  | 外键 → `agent.recovery_owners.owner_id` |
| `generation` | `bigint` | 否 |  | 本地行标识符（主键）。 |
| `event_type` | `text` | 否 |  | 本地行标识符（主键）。 |
| `claimed_at` | `timestamp with time zone` | 否 | `now()` |  |

键与关系：

- 外键 `recovery_owner_terminal_events_recovery_class_owner_id_fkey`: `recovery_class`, `owner_id` → `agent.recovery_owners` (`recovery_class`, `owner_id`)
- 主键 `recovery_owner_terminal_events_pkey`: `recovery_class`, `owner_id`, `generation`, `event_type`

#### `agent.recovery_owners`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `recovery_class` | `text` | 否 |  | 本地行标识符（主键）。 |
| `owner_id` | `text` | 否 |  | 本地行标识符（主键）。 |
| `generation` | `bigint` | 否 | `1` |  |
| `agent_id` | `text` | 否 |  | 软本地引用 → `agent.agents.agent_id`。 |
| `message_id` | `text` | 否 |  | 软本地引用 → `runtime.messages.message_id`。 |
| `active_at` | `timestamp with time zone` | 否 | `now()` |  |
| `terminal_at` | `timestamp with time zone` | 是 |  |  |
| `terminal_status` | `text` | 是 |  |  |

键与关系：

- 主键 `recovery_owners_pkey`: `recovery_class`, `owner_id`
- 唯一键 `recovery_owners_agent_id_key`: `agent_id`

#### `agent.runtime_restart_checkpoints`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `agent_id` | `text` | 否 |  | 本地行标识符（主键）。 |
| `mode` | `text` | 否 |  |  |
| `payload_json` | `text` | 是 |  |  |
| `created_at` | `bigint` | 否 |  |  |
| `updated_at` | `bigint` | 否 |  |  |
| `recovery_class` | `text` | 是 |  |  |
| `recovery_owner_id` | `text` | 是 |  | 软本地引用 → `agent.recovery_owners.owner_id`。 |
| `execution_config_json` | `text` | 是 |  |  |

键与关系：

- 主键 `runtime_restart_checkpoints_pkey`: `agent_id`

#### `agent.runtime_state`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `agent_id` | `text` | 否 |  | 本地行标识符（主键）。 |
| `current_context_seq` | `integer` | 否 | `0` |  |
| `last_event_seq` | `bigint` | 是 |  | 外键 → `runtime.events.event_seq` |
| `updated_at` | `timestamp with time zone` | 否 | `now()` |  |

键与关系：- 外键 `runtime_state_last_event_seq_fkey`: `last_event_seq` → `runtime.events`（`event_seq`）
- 主键 `runtime_state_pkey`: `agent_id`

#### `agent.session_memory_capture_deadlines`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `root_session_id` | `text` | 否 |  | 外键 → `agent.agents.agent_id` |
| `session_id` | `text` | 是 |  | 软本地引用 → `agent.sessions.session_id`。 |
| `reason` | `text` | 否 |  |  |
| `due_at_ms` | `bigint` | 否 |  |  |
| `created_at_ms` | `bigint` | 否 |  |  |
| `updated_at_ms` | `bigint` | 否 |  |  |

键与关系：

- 外键 `session_memory_capture_deadlines_root_session_id_fkey`: `root_session_id` → `agent.agents`（`agent_id`）
- 主键 `session_memory_capture_deadlines_pkey`: `root_session_id`

#### `agent.sessions`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `session_id` | `text` | 否 |  | 本地行标识符（主键）。 |
| `root_request_id` | `text` | 是 |  | 外键 → `runtime.requests.request_id` |
| `created_at` | `timestamp with time zone` | 否 | `now()` |  |
| `updated_at` | `timestamp with time zone` | 否 | `now()` |  |
| `archived_at` | `timestamp with time zone` | 是 |  |  |

键与关系：

- 外键 `sessions_root_request_id_fkey`: `root_request_id` → `runtime.requests`（`request_id`）
- 主键 `sessions_pkey`: `session_id`

#### `agent.subagent_monitor_decisions`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `decision_id` | `bigint` | 否 | `nextval('agent.subagent_monitor_decisions_decision_id_seq'::regclass)` | 本地行标识符（主键）。 |
| `coordinator_agent_id` | `text` | 否 |  | 软本地引用 → `agent.agents.agent_id`。 |
| `child_agent_id` | `text` | 否 |  | 软本地引用 → `agent.agents.agent_id`。 |
| `parent_agent_id` | `text` | 否 |  | 软本地引用 → `agent.agents.agent_id`。 |
| `parent_message_id` | `text` | 否 |  | 软本地引用 → `runtime.messages.message_id`。 |
| `monitor_event_id` | `bigint` | 否 |  | 本地监控事件引用；可通过 `monitor_event_kind` 解析出 `agent.subagent_progress_message_events.event_id` 或 `agent.subagent_progress_tool_events.event_id`。 |
| `monitor_event_kind` | `text` | 否 |  |  |
| `inference_request_id` | `text` | 否 |  | 推理/遥测关联标识符；无 Muse PostgreSQL 所有者表。 |
| `assistant_text` | `text` | 是 |  |  |
| `tool_call_id` | `text` | 是 |  | 软本地引用 → `runtime.tool_calls.tool_call_id`。 |
| `tool_name` | `text` | 是 |  |  |
| `tool_arguments` | `text` | 是 |  |  |
| `tool_result_json` | `text` | 否 |  |  |
| `decision_kind` | `text` | 否 |  |  |
| `created_at` | `bigint` | 否 |  |  |

键与关系：

- 主键 `subagent_monitor_decisions_pkey`: `decision_id`
- 唯一约束 `subagent_monitor_decisions_inference_request_id_key`: `inference_request_id`

#### `agent.subagent_progress`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `progress_id` | `bigint` | 否 | `nextval('agent.subagent_progress_progress_id_seq'::regclass)` | 本地行标识符（主键）。 |
| `child_agent_id` | `text` | 否 |  | 软本地引用 → `agent.agents.agent_id`。 |
| `event_seq` | `bigint` | 是 |  | 外键 → `runtime.events.event_seq` |
| `status` | `text` | 否 |  |  |
| `created_at` | `timestamp with time zone` | 否 | `now()` |  |
| `progress_text` | `text` | 是 |  |  |

键与关系：

- 外键 `subagent_progress_event_seq_fkey`: `event_seq` → `runtime.events`（`event_seq`）
- 主键 `subagent_progress_pkey`: `progress_id`

#### `agent.subagent_progress_message_events`| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `event_id` | `bigint` | 否 | `nextval('agent.subagent_progress_message_events_event_id_seq'::regclass)` | 本地行标识符（主键）。 |
| `child_agent_id` | `text` | 否 |  | 软本地外键 → `agent.agents.agent_id`。 |
| `parent_agent_id` | `text` | 否 |  | 软本地外键 → `agent.agents.agent_id`。 |
| `parent_message_id` | `text` | 否 |  | 软本地外键 → `runtime.messages.message_id`。 |
| `message_text` | `text` | 否 |  |  |
| `created_at` | `bigint` | 否 |  |  |
| `delivered_at` | `bigint` | 是 |  |  |

键与关系：

- 主键 `subagent_progress_message_events_pkey`: `event_id`

#### `agent.subagent_progress_tool_events`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `event_id` | `bigint` | 否 | `nextval('agent.subagent_progress_tool_events_event_id_seq'::regclass)` | 本地行标识符（主键）。 |
| `child_agent_id` | `text` | 否 |  | 软本地外键 → `agent.agents.agent_id`。 |
| `parent_agent_id` | `text` | 否 |  | 软本地外键 → `agent.agents.agent_id`。 |
| `parent_message_id` | `text` | 否 |  | 软本地外键 → `runtime.messages.message_id`。 |
| `tool_name` | `text` | 否 |  |  |
| `tool_description` | `text` | 否 |  |  |
| `tool_status` | `text` | 否 |  |  |
| `tool_result_preview` | `text` | 是 |  |  |
| `created_at` | `bigint` | 否 |  |  |
| `delivered_at` | `bigint` | 是 |  |  |
| `run_id` | `text` | 是 |  | 生产者特定的工具或工作流运行标识符；无单一所有者表。 |

键与关系：

- 主键 `subagent_progress_tool_events_pkey`: `event_id`

#### `agent.subagent_spawns`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `spawn_id` | `bigint` | 否 | `nextval('agent.subagent_spawns_spawn_id_seq'::regclass)` | 本地行标识符（主键）。 |
| `parent_agent_id` | `text` | 否 |  | 软本地外键 → `agent.agents.agent_id`。 |
| `child_agent_id` | `text` | 否 |  | 软本地外键 → `agent.agents.agent_id`。 |
| `parent_message_id` | `text` | 是 |  | 软本地外键 → `runtime.messages.message_id`。 |
| `parent_request_id` | `text` | 是 |  | 软本地外键 → `runtime.requests.request_id`。 |
| `root_request_id` | `text` | 是 |  | 软本地外键 → `runtime.requests.request_id`。 |
| `root_message_execution_id` | `text` | 是 |  | 软本地外键 → `runtime.messages.message_id`。 |
| `request_id` | `text` | 是 |  | 外键 → `runtime.requests.request_id`。 |
| `created_at` | `bigint` | 否 | `(EXTRACT(epoch FROM now()))::bigint` |  |
| `status` | `text` | 是 |  |  |
| `deferred_terminal_status` | `text` | 是 |  |  |
| `final_response` | `text` | 是 |  |  |
| `completed_at` | `bigint` | 是 |  |  |
| `prompt` | `text` | 是 |  |  |
| `requester_source` | `text` | 是 |  |  |
| `requester_chat_context_json` | `text` | 是 |  |  |
| `seen_at` | `bigint` | 是 |  |  |
| `child_depth` | `integer` | 否 | `0` |  |
| `agent_type` | `text` | 是 |  |  |
| `metadata_json` | `text` | 是 |  |  |
| `spawn_call_id` | `text` | 是 |  | 软本地外键 → `runtime.tool_calls.tool_call_id`。 |

键与关系：

- 外键 `subagent_spawns_request_id_fkey`: `request_id` → `runtime.requests`（`request_id`）
- 主键 `subagent_spawns_pkey`: `spawn_id`
- 唯一约束 `subagent_spawns_child_agent_id_key`: `child_agent_id`
- 唯一约束 `subagent_spawns_parent_agent_id_child_agent_id_key`: `parent_agent_id`, `child_agent_id`

#### `agent.token_usage`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `token_usage_id` | `bigint` | 否 | `nextval('agent.token_usage_token_usage_id_seq'::regclass)` | 本地行标识符（主键）。 |
| `agent_id` | `text` | 否 |  | 软本地外键 → `agent.agents.agent_id`。 |
| `request_id` | `text` | 是 |  | 外键 → `runtime.requests.request_id` |
| `input_tokens` | `integer` | 否 | `0` |  |
| `output_tokens` | `integer` | 否 | `0` |  |
| `cached_input_tokens` | `integer` | 否 | `0` |  |
| `reasoning_tokens` | `integer` | 否 | `0` |  |
| `created_at` | `timestamp with time zone` | 否 | `now()` |  |

键与关系：

- 外键 `token_usage_request_id_fkey`: `request_id` → `runtime.requests` (`request_id`)
- 主键 `token_usage_pkey`: `token_usage_id`

#### `agent.volatile_context_pins`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `agent_id` | `text` | 否 |  | 本地行标识符（主键）。 |
| `variant_hash` | `text` | 否 |  | 本地行标识符（主键）。 |
| `variant_json` | `text` | 否 |  |  |
| `prompt_floor_seq` | `bigint` | 否 |  |  |
| `checkpoint_seq` | `bigint` | 否 |  |  |
| `body` | `text` | 是 |  |  |
| `created_at` | `timestamp with time zone` | 否 | `now()` |  |

键与关系：

- 主键 `volatile_context_pins_pkey`: `agent_id`, `variant_hash`

### `chat`

#### `chat.chats`

标准的聊天元数据。`chat_id` 即现有的根会话 UUID。

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `chat_id` | `text` | 否 |  | 本地行标识符（主键）。 |
| `origin` | `text` | 否 |  |  |
| `lifecycle` | `text` | 否 |  |  |
| `status` | `text` | 否 |  |  |
| `thread_title` | `text` | 是 |  |  |
| `source_session_id` | `text` | 是 |  | 软本地外键 → `agent.sessions.session_id`。 |
| `source_prompt_seq_upper_bound` | `bigint` | 是 |  |  |
| `source_message_id_boundary` | `text` | 是 |  |  |
| `created_at` | `bigint` | 否 |  |  |
| `updated_at` | `bigint` | 否 |  |  |
| `pinned` | `boolean` | 否 | `false` |  |
| `pinned_order` | `bigint` | 是 |  |  |

键与关系：

- 主键 `chats_pkey`: `chat_id`

#### `chat.event_transports`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `provider` | `text` | 否 |  | 本地行标识符（主键）。 |
| `event_seq` | `bigint` | 否 |  | 外键 → `runtime.events.event_seq` |

键与关系：

- 外键 `event_transports_event_seq_fkey`: `event_seq` → `runtime.events` (`event_seq`)
- 主键 `event_transports_pkey`: `provider`, `event_seq`

#### `chat.message_bindings`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `binding_id` | `bigint` | 否 | `nextval('chat.message_bindings_binding_id_seq'::regclass)` | 本地行标识符（主键）。 |
| `transport_message_id` | `text` | 是 |  | 提供商原生通道的消息标识符；无 Muse PostgreSQL 所属表。 |
| `transport` | `text` | 是 |  |  |
| `jarvis_message_id` | `text` | 否 |  | 软本地外键 → `runtime.messages.message_id`。 |
| `provider` | `text` | 是 |  |  |
| `native_conversation_id` | `text` | 是 |  | 不透明的相关标识符；未声明本地表关系。 |
| `native_message_id` | `text` | 是 |  | 不透明的相关标识符；未声明本地表关系。 |
| `created_at_ms` | `bigint` | 是 |  |  |
| `created_at` | `timestamp with time zone` | 否 | `now()` |  |
| `chat_id` | `text` | 是 |  | 软本地外键 → `chat.chats.chat_id`。 |
| `binding_epoch` | `bigint` | 是 |  |  |
| `admitted_at` | `timestamp with time zone` | 是 |  |  |

键与关系：

- 主键 `message_bindings_pkey`: `binding_id`
- 唯一约束 `message_bindings_chat_identity`: `chat_id`, `binding_epoch`, `transport`, `transport_message_id`

### `device`

#### `device.calendar_events`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `node_id` | `text` | 否 |  | 外键 → `device.nodes.node_id` |
| `external_event_id` | `text` | 否 |  | 本地行标识（主键）。 |
| `title` | `text` | 否 |  |  |
| `start_at` | `timestamp with time zone` | 否 |  |  |
| `end_at` | `timestamp with time zone` | 否 |  |  |
| `is_all_day` | `boolean` | 否 |  |  |
| `external_calendar_id` | `text` | 否 |  | 外部/提供商标识；无 Muse PostgreSQL 所有者表。 |
| `calendar_name` | `text` | 否 |  |  |
| `availability` | `text` | 否 |  |  |
| `location` | `text` | 是 |  |  |
| `notes` | `text` | 是 |  |  |
| `time_zone` | `text` | 是 |  |  |
| `start_local_date` | `date` | 是 |  |  |
| `end_local_date` | `date` | 是 |  |  |
| `is_recurring` | `boolean` | 否 |  |  |
| `url` | `text` | 是 |  |  |
| `event_status` | `text` | 否 |  |  |
| `calendar_color` | `text` | 是 |  |  |

键与关系：

- 外键 `calendar_events_node_id_fkey`: `node_id` → `device.nodes` (`node_id`)
- 主键 `calendar_events_pkey`: `node_id`, `external_event_id`

#### `device.call_log`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `call_log_id` | `bigint` | 否 | `nextval('device.call_log_call_log_id_seq'::regclass)` | 本地行标识（主键）。 |
| `producer_id` | `text` | 否 |  | 设备数据中提供的来源-生产者标识；无独立所有者表。 |
| `external_id` | `text` | 否 |  | 外部/提供商标识；无 Muse PostgreSQL 所有者表。 |
| `phone_number` | `text` | 是 |  |  |
| `contact_name` | `text` | 是 |  |  |
| `call_type` | `text` | 是 |  |  |
| `occurred_at_unix_ms` | `bigint` | 是 |  |  |
| `duration_seconds` | `bigint` | 是 |  |  |
| `platform` | `text` | 是 |  |  |
| `created_at` | `timestamp with time zone` | 否 | `now()` |  |

键与关系：

- 主键 `call_log_pkey`: `call_log_id`
- 唯一约束 `call_log_producer_id_external_id_key`: `producer_id`, `external_id`

#### `device.client_contexts`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `connection_id` | `text` | 否 |  | 本地行标识（主键）。 |
| `device_id` | `text` | 否 |  | 客户提供的设备标识；与 `device.nodes.node_id` 不同。 |
| `session_id` | `text` | 是 |  | 软本地引用 → `agent.sessions.session_id`。 |
| `root_session_id` | `text` | 是 |  | 软本地引用 → `agent.sessions.session_id`。 |
| `platform` | `text` | 是 |  |  |
| `view` | `text` | 是 |  |  |
| `mode` | `text` | 是 |  |  |
| `is_visible` | `boolean` | 是 | `true` |  |
| `presence_status` | `text` | 否 |  |  |
| `metadata_json` | `jsonb` | 是 |  |  |
| `connected_at_ms` | `bigint` | 否 |  |  |
| `last_active_at_ms` | `bigint` | 否 |  |  |
| `updated_at_ms` | `bigint` | 否 |  |  |
| `created_at` | `timestamp with time zone` | 否 | `now()` |  |

键与关系：

- 主键 `client_contexts_pkey`: `connection_id`

#### `device.contact_addresses`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `contact_address_id` | `bigint` | 否 | `nextval('device.contact_addresses_contact_address_id_seq'::regclass)` | 本地行标识（主键）。 |
| `contact_id` | `bigint` | 否 |  | 外键 → `device.contacts.contact_id` |
| `label` | `text` | 是 |  |  |
| `street` | `text` | 是 |  |  |
| `city` | `text` | 是 |  |  |
| `state` | `text` | 是 |  |  |
| `postal_code` | `text` | 是 |  |  |
| `country` | `text` | 是 |  |  |

键与关系：

- 外键 `contact_addresses_contact_id_fkey`: `contact_id` → `device.contacts` (`contact_id`)
- 主键 `contact_addresses_pkey`: `contact_address_id`

#### `device.contact_emails`| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `contact_email_id` | `bigint` | 否 | `nextval('device.contact_emails_contact_email_id_seq'::regclass)` | 本地行标识符（主键）。 |
| `contact_id` | `bigint` | 否 |  | 外键 → `device.contacts.contact_id` |
| `label` | `text` | 是 |  |  |
| `email` | `text` | 否 |  |  |

键与关系：

- 外键 `contact_emails_contact_id_fkey`: `contact_id` → `device.contacts` (`contact_id`)
- 主键 `contact_emails_pkey`: `contact_email_id`

#### `device.contact_phones`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `contact_phone_id` | `bigint` | 否 | `nextval('device.contact_phones_contact_phone_id_seq'::regclass)` | 本地行标识符（主键）。 |
| `contact_id` | `bigint` | 否 |  | 外键 → `device.contacts.contact_id` |
| `label` | `text` | 是 |  |  |
| `phone_e164` | `text` | 是 |  |  |
| `phone_raw` | `text` | 否 |  |  |

键与关系：

- 外键 `contact_phones_contact_id_fkey`: `contact_id` → `device.contacts` (`contact_id`)
- 主键 `contact_phones_pkey`: `contact_phone_id`

#### `device.contacts`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `contact_id` | `bigint` | 否 | `nextval('device.contacts_contact_id_seq'::regclass)` | 本地行标识符（主键）。 |
| `node_id` | `text` | 否 |  | 外键 → `device.nodes.node_id` |
| `platform` | `text` | 否 |  |  |
| `external_contact_id` | `text` | 否 |  | 外部/提供商标识符；非 Muse PostgreSQL 所有者表。 |
| `display_name` | `text` | 是 |  |  |
| `given_name` | `text` | 是 |  |  |
| `family_name` | `text` | 是 |  |  |
| `organization` | `text` | 是 |  |  |
| `job_title` | `text` | 是 |  |  |
| `birthday_text` | `text` | 是 |  |  |
| `note` | `text` | 是 |  |  |
| `synced_at_text` | `text` | 否 |  |  |
| `deleted_at` | `timestamp with time zone` | 是 |  |  |
| `updated_at` | `timestamp with time zone` | 否 | `now()` |  |
| `thumbnail` | `text` | 是 |  |  |

键与关系：

- 外键 `contacts_node_id_fkey`: `node_id` → `device.nodes` (`node_id`)
- 主键 `contacts_pkey`: `contact_id`
- 唯一键 `contacts_node_id_external_contact_id_key`: `node_id`, `external_contact_id`

#### `device.data_sync_state`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `node_id` | `text` | 否 |  | 外键 → `device.nodes.node_id` |
| `data_source` | `text` | 否 |  | 本地行标识符（主键）。 |
| `next_full_sync_at` | `timestamp with time zone` | 是 |  |  |
| `current_full_sync_id` | `text` | 是 |  | 软本地引用 → `device.upload_sessions.upload_session_id`。 |
| `current_full_sync_expires_at` | `timestamp with time zone` | 是 |  |  |
| `current_full_sync_range_start` | `timestamp with time zone` | 是 |  |  |
| `current_full_sync_range_end` | `timestamp with time zone` | 是 |  |  |
| `changed_during_full_sync` | `boolean` | 否 | `false` |  |
| `last_full_sync_at` | `timestamp with time zone` | 是 |  |  |
| `last_full_sync_range_start` | `timestamp with time zone` | 是 |  |  |
| `last_full_sync_range_end` | `timestamp with time zone` | 是 |  |  |
| `search_trigger_pending` | `boolean` | 否 | `false` |  |
| `current_full_sync_requester_root_session_id` | `text` | 是 |  | 软本地引用 → `agent.sessions.session_id`。 |
| `current_full_sync_requester_presentation_locale` | `text` | 是 |  |  |

键与关系：

- 外键 `data_sync_state_node_id_fkey`: `node_id` → `device.nodes` (`node_id`)
- 主键 `data_sync_state_pkey`: `node_id`, `data_source`

#### `device.media_upload_batches`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `lane` | `text` | 否 |  | 本地行标识（主键）。 |
| `batch_id` | `text` | 否 |  | 软本地外键 → `device.media_upload_batches.batch_id`。 |
| `high_water_global_seq` | `bigint` | 否 |  |  |
| `pending_count` | `bigint` | 否 |  |  |
| `oldest_received_at_unix_ms` | `bigint` | 否 |  |  |
| `claimed_at` | `timestamp with time zone` | 否 | `now()` |  |

键与关系：

- 主键 `media_upload_batches_pkey`: `lane`
- 唯一键 `media_upload_batches_batch_id_key`: `batch_id`

#### `device.media_upload_events`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `media_upload_event_id` | `bigint` | 否 | `nextval('device.media_upload_events_media_upload_event_id_seq'::regclass)` | 本地行标识（主键）。 |
| `global_seq` | `bigint` | 否 | `nextval('device.media_upload_events_global_seq_seq'::regclass)` |  |
| `event_id` | `text` | 否 |  | 设备来源的事件标识；无 Muse PostgreSQL 所有者表。 |
| `media_id` | `text` | 否 |  | 软本地外键，指向 `media.items.media_id`。 |
| `node_id` | `text` | 是 |  | 外键 → `device.nodes.node_id` |
| `lane` | `text` | 否 |  |  |
| `received_at_text` | `text` | 否 |  |  |
| `received_at_unix_ms` | `bigint` | 否 |  |  |
| `received_at` | `timestamp with time zone` | 否 | `now()` |  |
| `status` | `text` | 否 |  |  |
| `processed_at_text` | `text` | 是 |  |  |
| `processed_at_unix_ms` | `bigint` | 是 |  |  |
| `processed_at` | `timestamp with time zone` | 是 |  |  |
| `handoff_message_id` | `text` | 是 |  | 软本地外键 → `runtime.messages.message_id`。 |
| `summary_preview` | `text` | 是 |  |  |
| `failure_code` | `text` | 是 |  |  |
| `failure_message` | `text` | 是 |  |  |

键与关系：

- 外键 `media_upload_events_node_id_fkey`: `node_id` → `device.nodes`（`node_id`）
- 主键 `media_upload_events_pkey`: `media_upload_event_id`
- 唯一键 `media_upload_events_event_id_key`: `event_id`
- 唯一键 `media_upload_events_global_seq_key`: `global_seq`

#### `device.nodes`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `node_id` | `text` | 否 |  | 本地行标识（主键）。 |
| `node_kind` | `text` | 否 |  |  |
| `display_name` | `text` | 是 |  |  |
| `platform` | `text` | 否 | `'unknown'::text` |  |
| `enabled_permissions_json` | `text` | 否 | `'[]'::text` |  |
| `commands_json` | `text` | 否 | `'[]'::text` |  |
| `version` | `text` | 是 |  |  |
| `device_family` | `text` | 是 |  |  |
| `model_id` | `text` | 是 |  | 模型或设备提供商标识；无单一 Muse PostgreSQL 所有者表。 |
| `is_wakeup_supported` | `boolean` | 是 |  |  |
| `delivery_app` | `text` | 是 |  |  |
| `paired_at_text` | `text` | 是 |  |  |
| `created_at` | `timestamp with time zone` | 否 | `now()` |  |
| `last_seen_at_text` | `text` | 是 |  |  |
| `last_seen_at` | `timestamp with time zone` | 是 |  |  |
| `status` | `text` | 否 | `'registered'::text` |  |
| `revoked` | `boolean` | 否 | `false` |  |
| `contacts_last_synced_at` | `text` | 是 |  |  |
| `contacts_last_received_at` | `text` | 是 |  |  |
| `contacts_last_sync_status` | `text` | 否 | `'never'::text` |  |
| `contacts_last_sync_error` | `text` | 是 |  |  |
| `contacts_contact_count` | `bigint` | 否 | `0` |  |
| `health_last_received_at` | `text` | 是 |  |  |
| `health_last_sync_status` | `text` | 否 | `'never'::text` |  |
| `health_last_sync_error` | `text` | 是 |  |  |
| `data_sources_json` | `text` | 否 | `'{}'::text` |  |
| `invoke_protocol` | `text` | 否 | `'node'::text` |  |
| `pairing_status` | `text` | 否 | `'ok'::text` |  |
| `contacts_content_sha256` | `text` | 是 |  |  |
| `metadata_json` | `text` | 否 | `'{}'::text` |  |
| `location_sharing_mode` | `text` | 是 |  |  |

键与关系：

- 主键 `nodes_pkey`: `node_id`

#### `device.upload_chunks`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `upload_session_id` | `text` | 否 |  | 外键 → `device.upload_sessions.upload_session_id` |
| `chunk_index` | `integer` | 否 |  | 本地行标识（主键）。 |
| `payload_digest` | `text` | 否 |  |  |
| `created_at_text` | `text` | 否 |  |  |
| `created_at` | `timestamp with time zone` | 否 | `now()` |  |
| `payload` | `text` | 否 |  |  |

键与关系：

- 外键 `upload_chunks_upload_session_id_fkey`: `upload_session_id` → `device.upload_sessions` (`upload_session_id`)
- 主键 `upload_chunks_pkey`: `upload_session_id`, `chunk_index`

#### `device.upload_sessions`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `upload_session_id` | `text` | 否 |  | 本地行标识（主键）。 |
| `node_id` | `text` | 否 |  | 软本地引用 → `device.nodes.node_id`。 |
| `route_kind` | `text` | 否 |  |  |
| `request_id` | `text` | 是 |  | 软本地引用 → `runtime.requests.request_id`。 |
| `datatype` | `text` | 否 |  |  |
| `sync_mode` | `text` | 是 |  |  |
| `chunk_count` | `integer` | 否 |  |  |
| `status` | `text` | 否 |  |  |
| `created_at_text` | `text` | 否 |  |  |
| `updated_at_text` | `text` | 否 |  |  |
| `expires_at_text` | `text` | 否 |  |  |
| `expires_at` | `timestamp with time zone` | 否 |  |  |
| `created_at` | `timestamp with time zone` | 否 | `now()` |  |
| `response` | `text` | 是 |  |  |

键与关系：

- 主键 `upload_sessions_pkey`: `upload_session_id`

### `feed`

#### `feed.fleet_engagement_contributions`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `contribution_id` | `text` | 否 |  | 本地行标识（主键）。 |
| `origin_id` | `text` | 否 |  | 集群学习的来源标识；其所有者在本虚拟机数据库之外。 |
| `engagement_version` | `bigint` | 否 | `0` |  |
| `created_at_ms` | `bigint` | 否 |  |  |
| `updated_at_ms` | `bigint` | 否 |  |  |

键与关系：

- 主键 `fleet_engagement_contributions_pkey`: `contribution_id`

#### `feed.fleet_engagement_outbox`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `contribution_id` | `text` | 否 |  | 本地行标识（主键）。 |
| `origin_id` | `text` | 否 |  | 集群学习的来源标识；其所有者在本虚拟机数据库之外。 |
| `deleted_count` | `bigint` | 否 |  |  |
| `discuss_count` | `bigint` | 否 |  |  |
| `share_count` | `bigint` | 否 |  |  |
| `seed_use_count` | `bigint` | 否 |  |  |
| `mutation_version` | `bigint` | 否 |  |  |
| `attempt_count` | `integer` | 否 | `0` |  |
| `next_attempt_at_ms` | `bigint` | 否 |  |  |
| `created_at_ms` | `bigint` | 否 |  |  |
| `updated_at_ms` | `bigint` | 否 |  |  |

键与关系：

- 主键 `fleet_engagement_outbox_pkey`: `contribution_id`

#### `feed.fleet_fetch_receipts`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `payload_hash` | `text` | 否 |  | 本地行标识（主键）。 |
| `first_fetched_at_ms` | `bigint` | 否 |  |  |
| `origin_id` | `text` | 是 |  | 集群学习的来源标识；其所有者在本虚拟机数据库之外。

键与关系：

- 主键 `fleet_fetch_receipts_pkey`: `payload_hash`

#### `feed.fleet_publish_state`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `singleton` | `boolean` | 否 | `true` | 本地行标识（主键）。 |
| `published_watermark_ms` | `bigint` | 否 |  |  |

键与关系：

- 主键 `fleet_publish_state_pkey`: `singleton`

#### `feed.fleet_reaction_outbox`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `unit_id` | `text` | 否 |  | 本地行标识符（主键）。 |
| `origin_id` | `text` | 否 |  | 舰队学习的来源标识符；其归属方不在本虚拟机数据库内。 |
| `liked` | `boolean` | 否 |  |  |
| `mutation_version` | `bigint` | 否 |  |  |
| `attempt_count` | `integer` | 否 | `0` |  |
| `next_attempt_at_ms` | `bigint` | 否 |  |  |
| `created_at_ms` | `bigint` | 否 |  |  |
| `updated_at_ms` | `bigint` | 否 |  |  |

键与关系：

- 主键 `fleet_reaction_outbox_pkey`: `unit_id`

#### `feed.interactions`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `interaction_id` | `text` | 否 |  | 本地行标识符（主键）。 |
| `unit_id` | `text` | 否 |  | 可能的本地引用；需根据领域上下文在以下表中解析：`feed.fleet_reaction_outbox.unit_id`、`feed.promptless_unit_orders.unit_id`、`feed.units.unit_id`。 |
| `kind` | `text` | 否 |  |  |
| `value` | `text` | 是 |  |  |
| `created_at_ms` | `bigint` | 否 |  |  |

键与关系：

- 主键 `interactions_pkey`: `interaction_id`

#### `feed.null_state_seed`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `singleton` | `boolean` | 否 | `true` | 本地行标识符（主键）。 |
| `seeded_at_ms` | `bigint` | 否 |  |  |

键与关系：

- 主键 `null_state_seed_pkey`: `singleton`

#### `feed.preferences_projection`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `singleton_id` | `smallint` | 否 | `1` | 本地行标识符（主键）。 |
| `preferences_md` | `text` | 否 |  |  |
| `folded_through_ms` | `bigint` | 否 |  |  |
| `updated_by_run` | `text` | 否 |  |  |
| `updated_at_ms` | `bigint` | 否 |  |  |

键与关系：

- 主键 `preferences_projection_pkey`: `singleton_id`

#### `feed.prompt_scope_verdict`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `singleton` | `boolean` | 否 | `true` | 本地行标识符（主键）。 |
| `prompt_sha256` | `text` | 否 |  |  |
| `needs_interpretation` | `boolean` | 否 |  |  |
| `reason` | `text` | 否 |  |  |
| `classifier_version` | `integer` | 否 |  |  |
| `classified_at_ms` | `bigint` | 否 |  |  |

键与关系：

- 主键 `prompt_scope_verdict_pkey`: `singleton`

#### `feed.prompt_seed`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `singleton` | `boolean` | 否 | `true` | 本地行标识符（主键）。 |
| `seeded_at_ms` | `bigint` | 否 |  |  |

键与关系：

- 主键 `prompt_seed_pkey`: `singleton`

#### `feed.promptless_unit_orders`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `prompt_id` | `text` | 否 |  | 本地行标识符（主键）。 |
| `unit_id` | `text` | 否 |  | 外键 → `feed.units.unit_id` |
| `local_date` | `date` | 否 |  |  |
| `manual_order` | `double precision` | 否 |  |  |

键与关系：

- 外键 `promptless_unit_orders_unit_id_fkey`: `unit_id` → `feed.units`（`unit_id`）
- 主键 `promptless_unit_orders_pkey`: `prompt_id`, `unit_id`

#### `feed.prompts`| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `prompt_id` | `text` | 否 |  | 本地行标识符（主键）。 |
| `template_id` | `text` | 是 |  | 旧版内置Feed模板键；模板目录由代码管理，而非表。 |
| `slot_values` | `jsonb` | 否 | `'{}'::jsonb` |  |
| `free_text` | `text` | 否 | `''::text` |  |
| `enabled` | `boolean` | 否 | `true` |  |
| `created_at_ms` | `bigint` | 否 |  |  |
| `updated_at_ms` | `bigint` | 否 |  |  |
| `slot_ingredients` | `jsonb` | 否 | `'[]'::jsonb` |  |
| `generation_queued_at_ms` | `bigint` | 是 |  |  |
| `full_prompt` | `text` | 是 |  |  |
| `generation_queued_requester` | `jsonb` | 是 |  |  |

键与关系：

- 主键 `prompts_pkey`: `prompt_id`

#### `feed.run_steps`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `run_id` | `text` | 否 |  | 本地行标识符（主键）。 |
| `step_id` | `text` | 否 |  | 本地行标识符（主键）。 |
| `attempt` | `integer` | 否 | `0` | 本地行标识符（主键）。 |
| `input_hash` | `text` | 否 |  |  |
| `status` | `text` | 否 |  |  |
| `agent_id` | `text` | 是 |  | 软本地引用 → `agent.agents.agent_id`。 |
| `message_id` | `text` | 是 |  | 软本地引用 → `runtime.messages.message_id`。 |
| `output_json` | `jsonb` | 否 | `'{}'::jsonb` |  |
| `error` | `text` | 是 |  |  |
| `started_at_ms` | `bigint` | 否 |  |  |
| `finished_at_ms` | `bigint` | 是 |  |  |
| `updated_at_ms` | `bigint` | 否 |  |  |
| `submitted_at_ms` | `bigint` | 是 |  |  |

键与关系：

- 主键 `run_steps_pkey`: `run_id`, `step_id`, `attempt`

#### `feed.runs`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `run_id` | `text` | 否 |  | 本地行标识符（主键）。 |
| `trigger` | `text` | 否 |  |  |
| `edition_kind` | `text` | 否 |  |  |
| `local_date` | `date` | 否 |  |  |
| `tz` | `text` | 否 |  |  |
| `prompt_id` | `text` | 是 |  | 潜在的本地引用；可通过领域上下文解析：`feed.promptless_unit_orders.prompt_id`、`feed.prompts.prompt_id`。 |
| `prompt_snapshot` | `text` | 是 |  |  |
| `requester` | `jsonb` | 是 |  |  |
| `slot_fulfillment` | `jsonb` | 否 | `'{}'::jsonb` |  |
| `interactions_watermark_ms` | `bigint` | 是 |  |  |
| `status` | `text` | 否 | `'queued'::text` |  |
| `failure_reason` | `text` | 是 |  |  |
| `started_at_ms` | `bigint` | 是 |  |  |
| `finished_at_ms` | `bigint` | 是 |  |  |
| `created_at_ms` | `bigint` | 否 |  |  |
| `updated_at_ms` | `bigint` | 否 |  |  |
| `hour_slot` | `smallint` | 是 |  |  |
| `search_query_keys` | `text[]` | 是 |  |  |

键与关系：

- 主键 `runs_pkey`: `run_id`

#### `feed.surface_state`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `singleton` | `boolean` | 否 | `true` | 本地行标识符（主键）。 |
| `first_fetched_at_ms` | `bigint` | 否 |  |  |

键与关系：

- 主键 `surface_state_pkey`: `singleton`

#### `feed.units`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `unit_id` | `text` | 否 |  | 本地行标识（主键）。 |
| `run_id` | `text` | 否 |  | 潜在的本地引用；需根据领域上下文解析，如：`feed.run_steps.run_id`、`feed.runs.run_id`。 |
| `prompt_id` | `text` | 是 |  | 潜在的本地引用；需根据领域上下文解析，如：`feed.promptless_unit_orders.prompt_id`、`feed.prompts.prompt_id`。 |
| `edition_kind` | `text` | 否 |  |  |
| `edition_local_date` | `date` | 否 |  |  |
| `edition_generated_at_ms` | `bigint` | 否 |  |  |
| `position` | `integer` | 否 |  |  |
| `kicker` | `text` | 否 |  |  |
| `body_md` | `text` | 否 |  |  |
| `attachment_kind` | `text` | 否 |  |  |
| `header_image_path` | `text` | 是 |  |  |
| `image_urls` | `text[]` | 是 |  |  |
| `widget_html` | `text` | 是 |  |  |
| `social_embed_url` | `text` | 是 |  |  |
| `category` | `text` | 否 |  |  |
| `connector_attribution` | `text` | 是 |  |  |
| `stats` | `jsonb` | 是 |  |  |
| `reaction` | `text` | 是 |  |  |
| `reaction_updated_at_ms` | `bigint` | 是 |  |  |
| `share_count` | `bigint` | 否 | `0` |  |
| `discuss_count` | `bigint` | 否 | `0` |  |
| `origin` | `text` | 否 |  |  |
| `share_instructions` | `jsonb` | 是 |  |  |
| `share_artifact_generated_at_ms` | `bigint` | 是 |  |  |
| `share_artifact_stale` | `boolean` | 否 | `false` |  |
| `created_at_ms` | `bigint` | 否 |  |  |
| `updated_at_ms` | `bigint` | 否 |  |  |
| `embedding` | `vector(384)` | 是 |  |  |
| `manual_order` | `double precision` | 是 |  |  |
| `seen_at_ms` | `bigint` | 是 |  |  |
| `title` | `text` | 是 |  |  |
| `search_vector` | `tsvector` | 是 | `to_tsvector('simple'::regconfig, "left"(((((((COALESCE(kicker, ''::text) \|\| ' '::text) \|\| COALESCE(title, ''::text)) \|\| ' '::text) \|\| COALESCE(body_md, ''::text)) \|\| ' '::text) \|\| COALESCE(category, ''::text)), 200000))` |  |
| `last_seen_at_ms` | `bigint` | 是 |  |  |
| `timespent_ms` | `bigint` | 是 |  |  |
| `social_thumbnail_url` | `text` | 是 |  |  |
| `social_thumbnail_aspect_ratio` | `double precision` | 是 |  |  |
| `social_attribution_text` | `text` | 是 |  |  |
| `social_post_username` | `text` | 是 |  |  |
| `why_did_i_see_this` | `text` | 是 |  |  |
| `source_post_url` | `text` | 是 |  |  |
| `video_media_path` | `text` | 是 |  |  |
| `carousel_clip_paths` | `text[]` | 是 |  |  |
| `source_fleet_origin_id` | `text` | 是 |  | 舰队学习的来源标识；其所有者不在本虚拟机数据库内。 |
| `fleet_reaction_version` | `bigint` | 否 | `0` |  |
| `emoji` | `text` | 是 |  |  |
| `source_idea_id` | `text` | 是 |  | 软性本地引用 → `ideas.ideas.idea_id`。 |
| `icon_key` | `text` | 是 |  |  |
| `source_url_keys` | `text[]` | 是 |  |  |
| `activity_tier` | `text` | 是 |  |  |

键与关系：

- 主键 `units_pkey`: `unit_id`

### `goals`

#### `goals.actions`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `action_id` | `bigint` | 否 | `nextval('goals.actions_action_id_seq'::regclass)` | 本地行标识（主键）。 |
| `goal_id` | `text` | 否 |  | 外键 → `goals.goals.goal_id` |
| `status` | `text` | 否 |  |  |
| `created_at` | `timestamp with time zone` | 否 |  |  |
| `updated_at` | `timestamp with time zone` | 否 |  |  |
| `action_text` | `text` | 否 |  |  |

键与关系：

- 外键 `actions_goal_id_fkey`: `goal_id` → `goals.goals`（`goal_id`）
- 主键 `actions_pkey`: `action_id`

#### `goals.associations`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `association_id` | `text` | 否 |  | 本地行标识符（主键）。 |
| `goal_id` | `text` | 否 |  | 外键 → `goals.goals.goal_id` |
| `association_type` | `text` | 否 |  |  |
| `target_id` | `text` | 否 |  | 由 `association_type` 指定的多态目标：计划任务 ID (`scheduler.jobs.job_id`)、制品 slug (`spaces.spaces.space_slug`)，或磁盘上的文档路径。 |
| `details_json` | `text` | 否 | `'{}'::text` |  |
| `created_at` | `text` | 否 |  |  |
| `updated_at` | `text` | 否 |  |  |
| `created_at_ts` | `timestamp with time zone` | 是 |  |  |
| `updated_at_ts` | `timestamp with time zone` | 是 |  |  |

键与关系：

- 外键 `associations_goal_id_fkey`: `goal_id` → `goals.goals` (`goal_id`)
- 主键 `associations_pkey`: `association_id`
- 唯一键 `associations_goal_id_association_type_target_id_key`: `goal_id`, `association_type`, `target_id`

#### `goals.briefings`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `briefing_id` | `text` | 否 |  | 本地行标识符（主键）。 |
| `goal_id` | `text` | 否 |  | 外键 → `goals.goals.goal_id` |
| `path` | `text` | 否 |  |  |
| `created_at` | `text` | 否 |  |  |
| `updated_at` | `text` | 否 |  |  |
| `last_opened_at` | `text` | 是 |  |  |
| `created_at_ts` | `timestamp with time zone` | 是 |  |  |
| `updated_at_ts` | `timestamp with time zone` | 是 |  |  |
| `last_opened_at_ts` | `timestamp with time zone` | 是 |  |  |
| `hero_image_path` | `text` | 是 |  |  |

键与关系：

- 外键 `briefings_goal_id_fkey`: `goal_id` → `goals.goals` (`goal_id`)
- 主键 `briefings_pkey`: `briefing_id`
- 唯一键 `briefings_goal_id_path_key`: `goal_id`, `path`

#### `goals.engagement_events`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `event_id` | `text` | 否 |  | 本地行标识符（主键）。 |
| `goal_id` | `text` | 否 |  | 外键 → `goals.goals.goal_id` |
| `event_type` | `text` | 否 |  | 参与/反馈事件类型。规范词汇（与 ideas.idea_events 共享）：展示、点击、参与、正面反馈、负面反馈。 |
| `surface` | `text` | 是 |  |  |
| `value` | `double precision` | 是 |  | 参与行的可选数值深度（如停留秒数/权重值）；展示/点击/反馈行则为 null。 |
| `request_id` | `text` | 否 |  | 软本地引用 → `runtime.requests.request_id`。 |
| `metadata` | `jsonb` | 否 | `'{}'::jsonb` |  |
| `created_at` | `timestamp with time zone` | 否 | `now()` |  |

键与关系：

- 外键 `engagement_events_goal_id_fkey`: `goal_id` → `goals.goals` (`goal_id`)
- 主键 `engagement_events_pkey`: `event_id`

#### `goals.goals`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `goal_id` | `text` | 否 |  | 本地行标识符（主键）。 |
| `activity_key` | `text` | 是 |  |  |
| `slug` | `text` | 是 |  |  |
| `title` | `text` | 否 |  |  |
| `summary` | `text` | 是 |  |  |
| `description` | `text` | 是 |  |  |
| `momentum` | `text` | 是 |  |  |
| `momentum_status` | `text` | 是 |  |  |
| `emoji` | `text` | 是 |  |  |
| `image_relpath` | `text` | 是 |  |  |
| `icon_generation_status` | `text` | 是 |  |  |
| `status` | `text` | 否 |  |  |
| `created_at` | `text` | 否 |  |  |
| `updated_at` | `text` | 否 |  |  |
| `parent_goal_id` | `text` | 是 |  | 外键 → `goals.goals.goal_id` |
| `sort_order` | `double precision` | 否 | `0` |  |
| `completed_at` | `text` | 是 |  |  |
| `created_at_ts` | `timestamp with time zone` | 是 |  |  |
| `updated_at_ts` | `timestamp with time zone` | 是 |  |  |
| `last_activity_at` | `text` | 是 |  |  |
| `last_activity_at_ts` | `timestamp with time zone` | 是 |  |  |
| `pushable` | `boolean` | 否 | `true` |  |
| `category` | `text` | 是 |  |  |
| `goal_kind` | `text` | 是 |  |  |
| `value_alignment` | `text` | 是 |  |  |
| `user_words` | `text` | 是 |  |  |
| `assistant_distillation` | `text` | 是 |  |  |
| `woop` | `jsonb` | 是 |  |  |
| `implementation_intentions` | `jsonb` | 是 |  |  |
| `monitoring_signal` | `text` | 是 |  |  |
| `review_cadence` | `text` | 是 |  |  |
| `next_review_question` | `text` | 是 |  |  |
| `momentum_dimensions` | `jsonb` | 是 |  |  |
| `adjustment_recommendation` | `text` | 是 |  |  |
| `provenance` | `jsonb` | 是 |  |  |
| `completed_at_ts` | `timestamp with time zone` | 是 |  |  |
| `source` | `text` | 否 | `'user_goal'::text` |  |
| `attention_kind` | `text` | 否 | `'does_not_need_attention'::text` |  |
| `attention_updated_at` | `timestamp with time zone` | 否 | `now()` |  |
| `next_due_at` | `timestamp with time zone` | 是 |  |  |
| `escalation` | `jsonb` | 是 |  |  |
| `escalation_due_at` | `timestamp with time zone` | 是 |  |  |
| `fixed_position` | `bigint` | 是 |  |  |

键与关系：

- 外键 `goals_parent_goal_id_fkey`: `parent_goal_id` → `goals.goals`（`goal_id`）
- 主键 `goals_pkey`: `goal_id`
- 唯一键 `goals_slug_key`: `slug`

#### `goals.learning_state`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `goal_id` | `text` | 否 |  | 外键 → `goals.goals.goal_id` |
| `target_skill` | `text` | 是 |  |  |
| `prerequisites` | `jsonb` | 否 | `'[]'::jsonb` |  |
| `mastery_estimate` | `double precision` | 是 |  |  |
| `mastery_evidence` | `jsonb` | 否 | `'[]'::jsonb` |  |
| `misconceptions` | `jsonb` | 否 | `'[]'::jsonb` |  |
| `last_retrieval_at` | `timestamp with time zone` | 是 |  |  |
| `next_review_at` | `timestamp with time zone` | 是 |  |  |
| `confidence` | `double precision` | 是 |  |  |
| `transfer_status` | `text` | 是 |  |  |
| `study_status` | `text` | 否 | `'queued'::text` |  |
| `last_studied_at` | `timestamp with time zone` | 是 |  |  |
| `updated_at_ts` | `timestamp with time zone` | 否 | `now()` |  |

键与关系：

- 外键 `learning_state_goal_id_fkey`: `goal_id` → `goals.goals`（`goal_id`）
- 主键 `learning_state_pkey`: `goal_id`

#### `goals.momentum_history`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `history_id` | `bigint` | 否 | `nextval('goals.momentum_history_history_id_seq'::regclass)` | 本地行标识（主键）。 |
| `goal_id` | `text` | 否 |  | 外键 → `goals.goals.goal_id` |
| `momentum_status` | `text` | 是 |  |  |
| `momentum_dimensions` | `jsonb` | 是 |  |  |
| `adjustment_recommendation` | `text` | 是 |  |  |
| `source` | `text` | 否 |  |  |
| `run_id` | `text` | 是 |  | 生产者运行关联标识；无单一的 Muse PostgreSQL 所有者表。 |
| `observed_at` | `timestamp with time zone` | 否 | `now()` |  |
| `created_at` | `timestamp with time zone` | 否 | `now()` |  |

键与关系：

- 外键 `momentum_history_goal_id_fkey`: `goal_id` → `goals.goals` (`goal_id`)
- 主键 `momentum_history_pkey`: `history_id`

#### `goals.sessions`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `root_goal_id` | `text` | 否 |  | 外键 → `goals.goals.goal_id` |
| `root_agent_id` | `text` | 否 |  | 软本地引用 → `agent.agents.agent_id`。 |
| `session_id` | `text` | 否 |  | 软本地引用 → `agent.sessions.session_id`。 |

键与关系：

- 外键 `sessions_root_goal_id_fkey`: `root_goal_id` → `goals.goals` (`goal_id`)
- 主键 `sessions_pkey`: `root_goal_id`

#### `goals.suggestions`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `suggestion_id` | `text` | 否 |  | 本地行标识（主键）。 |
| `goal_id` | `text` | 否 |  | 外键 → `goals.goals.goal_id` |
| `idea_id` | `text` | 否 |  | 软本地引用 → `ideas.ideas.idea_id`。 |
| `status` | `text` | 否 |  |  |
| `created_at` | `text` | 否 |  |  |
| `updated_at` | `text` | 否 |  |  |
| `created_at_ts` | `timestamp with time zone` | 是 |  |  |
| `updated_at_ts` | `timestamp with time zone` | 是 |  |  |

键与关系：

- 外键 `suggestions_goal_id_fkey`: `goal_id` → `goals.goals` (`goal_id`)
- 主键 `suggestions_pkey`: `suggestion_id`
- 唯一约束 `suggestions_goal_id_idea_id_key`: `goal_id`, `idea_id`

#### `goals.thread_actions`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `thread_action_id` | `bigint` | 否 | `nextval('goals.thread_actions_thread_action_id_seq'::regclass)` | 本地行标识（主键）。 |
| `thread_id` | `bigint` | 否 |  | 外键 → `goals.threads.thread_id` |
| `action_id` | `bigint` | 是 |  | 外键 → `goals.actions.action_id` |
| `created_at` | `timestamp with time zone` | 否 | `now()` |  |

键与关系：

- 外键 `thread_actions_action_id_fkey`: `action_id` → `goals.actions` (`action_id`)
- 外键 `thread_actions_thread_id_fkey`: `thread_id` → `goals.threads` (`thread_id`)
- 主键 `thread_actions_pkey`: `thread_action_id`

#### `goals.threads`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `thread_id` | `bigint` | 否 | `nextval('goals.threads_thread_id_seq'::regclass)` | 本地行标识（主键）。 |
| `goal_id` | `text` | 否 |  | 外键 → `goals.goals.goal_id` |
| `request_id` | `text` | 是 |  | 外键 → `runtime.requests.request_id` |
| `created_at` | `timestamp with time zone` | 否 | `now()` |  |

键与关系：

- 外键 `threads_goal_id_fkey`: `goal_id` → `goals.goals` (`goal_id`)
- 外键 `threads_request_id_fkey`: `request_id` → `runtime.requests` (`request_id`)
- 主键 `threads_pkey`: `thread_id`

#### `goals.updates`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `update_id` | `text` | 否 |  | 本地行标识（主键）。 |
| `goal_id` | `text` | 否 |  | 外键 → `goals.goals.goal_id` |
| `title` | `text` | 是 |  |  |
| `summary` | `text` | 是 |  |  |
| `description` | `text` | 是 |  |  |
| `effective_at` | `text` | 是 |  |  |
| `created_at` | `text` | 否 |  |  |
| `updated_at` | `text` | 否 |  |  |
| `effective_at_ts` | `timestamp with time zone` | 是 |  |  |
| `updated_at_ts` | `timestamp with time zone` | 是 |  |  |
| `author_source` | `text` | 否 | `'user'::text` |  |

键与关系：

- 外键 `updates_goal_id_fkey`: `goal_id` → `goals.goals` (`goal_id`)
- 主键 `updates_pkey`: `update_id`

### `health`

#### `health.aggregates`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `aggregate_id` | `bigint` | 否 | `nextval('health.aggregates_aggregate_id_seq'::regclass)` | 本地行标识（主键）。 |
| `provider` | `text` | 否 |  |  |
| `node_id` | `text` | 是 |  | 软本地引用 → `device.nodes.node_id`。 |
| `timezone` | `text` | 是 |  |  |
| `aggregate_type` | `text` | 否 |  |  |
| `period_start` | `timestamp with time zone` | 否 |  |  |
| `period_end` | `timestamp with time zone` | 否 |  |  |
| `numeric_value` | `double precision` | 是 |  |  |
| `unit` | `text` | 是 |  |  |

键与关系：

- 主键 `aggregates_pkey`: `aggregate_id`
- 唯一键 `aggregates_provider_node_id_aggregate_type_period_start_per_key`: `provider`, `node_id`, `aggregate_type`, `period_start`, `period_end`

#### `health.events`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `health_event_id` | `bigint` | 否 | `nextval('health.events_health_event_id_seq'::regclass)` | 本地行标识（主键）。 |
| `provider` | `text` | 否 |  |  |
| `node_id` | `text` | 是 |  | 软本地引用 → `device.nodes.node_id`。 |
| `external_event_id` | `text` | 是 |  | 外部/提供方标识；Muse PostgreSQL 中无所有者表。 |
| `bundle_id` | `text` | 是 |  | 来源应用的捆绑包标识。 |
| `timezone` | `text` | 是 |  |  |
| `event_type` | `text` | 否 |  |  |
| `event_at` | `timestamp with time zone` | 否 |  |  |
| `event_end_at` | `timestamp with time zone` | 是 |  |  |
| `event_text` | `text` | 是 |  |  |

键与关系：

- 主键 `events_pkey`: `health_event_id`

#### `health.record_values`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `record_value_id` | `bigint` | 否 | `nextval('health.record_values_record_value_id_seq'::regclass)` | 本地行标识（主键）。 |
| `record_table` | `text` | 否 |  |  |
| `record_id` | `bigint` | 否 |  | 多态本地引用，由 `record_table` 指定：`health.aggregates.aggregate_id`、`health.events.health_event_id`、`health.sleep_sessions.sleep_session_id` 或 `health.workouts.workout_id`。 |
| `value_name` | `text` | 否 |  |  |
| `unit` | `text` | 是 |  |  |
| `numeric_value` | `double precision` | 是 |  |  |
| `text_value` | `text` | 是 |  |  |

键与关系：

- 主键 `record_values_pkey`: `record_value_id`
- 唯一键 `record_values_record_table_record_id_value_name_key`: `record_table`, `record_id`, `value_name`

#### `health.sample_values`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `sample_value_id` | `bigint` | 否 | `nextval('health.sample_values_sample_value_id_seq'::regclass)` | 本地行标识（主键）。 |
| `sample_id` | `bigint` | 否 |  | 外键 → `health.samples.sample_id` |
| `value_name` | `text` | 否 |  |  |
| `unit` | `text` | 是 |  |  |
| `numeric_value` | `double precision` | 是 |  |  |
| `text_value` | `text` | 是 |  |  |

键与关系：- 外键 `sample_values_sample_id_fkey`: `sample_id` → `health.samples`（`sample_id`）
- 主键 `sample_values_pkey`: `sample_value_id`
- 唯一约束 `sample_values_sample_id_value_name_key`: `sample_id`, `value_name`

#### `health.samples`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `sample_id` | `bigint` | 否 | `nextval('health.samples_sample_id_seq'::regclass)` | 本地行标识符（主键）。 |
| `provider` | `text` | 否 |  |  |
| `node_id` | `text` | 是 |  | 软引用，指向 `device.nodes.node_id`。 |
| `external_sample_id` | `text` | 否 |  | 外部/提供商标识符；不属于 Muse PostgreSQL 的所有者表。 |
| `bundle_id` | `text` | 是 |  | 来源应用的包标识符。 |
| `sample_type` | `text` | 否 |  |  |
| `start_at` | `timestamp with time zone` | 否 |  |  |
| `end_at` | `timestamp with time zone` | 是 |  |  |
| `unit` | `text` | 是 |  |  |
| `numeric_value` | `double precision` | 是 |  |  |
| `text_value` | `text` | 是 |  |  |
| `source_name` | `text` | 是 |  |  |
| `created_at` | `timestamp with time zone` | 否 | `now()` |  |
| `timezone` | `text` | 是 |  |  |

键与关系：

- 主键 `samples_pkey`: `sample_id`

#### `health.sleep_sessions`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `sleep_session_id` | `bigint` | 否 | `nextval('health.sleep_sessions_sleep_session_id_seq'::regclass)` | 本地行标识符（主键）。 |
| `provider` | `text` | 否 |  |  |
| `node_id` | `text` | 是 |  | 软引用，指向 `device.nodes.node_id`。 |
| `external_sleep_id` | `text` | 否 |  | 外部/提供商标识符；不属于 Muse PostgreSQL 的所有者表。 |
| `bundle_id` | `text` | 是 |  | 来源应用的包标识符。 |
| `timezone` | `text` | 是 |  |  |
| `start_at` | `timestamp with time zone` | 否 |  |  |
| `end_at` | `timestamp with time zone` | 否 |  |  |
| `quality_score` | `double precision` | 是 |  |  |
| `awake_seconds` | `double precision` | 是 |  |  |
| `core_seconds` | `double precision` | 是 |  |  |
| `deep_seconds` | `double precision` | 是 |  |  |
| `rem_seconds` | `double precision` | 是 |  |  |
| `asleep_unspecified_seconds` | `double precision` | 是 |  |  |
| `asleep_seconds` | `integer` | 是 |  |  |
| `in_bed_seconds` | `integer` | 是 |  |  |
| `created_at` | `timestamp with time zone` | 否 | `now()` |  |

键与关系：

- 主键 `sleep_sessions_pkey`: `sleep_session_id`

#### `health.synced_ranges`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `synced_range_id` | `bigint` | 否 | `nextval('health.synced_ranges_synced_range_id_seq'::regclass)` | 本地行标识符（主键）。 |
| `provider` | `text` | 否 |  |  |
| `node_id` | `text` | 是 |  | 软引用，指向 `device.nodes.node_id`。 |
| `category` | `text` | 否 |  |  |
| `span` | `tstzrange` | 否 |  |  |
| `updated_at` | `timestamp with time zone` | 否 | `now()` |  |

键与关系：

- 主键 `synced_ranges_pkey`: `synced_range_id`

#### `health.workouts`| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `workout_id` | `bigint` | 否 | `nextval('health.workouts_workout_id_seq'::regclass)` | 本地行标识（主键）。 |
| `provider` | `text` | 否 |  |  |
| `node_id` | `text` | 是 |  | 软本地引用 → `device.nodes.node_id`。 |
| `external_workout_id` | `text` | 否 |  | 外部/提供商标识；无 Muse PostgreSQL 所有者表。 |
| `bundle_id` | `text` | 是 |  | 来源应用的包标识。 |
| `timezone` | `text` | 是 |  |  |
| `workout_type` | `text` | 否 |  |  |
| `start_at` | `timestamp with time zone` | 否 |  |  |
| `end_at` | `timestamp with time zone` | 是 |  |  |
| `active_seconds` | `double precision` | 是 |  |  |
| `energy_kcal` | `double precision` | 是 |  |  |
| `distance_meters` | `double precision` | 是 |  |  |
| `hr_max_bpm` | `double precision` | 是 |  |  |
| `hr_average_bpm` | `double precision` | 是 |  |  |
| `created_at` | `timestamp with time zone` | 否 | `now()` |  |

键与关系：

- 主键 `workouts_pkey`: `workout_id`

### `ideas`

#### `ideas.bandit_arm_state`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `policy_id` | `text` | 否 |  | 外键 → `ideas.explore_policy.policy_id` |
| `domain` | `text` | 否 |  | 本地行标识（主键）。 |
| `lane` | `text` | 否 |  | 本地行标识（主键）。 |
| `decision_points` | `bigint` | 否 | `0` |  |
| `terminal_rewards` | `bigint` | 否 | `0` |  |
| `terminal_observations` | `bigint` | 否 | `0` |  |
| `fast_reward_sum` | `double precision` | 否 | `0` |  |
| `fast_observations` | `bigint` | 否 | `0` |  |
| `updated_at` | `timestamp with time zone` | 否 | `now()` |  |

键与关系：

- 外键 `bandit_arm_state_policy_id_fkey`: `policy_id` → `ideas.explore_policy`（`policy_id`）
- 主键 `bandit_arm_state_pkey`: `policy_id`, `domain`, `lane`

#### `ideas.bandit_fold_state`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `policy_id` | `text` | 否 |  | 外键 → `ideas.explore_policy.policy_id` |
| `folded_until` | `timestamp with time zone` | 是 |  |  |
| `folded_event_id` | `text` | 是 |  | 软本地引用 → `ideas.bandit_folded_events.event_id`。 |
| `updated_at` | `timestamp with time zone` | 否 | `now()` |  |

键与关系：

- 外键 `bandit_fold_state_policy_id_fkey`: `policy_id` → `ideas.explore_policy`（`policy_id`）
- 主键 `bandit_fold_state_pkey`: `policy_id`

#### `ideas.bandit_folded_events`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `policy_id` | `text` | 否 |  | 外键 → `ideas.explore_policy.policy_id` |
| `event_id` | `text` | 否 |  | 外键 → `ideas.idea_events.event_id` |
| `folded_at` | `timestamp with time zone` | 否 | `now()` |  |

键与关系：

- 外键 `bandit_folded_events_event_id_fkey`: `event_id` → `ideas.idea_events`（`event_id`）
- 外键 `bandit_folded_events_policy_id_fkey`: `policy_id` → `ideas.explore_policy`（`policy_id`）
- 主键 `bandit_folded_events_pkey`: `policy_id`, `event_id`

#### `ideas.discovery_pool_history`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `snapshot_id` | `text` | 否 |  | 本地行标识（主键）。 |
| `source` | `text` | 否 |  |  |
| `payload_json` | `jsonb` | 否 |  |  |
| `created_at` | `timestamp with time zone` | 否 | `now()` |  |

键与关系：

- 主键 `flock_explore_history_pkey`: `snapshot_id`

#### `ideas.discovery_pool_meta`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `key` | `text` | 否 |  | 本地行标识（主键）。 |
| `source` | `text` | 否 |  |  |
| `payload_json` | `jsonb` | 否 |  |  |
| `updated_at` | `timestamp with time zone` | 否 | `now()` |  |

键与关系：

- 主键 `flock_explore_meta_pkey`: `key`#### `ideas.explore_policy`

| 列名 | 类型 | 可空 | 默认值 | 键/标识符含义 |
|---|---|---:|---|---|
| `policy_id` | `text` | 否 |  | 本地行标识符（主键）。 |
| `explore_floor` | `integer` | 否 |  |  |
| `per_domain_min` | `integer` | 否 |  |  |
| `version` | `text` | 否 |  |  |
| `updated_at` | `timestamp with time zone` | 否 | `now()` |  |

键与关系：

- 主键 `explore_policy_pkey`: `policy_id`

#### `ideas.feed_snapshot_cards`

| 列名 | 类型 | 可空 | 默认值 | 键/标识符含义 |
|---|---|---:|---|---|
| `snapshot_id` | `text` | 否 |  | 外键 → `ideas.feed_snapshot_sections.snapshot_id` |
| `section_id` | `text` | 否 |  | 外键 → `ideas.feed_snapshot_sections.section_id` |
| `position` | `integer` | 否 |  | 本地行标识符（主键）。 |
| `source_key` | `text` | 否 |  |  |
| `origin` | `text` | 否 |  |  |
| `display_title` | `text` | 是 |  |  |
| `display_summary` | `text` | 是 |  |  |
| `context_label` | `text` | 是 |  |  |
| `detail_description` | `text` | 是 |  |  |
| `build_summary` | `text` | 是 |  |  |
| `type_label` | `text` | 是 |  |  |
| `prerequisite_notes` | `text` | 是 |  |  |

键与关系：

- 外键 `feed_snapshot_cards_snapshot_id_section_id_fkey`: `snapshot_id`, `section_id` → `ideas.feed_snapshot_sections`（`snapshot_id`, `section_id`）
- 主键 `feed_snapshot_cards_pkey`: `snapshot_id`, `section_id`, `position`

#### `ideas.feed_snapshot_sections`

| 列名 | 类型 | 可空 | 默认值 | 键/标识符含义 |
|---|---|---:|---|---|
| `snapshot_id` | `text` | 否 |  | 外键 → `ideas.feed_snapshots.snapshot_id` |
| `section_id` | `text` | 否 |  | 本地行标识符（主键）。 |
| `position` | `integer` | 否 |  |  |
| `role` | `text` | 否 |  |  |
| `title` | `text` | 否 |  |  |
| `subtitle` | `text` | 是 |  |  |

键与关系：

- 外键 `feed_snapshot_sections_snapshot_id_fkey`: `snapshot_id` → `ideas.feed_snapshots`（`snapshot_id`）
- 主键 `feed_snapshot_sections_pkey`: `snapshot_id`, `section_id`
- 唯一键 `feed_snapshot_sections_snapshot_id_position_key`: `snapshot_id`, `position`

#### `ideas.feed_snapshots`

| 列名 | 类型 | 可空 | 默认值 | 键/标识符含义 |
|---|---|---:|---|---|
| `snapshot_id` | `text` | 否 |  | 本地行标识符（主键）。 |
| `generated_at` | `timestamp with time zone` | 否 | `now()` |  |
| `is_active` | `boolean` | 否 | `false` |  |

键与关系：

- 主键 `feed_snapshots_pkey`: `snapshot_id`

#### `ideas.icon_embeddings`

| 列名 | 类型 | 可空 | 默认值 | 键/标识符含义 |
|---|---|---:|---|---|
| `icon_key` | `text` | 否 |  | 本地行标识符（主键）。 |
| `model_name` | `text` | 否 |  | 本地行标识符（主键）。 |
| `descriptions_digest` | `text` | 否 |  | 本地行标识符（主键）。 |
| `embedding` | `vector(384)` | 否 |  |  |
| `created_at` | `timestamp with time zone` | 否 | `now()` |  |

键与关系：

- 主键 `icon_embeddings_pkey`: `icon_key`, `model_name`, `descriptions_digest`

#### `ideas.idea_anchors`

| 列名 | 类型 | 可空 | 默认值 | 键/标识符含义 |
|---|---|---:|---|---|
| `idea_id` | `text` | 否 |  | 外键 → `ideas.ideas.idea_id` |
| `anchor_kind` | `text` | 否 |  | 本地行标识符（主键）。 |
| `anchor_id` | `text` | 否 |  | 本地行标识符（主键）。 |
| `lifecycle_status` | `text` | 否 | `'candidate'::text` |  |
| `build_status` | `text` | 是 |  |  |
| `surfaced_at` | `timestamp with time zone` | 是 |  |  |
| `impression_count` | `integer` | 否 | `0` |  |
| `decided_at` | `timestamp with time zone` | 是 |  |  |
| `dismiss_reason` | `text` | 是 |  |  |
| `created_at` | `timestamp with time zone` | 否 | `now()` |  |
| `updated_at` | `timestamp with time zone` | 否 | `now()` |  |

键与关系：- 外键 `idea_anchors_idea_id_fkey`：`idea_id` → `ideas.ideas`（`idea_id`）
- 主键 `idea_anchors_pkey`：`idea_id`、`anchor_kind`、`anchor_id`

#### `ideas.idea_build_status`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `source_namespace` | `text` | 否 |  | 本地行标识符（主键）。 |
| `source_kind` | `text` | 否 |  | 本地行标识符（主键）。 |
| `source_id` | `text` | 否 |  | 本地行标识符（主键）。 |
| `status` | `text` | 否 |  |  |
| `updated_at` | `timestamp with time zone` | 否 | `now()` |  |
| `selected_item_ids` | `text[]` | 是 |  | 软本地引用 → `ideas.idea_items.idea_item_id`。构建过程中被接受的创意项 ID；NULL 表示无选择来源，空数组表示全卡接受。 |

键与关系：

- 主键 `idea_build_status_pkey`：`source_namespace`、`source_kind`、`source_id`

#### `ideas.idea_card_feeds`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `feed_id` | `text` | 否 |  | 本地行标识符（主键）。 |
| `source` | `text` | 否 |  |  |
| `payload_json` | `jsonb` | 否 |  |  |
| `generated_at` | `timestamp with time zone` | 否 | `now()` |  |
| `updated_at` | `timestamp with time zone` | 否 | `now()` |  |

键与关系：

- 主键 `idea_card_feeds_pkey`：`feed_id`

#### `ideas.idea_dedup`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `idea_id` | `text` | 否 |  | 外键 → `ideas.ideas.idea_id` |
| `content_fingerprint` | `text` | 否 |  |  |
| `embedding_ref` | `text` | 是 |  |  |
| `canonical_idea_id` | `text` | 是 |  | 外键 → `ideas.ideas.idea_id` |
| `merged_at` | `timestamp with time zone` | 是 |  |  |
| `merge_reason` | `text` | 是 |  |  |
| `created_at` | `timestamp with time zone` | 否 | `now()` |  |

键与关系：

- 外键 `idea_dedup_canonical_idea_id_fkey`：`canonical_idea_id` → `ideas.ideas`（`idea_id`）
- 外键 `idea_dedup_idea_id_fkey`：`idea_id` → `ideas.ideas`（`idea_id`）
- 主键 `idea_dedup_pkey`：`idea_id`

#### `ideas.idea_events`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `event_id` | `text` | 否 |  | 本地行标识符（主键）。 |
| `idea_id` | `text` | 否 |  | 外键 → `ideas.ideas.idea_id` |
| `event_type` | `text` | 否 |  | 参与/生命周期事件类型。参与类事件的规范词汇（与 goals.engagement_events 共用）：曝光、点击、参与、正反馈、负反馈。生命周期/决策类事件（浮现、接受、驳回、构建等）也通过此记录流转。 |
| `dismissal_reason` | `text` | 是 |  |  |
| `anchor_kind` | `text` | 是 |  |  |
| `anchor_id` | `text` | 是 |  | 潜在的本地引用；可通过领域上下文在 `ideas.idea_anchors.anchor_id` 中解析。 |
| `lane` | `text` | 是 |  |  |
| `request_id` | `text` | 否 |  | 软本地引用 → `runtime.requests.request_id` |
| `metadata` | `jsonb` | 否 | `'{}'::jsonb` |  |
| `created_at` | `timestamp with time zone` | 否 | `now()` |  |

键与关系：

- 外键 `idea_events_idea_id_fkey`：`idea_id` → `ideas.ideas`（`idea_id`）
- 主键 `idea_events_pkey`：`event_id`

#### `ideas.idea_feedback_state`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `idea_id` | `text` | 否 |  | 外键 → `ideas.ideas.idea_id` |
| `feedback` | `text` | 否 |  |  |
| `reason` | `text` | 是 |  |  |
| `event_id` | `text` | 否 |  | 潜在的本地引用；可通过领域上下文在 `ideas.bandit_folded_events.event_id` 和 `ideas.idea_events.event_id` 中解析。 |
| `surface` | `text` | 是 |  |  |
| `created_at` | `timestamp with time zone` | 否 | `now()` |  |
| `updated_at` | `timestamp with time zone` | 否 | `now()` |  |

键与关系：- 外键 `idea_feedback_state_idea_id_fkey`：`idea_id` → `ideas.ideas`（`idea_id`）
- 主键 `idea_feedback_state_pkey`：`idea_id`

#### `ideas.idea_install_assets`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `asset_id` | `text` | 否 |  | 本地行标识符（主键）。 |
| `idea_id` | `text` | 否 |  | 外键 → `ideas.ideas.idea_id` |
| `asset_type` | `text` | 否 |  |  |
| `asset_json` | `jsonb` | 否 |  |  |
| `created_at` | `timestamp with time zone` | 否 | `now()` |  |
| `updated_at` | `timestamp with time zone` | 否 | `now()` |  |

键和关系：

- 外键 `idea_install_assets_idea_id_fkey`：`idea_id` → `ideas.ideas`（`idea_id`）
- 主键 `idea_install_assets_pkey`：`asset_id`

#### `ideas.idea_items`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `idea_item_id` | `text` | 否 |  | 本地行标识符（主键）。 |
| `idea_id` | `text` | 否 |  | 外键 → `ideas.ideas.idea_id` |
| `position` | `integer` | 否 |  |  |
| `kind` | `text` | 否 |  |  |
| `title` | `text` | 否 |  |  |
| `summary` | `text` | 否 |  |  |
| `detail_description` | `text` | 是 |  |  |
| `instructions` | `text` | 是 |  |  |
| `build_plan_markdown` | `text` | 是 |  |  |
| `status` | `text` | 是 |  |  |
| `created_at` | `timestamp with time zone` | 否 | `now()` |  |
| `updated_at` | `timestamp with time zone` | 否 | `now()` |  |

键和关系：

- 外键 `idea_items_idea_id_fkey`：`idea_id` → `ideas.ideas`（`idea_id`）
- 主键 `idea_items_pkey`：`idea_item_id`
- 唯一约束 `idea_items_idea_id_position_key`：`idea_id`, `position`

#### `ideas.idea_quality`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `idea_id` | `text` | 否 |  | 外键 → `ideas.ideas.idea_id` |
| `relevance_score` | `double precision` | 否 |  |  |
| `novelty_score` | `double precision` | 否 |  |  |
| `feasibility_score` | `double precision` | 否 |  |  |
| `composite_score` | `double precision` | 否 |  |  |
| `composite_version` | `text` | 否 |  |  |
| `scorer` | `text` | 否 |  |  |
| `decision_outcome` | `text` | 否 |  |  |
| `decision_reasons` | `jsonb` | 否 | `'[]'::jsonb` |  |
| `scored_at` | `timestamp with time zone` | 否 | `now()` |  |
| `value_score` | `double precision` | 否 | `0` |  |

键和关系：

- 外键 `idea_quality_idea_id_fkey`：`idea_id` → `ideas.ideas`（`idea_id`）
- 主键 `idea_quality_pkey`：`idea_id`

#### `ideas.idea_sources`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `source_namespace` | `text` | 否 |  | 本地行标识符（主键）。 |
| `source_kind` | `text` | 否 |  | 本地行标识符（主键）。 |
| `source_id` | `text` | 否 |  | 本地行标识符（主键）。 |
| `idea_id` | `text` | 否 |  | 外键 → `ideas.ideas.idea_id` |
| `position` | `integer` | 是 |  |  |
| `metadata_json` | `jsonb` | 否 | `'{}'::jsonb` |  |

键和关系：

- 外键 `idea_sources_idea_id_fkey`：`idea_id` → `ideas.ideas`（`idea_id`）
- 主键 `idea_sources_pkey`：`source_namespace`, `source_kind`, `source_id`

#### `ideas.idea_tags`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `idea_id` | `text` | 否 |  | 外键 → `ideas.ideas.idea_id` |
| `position` | `integer` | 否 |  | 本地行标识符（主键）。 |
| `tag` | `text` | 否 |  |  |

键和关系：

- 外键 `idea_tags_idea_id_fkey`：`idea_id` → `ideas.ideas`（`idea_id`）
- 主键 `idea_tags_pkey`：`idea_id`, `position`

#### `ideas.ideas`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `idea_id` | `text` | 否 |  | 本地行标识符（主键）。 |
| `kind` | `text` | 否 |  |  |
| `title` | `text` | 否 |  |  |
| `summary` | `text` | 否 |  |  |
| `rationale` | `text` | 是 |  |  |
| `category_label` | `text` | 是 |  |  |
| `date_label` | `text` | 是 |  |  |
| `audience` | `text` | 是 |  |  |
| `lane` | `text` | 是 |  |  |
| `install_markdown` | `text` | 是 |  |  |
| `created_at` | `timestamp with time zone` | 否 | `now()` |  |
| `updated_at` | `timestamp with time zone` | 否 | `now()` |  |
| `domain` | `text` | 否 |  |  |
| `dedup_key` | `text` | 是 |  |  |
| `expires_at` | `timestamp with time zone` | 是 |  |  |
| `generator` | `text` | 是 |  |  |
| `search_vector` | `tsvector` | 是 | `to_tsvector('simple'::regconfig, "left"(((((((COALESCE(title, ''::text) \|\| ' '::text) \|\| COALESCE(summary, ''::text)) \|\| ' '::text) \|\| COALESCE(rationale, ''::text)) \|\| ' '::text) \|\| COALESCE(category_label, ''::text)), 200000))` |  |
| `prerequisite_notes` | `text` | 是 |  |  |
| `build_summary` | `text` | 是 |  |  |
| `category_index` | `bigint` | 是 |  |  |
| `embedding` | `vector(384)` | 是 |  |  |

键与关系：

- 主键 `ideas_pkey`: `idea_id`

### `ingest`

#### `ingest.data_source_events`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `event_id` | `bigint` | 否 | `nextval('ingest.data_source_events_event_id_seq'::regclass)` | 本地行标识符（主键）。 |
| `global_seq` | `bigint` | 否 | `nextval('ingest.data_source_events_global_seq_seq'::regclass)` |  |
| `ingest_id` | `text` | 否 |  | 软本地引用 → `ingest.data_source_events.ingest_id`。 |
| `producer_id` | `text` | 否 |  | 数据源生产者标识符；无独立的所有者表。 |
| `source` | `text` | 否 |  |  |
| `origin` | `text` | 否 |  |  |
| `processing_lane` | `text` | 否 |  |  |
| `received_at_text` | `text` | 否 |  |  |
| `received_at_unix_ms` | `bigint` | 否 |  |  |
| `payload_representation` | `text` | 否 |  |  |
| `payload_sha256` | `text` | 是 |  |  |
| `status` | `text` | 否 |  |  |
| `received_at` | `timestamp with time zone` | 否 | `now()` |  |
| `processed_at_text` | `text` | 是 |  |  |
| `processed_at_unix_ms` | `bigint` | 是 |  |  |
| `processed_at` | `timestamp with time zone` | 是 |  |  |
| `handoff_message_id` | `text` | 是 |  | 软本地引用 → `runtime.messages.message_id`。 |
| `summary_preview` | `text` | 是 |  |  |
| `failure_code` | `text` | 是 |  |  |
| `failure_message` | `text` | 是 |  |  |
| `payload` | `text` | 否 |  |  |
| `presentation_locale` | `text` | 是 |  |  |

键与关系：

- 主键 `data_source_events_pkey`: `event_id`
- 唯一键 `data_source_events_global_seq_key`: `global_seq`
- 唯一键 `data_source_events_ingest_id_key`: `ingest_id`

### `media`

#### `media.descriptions`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `media_id` | `text` | 否 |  | 本地行标识符（主键）。 |
| `model` | `text` | 是 |  |  |
| `version` | `bigint` | 否 | `1` |  |
| `created_at` | `timestamp with time zone` | 否 | `now()` |  |
| `description_text` | `text` | 否 |  |  |
| `summary_short_text` | `text` | 否 |  |  |
| `summary_full_text` | `text` | 是 |  |  |
| `people_text` | `text` | 是 |  |  |
| `activity_text` | `text` | 是 |  |  |
| `objects_text` | `text` | 是 |  |  |
| `ocr_text` | `text` | 是 |  |  |
| `location_hint_text` | `text` | 是 |  |  |

键与关系：

- 主键 `descriptions_pkey`: `media_id`

#### `media.exif_values`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `exif_value_id` | `bigint` | 否 | `nextval('media.exif_values_exif_value_id_seq'::regclass)` | 本地行标识（主键）。 |
| `media_id` | `text` | 否 |  | 潜在的本地引用；可通过以下字段的领域上下文解析：`media.descriptions.media_id`、`media.items.media_id`、`media.locations.media_id`。 |
| `tag_name` | `text` | 否 |  |  |
| `scalar_type` | `text` | 否 |  |  |
| `scalar_value` | `text` | 否 |  |  |

键与关系：

- 主键 `exif_values_pkey`: `exif_value_id`
- 唯一键 `exif_values_media_id_tag_name_key`: `media_id`, `tag_name`

#### `media.items`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `media_id` | `text` | 否 |  | 本地行标识（主键）。 |
| `source` | `text` | 否 |  |  |
| `source_media_id` | `text` | 否 |  | 软本地引用 → `media.items.media_id`。 |
| `node_id` | `text` | 是 |  | 软本地引用 → `device.nodes.node_id`。 |
| `file_uri` | `text` | 否 |  |  |
| `media_type` | `text` | 否 |  |  |
| `sha256` | `bytea` | 是 |  |  |
| `byte_len` | `bigint` | 是 |  |  |
| `taken_at` | `timestamp with time zone` | 是 |  |  |
| `taken_at_local` | `text` | 是 |  |  |
| `taken_at_local_date` | `date` | 是 |  |  |
| `uploaded_at` | `timestamp with time zone` | 否 | `now()` |  |
| `uploaded_at_unix` | `bigint` | 否 | `0` |  |
| `local_identifier` | `text` | 是 |  |  |
| `description_status` | `text` | 否 | `'pending'::text` |  |
| `description_attempt_count` | `integer` | 否 | `0` |  |
| `description_next_retry_at_unix` | `bigint` | 是 |  |  |

键与关系：

- 主键 `items_pkey`: `media_id`

#### `media.locations`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `media_id` | `text` | 否 |  | 本地行标识（主键）。 |
| `latitude` | `double precision` | 是 |  |  |
| `longitude` | `double precision` | 是 |  |  |
| `altitude_meters` | `double precision` | 是 |  |  |
| `location_source` | `text` | 是 |  |  |
| `geocode_attempt_count` | `integer` | 否 | `0` |  |
| `geocode_next_retry_at_unix` | `bigint` | 是 |  |  |
| `location_text` | `text` | 是 |  |  |

键与关系：

- 主键 `locations_pkey`: `media_id`

### `memory`

#### `memory.claims`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `claim_id` | `text` | 否 |  | 本地行标识（主键）。 |
| `run_id` | `text` | 否 |  | 不透明的相关标识；未声明本地表关系。 |
| `kind` | `text` | 否 |  |  |
| `salience` | `text` | 否 |  |  |
| `claim_text` | `text` | 否 |  |  |
| `quote` | `text` | 是 |  |  |
| `speaker` | `text` | 否 |  |  |
| `evidence_handles` | `jsonb` | 否 | `'[]'::jsonb` |  |
| `supersedes_claim_id` | `text` | 是 |  | 不透明的相关标识；未声明本地表关系。 |
| `status` | `text` | 否 | `'active'::text` |  |
| `confidence` | `double precision` | 否 |  |  |
| `first_seen` | `timestamp with time zone` | 否 | `now()` |  |
| `reinforced_at` | `timestamp with time zone` | 否 | `now()` |  |
| `valid_until` | `timestamp with time zone` | 是 |  |  |
| `source_path` | `text` | 否 |  |  |
| `source_line` | `bigint` | 否 |  |  |
| `created_at` | `timestamp with time zone` | 否 | `now()` |  |
| `updated_at` | `timestamp with time zone` | 否 | `now()` |  |

键与关系：

- 主键 `claims_pkey`: `claim_id`

#### `memory.embedding_models`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `embedding_model_id` | `bigint` | 否 | `nextval('memory.embedding_models_embedding_model_id_seq'::regclass)` | 本地行标识（主键）。 |
| `model_name` | `text` | 否 |  |  |
| `dimensions` | `integer` | 否 |  |  |
| `distance_metric` | `text` | 否 |  |  |
| `created_at` | `timestamp with time zone` | 否 | `now()` |  |

键和关系：

- 主键 `embedding_models_pkey`：`embedding_model_id`
- 唯一键 `embedding_models_model_name_dimensions_distance_metric_key`：`model_name`, `dimensions`, `distance_metric`

#### `memory.embeddings`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `memory_embedding_id` | `bigint` | 否 | `nextval('memory.embeddings_memory_embedding_id_seq'::regclass)` | 本地行标识符（主键）。 |
| `memory_entry_id` | `bigint` | 否 |  | 外键 → `memory.entries.memory_entry_id` |
| `embedding_model_id` | `bigint` | 否 |  | 外键 → `memory.embedding_models.embedding_model_id` |
| `embedding` | `vector(384)` | 否 |  |  |
| `created_at` | `timestamp with time zone` | 否 | `now()` |  |

键和关系：

- 外键 `embeddings_embedding_model_id_fkey`：`embedding_model_id` → `memory.embedding_models`（`embedding_model_id`）
- 外键 `embeddings_memory_entry_id_fkey`：`memory_entry_id` → `memory.entries`（`memory_entry_id`）
- 主键 `embeddings_pkey`：`memory_embedding_id`
- 唯一键 `embeddings_memory_entry_id_embedding_model_id_key`：`memory_entry_id`, `embedding_model_id`

#### `memory.entries`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `memory_entry_id` | `bigint` | 否 | `nextval('memory.entries_memory_entry_id_seq'::regclass)` | 本地行标识符（主键）。 |
| `memory_uri` | `text` | 否 |  |  |
| `chunk_id` | `text` | 否 |  | 本行所属的记忆块标识，无单独的归属表。 |
| `source_type` | `text` | 否 |  |  |
| `status` | `text` | 否 |  |  |
| `privacy_class` | `text` | 否 |  |  |
| `confidence` | `double precision` | 否 | `1.0` |  |
| `citation_path` | `text` | 是 |  |  |
| `line_start` | `bigint` | 否 | `0` |  |
| `line_end` | `bigint` | 否 | `0` |  |
| `created_at` | `timestamp with time zone` | 否 | `now()` |  |
| `created_at_unix` | `bigint` | 否 | `0` |  |
| `title_text` | `text` | 是 |  |  |
| `body_text` | `text` | 否 |  |  |
| `reason_text` | `text` | 是 |  |  |

键和关系：

- 主键 `entries_pkey`：`memory_entry_id`
- 唯一键 `entries_memory_uri_key`：`memory_uri`

#### `memory.entry_attributes`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `memory_entry_attribute_id` | `bigint` | 否 | `nextval('memory.entry_attributes_memory_entry_attribute_id_seq'::regclass)` | 本地行标识符（主键）。 |
| `memory_entry_id` | `bigint` | 否 |  | 外键 → `memory.entries.memory_entry_id` |
| `attribute_name` | `text` | 否 |  |  |
| `scalar_value` | `text` | 否 |  |  |

键和关系：

- 外键 `entry_attributes_memory_entry_id_fkey`：`memory_entry_id` → `memory.entries`（`memory_entry_id`）
- 主键 `entry_attributes_pkey`：`memory_entry_attribute_id`
- 唯一键 `entry_attributes_memory_entry_id_attribute_name_key`：`memory_entry_id`, `attribute_name`

#### `memory.metadata`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `key` | `text` | 否 |  | 本地行标识符（主键）。 |
| `value` | `text` | 否 |  |  |
| `updated_at` | `timestamp with time zone` | 否 | `now()` |  |

键和关系：

- 主键 `metadata_pkey`：`key`

### `podcasts`

#### `podcasts.episodes`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `slug` | `text` | 否 |  | 本地行标识符（主键）。 |
| `title` | `text` | 否 |  |  |
| `description` | `text` | 否 | `''::text` |  |
| `created_at` | `text` | 否 |  |  |
| `duration_secs` | `bigint` | 否 |  |  |
| `audio_path` | `text` | 否 |  |  |
| `chunk_count` | `bigint` | 否 |  |  |
| `cover_path` | `text` | 是 |  |  |
| `episode_url` | `text` | 是 |  |  |
| `feed_url` | `text` | 是 |  |  |
| `updated_at` | `timestamp with time zone` | 否 | `now()` |  |
| `spotify_episode_uri` | `text` | 是 |  |  |
| `script` | `text` | 是 |  |  |
| `topics` | `text` | 是 |  |  |

键与关系：

- 主键 `episodes_pkey`: `slug`

#### `podcasts.feed`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `singleton` | `boolean` | 否 | `true` | 本地行标识符（主键）。 |
| `feed_title` | `text` | 否 |  |  |
| `feed_id` | `text` | 是 |  | 软本地引用 → `podcasts.feeds.feed_id`，指向最近发布的那个 feed。`podcasts.feeds` 是完整目录；应通过 `feed_url` 将 episode 与此表关联，而非通过本行。 |
| `feed_url` | `text` | 是 |  |  |
| `cover_path` | `text` | 是 |  |  |
| `updated_at` | `timestamp with time zone` | 否 | `now()` |  |
| `spotify_show_url` | `text` | 是 |  |  |
| `spotify_show_id` | `text` | 是 |  | Spotify 提供商标识符；Muse PostgreSQL 中无所有者表。 |

键与关系：

- 主键 `feed_pkey`: `singleton`

#### `podcasts.feeds`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `feed_id` | `text` | 否 |  | 本地行标识符（主键）。 |
| `feed_title` | `text` | 否 |  |  |
| `feed_url` | `text` | 是 |  |  |
| `cover_path` | `text` | 是 |  |  |
| `spotify_show_url` | `text` | 是 |  |  |
| `spotify_show_id` | `text` | 是 |  | Spotify 提供商标识符；Muse PostgreSQL 中无所有者表。 |
| `created_at` | `timestamp with time zone` | 否 | `now()` |  |
| `updated_at` | `timestamp with time zone` | 否 | `now()` |  |

键与关系：

- 主键 `feeds_pkey`: `feed_id`

### `runtime`

#### `runtime.agent_todo_snapshots`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `conversation_id` | `text` | 否 |  | 本地行标识符（主键）。 |
| `agent_identity` | `text` | 否 |  | 本地行标识符（主键）。 |
| `items_json` | `text` | 否 | `'[]'::text` |  |
| `revision` | `bigint` | 否 | `1` |  |
| `updated_at_ms` | `bigint` | 否 | `((EXTRACT(epoch FROM now()) * (1000)::numeric))::bigint` |  |

键与关系：

- 主键 `agent_todo_snapshots_pkey`: `conversation_id`, `agent_identity`

#### `runtime.avatar_state`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `singleton` | `boolean` | 否 | `true` | 本地行标识符（主键）。 |
| `active_stem` | `text` | 否 | `''::text` |  |
| `image_path` | `text` | 否 | `''::text` |  |
| `darkmode_image_path` | `text` | 是 |  |  |
| `chat_theme_color` | `text` | 是 |  |  |
| `chat_theme_color_override` | `text` | 是 |  |  |
| `static_frame_paths` | `text[]` | 否 | `'{}'::text[]` |  |
| `video_variants_json` | `text` | 否 | `'{}'::text` |  |
| `darkmode_video_variants_json` | `text` | 否 | `'{}'::text` |  |
| `video_variant_repair_json` | `text` | 否 | `'{}'::text` |  |
| `darkmode_video_variant_repair_json` | `text` | 否 | `'{}'::text` |  |
| `revision` | `bigint` | 否 | `0` |  |
| `updated_at_ms` | `bigint` | 否 | `((EXTRACT(epoch FROM now()) * (1000)::numeric))::bigint` |  |
| `image_variant_assets_json` | `text` | 否 | `'{"assets":[]}'::text` |  |
| `avatar_asset_repair_json` | `text` | 否 | `'{"repairs":[]}'::text` |  |
| `choreography_profile_json` | `text` | 是 |  |  |
| `finalization_id` | `text` | 是 |  | 不透明的关联标识符；未声明本地表关系。 |
| `avatar_milestones_json` | `text` | 否 | `'{}'::text` |  |
| `legacy_folded_at_ms` | `bigint` | 是 |  |  |
| `advanced_avatar_variants_json` | `text` | 否 | `'{}'::text` |  |

键和关系：

- 主键 `avatar_state_pkey`: `singleton`

#### `runtime.browser_tasks`

| 列名 | 类型 | 可为空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `task_id` | `text` | 否 |  | 本地行标识符（主键）。 |
| `browser_session_id` | `text` | 否 | `'default'::text` | 浏览器运行时标识符；无 Muse PostgreSQL 所有者表。 |
| `root_session_id` | `text` | 否 |  | 软本地引用 → `agent.sessions.session_id`。 |
| `owner_agent_id` | `text` | 是 |  | 对于代理创建的任务，包括历史为 NULL 的 `owner_kind`，软本地引用 → `agent.agents.agent_id`。当 `owner_kind` 为 `user` 时，值为 NULL；无执行代理或所有者行。 |
| `root_message_id` | `text` | 否 |  | 对于代理创建的任务，包括历史为 NULL 的 `owner_kind`，软本地引用 → `runtime.messages.message_id`。当 `owner_kind` 为 `user` 时，值与本行的 `task_id` 相同；无消息所有者行。 |
| `stream_owner_message_id` | `text` | 否 |  | 对于代理创建的任务，包括历史为 NULL 的 `owner_kind`，软本地引用 → `runtime.messages.message_id`。当 `owner_kind` 为 `user` 时，值与本行的 `task_id` 相同；无消息所有者行。 |
| `status` | `text` | 否 |  |  |
| `title` | `text` | 否 | `'Browser task'::text` |  |
| `step_count` | `integer` | 否 | `0` |  |
| `chat_context_json` | `text` | 是 |  |  |
| `latest_action_id` | `text` | 是 |  | 浏览器运行时操作标识符；无 Muse PostgreSQL 所有者表。 |
| `latest_tab_json` | `text` | 是 |  |  |
| `latest_screenshot_json` | `text` | 是 |  |  |
| `created_at` | `timestamp with time zone` | 否 | `now()` |  |
| `updated_at` | `timestamp with time zone` | 否 | `now()` |  |
| `completed_at` | `timestamp with time zone` | 是 |  |  |
| `terminal_reason` | `text` | 是 |  |  |
| `admission_seq` | `bigint` | 是 |  |  |
| `presentation_root_session_id` | `text` | 是 |  | 软本地引用 → `agent.sessions.session_id`。 |
| `parent_agent_id` | `text` | 是 |  | 软本地引用 → `agent.agents.agent_id`。 |
| `tool_call_id` | `text` | 是 |  | 对于代理创建的任务，包括历史为 NULL 的 `owner_kind`，软本地引用 → `runtime.tool_calls.call_id`（文本关联标识符，而非其数值型 `tool_call_id`）。当 `owner_kind` 为 `user` 时，值与本行的 `task_id` 相同；无工具调用或所有者行。 |
| `request_trace_context_json` | `text` | 是 |  |  |
| `requester_source` | `text` | 是 |  |  |
| `requester_transport` | `text` | 是 |  |  |
| `requester_model` | `text` | 是 |  |  |
| `requester_effective_model` | `text` | 是 |  |  |
| `request_mode_json` | `text` | 是 |  |  |
| `max_training_tier_json` | `text` | 是 |  |  |
| `is_task_card_visible` | `boolean` | 否 | `false` |  |
| `initial_instruction` | `text` | 是 |  |  |
| `history_source_agent_id` | `text` | 是 |  | 软本地引用 → `agent.agents.agent_id`。 |
| `state_revision` | `bigint` | 否 | `0` |  |
| `deadline_at` | `timestamp with time zone` | 是 |  |  |
| `continuation_root_task_id` | `text` | 是 |  | 软本地引用 → `runtime.browser_tasks.task_id`。 |
| `terminal_interrupt_subtype` | `text` | 是 |  |  |
| `outcome_status` | `text` | 是 |  |  |
| `outcome_reason` | `text` | 是 |  |  |
| `outcome_at` | `timestamp with time zone` | 是 |  |  |
| `retention_end_reason` | `text` | 是 |  |  |
| `card_generation` | `bigint` | 否 | `0` |  |
| `input_grants_json` | `text` | 否 | `'{"version":1,"grants":[]}'::text` |  |
| `browser_navigation_attempted` | `boolean` | 否 | `false` |  |
| `terminal_user_update_due_at` | `timestamp with time zone` | 是 |  | 当此终端实例尚需向用户发送结束更新时该字段不为空；同时也作为工作者的重试/认领时间。 |
| `egress_profile` | `text` | 是 |  |  |
| `run_number` | `bigint` | 是 |  |  |
| `run_started_at` | `timestamp with time zone` | 是 |  |  |
| `run_presentation_locale` | `text` | 是 |  | 当前 BrowserTask 运行期间冻结的 BCP 47 展示语言环境；遗留行若为空则默认为 en-US。 |
| `run_location_context_json` | `text` | 是 |  |  |
| `owner_kind` | `text` | 是 |  | 创建来源：用户、主代理或定时任务；未知时为 NULL。在接管和延续过程中保持不变。用户租约的 `owner_agent_id` 为 NULL，并将 `root_message_id`、`stream_owner_message_id` 和 `tool_call_id` 保留为 `task_id`，且无执行代理、聊天卡片或终端聊天交付。词汇和生命周期由 Rust 管理。 |
| `broker_instance` | `text` | 否 | `'user'::text` | 由可信准入机制选定的不可变物理浏览器所有者：用户或定时任务。与逻辑所有者类型无关，并在延续过程中保留。键和关系：

- 主键 `browser_tasks_pkey`：`task_id`

#### `runtime.chat_event_derived_write_backlog`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `event_seq` | `bigint` | 否 |  | 外键 → `runtime.events.event_seq` |
| `resources_json` | `text` | 否 | `'[]'::text` |  |
| `attempts` | `integer` | 否 | `0` |  |
| `last_error` | `text` | 是 |  |  |
| `created_at` | `timestamp with time zone` | 否 | `now()` |  |
| `updated_at` | `timestamp with time zone` | 否 | `now()` |  |
| `deadlettered_at` | `timestamp with time zone` | 是 |  |  |

键和关系：

- 外键 `chat_event_derived_write_backlog_event_seq_fkey`：`event_seq` → `runtime.events`（`event_seq`）
- 主键 `chat_event_derived_write_backlog_pkey`：`event_seq`

#### `runtime.checkout_spend_checkpoints`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `operation_id` | `text` | 否 |  | 外键 → `runtime.checkout_spend_operations.operation_id` |
| `provider` | `text` | 否 |  |  |
| `checkpoint_key` | `text` | 否 |  | 本地行标识符（主键）。 |
| `external_id` | `text` | 是 |  | 外部/提供商标识符；无 Muse PostgreSQL 所有者表。 |
| `detail` | `jsonb` | 是 |  |  |
| `claimed_at_ms` | `bigint` | 否 |  |  |
| `bound_at_ms` | `bigint` | 是 |  |  |

键和关系：

- 外键 `checkout_spend_checkpoints_operation_id_fkey`：`operation_id` → `runtime.checkout_spend_operations`（`operation_id`）
- 主键 `checkout_spend_checkpoints_pkey`：`operation_id`, `checkpoint_key`

#### `runtime.checkout_spend_operations`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `operation_id` | `text` | 否 |  | 本地行标识符（主键）。 |
| `client_task_key` | `text` | 否 |  |  |
| `checkout_request_id` | `text` | 是 |  | 本操作行拥有的结账关联键；无单独的所有者表。 |
| `wallet_request_id` | `text` | 是 |  | 钱包-提供商请求标识符；无 Muse PostgreSQL 所有者表。 |
| `approval_id` | `text` | 是 |  | 标志性审批标识符；Stripe Link 消费行在 `runtime.stripe_link_spend_requests` 中与其保持一致。 |
| `state` | `text` | 否 |  |  |
| `expires_at_ms` | `bigint` | 否 |  |  |
| `created_at_ms` | `bigint` | 否 |  |  |
| `updated_at_ms` | `bigint` | 否 |  |  |
| `terminal_result` | `jsonb` | 是 |  |  |
| `recovery_epoch` | `smallint` | 否 | `0` |  |
| `wallet_provider` | `text` | 否 | `'stripe-link'::text` |  |

键和关系：

- 主键 `checkout_spend_operations_pkey`：`operation_id`

#### `runtime.client_rendering_capabilities`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `singleton` | `boolean` | 否 | `true` | 本地行标识符（主键）。 |
| `client_id` | `text` | 否 |  | 客户端安装标识符；无单独的 Muse PostgreSQL 所有者表。 |
| `platform` | `text` | 否 |  |  |
| `supported_presentations` | `text[]` | 否 | `'{}'::text[]` |  |
| `supported_inline_presentations` | `text[]` | 否 | `'{}'::text[]` |  |
| `declared_at` | `timestamp with time zone` | 否 | `now()` |  |

键和关系：

- 主键 `client_rendering_capabilities_pkey`：`singleton`

#### `runtime.context_snapshots`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `snapshot_kind` | `text` | 否 |  | 本地行标识符（主键）。 |
| `owner_key` | `text` | 否 |  | 本地行标识符（主键）。 |
| `version` | `bigint` | 否 |  |  |
| `source_watermark` | `text` | 是 |  |  |
| `fingerprint` | `text` | 是 |  |  |
| `freshness_class` | `text` | 否 |  |  |
| `state_json` | `text` | 否 | `'{}'::text` |  |
| `refreshed_at` | `timestamp with time zone` | 否 | `now()` |  |
| `invalidated_at` | `timestamp with time zone` | 是 |  |  |
| `stale_after` | `timestamp with time zone` | 是 |  |  |
| `last_error` | `text` | 是 |  |  |
| `updated_at` | `timestamp with time zone` | 否 | `now()` |  |

键与关系：

- 主键 `context_snapshots_pkey`: `snapshot_kind`, `owner_key`

#### `runtime.dev_notice_watermark`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `singleton` | `boolean` | 否 | `true` | 本地行标识符（主键）。 |
| `last_seen_version_id` | `bigint` | 否 |  | 来自代码所有者开发者通知目录的版本；无所有者表。 |

键与关系：

- 主键 `dev_notice_watermark_pkey`: `singleton`

#### `runtime.event_hook_space_owners`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `hook_id` | `text` | 否 |  | 本地行标识符（主键）。 |
| `space_slug` | `text` | 否 |  |  |
| `created_at` | `timestamp with time zone` | 否 | `now()` |  |
| `updated_at` | `timestamp with time zone` | 否 | `now()` |  |

键与关系：

- 主键 `event_hook_space_owners_pkey`: `hook_id`

#### `runtime.event_payload_fields`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `event_payload_field_id` | `bigint` | 否 | `nextval('runtime.event_payload_fields_event_payload_field_id_seq'::regclass)` | 本地行标识符（主键）。 |
| `event_seq` | `bigint` | 否 |  | 外键 → `runtime.events.event_seq` |
| `field_path` | `text` | 否 |  |  |
| `scalar_type` | `text` | 否 |  |  |
| `scalar_value` | `text` | 是 |  |  |
| `text_value` | `text` | 是 |  |  |

键与关系：

- 外键 `event_payload_fields_event_seq_fkey`: `event_seq` → `runtime.events`（`event_seq`）
- 主键 `event_payload_fields_pkey`: `event_payload_field_id`
- 唯一键 `event_payload_fields_event_seq_field_path_key`: `event_seq`, `field_path`

#### `runtime.events`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `event_seq` | `bigint` | 否 | `nextval('runtime.events_event_seq_seq'::regclass)` | 本地行标识符（主键）。 |
| `event_id` | `uuid` | 否 | `gen_random_uuid()` | 该行拥有的稳定本地事件标识符；`event_seq`为其主键。 |
| `event_kind` | `runtime.event_kind` | 否 |  |  |
| `event_name` | `text` | 否 |  |  |
| `request_id` | `text` | 是 |  | 外键 → `runtime.requests.request_id` |
| `root_request_id` | `text` | 是 |  | 外键 → `runtime.requests.request_id` |
| `parent_request_id` | `text` | 是 |  | 外键 → `runtime.requests.request_id` |
| `transcript_surface` | `runtime.transcript_surface` | 否 |  |  |
| `visibility` | `runtime.visibility` | 否 |  |  |
| `source` | `text` | 否 |  |  |
| `role` | `runtime.message_role` | 是 |  |  |
| `chat_kind` | `text` | 否 | `'direct'::text` |  |
| `stream_lane` | `text` | 否 | `'main'::text` |  |
| `message_id` | `text` | 是 |  | 软本地引用 → `runtime.messages.message_id`。 |
| `parent_message_id` | `text` | 是 |  | 软本地引用 → `runtime.messages.message_id`。 |
| `reply_to_message_id` | `text` | 是 |  | 软本地引用 → `runtime.messages.message_id`。 |
| `reply_target_message_id` | `text` | 是 |  | 软本地引用 → `runtime.messages.message_id`。 |
| `parent_agent_id` | `text` | 是 |  | 软本地引用 → `agent.agents.agent_id`。 |
| `agent_id` | `text` | 是 |  | 软本地引用 → `agent.agents.agent_id`。 |
| `idempotency_key` | `text` | 是 |  |  |
| `display_text_ready` | `boolean` | 否 | `true` |  |
| `created_at` | `timestamp with time zone` | 否 | `now()` |  |
| `payload_json` | `text` | 是 |  |  |
| `chat_context_json` | `text` | 是 |  |  |

键与关系：

- 外键 `events_parent_request_id_fkey`: `parent_request_id` → `runtime.requests` (`request_id`)
- 外键 `events_request_id_fkey`: `request_id` → `runtime.requests` (`request_id`)
- 外键 `events_root_request_id_fkey`: `root_request_id` → `runtime.requests` (`request_id`)
- 主键 `events_pkey`: `event_seq`
- 唯一键 `events_event_id_key`: `event_id`
- 唯一键 `events_idempotency_key_key`: `idempotency_key`

#### `runtime.execute_resolve_runs`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `run_id` | `text` | 否 |  | 本地行标识符（主键）。 |
| `worker_kind` | `text` | 否 |  |  |
| `source_ref` | `text` | 否 |  |  |
| `lane_key` | `text` | 否 |  |  |
| `status` | `text` | 否 |  |  |
| `terminal_decision` | `text` | 是 |  |  |
| `terminal_message` | `text` | 是 |  |  |
| `handoff_message_id` | `text` | 是 |  | 软本地引用 → `runtime.messages.message_id`。 |
| `last_error_stage` | `text` | 是 |  |  |
| `last_error_message` | `text` | 是 |  |  |
| `metadata_json` | `jsonb` | 否 | `'{}'::jsonb` |  |
| `created_at` | `timestamp with time zone` | 否 | `now()` |  |
| `updated_at` | `timestamp with time zone` | 否 | `now()` |  |
| `worker_generation` | `bigint` | 否 | `0` |  |
| `worker_phase` | `text` | 是 |  |  |
| `worker_agent_id` | `text` | 是 |  | 软本地引用 → `agent.agents.agent_id`。 |
| `worker_started_at` | `timestamp with time zone` | 是 |  |  |
| `execute_message_id` | `text` | 是 |  | 软本地引用 → `runtime.messages.message_id`。 |
| `resolve_message_id` | `text` | 是 |  | 软本地引用 → `runtime.messages.message_id`。 |

键与关系：

- 主键 `execute_resolve_runs_pkey`: `run_id`
- 唯一键 `execute_resolve_runs_source_unique`: `worker_kind`, `source_ref`

#### `runtime.idea_execution_pending`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `activation_id` | `text` | 否 |  | 本地行标识符（主键）。 |
| `root_submission_message_id` | `text` | 否 |  | 软本地引用 → `runtime.messages.message_id`。 |
| `session_id` | `text` | 否 |  | 软本地引用 → `agent.sessions.session_id`。 |
| `idea_card_id` | `text` | 否 |  | 软本地引用 → `ideas.ideas.idea_id`。 |
| `idea_card_kind` | `text` | 否 |  |  |
| `dispatched_at` | `timestamp with time zone` | 否 | `now()` |  |
| `reconciled_at` | `timestamp with time zone` | 是 |  |  |

键与关系：

- 主键 `idea_execution_pending_pkey`: `activation_id`

#### `runtime.invite_badge_seen_state`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `singleton` | `boolean` | 否 | `true` | 本地行标识符（主键）。 |
| `last_seen_badge_version` | `bigint` | 否 | `0` |  |

键与关系：

- 主键 `invite_badge_seen_state_pkey`: `singleton`

#### `runtime.maintenance_markers`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `marker_key` | `text` | 否 |  | 本地行标识符（主键）。 |
| `completed_at` | `timestamp with time zone` | 否 | `now()` |  |
| `detail_json` | `jsonb` | 否 | `'{}'::jsonb` |  |

键与关系：

- 主键 `maintenance_markers_pkey`: `marker_key`

#### `runtime.message_attachments`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `attachment_id` | `bigint` | 否 | `nextval('runtime.message_attachments_attachment_id_seq'::regclass)` | 本地行标识符（主键）。 |
| `message_id` | `text` | 否 |  | 外键 → `runtime.messages.message_id` |
| `ordinal` | `integer` | 否 |  |  |
| `attachment_kind` | `text` | 否 |  |  |
| `file_uri` | `text` | 否 |  |  |
| `media_type` | `text` | 是 |  |  |
| `byte_len` | `bigint` | 是 |  |  |
| `sha256` | `bytea` | 是 |  |  |
| `caption_text` | `text` | 是 |  |  |
| `transcription_text` | `text` | 是 |  |  |

键与关系：

- 外键 `message_attachments_message_id_fkey`: `message_id` → `runtime.messages`（`message_id`）
- 主键 `message_attachments_pkey`: `attachment_id`
- 唯一键 `message_attachments_message_id_ordinal_key`: `message_id`, `ordinal`

#### `runtime.message_reactions`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `message_id` | `text` | 否 |  | 外键 → `runtime.messages.message_id` |
| `reaction_emoji` | `text` | 是 |  |  |
| `updated_at` | `timestamp with time zone` | 否 | `now()` |  |

键与关系：

- 外键 `message_reactions_message_id_fkey`: `message_id` → `runtime.messages`（`message_id`）
- 主键 `message_reactions_pkey`: `message_id`

#### `runtime.messages`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `message_id` | `text` | 否 |  | 本地行标识符（主键）。 |
| `event_seq` | `bigint` | 否 |  | 外键 → `runtime.events.event_seq` |
| `role` | `runtime.message_role` | 否 |  |  |
| `prompt_rendering_id` | `bigint` | 是 |  | 提示渲染标识符；无可查询的所有者表。 |
| `author_label` | `text` | 是 |  |  |
| `provider_message_id` | `text` | 是 |  | 外部/提供方标识符；无 Muse PostgreSQL 所有者表。 |
| `created_at` | `timestamp with time zone` | 否 | `now()` |  |
| `body` | `text` | 是 |  |  |

键与关系：

- 外键 `messages_event_seq_fkey`: `event_seq` → `runtime.events`（`event_seq`）
- 主键 `messages_pkey`: `message_id`
- 唯一键 `messages_event_seq_key`: `event_seq`

#### `runtime.product_improvements_preference`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `singleton` | `boolean` | 否 | `true` | 本地行标识（主键）。 |
| `enabled` | `boolean` | 否 |  |  |
| `updated_at` | `timestamp with time zone` | 否 | `now()` |  |

键与关系：

- 主键 `product_improvements_preference_pkey`: `singleton`

#### `runtime.raw_signal_collections`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `collection_id` | `uuid` | 否 |  | 本地行标识（主键）。 |
| `tool` | `text` | 否 |  |  |
| `mode` | `text` | 否 |  |  |
| `fetched_at` | `timestamp with time zone` | 否 |  |  |
| `partial` | `boolean` | 否 |  |  |
| `skipped` | `boolean` | 否 |  |  |
| `per_request_timeout_secs` | `integer` | 否 |  |  |
| `max_total_secs` | `integer` | 否 |  |  |
| `source_names` | `text[]` | 否 |  |  |
| `entry_count` | `integer` | 否 |  |  |
| `metadata_json` | `jsonb` | 否 | `'{}'::jsonb` |  |
| `created_at` | `timestamp with time zone` | 否 | `now()` |  |

键与关系：

- 主键 `raw_signal_collections_pkey`: `collection_id`

#### `runtime.raw_signal_entries`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `entry_id` | `bigint` | 否 | `nextval('runtime.raw_signal_entries_entry_id_seq'::regclass)` | 本地行标识（主键）。 |
| `collection_id` | `uuid` | 否 |  | 外键 → `runtime.raw_signal_collections.collection_id` |
| `ordinal` | `integer` | 否 |  |  |
| `source` | `text` | 否 |  |  |
| `name` | `text` | 否 |  |  |
| `method` | `text` | 否 |  |  |
| `logical_url` | `text` | 否 |  |  |
| `ok` | `boolean` | 否 |  |  |
| `status` | `integer` | 是 |  |  |
| `logical_path` | `text` | 是 |  |  |
| `error` | `text` | 是 |  |  |
| `duration_ms` | `bigint` | 否 |  |  |
| `bytes` | `bigint` | 是 |  |  |
| `body_json` | `jsonb` | 是 |  |  |
| `created_at` | `timestamp with time zone` | 否 | `now()` |  |

键与关系：

- 外键 `raw_signal_entries_collection_id_fkey`: `collection_id` → `runtime.raw_signal_collections`（`collection_id`）
- 主键 `raw_signal_entries_pkey`: `entry_id`
- 唯一约束 `raw_signal_entries_collection_ordinal_unique`: `collection_id`, `ordinal`

#### `runtime.requests`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `request_id` | `text` | 否 |  | 本地行标识（主键）。 |
| `root_request_id` | `text` | 是 |  | 外键 → `runtime.requests.request_id` |
| `parent_request_id` | `text` | 是 |  | 外键 → `runtime.requests.request_id` |
| `request_origin` | `text` | 否 |  |  |
| `transcript_surface` | `runtime.transcript_surface` | 否 |  |  |
| `root_work_class` | `text` | 否 |  |  |
| `created_at` | `timestamp with time zone` | 否 | `now()` |  |

键与关系：

- 外键 `requests_parent_request_id_fkey`: `parent_request_id` → `runtime.requests`（`request_id`）
- 外键 `requests_root_request_id_fkey`: `root_request_id` → `runtime.requests`（`request_id`）
- 主键 `requests_pkey`: `request_id`

#### `runtime.resources`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `resource_id` | `bigint` | 否 | `nextval('runtime.resources_resource_id_seq'::regclass)` | 本地行标识（主键）。 |
| `event_seq` | `bigint` | 否 |  | 外键 → `runtime.events.event_seq` |
| `ordinal` | `integer` | 否 | `0` |  |
| `resource_kind` | `text` | 否 |  |  |
| `resource_key` | `text` | 否 |  |  |
| `label` | `text` | 是 |  |  |
| `mime_type` | `text` | 是 |  |  |
| `size_bytes` | `bigint` | 是 |  |  |
| `metadata_json` | `text` | 是 |  |  |
| `created_at` | `timestamp with time zone` | 否 | `now()` |  |

键与关系：

- 外键 `resources_event_seq_fkey`: `event_seq` → `runtime.events`（`event_seq`）
- 主键 `resources_pkey`: `resource_id`
- 唯一约束 `resources_event_seq_ordinal_key`: `event_seq`, `ordinal`#### `runtime.search_documents`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `search_document_id` | `bigint` | 否 | `nextval('runtime.search_documents_search_document_id_seq'::regclass)` | 本地行标识（主键）。 |
| `owner_table` | `text` | 否 |  |  |
| `owner_key` | `text` | 否 |  |  |
| `language` | `regconfig` | 否 | `'english'::regconfig` |  |
| `search_vector` | `tsvector` | 否 |  |  |
| `updated_at` | `timestamp with time zone` | 否 | `now()` |  |
| `search_text` | `text` | 否 |  |  |

键与关系：

- 主键 `search_documents_pkey`: `search_document_id`
- 唯一键 `search_documents_owner_table_owner_key_key`: `owner_table`, `owner_key`

#### `runtime.skill_invalidation_state`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `skill_name` | `text` | 否 |  | 本地行标识（主键）。 |
| `used` | `boolean` | 否 | `false` |  |
| `pending_invalidation_hash` | `text` | 是 |  |  |
| `pending_manifest_rel_path` | `text` | 是 |  |  |
| `last_delivered_invalidation_hash` | `text` | 是 |  |  |

键与关系：

- 主键 `skill_invalidation_state_pkey`: `skill_name`

#### `runtime.stripe_link_spend_requests`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `approval_id` | `text` | 否 |  | 本地行标识（主键）。 |
| `continuation_root_task_id` | `text` | 否 |  | 软引用 → `runtime.browser_tasks.task_id`。 |
| `merchant_origin` | `text` | 否 |  |  |
| `checkout_metadata` | `jsonb` | 否 |  |  |
| `approved_amount_minor` | `bigint` | 否 |  |  |
| `approved_currency` | `text` | 否 |  |  |
| `lifecycle_state` | `text` | 否 |  |  |
| `create_attempt_id` | `uuid` | 否 |  | 此消费请求行拥有的幂等令牌。 |
| `stripe_spend_request_id` | `text` | 是 |  | Stripe Link 提供商标识；无 Muse PostgreSQL 所有者表。 |
| `expires_at` | `timestamp with time zone` | 否 |  |  |
| `assigned_browser_task_id` | `text` | 是 |  | 软引用 → `runtime.browser_tasks.task_id`。 |
| `assigned_at` | `timestamp with time zone` | 是 |  |  |
| `cancel_attempt_id` | `uuid` | 是 |  | 此消费请求行拥有的取消幂等令牌。 |
| `closed_at` | `timestamp with time zone` | 是 |  |  |
| `close_reason` | `text` | 是 |  |  |
| `created_at` | `timestamp with time zone` | 否 | `now()` |  |
| `owner_browser_task_id` | `text` | 是 |  | 软引用 → `runtime.browser_tasks.task_id`。当前 BrowserTask 的确切拥有者。仅对那些无法持久证明其所有者的携带行为空。 |
| `claiming_browser_task_lineage_id` | `text` | 是 |  | 浏览器运行时谱系声明；无独立的 Muse PostgreSQL 所有者表。可移动的当前所有者谱系声明；在结账成功后保留，并在取消、无效或过期时清除。 |
| `cancellation_confirmation_task_state_revision` | `bigint` | 是 |  | 应答前的确切暂停挑战修订版，应答后的确切恢复运行修订版。 |
| `cancellation_confirmation_response_message_id` | `text` | 是 |  | 软引用 → `runtime.messages.message_id`。被允许恢复保留的 BrowserTask 的真实用户授权消息。 |
| `wallet_provider` | `text` | 否 | `'stripe-link'::text` | 拥有此直接 BrowserTask 生命周期的钱包提供商。该表名仅为保持向后兼容性而保留。 |

键与关系：

- 主键 `stripe_link_spend_requests_pkey`: `approval_id`

#### `runtime.summaries`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `id` | `bigint` | 否 | `nextval('runtime.summaries_id_seq'::regclass)` | 本地行标识（主键）。 |
| `summary_key` | `text` | 是 |  |  |
| `summary_text` | `text` | 是 |  |  |
| `created_at_ms` | `bigint` | 否 | `((EXTRACT(epoch FROM now()))::bigint * 1000)` |  |

键与关系：- 主键 `summaries_pkey`：`id`

#### `runtime.tool_calls`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `tool_call_id` | `bigint` | 否 | `nextval('runtime.tool_calls_tool_call_id_seq'::regclass)` | 本地行标识（主键）。 |
| `event_seq` | `bigint` | 否 |  | 外键 → `runtime.events.event_seq` |
| `call_id` | `text` | 否 |  | 潜在的本地引用；可通过领域上下文在 `runtime.workflow_agent_calls.call_id` 中解析。 |
| `tool_name` | `text` | 否 |  |  |
| `server_name` | `text` | 是 |  |  |
| `status` | `text` | 否 |  |  |
| `created_at` | `timestamp with time zone` | 否 | `now()` |  |
| `arguments_json` | `text` | 是 |  |  |

键与关系：

- 外键 `tool_calls_event_seq_fkey`：`event_seq` → `runtime.events`（`event_seq`）
- 主键 `tool_calls_pkey`：`tool_call_id`
- 唯一键 `tool_calls_call_id_key`：`call_id`
- 唯一键 `tool_calls_event_seq_key`：`event_seq`

#### `runtime.tool_outputs`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `tool_output_id` | `bigint` | 否 | `nextval('runtime.tool_outputs_tool_output_id_seq'::regclass)` | 本地行标识（主键）。 |
| `event_seq` | `bigint` | 否 |  | 外键 → `runtime.events.event_seq` |
| `call_id` | `text` | 否 |  | 潜在的本地引用；可通过领域上下文在 `runtime.workflow_agent_calls.call_id` 中解析。 |
| `status` | `text` | 否 |  |  |
| `created_at` | `timestamp with time zone` | 否 | `now()` |  |
| `output_text` | `text` | 是 |  |  |
| `error_text` | `text` | 是 |  |  |

键与关系：

- 外键 `tool_outputs_event_seq_fkey`：`event_seq` → `runtime.events`（`event_seq`）
- 主键 `tool_outputs_pkey`：`tool_output_id`
- 唯一键 `tool_outputs_event_seq_key`：`event_seq`

#### `runtime.widgets`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `widget_id` | `text` | 否 |  | 本地行标识（主键）。 |
| `kind` | `text` | 否 |  |  |
| `data_json` | `text` | 否 |  |  |
| `display_text` | `text` | 是 |  |  |
| `state_bundle_json` | `text` | 否 | `'{}'::text` |  |
| `state_version` | `bigint` | 否 | `0` |  |
| `state_updated_at_ms` | `bigint` | 是 |  |  |
| `created_at_ms` | `bigint` | 否 |  |  |
| `updated_at_ms` | `bigint` | 否 |  |  |

键与关系：

- 主键 `widgets_pkey`：`widget_id`

#### `runtime.work_items`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `work_id` | `text` | 否 |  | 本地行标识（主键）。 |
| `root_work_id` | `text` | 是 |  | 外键 → `runtime.work_items.work_id` |
| `parent_work_id` | `text` | 是 |  | 外键 → `runtime.work_items.work_id` |
| `request_id` | `text` | 是 |  | 外键 → `runtime.requests.request_id` |
| `root_session_id` | `text` | 是 |  | 软本地引用 → `agent.sessions.session_id`。 |
| `session_id` | `text` | 是 |  | 软本地引用 → `agent.sessions.session_id`。 |
| `agent_id` | `text` | 是 |  | 软本地引用 → `agent.agents.agent_id`。 |
| `root_message_id` | `text` | 是 |  | 软本地引用 → `runtime.messages.message_id`。 |
| `stream_owner_message_id` | `text` | 是 |  | 软本地引用 → `runtime.messages.message_id`。 |
| `source` | `text` | 是 |  |  |
| `request_origin` | `text` | 是 |  |  |
| `work_class` | `text` | 否 |  |  |
| `transcript_surface` | `text` | 是 |  |  |
| `state` | `text` | 否 |  |  |
| `phase` | `text` | 是 |  |  |
| `subphase` | `text` | 是 |  |  |
| `priority` | `integer` | 否 | `0` |  |
| `preemptibility` | `text` | 否 |  |  |
| `deadline_at` | `timestamp with time zone` | 是 |  |  |
| `heartbeat_at` | `timestamp with time zone` | 否 | `now()` |  |
| `lease_owner` | `text` | 是 |  |  |
| `retry_policy_json` | `text` | 是 |  |  |
| `metadata_json` | `text` | 否 | `'{}'::text` |  |
| `terminal_reason` | `text` | 是 |  |  |
| `terminal_detail` | `text` | 是 |  |  |
| `diagnostic_json` | `text` | 是 |  |  |
| `created_at` | `timestamp with time zone` | 否 | `now()` |  |
| `updated_at` | `timestamp with time zone` | 否 | `now()` |  |
| `terminalized_at` | `timestamp with time zone` | 是 |  |  |

键与关系：

- 外键 `work_items_parent_work_id_fkey`: `parent_work_id` → `runtime.work_items` (`work_id`)
- 外键 `work_items_request_id_fkey`: `request_id` → `runtime.requests` (`request_id`)
- 外键 `work_items_root_work_id_fkey`: `root_work_id` → `runtime.work_items` (`work_id`)
- 主键 `work_items_pkey`: `work_id`

#### `runtime.workflow_agent_calls`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `call_id` | `text` | 否 |  | 本地行标识（主键）。 |
| `run_id` | `text` | 否 |  | 外键 → `runtime.workflow_runs.run_id` |
| `phase_run_id` | `text` | 是 |  | 外键 → `runtime.workflow_phase_runs.phase_run_id` |
| `replay_key` | `text` | 否 |  |  |
| `cache_key` | `text` | 否 |  |  |
| `call_ordinal` | `integer` | 否 |  |  |
| `prompt` | `text` | 否 |  |  |
| `options_json` | `jsonb` | 否 | `'{}'::jsonb` |  |
| `child_agent_id` | `text` | 是 |  | 软本地引用 → `agent.agents.agent_id`。 |
| `status` | `text` | 否 |  |  |
| `cached_from_call_id` | `text` | 是 |  | 外键 → `runtime.workflow_agent_calls.call_id` |
| `final_response` | `text` | 是 |  |  |
| `input_tokens` | `bigint` | 否 | `0` |  |
| `output_tokens` | `bigint` | 否 | `0` |  |
| `tool_call_count` | `bigint` | 否 | `0` |  |
| `duration_ms` | `bigint` | 是 |  |  |
| `created_at` | `timestamp with time zone` | 否 | `now()` |  |
| `started_at` | `timestamp with time zone` | 是 |  |  |
| `updated_at` | `timestamp with time zone` | 否 | `now()` |  |
| `completed_at` | `timestamp with time zone` | 是 |  |  |
| `error` | `text` | 是 |  |  |
| `child_message_id` | `text` | 是 |  | 软本地引用 → `runtime.messages.message_id`。 |

键与关系：

- 外键 `workflow_agent_calls_cached_from_call_id_fkey`: `cached_from_call_id` → `runtime.workflow_agent_calls` (`call_id`)
- 外键 `workflow_agent_calls_phase_run_id_fkey`: `phase_run_id` → `runtime.workflow_phase_runs` (`phase_run_id`)
- 外键 `workflow_agent_calls_run_id_fkey`: `run_id` → `runtime.workflow_runs` (`run_id`)
- 主键 `workflow_agent_calls_pkey`: `call_id`

#### `runtime.workflow_launch_occurrence_aliases`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `launch_occurrence_agent_id` | `text` | 否 |  | 本地行标识符（主键）。 |
| `launch_occurrence_message_id` | `text` | 否 |  | 本地行标识符（主键）。 |
| `launch_occurrence_tool_call_id` | `text` | 否 |  | 本地行标识符（主键）。 |
| `run_id` | `text` | 否 |  | 外键 → `runtime.workflow_runs.run_id` |
| `contract_fingerprint` | `text` | 否 |  |  |
| `created_at` | `timestamp with time zone` | 否 | `now()` |  |

键与关系：

- 外键 `workflow_launch_occurrence_aliases_run_id_fkey`: `run_id` → `runtime.workflow_runs` (`run_id`)
- 主键 `workflow_launch_occurrence_aliases_pkey`: `launch_occurrence_agent_id`, `launch_occurrence_message_id`, `launch_occurrence_tool_call_id`

#### `runtime.workflow_legacy_launch_occurrence_blocks`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `launch_occurrence_agent_id` | `text` | 否 |  | 本地行标识符（主键）。 |
| `launch_occurrence_message_id` | `text` | 否 |  | 本地行标识符（主键）。 |
| `launch_occurrence_tool_call_id` | `text` | 否 |  | 本地行标识符（主键）。 |
| `legacy_run_id` | `text` | 否 |  | 外键 → `runtime.workflow_runs.run_id` |
| `reason` | `text` | 否 |  |  |
| `created_at` | `timestamp with time zone` | 否 | `now()` |  |

键与关系：

- 外键 `workflow_legacy_launch_occurrence_blocks_legacy_run_id_fkey`: `legacy_run_id` → `runtime.workflow_runs` (`run_id`)
- 主键 `workflow_legacy_launch_occurrence_blocks_pkey`: `launch_occurrence_agent_id`, `launch_occurrence_message_id`, `launch_occurrence_tool_call_id`

#### `runtime.workflow_phase_runs`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `phase_run_id` | `text` | 否 |  | 本地行标识符（主键）。 |
| `run_id` | `text` | 否 |  | 外键 → `runtime.workflow_runs.run_id` |
| `phase_name` | `text` | 否 |  |  |
| `ordinal` | `integer` | 否 |  |  |
| `status` | `text` | 否 |  |  |
| `agent_total` | `integer` | 否 | `0` |  |
| `agent_completed` | `integer` | 否 | `0` |  |
| `input_tokens` | `bigint` | 否 | `0` |  |
| `output_tokens` | `bigint` | 否 | `0` |  |
| `started_at` | `timestamp with time zone` | 是 |  |  |
| `updated_at` | `timestamp with time zone` | 否 | `now()` |  |
| `completed_at` | `timestamp with time zone` | 是 |  |  |
| `error` | `text` | 是 |  |  |

键与关系：

- 外键 `workflow_phase_runs_run_id_fkey`: `run_id` → `runtime.workflow_runs` (`run_id`)
- 主键 `workflow_phase_runs_pkey`: `phase_run_id`
- 唯一约束 `workflow_phase_runs_unique_phase`: `run_id`, `phase_name`, `ordinal`

#### `runtime.workflow_runs`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `run_id` | `text` | 否 |  | 本地行标识符（主键）。 |
| `root_agent_id` | `text` | 否 |  | 软本地引用 → `agent.agents.agent_id`。 |
| `parent_message_id` | `text` | 是 |  | 软本地引用 → `runtime.messages.message_id`。 |
| `launching_agent_id` | `text` | 是 |  | 软本地引用 → `agent.agents.agent_id`。 |
| `launching_message_id` | `text` | 是 |  | 软本地引用 → `runtime.messages.message_id`。 |
| `launching_tool_call_id` | `text` | 是 |  | 软本地引用 → `runtime.tool_calls.tool_call_id`。 |
| `task_id` | `text` | 否 |  | 潜在的本地引用；需根据领域上下文解析为：`runtime.browser_tasks.task_id`。 |
| `resume_from_run_id` | `text` | 是 |  | 外键 → `runtime.workflow_runs.run_id` |
| `workflow_name` | `text` | 否 |  |  |
| `description` | `text` | 否 | `''::text` |  |
| `phases_json` | `jsonb` | 否 | `'[]'::jsonb` |  |
| `args_json` | `jsonb` | 是 |  |  |
| `workspace_root` | `text` | 否 |  |  |
| `script_path` | `text` | 否 |  |  |
| `script_sha256` | `text` | 否 |  |  |
| `request_trace_json` | `jsonb` | 是 |  |  |
| `executor_id` | `text` | 是 |  | 短暂的工作流租约持有者标识符；无单独的所有者表。 |
| `execution_attempt` | `integer` | 否 | `0` |  |
| `heartbeat_at` | `timestamp with time zone` | 否 | `now()` |  |
| `lease_expires_at` | `timestamp with time zone` | 是 |  |  |
| `recovery_count` | `integer` | 否 | `0` |  |
| `last_recovery_reason` | `text` | 是 |  |  |
| `last_recovered_at` | `timestamp with time zone` | 是 |  |  |
| `status` | `text` | 否 |  |  |
| `error` | `text` | 是 |  |  |
| `final_result` | `text` | 是 |  |  |
| `created_at` | `timestamp with time zone` | 否 | `now()` |  |
| `started_at` | `timestamp with time zone` | 是 |  |  |
| `updated_at` | `timestamp with time zone` | 否 | `now()` |  |
| `completed_at` | `timestamp with time zone` | 是 |  |  |
| `launch_mode` | `text` | 否 | `'sync'::text` |  |
| `terminal_handoff_transport` | `text` | 是 |  |  |
| `terminal_handoff_delivery_target` | `text` | 是 |  |  |
| `terminal_handoff_chat_context_json` | `jsonb` | 是 |  |  |
| `terminal_handoff_message_id` | `text` | 是 |  | 软本地引用 → `runtime.messages.message_id`。 |
| `terminal_handoff_delivered_at` | `timestamp with time zone` | 是 |  |  |
| `terminal_handoff_attempt_count` | `integer` | 否 | `0` |  |
| `terminal_handoff_last_attempt_at` | `timestamp with time zone` | 是 |  |  |
| `terminal_handoff_next_attempt_at` | `timestamp with time zone` | 是 |  |  |
| `terminal_handoff_last_error` | `text` | 是 |  |  |
| `launch_occurrence_agent_id` | `text` | 是 |  | 潜在的本地引用；需根据领域上下文解析为：`runtime.workflow_launch_occurrence_aliases.launch_occurrence_agent_id`、`runtime.workflow_legacy_launch_occurrence_blocks.launch_occurrence_agent_id`。 |
| `launch_occurrence_message_id` | `text` | 是 |  | 潜在的本地引用；需根据领域上下文解析为：`runtime.workflow_launch_occurrence_aliases.launch_occurrence_message_id`、`runtime.workflow_legacy_launch_occurrence_blocks.launch_occurrence_message_id`。 |
| `launch_occurrence_tool_call_id` | `text` | 是 |  | 潜在的本地引用；需根据领域上下文解析为：`runtime.workflow_launch_occurrence_aliases.launch_occurrence_tool_call_id`、`runtime.workflow_legacy_launch_occurrence_blocks.launch_occurrence_tool_call_id`。 |
| `sync_wait_deadline_at` | `timestamp with time zone` | 是 |  |  |
| `terminal_handoff_disposition` | `text` | 是 |  |  |
| `terminal_handoff_generation` | `integer` | 否 | `0` |  |
| `magi_workload_class` | `text` | 是 |  |  |

键与关系：

- 外键 `workflow_runs_resume_from_run_id_fkey`: `resume_from_run_id` → `runtime.workflow_runs`（`run_id`）
- 主键 `workflow_runs_pkey`: `run_id`
- 唯一约束 `workflow_runs_task_unique`: `task_id`

#### `runtime.writer_epoch`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `singleton` | `boolean` | 否 | `true` | 本地行标识符（主键）。 |
| `rv_epoch` | `bigint` | 否 | `0` |  |
| `updated_at` | `timestamp with time zone` | 否 | `now()` |  |

键与关系：

- 主键 `writer_epoch_pkey`: `singleton`

### `scheduler`

#### `scheduler.cron_mutations`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `mutation_seq` | `bigint` | 否 | `nextval('scheduler.cron_mutations_mutation_seq_seq'::regclass)` | 本地行标识符（主键）。 |
| `job_id` | `text` | 否 |  | 软本地外键 → `scheduler.jobs.job_id`。 |
| `action` | `text` | 否 |  |  |
| `occurred_at_ms` | `bigint` | 否 |  |  |
| `carrier_message_id` | `text` | 否 |  | 软本地外键 → `runtime.messages.message_id`。 |
| `request_id` | `text` | 否 |  | 软本地外键 → `runtime.requests.request_id`。 |
| `tool_call_id` | `text` | 否 |  | 软本地外键 → `runtime.tool_calls.tool_call_id`。 |
| `input_message_ids` | `text[]` | 否 |  | 软本地外键 → `runtime.messages.message_id`。 |
| `created_at` | `timestamp with time zone` | 否 | `now()` |  |

键与关系：

- 主键 `cron_mutations_pkey`: `mutation_seq`
- 唯一键 `cron_mutations_request_id_tool_call_id_job_id_key`: `request_id`, `tool_call_id`, `job_id`

#### `scheduler.delivery_outbox`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `delivery_key` | `text` | 否 |  | 本地行标识符（主键）。 |
| `job_id` | `text` | 否 |  | 软本地外键 → `scheduler.jobs.job_id`。 |
| `run_id` | `text` | 否 |  | 可能的本地外键；需根据领域上下文在以下表中解析：`scheduler.doctor_run_plans.run_id`、`scheduler.job_runs.run_id`、`scheduler.terminal_signals.run_id`。 |
| `payload_json` | `text` | 否 |  |  |
| `state` | `text` | 否 |  |  |
| `dispatch_boot_generation` | `text` | 是 |  |  |
| `attempt_count` | `integer` | 否 | `0` |  |
| `next_attempt_at_utc` | `bigint` | 否 |  |  |
| `last_error` | `text` | 是 |  |  |
| `created_at_utc` | `bigint` | 否 |  |  |
| `updated_at_utc` | `bigint` | 否 |  |  |
| `delivered_at_utc` | `bigint` | 是 |  |  |

键与关系：

- 主键 `delivery_outbox_pkey`: `delivery_key`

#### `scheduler.doctor_run_plans`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `run_id` | `text` | 否 |  | 外键 → `scheduler.job_runs.run_id` |
| `tasks_json` | `jsonb` | 否 |  |  |
| `created_at` | `timestamp with time zone` | 否 | `now()` |  |

键与关系：

- 外键 `doctor_run_plans_run_id_fkey`: `run_id` → `scheduler.job_runs`（`run_id`）
- 主键 `doctor_run_plans_pkey`: `run_id`

#### `scheduler.doctor_task_state`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `task_name` | `text` | 否 |  | 本地行标识符（主键）。 |
| `evidence_hash` | `text` | 否 |  |  |
| `last_success_run_id` | `text` | 否 |  | 软本地外键，指向 `scheduler.job_runs.run_id`。 |
| `last_success_scheduled_for_utc` | `bigint` | 否 |  |  |
| `updated_at` | `timestamp with time zone` | 否 | `now()` |  |

键与关系：

- 主键 `doctor_task_state_pkey`: `task_name`

#### `scheduler.events`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `scheduler_event_id` | `bigint` | 否 | `nextval('scheduler.events_scheduler_event_id_seq'::regclass)` | 本地行标识符（主键）。 |
| `job_id` | `text` | 是 |  | 软本地引用 → `scheduler.jobs.job_id`。 |
| `run_id` | `text` | 是 |  | 潜在的本地引用；需根据领域上下文解析，如：`scheduler.doctor_run_plans.run_id`、`scheduler.job_runs.run_id`、`scheduler.terminal_signals.run_id`。 |
| `event_name` | `text` | 否 |  |  |
| `detail` | `text` | 是 |  |  |
| `created_at` | `timestamp with time zone` | 否 | `now()` |  |

键与关系：

- 主键 `events_pkey`: `scheduler_event_id`

#### `scheduler.job_definitions`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `job_definition_id` | `bigint` | 否 | `nextval('scheduler.job_definitions_job_definition_id_seq'::regclass)` | 本地行标识符（主键）。 |
| `job_id` | `text` | 否 |  | 外键 → `scheduler.jobs.job_id` |
| `version` | `integer` | 否 |  |  |
| `prompt_template_id` | `bigint` | 是 |  | 提示模板标识符；无可查询的所有者表。 |
| `created_at` | `timestamp with time zone` | 否 | `now()` |  |
| `retired_at` | `timestamp with time zone` | 是 |  |  |
| `task_text` | `text` | 否 |  |  |

键与关系：

- 外键 `job_definitions_job_id_fkey`: `job_id` → `scheduler.jobs`（`job_id`）
- 主键 `job_definitions_pkey`: `job_definition_id`
- 唯一键 `job_definitions_job_id_version_key`: `job_id`, `version`

#### `scheduler.job_idea_scope`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `job_id` | `text` | 否 |  | 外键 → `scheduler.jobs.job_id` |
| `idea_provenance_json` | `text` | 否 |  |  |
| `created_at` | `timestamp with time zone` | 否 | `now()` |  |
| `updated_at` | `timestamp with time zone` | 否 | `now()` |  |

键与关系：

- 外键 `job_idea_scope_job_id_fkey`: `job_id` → `scheduler.jobs`（`job_id`）
- 主键 `job_idea_scope_pkey`: `job_id`

#### `scheduler.job_runs`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `run_id` | `text` | 否 |  | 本地行标识符（主键）。 |
| `job_id` | `text` | 否 |  | 外键 → `scheduler.jobs.job_id` |
| `job_definition_id` | `bigint` | 是 |  | 外键 → `scheduler.job_definitions.job_definition_id` |
| `request_id` | `text` | 是 |  | 外键 → `runtime.requests.request_id` |
| `scheduled_for` | `timestamp with time zone` | 否 |  |  |
| `scheduled_for_utc` | `bigint` | 否 |  |  |
| `trigger_reason` | `text` | 是 |  |  |
| `started_at` | `timestamp with time zone` | 是 |  |  |
| `started_at_utc` | `bigint` | 是 |  |  |
| `finished_at` | `timestamp with time zone` | 是 |  |  |
| `finished_at_utc` | `bigint` | 是 |  |  |
| `status` | `scheduler.run_status` | 否 |  |  |
| `result_summary` | `text` | 是 |  |  |
| `error_text` | `text` | 是 |  |  |
| `attempt` | `integer` | 否 | `1` |  |
| `worker_phase` | `text` | 是 |  |  |
| `worker_phase_attempt` | `integer` | 是 |  |  |
| `worker_agent_id` | `text` | 是 |  | 软本地引用 → `agent.agents.agent_id`。 |
| `worker_boot_generation` | `text` | 是 |  |  |
| `worker_claimed_at_utc` | `bigint` | 是 |  |  |
| `worker_execution_timed_out` | `boolean` | 否 | `false` |  |
| `worker_history_agent_id` | `text` | 是 |  | 软本地引用 → `agent.agents.agent_id`。 |
| `product_endpoint_deadline_at_ms` | `bigint` | 是 |  |  |
| `presentation_locale` | `text` | 否 | `'en-US'::text` | 本次运行生成的用户可见输出所用的不可变语言环境；并非用户、会话或设备偏好。 |
| `onboarding_tour_body` | `text` | 是 |  |  |
| `pending_terminal_result_json` | `text` | 是 |  |  |
| `worker_approval_wait_json` | `text` | 是 |  |  |
| `device_read_state_json` | `text` | 是 |  |  |
| `device_snapshot_hash` | `text` | 是 |  |  |
| `device_authority_hash` | `text` | 是 |  |  |
| `worker_timeouts_json` | `text` | 是 |  |  |

键与关系：

- 外键 `job_runs_job_definition_id_fkey`: `job_definition_id` → `scheduler.job_definitions` (`job_definition_id`)
- 外键 `job_runs_job_id_fkey`: `job_id` → `scheduler.jobs` (`job_id`)
- 外键 `job_runs_request_id_fkey`: `request_id` → `runtime.requests` (`request_id`)
- 主键 `job_runs_pkey`: `run_id`
- 唯一键 `job_runs_job_id_scheduled_for_utc_key`: `job_id`, `scheduled_for_utc`

#### `scheduler.jobs`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `job_id` | `text` | 否 |  | 本地行标识符（主键）。 |
| `is_heartbeat` | `boolean` | 否 | `false` |  |
| `source_path` | `text` | 是 |  |  |
| `source_hash` | `text` | 是 |  |  |
| `schedule_kind` | `text` | 否 |  |  |
| `enabled` | `boolean` | 否 | `true` |  |
| `schedule_expr` | `text` | 否 |  |  |
| `timezone` | `text` | 否 | `'UTC'::text` |  |
| `retry_on_failure` | `boolean` | 否 | `false` |  |
| `max_retries` | `integer` | 否 | `0` |  |
| `delivery_targets_json` | `text` | 否 | `'[]'::text` |  |
| `next_run_at` | `timestamp with time zone` | 是 |  |  |
| `next_run_at_utc` | `bigint` | 否 | `0` |  |
| `last_run_at_utc` | `bigint` | 是 |  |  |
| `last_success_at_utc` | `bigint` | 是 |  |  |
| `last_status` | `text` | 是 |  |  |
| `consecutive_failures` | `bigint` | 否 | `0` |  |
| `updated_at_utc` | `bigint` | 否 | `0` |  |
| `created_at` | `timestamp with time zone` | 否 | `now()` |  |
| `updated_at` | `timestamp with time zone` | 否 | `now()` |  |
| `blocked_reason` | `text` | 是 |  |  |
| `blocked_dependency` | `text` | 是 |  |  |
| `blocked_at_utc` | `bigint` | 是 |  |  |
| `next_blocked_probe_at_utc` | `bigint` | 是 |  |  |
| `device_authority_hash` | `text` | 是 |  |  |
| `device_read_checkpoints_json` | `text` | 否 | `'{}'::text` |  |

键与关系：

- 主键 `jobs_pkey`: `job_id`

#### `scheduler.scheduled_resume_registrations`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `resume_at_utc` | `bigint` | 否 |  | 本地行标识（主键）。 |
| `schedule_id` | `text` | 是 |  | 主调度器注册标识；无对应的Muse PostgreSQL所有者表。 |
| `dispatch_at_ms` | `bigint` | 是 |  |  |
| `acknowledgement_attempts` | `integer` | 否 | `0` |  |
| `created_at` | `timestamp with time zone` | 否 | `now()` |  |
| `job_name` | `text` | 是 |  |  |
| `shadow` | `boolean` | 否 | `false` |  |
| `registration_attempts` | `integer` | 否 | `0` |  |
| `acknowledgement_next_attempt_at_ms` | `bigint` | 否 | `0` |  |
| `environment_id` | `text` | 是 |  | 不透明的关联标识；未声明本地表关系。 |

键与关系：

- 主键 `scheduled_resume_registrations_pkey`: `resume_at_utc`

#### `scheduler.scheduled_resume_state`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `singleton` | `boolean` | 否 | `true` | 本地行标识（主键）。 |
| `next_resume_at_utc` | `bigint` | 是 |  |  |
| `registered_dispatch_at_ms` | `bigint` | 是 |  |  |
| `projection_complete` | `boolean` | 否 | `true` |  |
| `registration_resume_at_utc` | `bigint` | 是 |  |  |
| `registered_schedule_id` | `text` | 是 |  | 来自 `scheduler.scheduled_resume_registrations.schedule_id` 的主调度器注册标识镜像。 |
| `next_resume_job_name` | `text` | 是 |  |  |

键与关系：

- 主键 `scheduled_resume_state_pkey`: `singleton`

#### `scheduler.terminal_signals`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `run_id` | `text` | 否 |  | 本地行标识（主键）。 |
| `signal_type` | `text` | 否 |  |  |
| `message` | `text` | 是 |  |  |
| `producer_request_trace_json` | `text` | 是 |  |  |
| `created_at` | `timestamp with time zone` | 否 | `now()` |  |
| `created_at_utc` | `bigint` | 否 | `0` |  |
| `producer_agent_id` | `text` | 是 |  | 软本地引用 → `agent.agents.agent_id`。 |
| `producer_phase_attempt` | `integer` | 是 |  |  |
| `producer_boot_generation` | `text` | 是 |  |  |
| `blocked_reason` | `text` | 是 |  |  |
| `blocked_dependency` | `text` | 是 |  |  |

键与关系：

- 主键 `terminal_signals_pkey`: `run_id`

### `self_improvement`

#### `self_improvement.backfill_day_runs`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `backfill_day_run_id` | `text` | 否 |  | 本地行标识（主键）。 |
| `objective_id` | `text` | 否 |  | 潜在的本地引用；可通过领域上下文在以下表中解析：`self_improvement.handoff_dedupe.objective_id`、`self_improvement.objective_markers.objective_id`、`self_improvement.objective_state.objective_id`。 |
| `day` | `date` | 否 |  |  |
| `run_id` | `text` | 否 |  | 潜在的本地引用；可通过领域上下文在 `self_improvement.runs.run_id` 中解析。 |
| `request_id` | `text` | 否 |  | 软本地引用 → `runtime.requests.request_id`。 |
| `replay_hash` | `text` | 否 |  |  |
| `status` | `text` | 否 | `'pending'::text` |  |
| `diagnostics` | `jsonb` | 否 | `'{}'::jsonb` |  |
| `started_at` | `timestamp with time zone` | 是 |  |  |
| `finished_at` | `timestamp with time zone` | 是 |  |  |
| `created_at` | `timestamp with time zone` | 否 | `now()` |  |
| `updated_at` | `timestamp with time zone` | 否 | `now()` |  |

键与关系：

- 主键 `backfill_day_runs_pkey`: `backfill_day_run_id`

#### `self_improvement.calculation_records`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `replay_id` | `text` | 否 |  | 本地行标识符（主键）。 |
| `card_id` | `text` | 否 |  | 由代码定义的自我提升测量卡片键；无所有者表。 |
| `objective_id` | `text` | 否 |  | 可能的本地引用；需根据领域上下文在以下表中解析：`self_improvement.handoff_dedupe.objective_id`、`self_improvement.objective_markers.objective_id`、`self_improvement.objective_state.objective_id`。 |
| `config_hash` | `text` | 否 |  |  |
| `code_version` | `text` | 否 |  |  |
| `source_handles` | `jsonb` | 否 | `'[]'::jsonb` |  |
| `normalized_inputs` | `jsonb` | 否 | `'{}'::jsonb` |  |
| `formulas` | `jsonb` | 否 | `'{}'::jsonb` |  |
| `outputs` | `jsonb` | 否 | `'{}'::jsonb` |  |
| `created_at` | `timestamp with time zone` | 否 | `now()` |  |

键与关系：

- 主键 `calculation_records_pkey`: `replay_id`

#### `self_improvement.calibration_records`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `calibration_id` | `text` | 否 |  | 本地行标识符（主键）。 |
| `card_id` | `text` | 否 |  | 由代码定义的自我提升测量卡片键；无所有者表。 |
| `objective_id` | `text` | 否 |  | 可能的本地引用；需根据领域上下文在以下表中解析：`self_improvement.handoff_dedupe.objective_id`、`self_improvement.objective_markers.objective_id`、`self_improvement.objective_state.objective_id`。 |
| `change_id` | `text` | 否 |  | 由目标提供的已测量变化标识符；无所有者表。 |
| `classification` | `text` | 否 |  |  |
| `window_start` | `timestamp with time zone` | 是 |  |  |
| `window_end` | `timestamp with time zone` | 是 |  |  |
| `source_handles` | `jsonb` | 否 | `'[]'::jsonb` |  |
| `diagnostics` | `jsonb` | 否 | `'{}'::jsonb` |  |
| `created_at` | `timestamp with time zone` | 否 | `now()` |  |

键与关系：

- 主键 `calibration_records_pkey`: `calibration_id`

#### `self_improvement.connector_read_audit`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `audit_id` | `bigint` | 否 | `nextval('self_improvement.connector_read_audit_audit_id_seq'::regclass)` | 本地行标识符（主键）。 |
| `connector_id` | `text` | 是 |  | 由 authd 拥有的连接器身份；无 Muse PostgreSQL 所有者表。 |
| `device_node_id` | `text` | 是 |  | 软本地引用 → `device.nodes.node_id`。 |
| `method_key` | `text` | 否 |  |  |
| `purpose` | `text` | 否 |  |  |
| `request_id` | `text` | 否 |  | 软本地引用 → `runtime.requests.request_id`。 |
| `request_origin` | `text` | 否 |  |  |
| `sensitivity` | `text` | 否 |  |  |
| `source_handle` | `text` | 是 |  |  |
| `metadata` | `jsonb` | 否 | `'{}'::jsonb` |  |
| `created_at` | `timestamp with time zone` | 否 | `now()` |  |

键与关系：

- 主键 `connector_read_audit_pkey`: `audit_id`

#### `self_improvement.conversation_follow_up_attempts`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `occurrence` | `text` | 否 |  | 本地行标识（主键）。 |
| `delivery_submission_id` | `text` | 否 |  | 软本地外键 → `agent.message_mailbox.submission_id`。 |
| `selector_decision` | `text` | 是 |  |  |
| `selector_message` | `text` | 是 |  |  |
| `feedback_note` | `text` | 是 |  |  |
| `selected_at` | `timestamp with time zone` | 是 |  |  |
| `state` | `text` | 否 | `'pending'::text` |  |
| `disposition_reason` | `text` | 是 |  |  |
| `surfaced_at` | `timestamp with time zone` | 是 |  |  |
| `terminal_at` | `timestamp with time zone` | 是 |  |  |
| `created_at` | `timestamp with time zone` | 否 | `now()` |  |
| `updated_at` | `timestamp with time zone` | 否 | `now()` |  |
| `source_root_agent_id` | `text` | 是 |  | 软本地外键 → `agent.agents.agent_id`。 |
| `selector_priority` | `smallint` | 是 |  |  |
| `chat_id` | `text` | 是 |  | 软本地外键 → `chat.chats.chat_id`。 |
| `chat_binding_epoch` | `bigint` | 是 |  |  |

键与关系：

- 主键 `conversation_follow_up_attempts_pkey`: `occurrence`

#### `self_improvement.handoff_dedupe`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `objective_id` | `text` | 否 |  | 本地行标识（主键）。 |
| `content_hash` | `text` | 否 |  | 本地行标识（主键）。 |
| `run_id` | `text` | 否 |  | 潜在的本地引用；可通过领域上下文解析为：`self_improvement.runs.run_id`。 |
| `emitted_at` | `timestamp with time zone` | 否 | `now()` |  |

键与关系：

- 主键 `handoff_dedupe_pkey`: `objective_id`, `content_hash`

#### `self_improvement.learning_adoption_events`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `id` | `bigint` | 否 |  | 本地行标识（主键）。 |
| `learning_id` | `text` | 否 |  | 学习标识；无可查询的所有者表。 |
| `objective_id` | `text` | 否 |  | 潜在的本地引用；可通过领域上下文解析为：`self_improvement.handoff_dedupe.objective_id`、`self_improvement.objective_markers.objective_id`、`self_improvement.objective_state.objective_id`。 |
| `run_id` | `text` | 否 |  | 潜在的本地引用；可通过领域上下文解析为：`self_improvement.runs.run_id`。 |
| `outcome` | `text` | 否 |  |  |
| `detail` | `jsonb` | 否 | `'{}'::jsonb` |  |
| `created_at` | `timestamp with time zone` | 否 | `now()` |  |

键与关系：

- 主键 `learning_adoption_events_pkey`: `id`

#### `self_improvement.leases`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `lease_key` | `text` | 否 |  | 本地行标识（主键）。 |
| `owner` | `text` | 否 |  |  |
| `objective_id` | `text` | 是 |  | 潜在的本地引用；可通过领域上下文解析为：`self_improvement.handoff_dedupe.objective_id`、`self_improvement.objective_markers.objective_id`、`self_improvement.objective_state.objective_id`。 |
| `run_id` | `text` | 是 |  | 外键 → `self_improvement.runs.run_id`。 |
| `acquired_at` | `timestamp with time zone` | 否 | `now()` |  |
| `expires_at` | `timestamp with time zone` | 否 |  |  |
| `heartbeat_at` | `timestamp with time zone` | 否 | `now()` |  |
| `metadata` | `jsonb` | 否 | `'{}'::jsonb` |  |

键与关系：

- 外键 `leases_run_id_fkey`: `run_id` → `self_improvement.runs`（`run_id`）。  
- 主键 `leases_pkey`: `lease_key`

#### `self_improvement.objective_markers`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `objective_id` | `text` | 否 |  | 本地行标识（主键）。 |
| `marker` | `text` | 否 |  | 本地行标识（主键）。 |
| `set_at` | `timestamp with time zone` | 否 | `now()` |  |

键与关系：

- 主键 `objective_markers_pkey`: `objective_id`, `marker`

#### `self_improvement.objective_state`| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `objective_id` | `text` | 否 |  | 本地行标识符（主键）。 |
| `current_file_hash` | `text` | 是 |  |  |
| `projection_hash` | `text` | 是 |  |  |
| `status` | `text` | 否 |  |  |
| `last_run_utc` | `timestamp with time zone` | 是 |  |  |
| `state_json` | `jsonb` | 否 | `'{}'::jsonb` |  |
| `updated_at` | `timestamp with time zone` | 否 | `now()` |  |

键与关系：

- 主键 `objective_state_pkey`: `objective_id`

#### `self_improvement.relationship_briefs`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `brief_id` | `text` | 否 |  | 本地行标识符（主键）。 |
| `title` | `text` | 否 |  |  |
| `body_html` | `text` | 否 |  |  |
| `anchored_idea_ids` | `jsonb` | 否 | `'[]'::jsonb` | 软本地引用 → `ideas.ideas.idea_id`。 |
| `request_id` | `text` | 否 |  | 软本地引用 → `runtime.requests.request_id`。 |
| `created_at` | `timestamp with time zone` | 否 | `now()` |  |
| `last_opened_at` | `timestamp with time zone` | 是 |  |  |

键与关系：

- 主键 `relationship_briefs_pkey`: `brief_id`

#### `self_improvement.runs`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `run_id` | `text` | 否 |  | 本地行标识符（主键）。 |
| `objective_id` | `text` | 否 |  | 潜在的本地引用；可通过领域上下文解析为：`self_improvement.handoff_dedupe.objective_id`、`self_improvement.objective_markers.objective_id`、`self_improvement.objective_state.objective_id`。 |
| `state` | `text` | 否 |  |  |
| `request_id` | `text` | 否 |  | 软本地引用 → `runtime.requests.request_id`。 |
| `request_origin` | `text` | 否 |  |  |
| `phase` | `text` | 是 |  |  |
| `subphase` | `text` | 是 |  |  |
| `diagnostics` | `jsonb` | 否 | `'{}'::jsonb` |  |
| `queued_at` | `timestamp with time zone` | 否 | `now()` |  |
| `started_at` | `timestamp with time zone` | 是 |  |  |
| `finished_at` | `timestamp with time zone` | 是 |  |  |
| `created_at` | `timestamp with time zone` | 否 | `now()` |  |
| `updated_at` | `timestamp with time zone` | 否 | `now()` |  |
| `run_clock` | `timestamp with time zone` | 否 |  |  |
| `available_at` | `timestamp with time zone` | 否 | `now()` |  |
| `recovery_phase_version` | `integer` | 是 |  |  |
| `recovery_admitted_at` | `timestamp with time zone` | 是 |  |  |
| `recovery_admission_boot_id` | `text` | 是 |  | 守护进程启动时的生成标识符；无 Muse PostgreSQL 所有者表。 |

键与关系：

- 主键 `runs_pkey`: `run_id`

### `shell`

#### `shell.user_state`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `source` | `text` | 否 |  | 本地行标识符（主键）。 |
| `item_key` | `text` | 否 |  | 本地行标识符（主键）。 |
| `is_favorite` | `boolean` | 否 | `false` |  |
| `accessed_at_ms` | `bigint` | 是 |  |  |
| `frequency_score` | `double precision` | 是 |  |  |
| `favorite_order` | `double precision` | 是 |  |  |
| `display_name` | `text` | 是 |  |  |
| `icon` | `text` | 是 |  |  |
| `updated_at_ms` | `bigint` | 否 | `((EXTRACT(epoch FROM now()) * (1000)::numeric))::bigint` |  |
| `last_opened_at_ms` | `bigint` | 是 |  |  |

键与关系：

- 主键 `user_state_pkey`: `source`, `item_key`

### `spaces`

#### `spaces.action_arguments`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `action_argument_id` | `bigint` | 否 | `nextval('spaces.action_arguments_action_argument_id_seq'::regclass)` | 本地行标识符（主键）。 |
| `invocation_id` | `text` | 否 |  | 外键 → `spaces.action_invocations.invocation_id` |
| `argument_name` | `text` | 否 |  |  |
| `scalar_value` | `text` | 是 |  |  |
| `text_content` | `text` | 是 |  |  |

键与关系：- 外键 `action_arguments_invocation_id_fkey`: `invocation_id` → `spaces.action_invocations`（`invocation_id`）
- 主键 `action_arguments_pkey`: `action_argument_id`
- 唯一键 `action_arguments_invocation_id_argument_name_key`: `invocation_id`, `argument_name`

#### `spaces.action_invocations`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `global_seq` | `bigint` | 否 | `nextval('spaces.action_invocations_global_seq_seq'::regclass)` |  |
| `invocation_id` | `text` | 否 |  | 本地行标识符（主键）。 |
| `space_slug` | `text` | 否 |  |  |
| `space_display_name` | `text` | 是 |  |  |
| `action` | `text` | 否 |  |  |
| `transport` | `text` | 否 | `'unknown'::text` |  |
| `request_id` | `text` | 是 |  | 外键 → `runtime.requests.request_id` |
| `status` | `text` | 否 |  |  |
| `invoked_at_text` | `text` | 是 |  |  |
| `invoked_at_unix_ms` | `bigint` | 否 | `0` |  |
| `invoked_at` | `timestamp with time zone` | 否 |  |  |
| `settled_at_text` | `text` | 是 |  |  |
| `settled_at_unix_ms` | `bigint` | 是 |  |  |
| `duration_ms` | `bigint` | 是 |  |  |
| `args_preview` | `text` | 是 |  |  |
| `result_preview` | `text` | 是 |  |  |
| `error` | `text` | 是 |  |  |
| `stream_protocol_messages` | `bigint` | 是 |  |  |
| `stream_data_messages` | `bigint` | 是 |  |  |
| `finished_at` | `timestamp with time zone` | 是 |  |  |
| `source_kind` | `text` | 是 |  |  |
| `source_ref` | `text` | 是 |  |  |
| `trigger_ref` | `text` | 是 |  |  |

键与关系：

- 外键 `action_invocations_request_id_fkey`: `request_id` → `runtime.requests`（`request_id`）
- 主键 `action_invocations_pkey`: `invocation_id`
- 唯一键 `action_invocations_global_seq_key`: `global_seq`

#### `spaces.action_results`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `invocation_id` | `text` | 否 |  | 外键 → `spaces.action_invocations.invocation_id` |
| `result_text` | `text` | 是 |  |  |
| `error_text` | `text` | 是 |  |  |

键与关系：

- 外键 `action_results_invocation_id_fkey`: `invocation_id` → `spaces.action_invocations`（`invocation_id`）
- 主键 `action_results_pkey`: `invocation_id`

#### `spaces.backfill_markers`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `marker` | `text` | 否 |  | 本地行标识符（主键）。 |
| `completed_at` | `timestamp with time zone` | 否 | `now()` |  |

键与关系：

- 主键 `backfill_markers_pkey`: `marker`

#### `spaces.file_artifact_identities`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `slug` | `text` | 否 |  | 本地行标识符（主键）。 |
| `artifact_id` | `uuid` | 否 | `gen_random_uuid()` | 本行拥有的稳定本地文件标识符；`slug`为主键。 |
| `created_at` | `timestamp with time zone` | 否 | `now()` |  |

键与关系：

- 主键 `file_artifact_identities_pkey`: `slug`

#### `spaces.proposals`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `proposal_id` | `text` | 否 |  | 本地行标识符（主键）。 |
| `root_session_id` | `text` | 否 |  | 软性本地引用 → `agent.sessions.session_id`。 |
| `space_slug` | `text` | 否 |  |  |
| `proposed_name` | `text` | 否 |  |  |
| `params_json` | `text` | 否 |  |  |
| `confirmed_at_text` | `text` | 是 |  |  |
| `confirmed_at` | `timestamp with time zone` | 是 |  |  |
| `created_at_text` | `text` | 否 |  |  |
| `created_at` | `timestamp with time zone` | 否 | `now()` |  |

键与关系：

- 主键 `proposals_pkey`: `proposal_id`

#### `spaces.shares`| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `space_slug` | `text` | 否 |  | 外键 → `spaces.spaces.space_slug` |
| `share_type` | `text` | 否 |  |  |
| `shortcode` | `text` | 否 |  |  |
| `is_active` | `boolean` | 否 | `true` |  |
| `share_id` | `text` | 是 |  | 艺术品发布服务的共享标识符；无本地所有者表。 |
| `cloudflare_deploy_status` | `text` | 是 |  |  |
| `cloudflare_public_url` | `text` | 是 |  |  |
| `cloudflare_actions_url` | `text` | 是 |  |  |
| `created_at` | `timestamp with time zone` | 否 | `now()` |  |
| `updated_at` | `timestamp with time zone` | 否 | `now()` |  |
| `cloudflare_storage_mode` | `text` | 否 | `'d1_r2'::text` |  |
| `published_manifest_sha256` | `text` | 是 |  |  |

键和关系：

- 外键 `shares_space_slug_fkey`: `space_slug` → `spaces.spaces`（`space_slug`）
- 主键 `shares_pkey`: `space_slug`
- 唯一键 `shares_shortcode_key`: `shortcode`

#### `spaces.spaces`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `space_slug` | `text` | 否 |  | 本地行标识符（主键）。 |
| `display_name` | `text` | 否 |  |  |
| `space_root_path` | `text` | 是 |  |  |
| `db_path` | `text` | 是 |  |  |
| `force_order` | `bigint` | 是 |  |  |
| `session_id` | `text` | 是 |  | 软本地外键 → `agent.sessions.session_id`。 |
| `construction_status` | `text` | 是 |  |  |
| `construction_updated_at_text` | `text` | 是 |  |  |
| `created_at_text` | `text` | 是 |  |  |
| `updated_at_text` | `text` | 是 |  |  |
| `created_at` | `timestamp with time zone` | 否 | `now()` |  |
| `updated_at` | `timestamp with time zone` | 否 | `now()` |  |
| `archived_at` | `timestamp with time zone` | 是 |  |  |
| `current_build_id` | `text` | 是 |  | 当前存储于该记录中的构件-构建关联ID；构建工作区位于磁盘上，无单独的PostgreSQL所有者表。 |
| `created_from_proposal_id` | `text` | 是 |  | 软本地外键 → `spaces.proposals.proposal_id`。 |
| `space_id` | `uuid` | 否 | `gen_random_uuid()` | 软本地外键 → `spaces.spaces.space_id`。 |
| `source` | `text` | 否 | `'local'::text` |  |
| `shortcode` | `text` | 是 |  |  |
| `source_url` | `text` | 是 |  |  |
| `has_server_actions` | `boolean` | 否 | `true` |  |
| `cloudflare_share_manifest_sha256` | `text` | 是 |  |  |
| `content_share_allowed` | `boolean` | 是 |  |  |
| `content_share_review_sha256` | `text` | 是 |  |  |
| `declared_capabilities_json` | `jsonb` | 否 | `'{"connectors": [], "schema_version": 1, "public_web_read": false}'::jsonb` |  |
| `capability_manifest_revision` | `bigint` | 否 | `0` |  |
| `builder_provenance_pending_build_id` | `text` | 是 |  | 不透明关联标识符；未声明本地表关系。当前尚未完成构建者溯源捕获的构建ID。 |
| `builder_provenance_lost_build_id` | `text` | 是 |  | 不透明关联标识符；未声明本地表关系。已成功服务的构建ID，其构建者溯源已丢失；在后续编辑尝试中保留，直至有更新的构建记录证据。 |
| `builder_provenance_build_id` | `text` | 是 |  | 不透明关联标识符；未声明本地表关系。其限定的构建工具证据投影存储于此行的构建ID。 |
| `builder_provenance_evidence_jsonl` | `text` | 是 |  | 守护进程拥有的、受限范围内的JSONL格式文件，记录具备溯源能力的构建工具调用及输出。 |
| `share_disclosure_review` | `jsonb` | 是 |  |  |
| `is_promoted` | `boolean` | 否 | `false` | UI位置：false表示“库”，true表示侧边栏中的“空间”。与运行时类型及置顶状态无关。 |
| `artifact_audit_review` | `jsonb` | 是 |  | 运行时拥有的当前构建的构件审核记录：限定范围内的近期公开聊天记录（授权该构件）、审核覆盖的身份信息，以及保留的批评意见。 |
| `saved_at_ms` | `bigint` | 是 |  |  |
| `publisher_id` | `text` | 是 |  | 不透明关联标识符；未声明本地表关系。 |
| `publisher_name` | `text` | 是 |  |  |
| `saved_record_id` | `text` | 是 |  | 不透明关联标识符；未声明本地表关系。 |
| `saved_available` | `boolean` | 是 |  |  |
| `saved_target_shortcode` | `text` | 是 |  |  |
| `auto_publish_enabled` | `boolean` | 否 | `false` |  |

键与关系：

- 主键 `spaces_pkey`: `space_slug`

#### `spaces.user_state`

| 列名 | 类型 | 可空 | 默认值 | 键/标识含义 |
|---|---|---:|---|---|
| `space_slug` | `text` | 否 |  | 本地行标识（主键）。 |
| `is_favorite` | `boolean` | 否 | `false` |  |
| `accessed_at_ms` | `bigint` | 是 |  |  |
| `frequency_score` | `double precision` | 是 |  |  |
| `favorite_order` | `double precision` | 是 |  |  |
| `last_accessed_at` | `timestamp with time zone` | 是 |  |  |
| `pinned_at` | `timestamp with time zone` | 是 |  |  |
| `last_opened_at_ms` | `bigint` | 是 |  |  |
| `sharing_state` | `text` | 是 |  | 共享给该用户的 Space 的接收方选择：pending（待处理）、accepted（已接受）或ignored（已忽略）。不授予任何访问权限。NULL 表示无收到的共享记录；仅包含共享相关字段的行不属于导航历史。该选择在从“已保存”Space 中移除后仍会保留。 |
| `sharing_revision` | `uuid` | 是 |  | 接收方共享选择的不透明并发令牌；不是本地或全局标识。 |
| `sharing_saved_shortcode` | `text` | 是 |  | 在共享被接受时记录的历史“已保存”Space 短码。在“已保存”引用存在期间与 spaces.spaces.space_slug 匹配；但不能证明当前是否为已保存成员或是否位于侧边栏中。 |
| `sharing_updated_at_ms` | `bigint` | 是 |  | 最近一次接收方共享选择变更的 Unix 时间戳（毫秒）。 |

键和关系：

- 主键 `user_state_pkey`：`space_slug`
