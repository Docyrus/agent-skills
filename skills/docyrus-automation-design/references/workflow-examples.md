# Workflow Examples

A complete design → build → validate → test walkthrough for an automation, then reusable checklists. Commands assume an active session and target the `crm` app. Append `--json` to capture IDs.

## Table of Contents

1. [Worked example: notify + create-task on a new deal](#worked-example-notify--create-task-on-a-new-deal)
2. [Validation checklist](#validation-checklist)
3. [Test playbook](#test-playbook)

## Worked example: notify + create-task on a new deal

Goal: when a `Deal` record is created with `amount > 10000`, (a) notify the record owner and (b) create a follow-up `Task` linked to the deal.

### 1. Resolve the app and the data sources

```bash
docyrus apps list --json                                   # → appSlug "crm"
docyrus studio list-data-sources --appSlug crm --json      # → deals + tasks data source IDs
```

Capture `DEALS_DS_ID` and `TASKS_DS_ID`.

### 2. Create the automation + its record-created trigger

The create call makes the automation **and** its first trigger atomically. `--triggerType` is **camelCase**.

```bash
docyrus automation create --appSlug crm \
  --name "New big deal → notify + task" \
  --triggerType recordCreated --sourceDataSourceId DEALS_DS_ID --json
# → capture data.id as AUTO_ID (there is no slug)
```

### 3. Add the notification node (root, conditional)

Complex config (`data`, `condition`) goes through `--data`; the condition only fires for big deals.

```bash
docyrus automation create-node --appSlug crm --automationId AUTO_ID \
  --type send-notification --name "Notify owner" \
  --data '{
    "data": { "subject": "New big deal", "message": "{{name}} worth {{amount}} was created" },
    "condition": { "rules": [ { "field": "amount", "operator": ">", "value": 10000 } ] }
  }' --json
# → capture data.id as NOTIFY_NODE_ID
```

### 4. Add the create-task node, sequenced after the notification

`--targetDataSourceId` is a scalar flag; the field mapping is an object via `--data`. `parent` chains it after the notification node.

```bash
docyrus automation create-node --appSlug crm --automationId AUTO_ID \
  --type create-record --name "Create follow-up task" \
  --targetDataSourceId TASKS_DS_ID \
  --data '{
    "parent": "NOTIFY_NODE_ID",
    "field_mapping": { "name": "Follow up on {{name}}" },
    "dynamic_field_mapping": { "deal": "{{id}}" }
  }' --json
```

> `field_mapping` writes static/templated values; `dynamic_field_mapping` resolves expressions against the trigger record (here, linking the new task's `deal` relation to the triggering deal's `id`).

### 5. (Optional) a second trigger

If the same actions should also fire on manual run, add a `manual-activation` trigger (kebab `--type`):

```bash
docyrus automation create-trigger --appSlug crm --automationId AUTO_ID \
  --type manual-activation --sourceDataSourceId DEALS_DS_ID --json
```

Now validate, then test.

## Validation checklist

Run these reads and check each box.

- [ ] **Automation shape:** `docyrus automation get --appSlug crm --automationId AUTO_ID --json`
  → `name` correct; `status` = 1 (enabled); `source_data_source_id` resolves to Deals; `triggers[]` contains the recordCreated trigger.
- [ ] **Triggers:** `docyrus automation list-triggers --appSlug crm --automationId AUTO_ID --json`
  → each trigger's `type` is the expected camelCase value (`recordCreated`, `manualActivation`); watched data source correct; per-type config (e.g. `modified_columns`) present where applicable.
- [ ] **Nodes & sequence:** `docyrus automation list-nodes --appSlug crm --automationId AUTO_ID --json`
  → every node present with the right `type`; the create-task node's `parent` = the notify node's id (correct order); each `condition` and `field_mapping`/`dynamic_field_mapping` landed as written.
- [ ] **No silent drops:** the `data`/mapping you sent reads back intact. (Unknown keys are silently ignored under `whitelist:false`, so a missing key means a typo — re-check casing against the catalog.)

If any box fails, fix with `update-node` / `update-trigger` and re-read.

## Test playbook

Prove it fires, then clean up everything you created.

1. **Fire the trigger** — create a qualifying record:
   ```bash
   docyrus ds create crm deals --data '{"name":"Globex Expansion","amount":25000}' --json
   # → capture data.id as DEAL_ID
   ```

2. **Confirm the side effects:**
   - The follow-up task exists and links back:
     ```bash
     docyrus ds list crm tasks --columns "name, deal" --expand relation \
       --filters '{"rules":[{"field":"name","operator":"=","value":"Follow up on Globex Expansion"}]}' --json
     ```
   - The notification was produced (check the owner's notifications, or the automation run/job log if exposed).

3. **Test the condition boundary** — create a record that should NOT fire (`amount` below threshold) and confirm no task/notification appears:
   ```bash
   docyrus ds create crm deals --data '{"name":"Tiny deal","amount":500}' --json
   # → expect NO follow-up task created
   ```

4. **Manual run** (if you added the manual trigger) — trigger it from the record view / API and confirm the same actions run.

5. **Clean up:**
   ```bash
   docyrus ds delete crm deals DEAL_ID
   docyrus ds delete crm tasks <task-id>          # any tasks the automation created
   docyrus automation delete --appSlug crm --automationId AUTO_ID --json
   ```

Report to the user: which trigger fired, which nodes ran (and in what order), how the condition gated execution, and confirmation that test records and the throwaway automation were removed.

## Notes on what is and isn't enforced

- **DTO validation → 422** (not 400). Out-of-range numbers, bad enum values, missing `name`/`trigger_type` are caught here.
- **`external-action` → service-layer 400/404**: an `action_type_id` that isn't a `core_action` → 404; one whose group isn't `externalAction` → 400; a `data` object that fails the action's `input_json_schema` → 400.
- **`whitelist:false`**: extra JSON keys are accepted and ignored — never assume a key "worked" because the call succeeded; read the node back.
- **Recurrence won't fire instantly** in a test — verify the schedule fields and, if you need an immediate run, use a `manual-activation` trigger instead of waiting.
