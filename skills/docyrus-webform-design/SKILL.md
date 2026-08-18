---
name: docyrus-webform-design
description: Design, validate, and test a Docyrus webform — a public, unauthenticated form that collects external submissions into records of a data source — using the `docyrus studio` webform CLI commands. Use when the user wants a public intake/lead/contact/registration/survey form whose submissions create records (e.g. a website contact form that lands in a CRM data source), needs the form's public URL or embed snippet, wants to bind a form to a data source, or wants a submission to kick off an automation. Covers designing the form schema, binding it to a data source, activating it, getting the public/submit/embed URLs, and proving a submission creates a record. Triggers on "create a webform", "public form", "contact/lead/intake form", "form that creates records", "embed a form on a website", "form submission webhook", "studio create-webform", or any webform authoring + validation task. For the data source the form writes into, use docyrus-data-source-design; for an automation that fires on submission, use docyrus-automation-design.
---

# Docyrus Webform Design

Design a public webform with `docyrus studio create-webform`, then **validate** it and **test** that a submission lands as a record. A webform is a `tenant_webform` row, paired 1:1 with a webhook, that renders a public (no-login) form; each submission creates a record in the **bound data source** (or, if unbound, in a per-tenant `webform_record` table) and can fire an automation.

## How a webform works (read first)

- A webform is **bound to a data source** at create time. Each **form field's slug must match a field slug** on that data source — that's how a submission maps to record columns. Bind the data source first (see **docyrus-data-source-design**).
- Studio/builder-compatible forms persist two JSON documents: `schema` for ordered fields and per-field layout, and `options` for layout variant, theme, grid/wizard settings, and respondent-facing copy. The authoritative contract is [references/webform-design-schema.md](references/webform-design-schema.md).
- The public form is addressed by the **paired webhook's** short id + token, **not** the webform UUID. After create, read back `form_url` (render), `form_submit_url` (POST target), and `embed_code` (a `<script>` snippet).
- A submission `POST`s `{ "data": { …field values… } }` to the submit URL → it's queued → an edge function **asynchronously** creates the record in the bound data source and fires any active automation `webform` trigger. So records appear a moment after submission, not synchronously.

## Workflow

1. **Confirm app + auth, and the target data source.** The form writes into one data source whose field slugs the form keys must match.
   ```bash
   docyrus auth who --json
   docyrus apps list --json
   docyrus studio list-fields --appSlug crm --dataSourceSlug leads --json   # the slugs your form fields must use
   ```

2. **Design the form schema.** Decide which data-source fields the form collects, then build the separate `schema` and `options` documents. Each `schema.children[].field.slug` = the data-source field slug. Read [references/webform-design-schema.md](references/webform-design-schema.md) for the complete builder contract and start from [references/examples/webform-design.full.example.json](references/examples/webform-design.full.example.json) when a full example is useful. Use [references/schemas/webform-builder.schema.json](references/schemas/webform-builder.schema.json) and [references/schemas/webform-options.schema.json](references/schemas/webform-options.schema.json) for machine validation. Use [references/webform-model-and-submission.md](references/webform-model-and-submission.md) for the backend lifecycle and record-mapping details.

