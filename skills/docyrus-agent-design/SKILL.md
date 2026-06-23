---
name: docyrus-agent-design
description: Design, build, validate, and test a custom Docyrus AI agent end-to-end using the `docyrus agent` CLI commands and its sub-resources. Use when the user wants to create or configure a custom AI agent (assistant/skill/workflow agent) in a dev app — set its instructions, pick its models, toggle capabilities (tools, data sources, files, knowledge base, web search), attach data sources, attach tools, add knowledge docs, wire MCP servers, connect agents (forward/subagent), add dynamic contexts, build workflow steps, or create deployments. Covers choosing the agent shape, the required vs optional config, the sub-resource attach order, and proving the agent is well-formed. Triggers on "create an AI agent", "build an assistant", "configure an agent", "give the agent a data source / tool / knowledge", "agent workflow steps", "agent deployment", "connect agents", `docyrus agent create`, `docyrus agent tools`, `docyrus agent data-sources`, `docyrus agent workflow-steps`, or any custom-agent design + validation task. For authoring the tools an agent calls, use docyrus-app-ai-tools (define with `apps ai-tools`, then attach here with `agent tools`).
---

# Docyrus Agent Design

Build a custom AI agent with `docyrus agent`, then **validate** its configuration and **test** that it's well-formed. An agent is a row in `tenant_ai_agent` (identity + model + capability config) plus a set of **sub-resources** (tools, data sources, docs, MCPs, connections, dynamic contexts, workflow steps, deployments). This skill is the design workflow; the platform's conceptual capability catalog lives in **docyrus-platform** (`references/ai-capabilities.md`), and exhaustive CLI flag tables live in **docyrus-cli-app** (`references/cli-manifest.md`).

## Workflow

1. **Confirm app + auth.** Agents are dev-app scoped.
   ```bash
   docyrus auth who --json
   docyrus apps list --json          # target appSlug / appId
   ```

2. **Design the agent.** Decide its shape (is it a conversational **assistant**, a callable **skill**, or a **workflow** agent?), its instructions, its default/backup models, and which capabilities it needs (tools? data sources? files? knowledge base? web search?). Toggle a capability **on** before attaching the matching sub-resource. See [references/agent-model.md](references/agent-model.md) to map intent → fields.

