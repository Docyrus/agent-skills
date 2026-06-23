# Field Type Catalog

Canonical reference for choosing and configuring Docyrus field types. The authoritative source is `libs/shared/src/data-source/constants.ts` (`FIELD_TYPES`, `FIELD_DATABASE_TYPES`) and `libs/shared/src/data-source/fieldValidation.ts`.

Field `type` values exist in two forms: the **bare** name (`text`, `select`, `relation`) and the **canonical** `field-` name (`field-text`). The CLI/API accept either and normalise to the `field-` form. This catalog uses bare names in prose.

## Table of Contents

1. [Choosing a type (intent → type)](#choosing-a-type-intent--type)
2. [Configuration by type](#configuration-by-type)
3. [Stored database types](#stored-database-types)
4. [Default value rules](#default-value-rules)
5. [Reserved field slugs](#reserved-field-slugs)
6. [Data access levels](#data-access-levels)

## Choosing a type (intent → type)

| Need | Field type(s) |
|---|---|
| Short single-line text | `text` |
| Long / multi-line text | `textarea` |
| Markdown body | `markdown` |
| Email address | `email` |
| Phone number (+ country) | `phone` (companion `_country`) |
| Web link | `url` |
| Color swatch / icon picker | `color`, `icon` |
| Masked secret | `password` |
| Read-only echo of another field | `relatedField` |
| Rich text / WYSIWYG HTML | `htmlEditor` (companion `_html`) |
| Designed email body | `emailEditor` (companion `_html`) |
| Collaborative document | `docEditor` |
| Source code with syntax | `codeEditor`, `code` |
| Whole number / decimal | `number` |
| Money amount (+ currency) | `money` (companion `_currency`) |
| Currency code selector | `currency` |
| Time span / duration | `duration` (integer) |
| Star rating 1–5 | `rating` |
| Auto-incrementing counter | `autonumber`, `identity` (both read-only, no value) |
| Calendar date | `date` |
| Date + time | `dateTime` |
| Time of day | `time` |
| Start/end date range | `dateRange` |
| Yes/no | `checkbox`, `switch` |
| Pick one option | `select`, `radioGroup` |
| Pick one option + workflow metadata | `status` (companions `_secondary`, `_description`, `_followup_date`) |
| Pick many options | `multiSelect`, `tagSelect` |
| Pick one platform user | `userSelect` |
| Pick many platform users | `userMultiSelect` |
| Link to one record in another data source | `relation` (set `relationDataSourceId`) |
| Show records that point back here | `list` (virtual reverse lookup, read-only) |
| Nested array of sub-rows | `inlineData` |
| Nested single object | `inlineForm` |
| File / image attachment | `file`, `image` |
| Folder of files | `fileStorageFolder` |
| Computed expression (JSONata) | `formula` (read-only) |
| Read-only computed display | `display` (read-only) |
| Action button (no value) | `button` |
| Approval state | `approvalStatus` |
| Checklist / todos | `taskList`, `todo` |
| Geographic location | `locationSelect` |
| Arbitrary JSON blob | `json` |
| Saved query definition | `queryBuilder` |
| Dynamic/runtime-shaped value | `dynamic` |
| JSON-schema-described value(s) | `schema`, `schemaRepeater` |
| Messaging channel ref | `conversationChannel` |

**System-only types** (`systemEnum`, `systemBuffer`, `systemVector`, `systemTextArray`, `systemUuidArray`) back platform/system data sources — do not use them when authoring app data sources via `studio`.

## Configuration by type

Most simple types (`text`, `email`, `number`, `date`, `checkbox`, …) need only `name`, `slug`, `type`. The types below need extra config.

### Relations — `relation`
- **Required:** `relationDataSourceId` = the target (parent) data source's ID. Stored as a `uuid` FK.
- Put the `relation` field on the **child** (the side that holds one reference). Create the parent data source first so its ID exists.
- A record's value is the parent record's `id` (a UUID). When testing, set it to a real parent record ID.

### Reverse lookup — `list`
- Virtual/computed: no stored column, read-only, **no default value**, not writable.
- Lives on the **parent** to surface the child records that reference it. Configure it with the single `create-field --relationDataSourceId <childDsId>` flag (verified to persist); confirm with `list-fields` that `relation_data_source_id` is set, and test by pulling children with `--childQueries`.

### Selections & enums — `select`, `radioGroup`, `status`, `multiSelect`, `tagSelect`, `userSelect`, `userMultiSelect`
- Single-value selects store a `uuid` (the enum row ID); multi-selects store `uuid[]`.
- After creating the field, add options with `studio create-enums` (see SKILL.md). Each value the field stores is an **enum row UUID**, never the label.
- Reuse a shared option list across fields with `tenantEnumSetId` instead of duplicating options.
- `status` is a richer single-select with companion columns (`_secondary`, `_description`, `_followup_date`) and option flags (`isFinalOption`, `forceDescription`, `forceFollowupDate`) for pipeline/stage modelling.
- `userSelect`/`userMultiSelect` reference tenant users (resolved at query time via `--expand`), not custom enums.

### Numeric — `number`, `money`, `duration`, `rating`, `currency`
- `number`/`money`: decimal precision lives in `options.attributes` (`decimal: boolean`, `decimalPrecision: number`).
- `money` carries an amount (numeric) plus a `_currency` companion for the currency code; `currency` is a currency-code value (text).
- `duration` stores an integer; `rating` is an integer 1–5.

### Computed — `formula`, `display`
- `formula`: a JSONata/block expression evaluated at query time. Read-only, **no default**, no stored column. See the docyrus-platform / docyrus-api-dev **formula-design-guide-llm** reference for expression syntax.
- `display`: read-only computed/echo text; **no default**.

### Nested data — `inlineData`, `inlineForm`
- Stored as `jsonb`. `inlineData` = array of sub-rows; `inlineForm` = a single nested object. **No default value.**
- Shape is described via `options` (and may use `schema`/`schemaRepeater` for typed sub-fields).

### Files — `file`, `image`, `fileStorageFolder`
- Stored as `jsonb` metadata referencing storage objects. Upload record files with `docyrus ds files upload` when testing.

### Generic config keys (any field)
- `options` (JSON / `IFieldOptions`): `attributes` (numeric precision), `formatterOptions` (display formatting), `editorOptions` (input UI), `agentDescription` (hint for AI tools).
- `validations` (JSON array): named validation rules applied on write.
- `readOnly`, `status`, `sortOrder`: presentation/lifecycle metadata.
- `input_transformer` / `output_transformer`: server-side transform hooks (advanced; usually omitted).

## Stored database types

The physical column type per field (`FIELD_DATABASE_TYPES`). Useful for predicting filter/sort behaviour and what a value must look like.

| DB type | Field types |
|---|---|
| `text` | text, textarea, email, phone, url, color, icon, display, markdown, password, relatedField, htmlEditor, emailEditor, codeEditor, code, formula, currency, button, enum, systemEnum |
| `numeric` | money, number |
| `int` | duration, rating, identity, autonumber |
| `boolean` | checkbox, switch |
| `uuid` | relation, select, radioGroup, status, userSelect |
| `uuid[]` | multiSelect, tagSelect, userMultiSelect, systemUuidArray |
| `date` / `timestamptz` / `time` / `tstzrange` | date / dateTime / time / dateRange |
| `jsonb` | list, json, file, image, docEditor, conversationChannel, inlineData, inlineForm, taskList, todo, approvalStatus, locationSelect, queryBuilder, dynamic, schema, schemaRepeater, fileStorageFolder |
| `text[]` | systemTextArray |
| `bytea` / `vector` | systemBuffer / systemVector |

## Default value rules

`defaultValue` is always a **string** and is validated/cast server-side by field type. Violations return HTTP 400.

| Group | Field types | Rule |
|---|---|---|
| **UUID** | select, radioGroup, status, relation, enum | Must be a valid UUID — the **enum row ID** (selects/status) or a **record ID** (relation), **not the label**. Lowercased. |
| **Numeric** | number, money, duration, rating | Must parse as a number. `rating` = integer 1–5. `duration` = integer. |
| **Boolean** | checkbox, switch | `"true"`/`"1"` → true, `"false"`/`"0"` → false. |
| **No default allowed** | formula, list, button, taskList, display, autonumber, identity, approvalStatus, multiSelect, tagSelect, userMultiSelect, userSelect, inlineData, inlineForm, queryBuilder, dynamic, schema, schemaRepeater | Any default is dropped to `null`. |
| **As-is** | text, textarea, email, url, date, dateTime, time, etc. | Stored verbatim. |

Practical consequence: to give a `select`/`status` a default, **create the field → create its enums → `list-enums` to read the row IDs → set `defaultValue` to that UUID** (e.g. via `update-field`).

## Reserved field slugs

These slugs are built-in static columns on every simple data source and are **rejected** as custom field slugs (HTTP 400). Use the built-ins directly; do not recreate them.

```
id, name, description, autonumber_id, created_on, created_by,
last_modified_on, last_modified_by, followers, mentions, record_owner,
color, icon, data, docyment, type, tenant_id, tenant_data_source_id,
parent, parent_data_source_id, parent_record_id, editor_view_id,
style, sort_order, cursor_date
```

`name` is the record's title and `description` its notes — wire user-facing "title"/"notes" requirements to these, not to new fields.

## Data access levels

Set on the data source (`--data '{"data_access":...,"unit_peer_access":...}'`). Orgchart-aware: the record owner and their subordinates always keep full read/edit/delete; these levels control everyone else.

**`data_access`** (`enum_data_access`):
| Value | Grants to the rest of the tenant |
|---|---|
| `OPEN` | read / edit / delete |
| `PUBLIC_EDIT` | read / edit · delete reserved to owner + subordinates |
| `PUBLIC_READ` | read · edit / delete reserved to owner + subordinates |
| `PRIVATE` | nothing (owner + subordinates only) |

**`unit_peer_access`** (`enum_unit_peer_access`) — sideways grant to same-hierarchy-unit peers:
| Value | Peers can |
|---|---|
| `NONE` | nothing |
| `READ` | read |
| `EDIT` | read / edit |
| `FULL` | read / edit / delete |
