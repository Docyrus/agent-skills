---
name: docyrus-webform-design
description: Use when designing, publishing, embedding, or testing a generic Docyrus webform in Webforms Studio or with docyrus studio. Covers public intake, contact, lead, and survey forms, data-source binding, builder schema and options, hosted links, CAPTCHA, server submissions, and submission-triggered automations.
---

# Docyrus Webform Design

Design a generic Docyrus webform in Webforms Studio or with `docyrus studio`. A completed submission creates a record in the bound data source or, for an unbound form, in the tenant's webform records. Either can trigger an automation.

## Workflow

1. **Confirm the destination.** For a bound form, identify the app and data source and list its fields. Each answer slug must match the destination field slug. For an unbound form, choose stable slugs and review responses in Studio's per-form Submissions view.

   ```bash
   docyrus auth who --json
   docyrus apps list --json
   docyrus studio list-fields --appSlug crm --dataSourceSlug leads --json
   ```

2. **Design and validate.** Use separate `schema` and `options` documents. Follow the [builder specification](references/webform-design-schema.md) and check renderer support for the fields and layout you choose.
3. **Create and verify.** Use [Create a webform](#create-a-webform), then read back the design and binding with `get-webform`. Draft is `0`, active is `1`, and inactive is `2`. Publish when the form is ready for respondents.
4. **Share.** Copy the hosted link from **Publish** and the website embed from **Preview** (`#preview`). The hosted page uses `/w/<workspaceSlug>/<formSlug>` when both slugs exist, or `/s/<formId>/<formToken>` using the paired webhook's short ID and token. See [publishing and embed](references/webform-model-and-submission.md#publishing-and-embed) for the API handles and address behavior.
5. **Test the respondent journey.** Open the live page or the embed on an allowed website, complete the CAPTCHA, submit, and verify the queued record. For a server integration, use the form's [server-to-server URL](references/webform-model-and-submission.md#server-to-server-submission).

## Builder schema specification (read when authoring JSON)

Use the [Webforms builder specification](references/webform-design-schema.md) when authoring Studio-compatible JSON.

1. Read [references/webform-design-schema.md](references/webform-design-schema.md) for `schema`, `options`, all 30 palette field types, validation tokens, value shapes, Studio-side `json_schema`, submission behavior, and renderer compatibility notes.
2. Validate the two persisted documents independently with [references/schemas/webform-builder.schema.json](references/schemas/webform-builder.schema.json) and [references/schemas/webform-options.schema.json](references/schemas/webform-options.schema.json).
3. Use [references/schemas/webform-design-output.schema.json](references/schemas/webform-design-output.schema.json) only for a documentation/test wrapper containing both documents; the API stores them as separate properties.
4. Validate the public POST wrapper with [references/schemas/webform-submission-envelope.schema.json](references/schemas/webform-submission-envelope.schema.json), then check `data` against the intended field slugs and values. See [derived schemas](references/webform-design-schema.md#9-derived-json_schema) before using `json_schema` for local validation.
5. Use [references/examples/webform-design.full.example.json](references/examples/webform-design.full.example.json) as the exhaustive palette example. Reduce it to the fields the user needs; do not copy unrelated fields into a production form.

The native builder shape is `{ "version": 1, "children": [{ "type", "field", "enumOptions"?, "layout" }] }`. The runtime supplies the submit button and CAPTCHA.

## Command cheat-sheet

Use `--appId` or `--appSlug` to resolve a data-source slug. Write commands accept camelCase flags and JSON input via `--data` or `--from-file`. Flags override matching JSON fields.

Use the admin UUID for `get`, `update`, and `delete`. The optional public slug belongs to a brand workspace. `list-webforms` can filter by data source.

### Create a webform

```bash
docyrus studio create-webform --appSlug crm --dataSourceSlug leads \
  --name "Website lead form" --status 0 \
  --schema '{"version":1,"children":[
    {"type":"field-text","field":{"id":"lead-name","name":"Name","slug":"name","type":"field-text","validations":["required"]},"layout":{"colSpan":1,"rowSpan":1}},
    {"type":"field-email","field":{"id":"lead-email","name":"Email","slug":"email","type":"field-email","validations":["required"]},"layout":{"colSpan":1,"rowSpan":1}}
  ]}' --json
# Capture data.id (the admin UUID) for subsequent commands.
```

- Create requires `name`, a JSON object for `schema`, and an explicit `status` (0, 1, or 2). The Webforms Studio UI defaults to draft.
- Binding via `dataSourceId` is optional and create-only. Use `--dataSourceSlug` with the app selector to resolve it. Recreate the form to change the destination data source.
- In `--data` or `--from-file`, use `dataSourceId` and `options`. The corresponding options flag is `--webformOptions`. Use a JSON payload for API fields with no dedicated flag, such as `slug`, `tenantBrandId`, and `allowedOrigins`.

### Manage and inspect

```bash
docyrus studio list-webforms --appSlug crm --dataSourceSlug leads --json
docyrus studio get-webform --webformId <id> --json
docyrus studio update-webform --webformId <id> --status 1 --json
```

Updates are partial. Confirm `name`, `status`, `schema`, `options`, and `tenant_data_source_id` in the returned record. For a bound form, cross-check its answer slugs with `list-fields`. Keep existing slugs and choice IDs stable to preserve response mappings.

## Test

1. For a respondent test, open the live hosted page or the embed on an allowed origin. Complete the CAPTCHA and submit a uniquely identifiable response on a test form. An accepted submission is queued for record creation. An empty `data` ping stores no record.
2. Verify a bound submission in the destination data source after the queue processes it.

   ```bash
   docyrus ds list crm leads --columns "name,email" \
     --filters '{"rules":[{"field":"email","operator":"=","value":"jane@example.com"}]}' --json
   ```

   For an unbound form, use the per-form Submissions view or `docyrus curl "/v1/webforms/<webformId>/items"`.
3. If the form is connected to an automation, verify a run for the test submission. Use **docyrus-automation-design** to configure a `webform` trigger.

## References

- **[references/webform-design-schema.md](references/webform-design-schema.md)** — Authoritative Webforms builder specification: persisted `schema` and `options`, all 30 palette types, validation/value contracts, derived `json_schema`, submission envelope, round-trip rules, and current renderer compatibility gaps.
- **[references/schemas/webform-builder.schema.json](references/schemas/webform-builder.schema.json)** — Draft 2020-12 schema for the persisted builder `schema` document.
- **[references/schemas/webform-options.schema.json](references/schemas/webform-options.schema.json)** — Draft 2020-12 schema for normalized persisted `options`.
- **[references/schemas/webform-design-output.schema.json](references/schemas/webform-design-output.schema.json)** — Documentation wrapper that references both persisted-document schemas.
- **[references/schemas/webform-submission-envelope.schema.json](references/schemas/webform-submission-envelope.schema.json)** — Static schema for `{ data, captchaToken? }`, including the legacy token alias.
- **[references/examples/webform-design.full.example.json](references/examples/webform-design.full.example.json)** — Complete example containing every native builder palette type.
- **[references/webform-model-and-submission.md](references/webform-model-and-submission.md)** — Backend webform model, CLI/API casing, public submission flow, bound vs unbound record mapping, publishing, CAPTCHA, server submission, and automation linkage. For the current builder-generated JSON shape, defer to `webform-design-schema.md`.
- **docyrus-data-source-design** — the data source the form writes into (its field slugs = the form field slugs). **docyrus-automation-design** — the `webform` trigger that fires on submission. **docyrus-cli-app** — CLI command index; `docyrus studio …-webform --help` for flags.
