# Workflow Examples

A complete design → build → validate → test walkthrough for a custom agent, then reusable checklists. Commands assume an active session and target the `crm` app. Append `--json` to capture IDs.

## Table of Contents

1. [Worked example: a "Deal Coach" assistant](#worked-example-a-deal-coach-assistant)
2. [Validation checklist](#validation-checklist)
3. [Test playbook](#test-playbook)

## Worked example: a "Deal Coach" assistant

Goal: a conversational assistant that can read the `Deals` data source, call a "deal lookup" tool, and answer from a pricing knowledge doc.

### 1. Resolve app + dependencies

```bash
docyrus apps list --json                                  # → appSlug "crm"
docyrus studio list-data-sources --appSlug crm --json     # → DEALS_DS_ID
docyrus apps ai-tools list --appSlug crm --json           # → an existing tool id (TOOL_ID); or create one first
```

The tool must already exist app-scoped. To create one, see **docyrus-app-ai-tools** (`docyrus apps ai-tools create`), then attach it here.

### 2. Create the agent with capabilities ON

Turn on every capability you'll attach, in the create call. `--skillName` is the only hard requirement.

```bash
docyrus agent create --appSlug crm \
  --skillName "deal_coach" --name "Deal Coach" --agentName "deal-coach" \
  --description "Helps reps qualify and advance deals" \
  --instructions "You are a CRM assistant. Use the deals data source and the deal-lookup tool. Be concise." \
  --isAssistant true --mode conversation \
  --supportDataSources true --supportTools true --supportKnowledgeBase true --json
# → capture data.id as AGENT_ID
```

### 3. Attach a data source (read access)

```bash
docyrus agent data-sources create --appSlug crm --agentId AGENT_ID \
  --tenantDataSourceId DEALS_DS_ID --privilege read --json
```

### 4. Attach the tool

```bash
docyrus agent tools create --appSlug crm --agentId AGENT_ID \
  --coreAiToolId TOOL_ID --json
# optionally bind default params / a connection:
# --defaultParams '{"limit":5}' --tenantConnectionId <conn-id>
```

### 5. Add a knowledge document

```bash
docyrus agent docs create --appSlug crm --agentId AGENT_ID \
  --title "Pricing FAQ" \
  --content "Standard discount is 10%. Enterprise deals over 50k qualify for 20%." --json
```

Now validate, then test.

## Validation checklist

- [ ] **Agent shape:** `docyrus agent get --appSlug crm --agentId AGENT_ID --json`
  → `skill_name` set; `mode` = `conversation`; `ownership` = `CUSTOM`; `is_assistant` = true; **each capability toggle** (`support_data_sources`, `support_tools`, `support_knowledge_base`) = true.
- [ ] **Data sources:** `docyrus agent data-sources list --appSlug crm --agentId AGENT_ID --json`
  → DEALS_DS_ID present with the intended `privilege`.
- [ ] **Tools:** `docyrus agent tools list --appSlug crm --agentId AGENT_ID --json`
  → TOOL_ID attached; `default_params`/connection as set.
- [ ] **Docs:** `docyrus agent docs list --appSlug crm --agentId AGENT_ID --json`
  → the "Pricing FAQ" doc present.
- [ ] **No dangling refs:** every attached id (data source, tool, model) resolves to a real row. The capability toggles are consistent with what's attached (you didn't attach a tool while `support_tools` is false).

Fix mismatches with `agent update` / the sub-resource `update`, then re-read.

## Test playbook

1. **Well-formed:** all lists above return the expected rows; `agent get` toggles match attachments.
2. **Exercise (where a path exists):**
   - From an automation: create an `ai-agent` node referencing `tenant_ai_agent_id = AGENT_ID` (see **docyrus-automation-design**) and run it with a prompt that needs the deals data source / tool; confirm the agent answers using them.
   - Or invoke it from the app's chat/assistant surface.
3. **Negative/guard checks:**
   - Attaching a tool while `support_tools` is false → the agent won't use it. Confirm the toggle is on.
   - `agent tasks` / `recurring-tasks` will 404 — don't rely on them; use an automation `recurrence` trigger for schedules.
4. **Clean up:**
   ```bash
   docyrus agent docs         delete --appSlug crm --agentId AGENT_ID --id <doc-id> --json
   docyrus agent tools        delete --appSlug crm --agentId AGENT_ID --id <tool-row-id> --json
   docyrus agent data-sources delete --appSlug crm --agentId AGENT_ID --id <ds-row-id> --json
   docyrus agent delete --appSlug crm --agentId AGENT_ID --json
   ```

Report to the user: the agent's shape and instructions, which capabilities are on, what's attached (data sources/tools/docs), how it was exercised, and confirmation the throwaway agent + sub-resources were removed.

## Notes on what is and isn't enforced

- **Only `skill_name` is required by the agent create DTO.** Sub-resource "required on create" fields (`title`, `name`, `core_ai_tool_id`, `connected_ai_agent_id`, `connection_type`, `tenant_data_source_id`, workflow-step `input_schema`/`output_schema`) are enforced by the **backend**, returning a server error if omitted — the CLI sends the request regardless.
- **`createOnly`/`updateOnly`**: `--coreAiToolId` (tools) and `--tenantDataSourceId` (data-sources) only on create; `--archived` only on update. Passing them to the wrong op is rejected by the CLI (unknown flag).
- **Ids are FKs, not enums** — a fabricated model/tool/data-source id passes CLI validation but fails server-side.
- **Capability toggles are independent of attachments** — attaching without toggling on leaves the capability unused; toggling on without attaching gives the agent nothing to use. Keep them in sync.