3. **Create the agent.** The only required field is `--skillName`. Set identity + model + capability toggles in the same call. See [Create](#create-an-agent).

4. **Attach sub-resources** in this order (each is its own command group, see [Sub-resources](#sub-resources)):
   - **data-sources** the agent can read/write (turn on `--supportDataSources` first),
   - **tools** the agent can call (turn on `--supportTools`; author the tool first with **docyrus-app-ai-tools**, then attach it here),
   - **docs** (knowledge; turn on `--supportKnowledgeBase`), **mcps**, **connections** (forward/subagent), **dynamic-contexts**,
   - **workflow-steps** (only if `--hasWorkflow`), then **deployments** (+ deployment-tools / deployment-data-sources).

5. **Validate** — re-read the agent and each sub-resource list; confirm config + attachments landed. See [Validate](#validate).

6. **Test** — confirm the agent is well-formed (lists cleanly, references resolve) and, where possible, exercise it. See [Test](#test).

A full worked example (assistant with a data source, an attached tool, and a knowledge doc) is in [references/workflow-examples.md](references/workflow-examples.md).

## Command cheat-sheet

Selectors: `--appId | --appSlug`, `--agentId`. Sub-resource **row** id is the generic `--id` (not `--toolId`/`--dataSourceId`). Deployment-nested groups also need `--deploymentId`. Write commands take camelCase flags **or** `--data`/`--from-file` JSON (flags merge over JSON; API keys are **snake_case**). Append `--json`.

### Create an agent

```bash
docyrus agent create --appSlug crm \
  --skillName "deal_coach" --name "Deal Coach" --agentName "deal-coach" \
  --description "Helps reps qualify and advance deals" \
  --instructions "You are a CRM assistant. Be concise. Use the deals data source to answer." \
  --isAssistant true --mode conversation \
  --supportTools true --supportDataSources true --supportKnowledgeBase true --json
# → capture data.id as AGENT_ID
```

> ⚠️ **Known backend issue (verified on env=dev).** `agent create` currently fails with `cannot insert a non-DEFAULT value into column "name"`. The dev create service (`apps/api/src/dev/app/agent.service.ts` `createAgent`) inserts `name`, but `tenant_ai_agent.name` is a generated column (`GENERATED ALWAYS AS (COALESCE(agent_name, skill_name)) STORED`) — Postgres rejects **any** explicit insert into it, regardless of flags. Until the service stops inserting `name`, agents can't be created via this endpoint. The flag/usage below is correct and will work once fixed; in the meantime, operate on agents created through another path. The flag values themselves are unaffected — Postgres derives `name` from `agent_name`/`skill_name`.

- **`--skillName` is the only required field.** Everything else is optional with server defaults (`mode`→`conversation`, `ownership`→`CUSTOM`, `status`→1, `is_agent`→true, `is_assistant`/`is_skill`→false, `memory_token_limit`→4000).
- Identity: `--name` (display), `--agentName` (internal), `--description`, `--category`, `--welcomeMessage`.
- Models: `--defaultAiModelId`, `--backupAiModelId`, `--cotAiModelId`, `--defaultReasoningLevel`, `--temperature`, `--maxTokens`. Model ids are free-form FK strings (no enum) — resolve a real `core_ai_model` id, don't invent one.
- Shape toggles: `--isAgent`, `--isAssistant`, `--isSkill`, `--hasWorkflow`, `--multipleDeployment`, `--jsonOutput`.
- **Capability toggles gate the sub-resources**: `--supportTools`, `--supportDataSources`, `--supportFiles`, `--supportKnowledgeBase`, `--supportWebSearch`. Turn the relevant one **on** before attaching.
- `--ownership` enum is `APP | CUSTOM | PRODUCT | SYSTEM | USER` (default `CUSTOM`). ⚠️ The flag help text says `SYSTEM | APP | TENANT` — that is **wrong**; `TENANT` is not valid.
- JSON-only config (schemas, memory options) goes via `--data`: `instruction_schema`, `input_form_schema`, `output_render_schema`, `memory_options`, plus the long tail in [references/agent-model.md](references/agent-model.md).

### Sub-resources

Each group supports `list`/`get`/`create`/`update`/`delete` (a few exceptions noted). Always pass app + `--agentId`; the row is `--id`.

```bash
docyrus agent data-sources create --appSlug crm --agentId AGENT_ID --tenantDataSourceId <ds-id> --privilege read --json
docyrus agent tools        create --appSlug crm --agentId AGENT_ID --coreAiToolId <tool-id> --json     # define <tool-id> first via apps ai-tools (docyrus-app-ai-tools)
docyrus agent docs         create --appSlug crm --agentId AGENT_ID --title "Pricing FAQ" --content "..." --json
docyrus agent mcps         create --appSlug crm --agentId AGENT_ID --tenantMcpServerId <id> --json
docyrus agent connections  create --appSlug crm --agentId AGENT_ID --connectedAiAgentId <id> --connectionType subagent --json
docyrus agent dynamic-contexts create --appSlug crm --agentId AGENT_ID --name "Today" --systemMessage "..." --json
docyrus agent workflow-steps create --appSlug crm --agentId AGENT_ID --name "Step 1" --type prompt --inputSchema '{}' --outputSchema '{}' --json
docyrus agent deployments  create --appSlug crm --agentId AGENT_ID --name "Production" --json
docyrus agent list --appSlug crm --json
docyrus agent get  --appSlug crm --agentId AGENT_ID --json
```

The full sub-resource table (required-on-create fields, createOnly fields, enums) is in [references/agent-model.md](references/agent-model.md#sub-resources).

## Critical rules

- **Only `--skillName` is required** to create an agent. Don't over-specify; rely on server defaults and add what the use case needs.
- **Capability toggle before attachment.** Attaching a data source/tool/knowledge doc without turning on the matching `--support*` flag leaves the agent unable to use it. Set the toggle on `create` (or `update`) first.
- **`--ownership` valid values are `APP | CUSTOM | PRODUCT | SYSTEM | USER`** (default `CUSTOM`); the flag help is inaccurate.
- **Tools are a two-step, separate flow.** `docyrus agent tools create --coreAiToolId <id>` only **attaches** an existing app-scoped tool (a `tenant_ai_tool`/`core_ai_tool`) to the agent. You define the tool first via `docyrus apps ai-tools` — see **docyrus-app-ai-tools**.
- **Sub-resource rows use `--id`**, not `--toolId`/`--dataSourceId` (deliberate, to avoid clashing with body flags). `createOnly` fields (`--coreAiToolId` on tools, `--tenantDataSourceId` on data-sources) are rejected on `update`; `--archived` is update-only.
- **`agent tasks` and `agent recurring-tasks` are non-functional** against the current backend (no matching dev routes → 404, verified: `Cannot GET …/agents/:id/tasks`). Don't design around them; schedule recurring *work* with an automation `recurrence` trigger (**docyrus-automation-design**) instead.
- **Sub-resource `create` ops need an agent you can see under RLS.** Sub-resource creates run inside an RLS-enforced write transaction that re-checks the agent via `agentExists` (id + tenant + app). An agent owned by another user (or with a null owner) is invisible there and the create returns 404 "resource not found" — even though `list`/`get` (broader reads) show it. Operate on agents you own. (Sub-resource `list` ops are verified working; `create` happy-path is currently gated by the create bug above.)
- **`agent get/list/create/upload` etc. don't error on extra JSON keys** — sub-resource "required on create" fields (e.g. `title`, `name`, `core_ai_tool_id`, `connection_type`) are enforced by the **backend DTO**, so a missing one returns a server error, not a CLI complaint. Read back to confirm.
- **Model/data-source/tool ids are real FKs**, not enums — resolve them (e.g. from `list-data-sources`, `apps ai-tools list`) before attaching; a fabricated id fails server-side.
- **Validate then test, every time.** Re-read the agent + sub-resource lists. Delete throwaway agents you create.

## Validate

1. `docyrus agent get --appSlug <app> --agentId <id> --json` — confirm identity, models, `mode`, `ownership`, and that each `--support*` toggle is set as intended.
2. For each attached kind: `docyrus agent data-sources list … `, `docyrus agent tools list … `, `docyrus agent docs list … `, etc. — confirm the attachments exist and reference the right ids.
3. If `--hasWorkflow`: `docyrus agent workflow-steps list …` — confirm steps, their `type`, and `parent`/`sort_order` sequencing.
4. If deployed: `docyrus agent deployments list …` and the deployment-tools/data-sources lists.

Per-field "what correct looks like" is in [references/workflow-examples.md](references/workflow-examples.md#validation-checklist).

## Test

1. **Well-formed check:** every list above returns the expected rows and every referenced id resolves (no dangling FK). The agent `get` shows capability toggles consistent with its attachments.
2. **Exercise it** where a path exists — invoke the agent from an automation `ai-agent` node (**docyrus-automation-design**) or the app's chat surface, give it a prompt that needs an attached data source/tool, and confirm it uses them.
3. **Clean up:** delete throwaway sub-resources, then `docyrus agent delete --appSlug <app> --agentId <id> --json`.

Full test playbook in [references/workflow-examples.md](references/workflow-examples.md#test-playbook).

## References

- **[references/agent-model.md](references/agent-model.md)** — Agent fields grouped by purpose (identity, models, shape, capabilities, render/output, memory, multi-agent), the enum values and server defaults, and the full sub-resource table (groups, required-on-create, createOnly, enums).
- **[references/workflow-examples.md](references/workflow-examples.md)** — End-to-end worked agent (assistant + data source + tool + knowledge doc), validation checklist, test playbook.
- **docyrus-platform** → `references/ai-capabilities.md` — what the agent builder can do. **docyrus-cli-app** → `references/cli-manifest.md` — exhaustive `agent` + sub-resource flag tables. **docyrus-app-ai-tools** — author the AI tools an agent calls (`apps ai-tools`); attach them to the agent here with `agent tools create --coreAiToolId`.
