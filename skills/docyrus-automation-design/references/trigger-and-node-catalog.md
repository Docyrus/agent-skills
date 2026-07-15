# Trigger & Action-Node Catalog

Reference for choosing and configuring Docyrus automation triggers and action nodes via `docyrus automation`. The conceptual descriptions ("what each one means") live in the **docyrus-platform** skill's `references/automation-and-workflows.md`; this file is the **CLI-shaped** reference — exact `--type` values, scalar flags, and the JSON `data` shapes you pass via `--data`. Source of truth: `apps/cli/src/commands/automationCommands.ts`, `apps/api/src/dev/dto/automation/*`, `libs/shared/src/automation/constants.ts`.

## Table of Contents

1. [Casing rules (read first)](#casing-rules-read-first)
2. [Triggers](#triggers)
3. [Action nodes](#action-nodes)
4. [Conditions, field mappings & sequencing](#conditions-field-mappings--sequencing)
5. [Validation & casing gotchas](#validation--casing-gotchas)

## Casing rules (read first)

Three different casings are in play — getting them right is most of the battle:

| Context | Casing | Example |
|---|---|---|
| `automation create --triggerType` value | **camelCase** | `recordCreated` |
| trigger/node command `--type` value | **kebab-case** | `record-created` |
| CLI flags | camelCase **or** kebab (both accepted) | `--sourceDataSourceId` = `--source-data-source-id` |
| top-level keys in `--data` (resource columns) | **snake_case** | `source_data_source_id`, `field_mapping` |
| keys inside some nested `data` objects | **camelCase** | `columns`, `filterKeyword`, `runPerRecord`, `delaySeconds` |

When a flag exists for a value, prefer the flag (the CLI maps it to the right snake_case key). Reach for `--data` only for objects that have no flag.

## Triggers

The trigger is the event that starts the automation. The **first** trigger is created with `automation create --triggerType <camelCase>`; additional/edited triggers use `create-trigger`/`update-trigger --type <kebab>`. Shared flag: `--active <bool>`.

| Intent | `--triggerType` (create) | `--type` (trigger cmds) | Key config |
|---|---|---|---|
| A record is added | `recordCreated` | `record-created` | `--sourceDataSourceId`, `--maxRunPerRecord` |
| A record changes | `recordModified` | `record-modified` | `--sourceDataSourceId`, `--modifiedColumns` (CSV), `--modifiedColumnsCondition` (`all`\|`any`), `--maxRunPerRecord` |
| A record is removed | `recordDeleted` | `record-deleted` | `--sourceDataSourceId`, `--maxRunPerRecord` |
| On a schedule | `recurrence` | `recurrence` | see recurrence block below |
| A connector/app event | `appEvent` | `app-event` | `--dataProviderId`, `--dataProviderWebhookId` (→ keys `core_data_provider_id`, `core_data_provider_webhook_id`) |
| Inbound HTTP webhook | `webhook` | `webhook` | `--webhookId` **or** `--webhookName` (name auto-creates a `tenant_webhook` of type WEBHOOK; create-only) |
| Inbound email | `emailhook` | `emailhook` | `--webhookId` **or** `--webhookName` (auto-creates an EMAIL webhook; create-only) |
| A webform submission | `webform` | `webform` | `--webformId` (→ `tenant_webform_id`) |
| A record-view button | `buttonActivation` | `button-activation` | `--sourceDataSourceId`; button label/icon/color via `--data.data` |
| Manual run | `manualActivation` | `manual-activation` | `--sourceDataSourceId` |

### recurrence config

Flags: `--sourceDataSourceId`, `--recurrenceFrequency` (`hour`\|`day`\|`week`\|`month`\|`year`), `--recurrenceInterval` (int ≥1), `--recurrenceMinutes` (one of `0,15,30,45`), `--recurrenceWeekDays` (CSV of `MON`…`SUN`), `--recurrenceMonthDays` (`DAY_OF_MONTH`\|`DAY_OF_WEEK`), `--recurrenceStartDate`/`--recurrenceEndDate` (ISO), `--recurrenceRunAt` (`HH:mm`), `--maxRunPerRecord`. Per-run ordering lives in `--data.data` (`{"ordering":<fieldId>,"sortDir":"ASC|DESC"}`).

```bash
docyrus automation create-trigger --appSlug crm --automationId <id> --type recurrence \
  --sourceDataSourceId <ds> --recurrenceFrequency day --recurrenceInterval 1 \
  --recurrenceRunAt "09:00" --json
```

## Action nodes

A node runs work. Created/updated **by type** (kebab `--type`), deleted type-independently. Every node accepts the **base flags**: `--name` (required), `--description`, `--subType`, `--parent` (sequencing), `--active`. `--condition` is an object → pass via `--data`. Below: the per-type scalar flags, then the `data` object you pass via `--data`/`--from-file`.

| `--type` | Purpose | Scalar flags | `data` object (via `--data`) |
|---|---|---|---|
| `send-email` | Send an email | base only | `data`: `sender_type` (`system`\|`custom`), `subject`, `from_name`, `reply_to`, `to_other_users[]`, `to_external[]`, `cc`, `bcc`, `files`, `template:{id}`, `sender:{id}`, `custom_template`, `assignees[]`; `field_mapping:{to}`; `input_template:{subject,message}` |
| `send-notification` | In-app/push notification | base only | `data`: `subject`, `message`, `notify_to`, `notify_users[]`, `assignees[]` |
| `create-record` | Insert a record | `--targetDataSourceId` | `field_mapping`, `dynamic_field_mapping`, `data` |
| `update-records` | Update matching records | `--targetDataSourceId`, `--targetDataSourceFieldId` | `target_data_source_condition`, `field_mapping`, `dynamic_field_mapping`, `data` |
| `request-approval` | Pause for an approval | base only | `data`, `field_mapping`, `input_data_source_id` |
| `request-input` | Pause for user input | base only | `data`, `field_mapping`, `input_data_source_id` |
| `http-request` | Call an external API | `--connectionId`, `--connectionAccountId`, `--targetDataSourceId`, `--requestMethod` (`GET`…`DELETE`), `--contentType`, `--customEndpoint`, `--relativeEndpoint`, `--batch`, `--batchSize` (1–10000), `--outputTransformer`, `--batchTransformer`, `--errorTransformer` | `data`, `field_mapping`, `dynamic_field_mapping`, `custom_headers[]` (`{key,value}`, ≤100), `input_transformer`, `pre_action_request`, `post_action_request` |
| `data-source-query` | Read one data source | `--targetDataSourceId` | `data` (camelCase: `columns`, `filterKeyword`, `filters`, `formulas`, `childQueries`, `calculations`, `orderBy`, `limit` 0–100000, `offset`, `fullCount`, `recordId`, `runPerRecord`) |
| `custom-query` | Run a saved custom query | base only | `data`: `custom_query_id`, `filters`, `limit` (0–50000), `offset`, `runPerRecord` |
| `generate-document` | Render a file | base only | `data`: `format` (`pdf`\|`csv`\|`excel`\|`markdown`), `fileName`, `template`, `columns`, `pageFormat`, `pageOrientation`, `margins`, `headerTemplate`, `footerTemplate`, `styles` |
| `ai-prompt` | One LLM call | base only | `data`: `core_ai_model_id`, `systemPrompt`, `userPrompt`, `response_format` (`text`\|`json`), `output_json_schema`, `target_data_source_id`; `dynamic_field_mapping` |
| `ai-agent` | Invoke a custom agent | base only | `data`: `tenant_ai_agent_id`; `dynamic_field_mapping` |
| `execute-script` | Run sandboxed JS | base only | `data`: `code`, `language` (`javascript`), `timeoutMs` (1–600000), `memoryMb` (1–4096), `allowNetwork`. The `code` runs with an injected `api`/`record`/`data` SDK — see [in-sandbox SDK](#execute-script-in-sandbox-sdk) below. |
| `wait-for` | Delay the chain | base only | `data`: `delaySeconds` (0–2592000) **or** `delayValue`+`delayUnit` (`seconds`\|`minutes`\|`hours`\|`days`) |
| `external-action` | Run a connector/core action | `--actionTypeId` (**required**, a `core_action` of group `externalAction`), `--sourceDataSourceId`, `--targetDataSourceId`, `--connectionId`, `--connectionAccountId`, `--webhookId` | `data` (Ajv-validated against the action's `input_json_schema`), `field_mapping`, `dynamic_field_mapping` |

> Internal node types (everything except `external-action`) auto-assign their `action_type_id` from the kebab `--type` — you never set it. Only `external-action` requires `--actionTypeId`.

### `assignees[]` item shape (send-email / send-notification / request-*)

`{ "type": <ASSIGNMENT_RULE_TYPE>, "userId"?, "userIds"?[], "fieldSlug"?, "teamId"?, "roleId"?, "unitId"?, "anchorField"?, "includeDescendants"?, "immediateOnly"? }`. `type` is required; the other keys depend on the rule type.

### `execute-script` in-sandbox SDK

The `code` you put in an `execute-script` node runs in a locked-down JS sandbox (no npm imports, no host filesystem or env). Three globals are injected:

- **`record`** — the triggering/source record the action is running against.
- **`data`** — the node's resolved input `data` (your field mappings + config).
- **`api`** — the Docyrus SDK. The bearer token lives in a closure, so `code` can read/write data but cannot exfiltrate the token. Every method is `async` and runs as the automation's own identity (it sees only what that identity may access). Methods:

| Call | Does | Returns |
|---|---|---|
| `api.query(dsql, params?)` | **Read-only DSQL** SELECT over logical `appSlug.dataSourceSlug` tables. With `params`, `dsql` is first rendered through a tiny Handlebars subset — `{{ path }}` = raw substitution, `{{ q path }}` = safe SQL-quoted (`'…'`, or `NULL` when missing/null). Without `params`, `dsql` is sent verbatim. | `{ data, meta }` |
| `api.ds.list(appSlug, dsSlug, params?)` | List records (REST; `params` = same query object as `ds list`) | `{ data, … }` |
| `api.ds.get(appSlug, dsSlug, recordId, params?)` | Read one record | the record |
| `api.ds.create(appSlug, dsSlug, data)` | Insert a record | the created record |
| `api.ds.update(appSlug, dsSlug, recordId, data)` | Patch a record | the updated record |
| `api.ds.delete(appSlug, dsSlug, recordId)` | Delete a record | delete result |

**Return output** by assigning `module.exports = {…}` (or `exports.x = …`). It becomes the node's output `data` — merged with a `__meta: { durationMs, stdout, stderr }` block, where `console.log` is captured into `__meta.stdout`. Downstream nodes template that output like any other node's.

**Network:** with `allowNetwork` unset/`false` (default), only the Docyrus API host is reachable — `api.*` works but any other `fetch()` is blocked. Set `allowNetwork: true` in the node `data` to reach external hosts.

**Prefer `api.query` for reads.** One DSQL round-trip can filter/join/aggregate across data sources, versus many `api.ds.list` calls plus JS glue. Reach for `api.ds.*` when you need to write (create/update/delete) or fetch a single record by id. For DSQL syntax (functions, table naming, row limits, rejection rules) see the **docyrus-dsql-query-design** skill.

Example — from a `record`-created trigger on `crm.projects`, roll up the project's open tasks with DSQL and write the count back:

```javascript
// `record` is the project row that fired the trigger.
const { data: byStatus } = await api.query(
  `select status, count(*) as n
     from crm.tasks
    where project = {{ q record.id }} and status != 'done'
    group by status`,
  { record },
);
const openTotal = byStatus.reduce((sum, r) => sum + Number(r.n), 0);

// Write the rollup back onto the triggering project.
await api.ds.update("crm", "projects", record.id, { open_task_count: openTotal });

module.exports = { openTotal, byStatus }; // → node output.data (plus __meta)
```

> `{{ q record.id }}` quotes the id as a SQL literal (never string-concatenate untrusted values into `dsql` — use `{{ q … }}`). Use `{{ path }}` (unquoted) only for values you control, e.g. a column name or a numeric literal.

## Conditions, field mappings & sequencing

- **`condition`** (object, via `--data`): gates whether a node runs. Built from field/operator/value rules over the trigger record (same rule shape as `ds list --filters`). Omit it to always run.
- **`field_mapping`** (object): maps target fields → static or templated values. **`dynamic_field_mapping`**: maps target fields → expressions/formulas resolved at run time against the trigger record. For the expression syntax see **docyrus-api-dev** → `references/formula-design-guide-llm.md`.
- **`parent`** (UUID): the node that must run before this one. This is how you sequence and branch — there is no order column. A node with no `parent` runs directly off the trigger. Branch by giving two nodes the same `parent` with different `condition`s.

## Validation & casing gotchas

- **422, not 400, for DTO failures.** Body `{"message":"Invalid data received for parameters", ...errors}`. The service's external-action checks are separate and return 400 (`"… is not an external action"`, `"External action data validation failed"`) or 404 (`"Core action not found"`).
- **`whitelist:false`** — extra/misspelled JSON keys are accepted and silently ignored (not stripped, not rejected). A typo in a `data` key is a silent no-op, not an error. Verify by reading the node back.
- **camelCase `--triggerType` vs kebab `--type`** — the most common live failure, and a *silent* one. `automation create --triggerType` is **not validated/normalized**: a kebab value (`record-created`) is stored verbatim and the trigger silently won't fire (verified — no 422). `create-trigger`/`create-node --type` take the **kebab** form and **are** normalized to the camelCase stored value. So: `automation create --triggerType recordCreated`; `create-trigger --type record-created` (→ stored `recordModified`/`recordCreated`/…).
- **Create responses are sparse — validate via GET/list.** `automation create` returns just `{id}` (no `status`/`triggers`); `create-trigger`/`create-node` responses omit `type`/`source_data_source_id`/`modified_columns`/`field_mapping`/`data` even though they persist. Confirm with `get`/`list-triggers`/`list-nodes`/`get-node`.
- **`--sourceDataSourceId` on `automation create` is automation-level**, not the first trigger's — the trigger's own `source_data_source_id` stays null. Set a trigger's own source with `create-trigger --sourceDataSourceId`.
- **Internal nodes read back `type: "action"`** — the specific action is the auto-assigned `action_type_id`, not the kebab `--type`.
- **Nested-object casing splits**: `data-source-query` `data` uses camelCase (`filterKeyword`, `runPerRecord`); most other node `data` objects use snake_case (`custom_query_id`, `sender_type`). Match the table.
- **Webhook/emailhook name auto-creates a webhook** on create only; on update pass `--webhookId`.
- **`delete-node`/`delete-trigger` are type-independent** routes; create/update are typed. Deletes return 204 (CLI shows `{deleted:true,id}`).
- **`recurrence_run_at` must match `HH:mm`**; `recurrence_minutes` ∈ `{0,15,30,45}`; `status` ∈ 0–5; numeric bounds as listed above — out-of-range → 422.
