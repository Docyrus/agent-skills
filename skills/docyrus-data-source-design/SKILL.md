---
name: docyrus-data-source-design
description: Design, validate, and test a Docyrus data source (schema/table/collection) end-to-end using the `docyrus studio` CLI commands. Use when the user wants to model a new entity or table, create or change a data source and its fields, choose the right field types, set up relations between data sources, add select/status/enum option lists, or apply data-access rules — and then confirm the schema is correct and prove it works with sample records. Triggers on "design a data source", "create a table/collection/entity", "add fields", "model a schema", "set up a relation", "add dropdown/status options", "studio create-data-source", "studio create-field", "studio create-enums", or any data-source schema modelling + validation + testing task in Docyrus.
---

# Docyrus Data Source Design

Design a data source's schema with `docyrus studio`, then **validate** the schema and **test** it with real records before handing it off. A data source is a collection of records with a typed schema (fields). This skill is the schema-authoring counterpart to record querying (`docyrus ds` / `dsql`).

For the full CLI flag reference see the **docyrus-cli-app** skill; for platform concepts see **docyrus-platform**; for general schema-design principles see the **database-architect** skill. This skill is the data-source-specific workflow that ties them together.

## Workflow

Follow these phases in order. Do not skip validation/testing — an unvalidated schema is not done.

1. **Confirm the app and auth.** Every data source belongs to an app. Resolve the app first:
   ```bash
   docyrus auth who --json            # confirm an active session + tenant
   docyrus apps list --json           # find the target appSlug / appId
   ```
   If there is no session, stop and ask the user to run `docyrus auth login` (interactive).

2. **Model the entities.** Turn the requirement into data sources and fields *before* issuing commands. Per entity decide: the singular/plural name, the slug, each attribute → field type, which fields are selections (need enums), and which are relations to other data sources. Sketch this for the user and confirm naming when the domain is ambiguous. See [references/field-types.md](references/field-types.md) to pick types.

