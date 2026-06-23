---
name: docyrus-automation-design
description: Design, build, validate, and test a Docyrus automation (workflow) end-to-end using the `docyrus automation` CLI commands. Use when the user wants to automate a process — fire actions when records are created/modified/deleted, on a schedule, from a webhook/email/webform, or on a button click — and chain action nodes (send email/notification, create/update records, request approval/input, HTTP request, query a data source, generate a document, run an AI prompt/agent, execute a script, wait). Covers picking the right trigger, composing the action-node graph (conditions, field mappings, parent/child sequencing), and proving it runs. Triggers on "create an automation", "when a record is created/updated do X", "send an email/notification automatically", "schedule a recurring job", "webhook/email trigger", "approval workflow", "build a workflow", "automation trigger", "action node", `docyrus automation create`, `docyrus automation create-trigger`, `docyrus automation create-node`, or any Docyrus automation/workflow design + validation + testing task.
---

# Docyrus Automation Design

Build an automation with `docyrus automation`, then **validate** its trigger/node graph and **test** that it actually fires. An automation = one **trigger** (the event) + an ordered graph of **action nodes** (what runs). This skill is the design workflow; the platform's conceptual catalog of every trigger and node type lives in the **docyrus-platform** skill (`references/automation-and-workflows.md`), and the exhaustive CLI flags are available via `docyrus automation … --help` (command index: **docyrus-cli-app**). This skill ties them together and records the gotchas that only surface when you actually run the commands.

## Workflow

Follow in order. An automation that hasn't been validated and test-run is not done.

1. **Confirm app + auth.** Every automation belongs to an app.
   ```bash
   docyrus auth who --json          # confirm session + tenant
   docyrus apps list --json         # find the target appSlug / appId
   ```
   No session → stop and ask the user to run `docyrus auth login`.

2. **Design before issuing commands.** Decide: the **trigger** (which event; which data source it watches), then the **action nodes** in order, each node's **type**, its **condition** (does it run?), and its **field mappings** (what data it writes/sends). Sketch the graph for the user and confirm. See [references/trigger-and-node-catalog.md](references/trigger-and-node-catalog.md) to choose types.

