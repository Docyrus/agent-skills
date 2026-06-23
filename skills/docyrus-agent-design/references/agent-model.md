# Agent Model & Sub-Resources

Reference for the custom-agent data model and its sub-resources, as exposed by `docyrus agent`. Concepts live in **docyrus-platform** → `references/ai-capabilities.md`; exhaustive flags via `docyrus agent <sub> --help` (command index: **docyrus-cli-app**). Source of truth: `apps/cli/src/commands/agentCommands.ts`, `apps/api/src/dev/dto/agent/*`, `libs/shared/src/types/KyselyCodegen.ts`.

## Table of Contents

1. [The one required field](#the-one-required-field)
2. [Agent fields by purpose](#agent-fields-by-purpose)
3. [Enum values](#enum-values)
4. [Server defaults](#server-defaults)
5. [Sub-resources](#sub-resources)
6. [Casing & gotchas](#casing--gotchas)

## The one required field

`skill_name` (CLI flag `--skillName`) is the **only** field `agent create` requires. Everything else is optional and either defaulted server-side or left null. Keep create calls minimal and add what the use case needs.

## Agent fields by purpose

CLI flags are camelCase; API keys are snake_case (shown as `--flag → api_key`). Plain scalars pass through; `list` = CSV→array; `json` = parsed JSON (pass via `--data` if it has no flag).

**Identity**
`--name → name`, `--agentName → agent_name`, `--skillName → skill_name` (required), `--description → description`, `--category → category`, `--welcomeMessage → welcome_message`, `--parent → parent`, `--ownership → ownership` (enum), `--tenantAppId → tenant_app_id`.

**Shape / behavior**
`--mode → mode` (e.g. `conversation`), `--isAgent → is_agent`, `--isAssistant → is_assistant`, `--isSkill → is_skill`, `--hasWorkflow → has_workflow`, `--multipleDeployment → multiple_deployment`, `--jsonOutput → json_output`, `--status → status` (int), `--developmentStatus → development_status` (int), `--sortOrder → sort_order`, `--instructions → instructions`.

**Models / inference**
`--defaultAiModelId → default_ai_model_id`, `--backupAiModelId → backup_ai_model_id`, `--cotAiModelId → cot_ai_model_id`, `--defaultReasoningLevel → default_reasoning_level`, `--temperature → temperature`, `--maxTokens → max_tokens`, `--workDataSourceId → work_data_source_id`, `--ownerProductId → owner_product_id` (list).

**Capability toggles** (gate the matching sub-resource)
`--supportTools → support_tools`, `--supportDataSources → support_data_sources`, `--supportFiles → support_files`, `--supportKnowledgeBase → support_knowledge_base`, `--supportWebSearch → support_web_search`.

**Schemas / UX** (JSON, via flag or `--data`)
`--instructionSchema → instruction_schema`, `--inputFormSchema → input_form_schema`, `--outputRenderSchema → output_render_schema`, `--standardSuggestions → standard_suggestions` (list), `--supportedFileFormats → supported_file_formats` (list), `--memoryOptions → memory_options` (json), `--helpDocs → help_docs` (json), `--archived → archived` (update only).

**No-flag fields (via `--data`/`--from-file`, all snake_case)**
Render/output: `output_render_type` (enum), `output_render_template`, `output_work_type` (enum), `strict_json_schema_mode`, `output_json_schema`, `input_json_schema`, `prompt_template`, `thread_title_template`.
Memory: `memory_write_enabled`, `memory_token_limit`, `memory_access_levels[]` (enum), `memory_filter_tools[]`, `custom_memory_extractor_prompt`.
Context compaction: `compaction_enabled`, `compaction_model_id`, `compaction_summary_prompt`, `compaction_token_threshold`, `context_clear_thinking`, `context_clear_tool_uses`.
Multi-agent: `multi_agent_strategy` (enum), `selection_rules`.
Features/flags: `auto_generate_suggestions`, `document_search`, `support_deep_research`, `support_webhook`, `support_emailhook`, `support_multiple_models`, `support_additional_instructions`, `support_work_canvas`, `feature_projects`, `feature_tasks`, `feature_tasks_recurring`, `feature_works`, `is_base_assistant`, `show_prompt_input`, `prompt_optimization_choice`, `optimization_prompt`, `custom_suggestion_generation_prompt`.

## Enum values

Validated server-side — use exactly these:

- **`ownership`**: `APP` | `CUSTOM` | `PRODUCT` | `SYSTEM` | `USER`. (Flag help wrongly says `SYSTEM|APP|TENANT`.)
- **`multi_agent_strategy`**: `manual` | `orchestrator` | `router`.
- **`output_render_type`**: `adaptive_card` | `auto_layout` | `html` | `markdown` | `none` | `schema`.
- **`output_work_type`**: `add_in` | `agent` | `app` | `chart` | `code` | `custom_query` | `custom_query_sql` | `data` | `data_source` | `deep_research` | `diagram` | `docx` | `docyment` | `formula` | `html` | `html_template` | `image` | `infographic` | `mermaid_diagram` | `pptx` | `record` | `text` | `view` | `xlsx`.
- **`memory_access_levels[]`**: `project` | `team` | `team_agent` | `tenant` | `tenant_agent` | `user` | `user_agent`.
- **connection `connection_type`**: `forward` | `subagent`.
- **workflow-step `type`**: `action` | `agent` | `formula` | `note` | `prompt` | `tool_call`; **`loop`**: `foreach` | `until` | `while`.

Model ids (`default_ai_model_id`, etc.), tool ids, data-source ids are **free-form FK strings**, not enums — resolve real ids.

## Server defaults

Applied when omitted on create: `is_agent`→`true`, `is_assistant`→`false`, `is_skill`→`false`, `mode`→`"conversation"`, `multi_agent_strategy`→`"orchestrator"`, `output_render_type`→`"auto_layout"`, `ownership`→`"CUSTOM"`, `prompt_optimization_choice`→`"default"`, `status`→`1`, `memory_token_limit`→`4000`, `archived`→`false`, most other booleans→`false`. The owning app must exist (active, in-tenant) or create returns 404.

## Sub-resources

Each is a nested group: `docyrus agent <group> <op>`. Ops are `list`/`get`/`create`/`update`/`delete` unless noted. Always pass app + `--agentId`; the row id is `--id`. CLI flags camelCase → snake_case API keys.

| Group | Create-required field(s) | Other create/update flags | Notes |
|---|---|---|---|
| `models` | — | `--coreAiModelId`, `--coreAiProviderId` | attach a model option |
| `tools` | `--coreAiToolId` (createOnly) | `--defaultParams` (json), `--tenantConnectionId`, `--tenantConnectionAccountId` | attaches an existing app-scoped tool (author it with **docyrus-app-ai-tools**) |
| `data-sources` | `--tenantDataSourceId` (createOnly) | `--privilege` | grant the agent a data source |
| `docs` | `--title` | `--content`, `--parent` | knowledge document |
| `mcps` | — | `--tenantMcpServerId`, `--archived` (update) | MCP server binding |
| `connections` | `--connectedAiAgentId`, `--connectionType` (`forward`\|`subagent`) | `--archived` (update) | agent-to-agent |
| `dynamic-contexts` | `--name` | `--description`, `--systemMessage`, `--systemTools` (list), `--integratedActionTools` (list) | runtime context injection |
| `workflow-steps` | `--name`, `--inputSchema`, `--outputSchema` | `--type` (enum), `--prompt`, `--condition`, `--loop`, `--parent`, `--sortOrder`, `--coreAiModelId`, `--coreAiToolId`, `--executorAgentId`, many more (json) | only meaningful if agent `--hasWorkflow` |
| `deployments` | — | `--name`, `--description`, `--instructions`, `--welcomeMessage`, `--defaultAiModelId`, `--tools` (json), `--configured`, `--ownership` | a publishable config; nested tools/data-sources below |
| `deployment-tools` | `--coreAiToolId` | `--archived` (update) | requires `--deploymentId` |
| `deployment-data-sources` | `--tenantDataSourceId` (createOnly) | `--privilege` | requires `--deploymentId` |
| `workflow-jobs` | — (read-only + `delete`) | extra op `traces` → job traces | inspection only; no create/update |

**Non-functional groups:** `tasks` and `recurring-tasks` exist in the CLI but have **no matching dev backend routes** → calls 404. Use an automation `recurrence` trigger for scheduled work instead.

## Casing & gotchas

- **CLI flags accept camelCase or kebab-case** (`--agentId` = `--agent-id`); API/DTO keys are snake_case. `--data` keys must be snake_case.
- **`--id` for the sub-resource row** (not `--toolId`/etc.). But `docyrus apps ai-tools` (a different group) uses `--toolId` — don't confuse the two.
- **`createOnly` fields** (`core_ai_tool_id` on tools, `tenant_data_source_id` on data-sources & deployment-data-sources) are rejected on `update`. **`--archived` is update-only.**
- **DTO validation lives on the server.** A missing "required on create" sub-resource field is caught by the backend (error), not by the CLI; the CLI happily sends an incomplete body. Read back to confirm.
- **Delete envelope:** CLI prints `{deleted:true,id}`; backend returns `{success:true}`.