3. **Create the data source.** See [Create a data source](#create-a-data-source). Create parent/independent data sources before the ones that relate to them (you need the parent's ID for relation fields).

4. **Add fields.** Batch-create where possible (see [Add fields](#add-fields)). Set relation fields' `relationDataSourceId` to the target data source's ID.

5. **Add enum options** for every selection/status field. See [Add enum options](#add-enum-options).

6. **Validate the schema.** Re-read the data source with its fields and confirm every field landed with the right type/config. See [Validate](#validate).

7. **Test with sample records.** Create, query (with relation expansion), and then delete throwaway records to prove the schema actually accepts and returns data. See [Test](#test).

A complete worked example (a 2-table CRM with a relation, enums, validation, and tests) is in [references/workflow-examples.md](references/workflow-examples.md). Read it before doing your first end-to-end build.

## Studio command cheat-sheet

Selectors are exclusive pairs — pass exactly one of each required pair, the CLI resolves the other:
`--appId | --appSlug`, `--dataSourceId | --dataSourceSlug`, `--fieldId | --fieldSlug`.
Write commands take inline `--data '<json>'` **or** `--from-file ./payload.json` (JSON only); explicit flags override overlapping JSON keys. Append `--json` for machine-readable output.

### Create a data source

```bash
docyrus studio create-data-source --appSlug crm \
  --title "Contacts" --name "Contact" --slug "contacts" --icon "users" --json
```

- `title` = plural display label ("Contacts"); `name` = singular entity name ("Contact"); `slug` = url-safe lowercase id ("contacts"). All three are **required**. `--icon` persists but is not echoed in responses.
- **`studio` only creates `simple` data sources.** Omit `--type` — it defaults to `simple` and the server always stores `simple`. Passing any other value (`--type advanced`, `external`, …) is **rejected with HTTP 400** (`"Invalid data received for parameters"`), not silently coerced. `advanced`/`external`/`system` data sources are provisioned through other flows, not here.
- Access control is optional and goes through `--data` (the orgchart-aware RLS levels):
  ```bash
  docyrus studio create-data-source --appSlug crm --title "Deals" --name "Deal" --slug "deals" \
    --data '{"data_access":"PRIVATE","unit_peer_access":"READ"}' --json
  ```
  `data_access` ∈ `OPEN | PUBLIC_EDIT | PUBLIC_READ | PRIVATE`; `unit_peer_access` ∈ `NONE | READ | EDIT | FULL`. See [references/field-types.md](references/field-types.md#data-access-levels) for what each level grants.
- Simple data sources ship with built-in static fields you never create yourself (and cannot reuse as slugs): `id`, `name`, `description`, `created_on`, `created_by`, `last_modified_on`, `last_modified_by`, `record_owner`, `followers`, `mentions`, `color`, `icon`, `type`, `parent`, `sort_order`, plus internal columns. The full reserved set is in [references/field-types.md](references/field-types.md#reserved-field-slugs). **`name` already exists as the record title — do not add a `name` field; use it directly.**

### Add fields

One field:
```bash
docyrus studio create-field --appSlug crm --dataSourceSlug contacts \
  --name "Email" --slug "email" --type "email" --json
```

Batch (preferred — one round-trip per data source):
```bash
docyrus studio create-fields-batch --appSlug crm --dataSourceSlug contacts --data '[
  { "name": "Email",        "slug": "email",        "type": "email" },
  { "name": "Phone",        "slug": "phone",        "type": "phone" },
  { "name": "Lifetime value","slug": "ltv",         "type": "money" },
  { "name": "Status",       "slug": "status",       "type": "select" },
  { "name": "Account",      "slug": "account",      "type": "relation",
    "relation_data_source_id": "<accounts-data-source-id>" }
]' --json
```

- `name`, `slug`, `type` are **required** per field. `type` accepts the bare form (`text`, `select`, `relation`) or the canonical `field-` form (`field-text`) — both work.
- **Flags vs. batch-item keys differ in casing.** The single `create-field` command takes camelCase **flags** (`--relationDataSourceId`, `--defaultValue`, `--readOnly`, `--sortOrder`, `--tenantEnumSetId`). `create-fields-batch` JSON **items** must use the API's **snake_case** keys (`relation_data_source_id`, `default_value`, `read_only`, `sort_order`, `tenant_enum_set_id`, `options`, `validations`). ⚠️ camelCase keys inside batch items are **silently dropped** — a relation built with `relationDataSourceId` in a batch item lands with a `null` target. (Enum batch items are the exception — they use camelCase: `sortOrder`, `isFinalOption`.)
- `defaultValue`/`default_value` is a **string** and is type-checked server-side at the schema level. Enum-backed selects/status/relation defaults must be the **enum row UUID, not the label** (a label returns HTTP 400). Many types reject defaults entirely. See [references/field-types.md](references/field-types.md#default-value-rules).
- Slugs are lowercased for simple data sources. Reserved slugs (see above) are rejected with HTTP 400.

### Add enum options

Selection fields (`select`, `multiSelect`, `status`, `radioGroup`, `tagSelect`, `userSelect`) hold UUIDs that reference enum rows. After creating the field, add its options:

```bash
docyrus studio create-enums --appSlug crm --dataSourceSlug contacts --fieldSlug status --data '[
  { "name": "Lead",     "color": "#94a3b8", "sortOrder": 1 },
  { "name": "Active",   "color": "#22c55e", "sortOrder": 2 },
  { "name": "Churned",  "color": "#ef4444", "sortOrder": 3, "isFinalOption": true }
]' --json
```

Per-option keys: `name` (required), `slug`, `color`, `icon`, `description`, `sortOrder`, `isFinalOption`, `forceDescription`, `forceFollowupDate`, `parent` (hierarchical). To reuse one option list across many fields, point each field at a shared set with `--tenantEnumSetId <id>` instead of recreating options.

### Manage / inspect

```bash
docyrus studio list-data-sources --appSlug crm --json                              # NB: --expand fields does NOT embed fields
docyrus studio get-data-source   --dataSourceId <id> --json                        # by data source ID only — no app/slug selectors
docyrus studio list-fields       --appSlug crm --dataSourceSlug contacts --json     # authoritative way to inspect fields
docyrus studio list-enums        --appSlug crm --dataSourceSlug contacts --fieldSlug status --json
docyrus studio update-field      --appSlug crm --dataSourceSlug contacts --fieldId <id> --data '{"name":"Primary email"}' --json
docyrus studio delete-field      --appSlug crm --dataSourceSlug contacts --fieldSlug email --json
docyrus studio delete-data-source --appId <id> --dataSourceSlug contacts --json    # archives (soft delete)
docyrus studio permanent-delete-data-source --appId <id> --dataSourceId <id> --json # hard delete (requires the ID)
```

`get-data-source` is the one studio read that takes only `--dataSourceId` (no `--appSlug`/`--dataSourceSlug`). `list-data-sources --expand fields` returns the data sources but **not** their fields — use `list-fields` to see fields.

## Relations

- **Link a child to a parent:** add a `relation` field on the **child** data source whose `relationDataSourceId` is the **parent's** data source ID (single-valued, stored as a UUID FK). Example: `contacts.account → relation → accounts`.
- **Many references:** use `multiSelect`/`userMultiSelect`/`tagSelect` (stored as `uuid[]`) when a record points to several rows.
- **Show the children from the parent side:** add a `list` field (a virtual/computed reverse lookup). `list` is read-only, has no stored column, and takes no default value.
- **Query across a relation** (for testing):
  - **Filter** with `rel_<relationSlug>/<fieldSlug>` (e.g. `rel_account/name`). This form is **filter-only** — it is not a column name.
  - **Expand to a nested object**: add the relation field to `--columns` and pass `--expand relation` → the value comes back as `{ "id": …, "name": … }`.
  - **Spread inline**: `...<relationFieldSlug>(field)` (e.g. `...account(name)`) flattens the related field into the row — alias to avoid collisions, since a related `name` overwrites the row's own `name`.
  - Or pull children with `--childQueries`. See the docyrus-cli-app **list-query-examples** reference.

Create parent data sources first so their IDs exist when you wire up relation fields.

## Critical rules

- **`studio` creates only `simple` data sources.** Omit `--type`; any non-simple value is rejected with HTTP 400.
- **Batch field items use snake_case keys** (`relation_data_source_id`, `default_value`, …); camelCase keys inside `create-fields-batch` JSON are silently dropped (e.g. a relation lands with a null target). The single `create-field` command uses camelCase *flags* instead. Enum batch items are camelCase (`sortOrder`, `isFinalOption`).
- **`get-data-source` resolves by `--dataSourceId` only** — no app/slug selectors. Get the ID from `list-data-sources`.
- **Simple data sources do not type-check record values on write.** Passing a label instead of a UUID (e.g. `status: "Lead"`) is stored as a raw string and simply won't resolve to the enum — it does **not** error. Always write enum row IDs / record IDs. (Strict type checks apply at the *schema* level — `create-field` defaults — and to `advanced` data sources.)
- **`name` and `description` already exist** on every simple data source (record title + notes). Reserved slugs are rejected — never recreate them as fields.
- **Enum-backed defaults are UUIDs**, not labels. Create the field, then read back the enum row IDs (`list-enums`) before setting a `defaultValue`.
- **Virtual/computed types take no value and no default**: `formula`, `list`, `button`, `taskList`, `display`, `autonumber`, `identity`. Don't try to set defaults or write to them.
- **Validate then test, every time.** Re-read the schema and round-trip at least one record. Delete any throwaway test records you create.
- Create independent data sources before dependent (relation-holding) ones.

## Validate

After authoring, confirm the schema is exactly what you intended:

1. `docyrus studio get-data-source --dataSourceId <id> --json` — confirm `type`, `slug`, access fields. (By ID only; grab the ID from `list-data-sources`.)
2. `docyrus studio list-fields --appSlug <app> --dataSourceSlug <ds> --json` — confirm every field exists with the right `type`, `relation_data_source_id`, `default_value`, `options`. (This is the authoritative field check — `list-data-sources --expand fields` does not return fields.)
3. For each selection field: `docyrus studio list-enums ... --json` — confirm options and capture their IDs.
4. `docyrus dsql schema data-source <app> <ds>` — confirm the queryable columns/relations the engine actually exposes.

Detailed checklist (what "correct" looks like per field type) is in [references/workflow-examples.md](references/workflow-examples.md#validation-checklist).

## Test

Prove the schema works against real data:

1. **Insert** a sample record covering required fields, a select (by enum UUID), and a relation (by parent record ID):
   ```bash
   docyrus ds create crm contacts --data '{"name":"Jane Doe","email":"jane@x.com","status":"<enum-id>","account":"<account-id>"}' --json
   ```
2. **Read it back** with expansion to confirm types resolve and relations join:
   ```bash
   docyrus ds list crm contacts --columns "name, email, status, account" --expand relation --json
   # status (enum) and account (relation) come back as nested { id, name } objects
   ```
3. **Exercise** a filter, a relation filter (`rel_account/name`), and a default value (omit a field that has a default and confirm it fills in).
4. **Clean up:** `docyrus ds delete crm contacts <recordId>` for every test record.

Full test playbook (edge cases, default-value checks, relation round-trip) is in [references/workflow-examples.md](references/workflow-examples.md#test-playbook).

## References

- **[references/field-types.md](references/field-types.md)** — Full field-type catalog: intent → type, configuration (`options`/`relationDataSourceId`/enums), stored DB type, default-value rules, reserved slugs, and access-control levels.
- **[references/workflow-examples.md](references/workflow-examples.md)** — End-to-end worked example (build → validate → test a related 2-table schema), plus the validation checklist and test playbook.