3. **Create the automation + its first trigger** (one atomic call — see [Create](#create-an-automation)). You **must** pass a `--triggerType`; the automation cannot exist without one.

4. **Add more triggers** only if the same actions should fire on multiple events (optional — see [Triggers](#triggers)).

5. **Add action nodes** in order. Simple scalar config goes via flags; all complex config (`data`, `field_mapping`, `condition`, …) goes via `--data`/`--from-file`. Sequence/branch with each node's `parent`. See [Action nodes](#action-nodes).

6. **Validate** the graph — re-read the automation, its triggers, and its nodes; confirm types, conditions, and mappings landed. See [Validate](#validate).

7. **Test** — trigger the automation for real (create/modify a record, hit the webhook, or run it manually) and confirm the actions happened; then clean up. See [Test](#test).

A full worked example (record-created → conditional notification + child-record creation) with validation and a run is in [references/workflow-examples.md](references/workflow-examples.md). Read it before your first build.

## Command cheat-sheet

Selectors: `--appId | --appSlug` (one of), `--automationId`, `--triggerId`, `--nodeId`. Write commands take `--data '<json>'` or `--from-file ./x.json`; explicit flags merge **over** the JSON. Output: append `--json` (or `--format json`).

> ⚠️ **`automation` has no `slug`.** Automations are addressed only by `--automationId` (grab it from `automation list`/`create`). Enabled/disabled state is the numeric `status` (0–5), not a boolean. Per-trigger and per-node on/off is the `active` boolean.

### Create an automation

```bash
docyrus automation create --appSlug crm \
  --name "Notify on new deal" --triggerType recordCreated \
  --sourceDataSourceId <deals-data-source-id> --json
```

- `--name` and `--triggerType` are **required**. The create call inserts the automation **and** its first trigger in one shot. ⚠️ The **create response is minimal** (`{id}` — no `status`, no `triggers[]`). The trigger and `status:1` are there; **GET the automation to see them**.
- **`--triggerType` must be the camelCase value** (`recordCreated`, `recordModified`, `recordDeleted`, `recurrence`, `appEvent`, `webhook`, `emailhook`, `webform`, `buttonActivation`, `manualActivation`). ⚠️ **`automation create` does NOT validate or normalize `--triggerType`** — a wrong value like `record-created` is stored **verbatim** (no error) and the trigger silently won't fire. (Verified: kebab `record-created` was saved as `record-created`, not normalized.) The `--type` on `create-trigger`/`create-node` is the **kebab** form (`record-created`) and **is** normalized server-side to the camelCase stored value. So: camelCase for `--triggerType`, kebab for `--type` — and the danger with `--triggerType` is silent corruption, not a 422.
- `--sourceDataSourceId` on `create` sets the **automation-level** `source_data_source_id` (verified) — the first trigger's own `source_data_source_id` stays null. That's fine for record triggers (they use the automation source); to set a trigger's own source explicitly, add it with `create-trigger --sourceDataSourceId`. `--status` defaults to `1` (enabled); `0–5` valid.

### Triggers

A trigger is created/updated **by type** (kebab-case `--type`); it is deleted type-independently.

```bash
docyrus automation create-trigger --appSlug crm --automationId <id> \
  --type record-modified --sourceDataSourceId <ds-id> \
  --modifiedColumns "status,amount" --modifiedColumnsCondition any --json

docyrus automation list-triggers --appSlug crm --automationId <id> --json   # derived from the automation
docyrus automation delete-trigger --appSlug crm --automationId <id> --triggerId <tid> --json
```

Trigger `--type` (kebab): `record-created`, `record-modified`, `record-deleted`, `recurrence`, `app-event`, `webhook`, `emailhook`, `webform`, `button-activation`, `manual-activation`. Per-type flags (recurrence schedule, modified-columns, webhook name, …) are in [references/trigger-and-node-catalog.md](references/trigger-and-node-catalog.md). `list-triggers`/`get-trigger` are **derived client-side** from the automation read — there is no separate trigger GET endpoint.

### Action nodes

```bash
# scalar config via flags
docyrus automation create-node --appSlug crm --automationId <id> \
  --type create-record --name "Create task" --targetDataSourceId <tasks-ds-id> --json

# complex config (mappings/data/condition) via --data
docyrus automation create-node --appSlug crm --automationId <id> \
  --type send-notification --name "Ping owner" \
  --data '{"data":{"subject":"New deal","message":"{{name}} created"},"condition":{...}}' --json

docyrus automation list-nodes  --appSlug crm --automationId <id> --json
docyrus automation update-node --appSlug crm --automationId <id> --nodeId <nid> --type create-record --data '{...}' --json
docyrus automation delete-node --appSlug crm --automationId <id> --nodeId <nid> --json
```

- Every node takes a `--name`. Node `--type` (kebab): `external-action`, `send-email`, `send-notification`, `create-record`, `update-records`, `request-approval`, `request-input`, `http-request`, `data-source-query`, `custom-query`, `generate-document`, `ai-prompt`, `ai-agent`, `execute-script`, `wait-for`.
- **Complex objects are never flattened into flags.** `data`, `field_mapping`, `dynamic_field_mapping`, `condition`, `input_template`, `target_data_source_condition` must be supplied through `--data`/`--from-file`. Per-type `data` shapes are in [references/trigger-and-node-catalog.md](references/trigger-and-node-catalog.md).
- **Sequencing/branching is by `parent`**, not array order: set a node's `parent` to the UUID of the node it runs after. Root nodes (no `parent`) run off the trigger. A node's `condition` object gates whether it runs.

## Critical rules

- **`--triggerType` (on `automation create`) is camelCase** (`recordCreated`) and is **not validated** — a wrong value is stored verbatim and silently breaks the trigger (no 422). **`--type` (on `create-trigger`/`create-node`) is kebab-case** (`record-created`) and **is** normalized server-side. Don't cross them.
- **Create responses are minimal.** `automation create`, `create-trigger`, and `create-node` echo little more than `{id}` — `status`, `triggers[]`, node `type`/`target`/mappings come back null/empty in the *create* response but **are** persisted. Always **GET/list to validate**, never trust the create response's blanks.
- **An automation is born with its first trigger.** You cannot create a triggerless automation — `--triggerType` is required on `create`.
- **Nodes/triggers require an existing automation.** Every node/trigger op first asserts the automation exists (404 otherwise). Create the automation first.
- **Complex node config goes in `--data`/`--from-file`, never flags.** Flags only carry scalar ids/strings/bools.
- **Graph order is `parent`-linked**, not insertion order. Wire `parent` to control sequence and branches.
- **Validation failures return HTTP 422** (`"Invalid data received for parameters"`), not 400. Service-layer rejections (e.g. external-action schema checks) return 400/404. (See the catalog for the external-action 400/404 rules.)
- **Unknown JSON keys are NOT rejected** (the API runs `whitelist:false`) — a mistyped `data` key passes validation and is silently ignored. Spell keys exactly (snake_case top-level; some nested `data` objects use camelCase — see catalog).
- **`status` is an int 0–5, not a boolean.** Per-trigger/per-node enable is the `active` boolean.
- **`external-action` needs a real `action_type_id`** (a `core_action` of group `externalAction`); internal node types auto-assign their `action_type_id` from the kebab `--type`.
- **`agent tasks`/`recurring-tasks` ≠ automations.** Scheduling recurring *work* is the `recurrence` trigger here; the `docyrus agent tasks` group is unrelated (and currently non-functional).
- **Validate then test, every time.** Re-read the graph and fire the trigger once. Delete throwaway records/automations you create.

## Validate

After authoring, confirm the graph is exactly what you intended:

1. `docyrus automation get --appSlug <app> --automationId <id> --json` — confirm `name`, `status`, `source_data_source_id`, and the embedded `triggers[]`.
2. `docyrus automation list-triggers --appSlug <app> --automationId <id> --json` — confirm each trigger's `type`, watched data source, and per-type config (`recurrence_*`, `modified_columns`, …).
3. `docyrus automation list-nodes --appSlug <app> --automationId <id> --json` — confirm every node's `name`, `parent` (sequence), `condition`, `data`, and `field_mapping`. ⚠️ Every internal node's `type` reads back as `"action"` (the generic node type) — the specific action identity is the auto-assigned `action_type_id`, not the kebab `--type` you passed. Use `get-node` to confirm `data`/`field_mapping`/`target_data_source_id` landed (the create response omits them).

Detailed "what correct looks like" checklist is in [references/workflow-examples.md](references/workflow-examples.md#validation-checklist).

## Test

Prove it fires:

1. **Record/button/manual triggers:** create or modify a record in the watched data source (`docyrus ds create … `) so the trigger condition matches, then read back the side effects (the created/updated record, a notification, etc.).
2. **Recurrence:** confirm the schedule fields; for an immediate check, also attach a `manual-activation` trigger and run it, or lower the interval temporarily.
3. **Webhook/emailhook/webform:** confirm the auto-created `tenant_webhook`/webform binding exists, then post to it.
4. **Clean up:** delete throwaway records, then `docyrus automation delete --appSlug <app> --automationId <id>` for any test automation.

Full test playbook is in [references/workflow-examples.md](references/workflow-examples.md#test-playbook).

## References

- **[references/trigger-and-node-catalog.md](references/trigger-and-node-catalog.md)** — Every trigger type and action-node type: when to use it, its scalar flags, and its `data`/mapping shape (with key casing). Plus the validation/casing gotchas per type.
- **[references/workflow-examples.md](references/workflow-examples.md)** — End-to-end worked automation (trigger + conditional multi-node graph), the validation checklist, and the run/test playbook.
- **docyrus-platform** → `references/automation-and-workflows.md` — the conceptual catalog (what each trigger/node *means*). **docyrus-cli-app** — CLI command index; `docyrus automation … --help` for exhaustive flags. For `data-source-query`/`custom-query` node payloads and field mappings, see **docyrus-api-dev** → `references/data-source-query-guide.md` and `references/formula-design-guide-llm.md`.