3. **Create the webform** (`create-webform`) bound to the data source, `status: 1` (active). See [Create](#create-a-webform).

4. **Get the public URLs** from the response/`get-webform` (`form_url`, `form_submit_url`, `embed_code`).

5. **Validate** the shape + binding. See [Validate](#validate).

6. **Test** a submission and confirm a record is created in the data source. See [Test](#test).

A worked example (a "Leads" intake form bound to a CRM data source, submitted, record confirmed) is in [references/webform-model-and-submission.md](references/webform-model-and-submission.md#worked-example).

## Builder schema specification (read when authoring JSON)

Use the copied Webforms builder specification as the source of truth whenever the task requires a studio-compatible form design rather than only a minimal backend payload:

1. Read [references/webform-design-schema.md](references/webform-design-schema.md) for `schema`, `options`, all 30 palette field types, validation tokens, value shapes, derived `json_schema`, submission behavior, and renderer compatibility notes.
2. Validate the two persisted documents independently with [references/schemas/webform-builder.schema.json](references/schemas/webform-builder.schema.json) and [references/schemas/webform-options.schema.json](references/schemas/webform-options.schema.json).
3. Use [references/schemas/webform-design-output.schema.json](references/schemas/webform-design-output.schema.json) only for a documentation/test wrapper containing both documents; the API stores them as separate properties.
4. Validate the public POST wrapper with [references/schemas/webform-submission-envelope.schema.json](references/schemas/webform-submission-envelope.schema.json), then validate its `data` property against the webform-specific derived `json_schema`.
5. Use [references/examples/webform-design.full.example.json](references/examples/webform-design.full.example.json) as the exhaustive palette example. Reduce it to the fields the user needs; do not copy unrelated fields into a production form.

The native builder shape is `{ "version": 1, "children": [{ "type", "field", "enumOptions"?, "layout" }] }`. Do not substitute the older `FieldText` + `options.key/options.label` shape when the user asks for output compatible with the current Webforms builder.

## Command cheat-sheet

Selectors: `--appId | --appSlug` (to resolve a slug), `--dataSourceId | --dataSourceSlug` (the binding), `--webformId` (the form). Write commands take camelCase flags **or** `--data`/`--from-file`; flags merge over JSON. Append `--json`.

> ⚠️ **Webforms have no slug** — `get`/`update`/`delete` address the form only by `--webformId`. `list-webforms` can filter by data source.

### Create a webform

```bash
docyrus studio create-webform --appSlug crm --dataSourceSlug leads \
  --name "Website lead form" --status 1 \
  --schema '{"version":1,"children":[
    {"type":"field-text","field":{"id":"lead-name","name":"Name","slug":"name","type":"field-text","validations":["required"]},"layout":{"colSpan":1,"rowSpan":1}},
    {"type":"field-email","field":{"id":"lead-email","name":"Email","slug":"email","type":"field-email","validations":["required"]},"layout":{"colSpan":1,"rowSpan":1}}
  ]}' \
  --css "body{font-family:sans-serif}" --json
# → capture data.id (webform UUID) and data.form_url / data.form_submit_url / data.embed_code
```

- **Required:** `--name`, `--schema` (a non-empty **JSON object**), and `--status` (**1 = active, 2 = inactive**). Omitting any → validation error (the CLI marks them optional, but the backend rejects).
- **Binding is optional but create-only:** `--dataSourceSlug`/`--dataSourceId` sets the data source. ⚠️ `update-webform` **cannot rebind** — to change the data source, recreate the form. An **unbound** form still captures submissions (into `webform_record`), just not into a data source.
- `--webformOptions` (JSON) and `--css` are optional builder config; `--sandbox true` marks it a test form (separate sandbox URL).
- **Each `schema.children[].field.slug` must equal a bound data-source field slug** — that's what makes a submission populate the record. Confirm with `list-fields` first. Also require non-empty unique field slugs and stable field/option/grid-row IDs as specified in [references/webform-design-schema.md](references/webform-design-schema.md#12-authoring-and-validation-checklist).

### Manage / inspect

```bash
docyrus studio list-webforms --appSlug crm --dataSourceSlug leads --json   # filter by data source; omit to list all
docyrus studio get-webform    --webformId <id> --json                       # full schema + form_url/form_submit_url/embed_code
docyrus studio update-webform --webformId <id> --status 2 --json            # PATCH; partial; NO data-source rebind
docyrus studio delete-webform --webformId <id> --json                       # 204; also deletes the paired webhook
```

- `get`/create responses expose the public handles: `form_id`/`form_token` (the webhook short id + token), `form_url` (render), `form_submit_url` (POST target), `form_sandbox_url`, and `embed_code` (a ready `<script>` snippet).
- `update` is PATCH (partial). `delete` returns 204 and cascades to the paired webhook.

## Critical rules

- **`name` + `schema` (JSON object) + `status` (1|2) are required** on create; `status` must be `1` or `2` (any other value rejected).
- **`schema.children[].field.slug` = data-source field slug.** A submission's `data` keys are mapped to records by slug — mismatched slugs won't populate the record. Validate slugs with `list-fields` before authoring the schema and follow the builder specification's uniqueness/stability rules.
- **Binding is create-only.** `--dataSourceSlug`/`--dataSourceId` only applies on `create-webform`; `update-webform` has no rebind. Recreate to move a form to a different data source.
- **Two CLI casing traps:** in a raw `--data` payload, the data-source key is **`dataSourceId` (camelCase, not `tenant_data_source_id`)**, and the options key is **`options`** even though the flag is `--webformOptions`. Prefer the flags, which map correctly.
- **Public address = webhook id + token, not the webform UUID.** Use `form_url`/`form_submit_url`/`embed_code` from the response; don't construct URLs from the webform id.
- **Submissions create records asynchronously** (queued → edge function). Expect a short delay; a record isn't created in the same request. An empty/empty-object `data` POST is treated as a verification ping and stores nothing.
- **Unbound webforms are valid** — submissions go to the per-tenant `webform_record` table instead of a data source (queryable via the `webforms/{id}/items` API).
- **Submission fires automations:** an active automation `webform` trigger bound to this form (`--webformId`) runs on each submission (see **docyrus-automation-design**).
- **Validate then test.** Confirm the schema/binding, then submit once and confirm the record. Delete throwaway forms (and the records they created).

## Validate

1. `docyrus studio get-webform --webformId <id> --json` — `name`, `status`, `schema`, `tenant_data_source_id` (admin field) as intended; `form_url`/`form_submit_url`/`embed_code` present.
2. Validate `schema` and `options` against the copied machine schemas, then cross-check every `schema.children[].field.slug` against `docyrus studio list-fields --appSlug crm --dataSourceSlug leads --json` — each maps to a real field slug (unmapped slugs silently won't populate the record).
3. `docyrus studio list-webforms --appSlug crm --dataSourceSlug leads --json` — the form appears under its bound data source.

## Test

Prove a submission becomes a record:

1. **Submit** to the public endpoint (no auth needed — the id+token is the credential). Use `form_submit_url` from `get-webform`:
   ```bash
   curl -X POST "<form_submit_url>" -H 'Content-Type: application/json' \
     -d '{"data":{"name":"Jane Tester","email":"jane@example.com"}}'
   # → { "id": ..., "created_at": ... }   (queued; record is created asynchronously)
   ```
2. **Confirm the record** lands in the bound data source (allow a moment for the queue/edge function):
   ```bash
   docyrus ds list crm leads --columns "name, email" \
     --filters '{"rules":[{"field":"email","operator":"=","value":"jane@example.com"}]}' --json
   ```
   For an **unbound** form, read submissions via `docyrus curl "/v1/webforms/<webformId>/items"` instead.
3. **Clean up:** delete the test record(s), then `docyrus studio delete-webform --webformId <id> --json`.

> Submission → record runs through an async queue + edge function; in a local dev environment that pipeline may not execute, so the submit may return `{id}` without a record appearing. Treat a clean `create`/`get` (with a valid `form_submit_url`) and matching field slugs as the schema-correctness check, and run the full submission round-trip in a real environment.

Full submission/record lifecycle details and the `webform_record` items API are in [references/webform-model-and-submission.md](references/webform-model-and-submission.md). The current builder-generated `schema`/`options` contract is in [references/webform-design-schema.md](references/webform-design-schema.md).

## References

- **[references/webform-design-schema.md](references/webform-design-schema.md)** — Authoritative Webforms builder specification: persisted `schema` and `options`, all 30 palette types, validation/value contracts, derived `json_schema`, submission envelope, round-trip rules, and current renderer compatibility gaps.
- **[references/schemas/webform-builder.schema.json](references/schemas/webform-builder.schema.json)** — Draft 2020-12 schema for the persisted builder `schema` document.
- **[references/schemas/webform-options.schema.json](references/schemas/webform-options.schema.json)** — Draft 2020-12 schema for normalized persisted `options`.
- **[references/schemas/webform-design-output.schema.json](references/schemas/webform-design-output.schema.json)** — Documentation wrapper that references both persisted-document schemas.
- **[references/schemas/webform-submission-envelope.schema.json](references/schemas/webform-submission-envelope.schema.json)** — Static schema for `{ data, turnstileToken? }`; validate `data` against the form-specific derived `json_schema`.
- **[references/examples/webform-design.full.example.json](references/examples/webform-design.full.example.json)** — Complete example containing every native builder palette type.
- **[references/webform-model-and-submission.md](references/webform-model-and-submission.md)** — Backend webform model, CLI/API casing, public submission flow, bound vs unbound record mapping, response handles, automation linkage, and worked lifecycle example. For the current builder-generated JSON shape, defer to `webform-design-schema.md`.
- **docyrus-data-source-design** — the data source the form writes into (its field slugs = the form field slugs). **docyrus-automation-design** — the `webform` trigger that fires on submission. **docyrus-cli-app** — CLI command index; `docyrus studio …-webform --help` for flags.
