# Email Template Authoring & Testing

Reference for writing Docyrus email-template subjects/bodies and proving they render. Source of truth: `apps/cli/src/commands/studioCommands.ts` (`emailTemplateFlags`, email cmds), `apps/api/src/dev/dto/email-template.dto.ts`, `apps/api/src/dev/email-template.service.ts`, `apps/api/src/helpers/utils/templateCompiler.ts` (Handlebars). The render engine for the data-bound paths is **Handlebars** (`Handlebars.compile`).

## Table of Contents

1. [The render context](#the-render-context)
2. [Handlebars patterns](#handlebars-patterns)
3. [Data model](#data-model)
4. [Worked example](#worked-example)
5. [Validation checklist](#validation-checklist)
6. [Test playbook](#test-playbook)

## The render context

At send time the template is compiled against the **expanded record** — the same object `ds get`/`ds list --expand` returns. The record's fields are at the **root** of the context:

- **Bare field:** `{{slug}}` → the field's value (e.g. `{{amount}}`, `{{email}}`).
- **Built-ins:** `{{name}}` (record title), `{{description}}` (notes), `{{autonumber_id}}`, `{{id}}`, `{{created_on}}`.
- **Expanded enum/relation/user fields are nested objects** (`{id, name, …}`) → use a sub-path: `{{stage.name}}`, `{{account.name}}`, `{{owner.name}}`. A bare `{{stage}}` would render the object/UUID, not the label.
- **Missing keys render empty** — Handlebars does not error on an unknown placeholder. This is why you validate placeholders against `list-fields` rather than relying on a runtime failure.

## Handlebars patterns

Standard Handlebars (`{{ }}`), not JSONata or `${}`:

```handlebars
Hi {{owner.name}},

The deal {{name}} ({{autonumber_id}}) is now {{stage.name}}.

{{#if amount}}Value: {{amount}}{{/if}}

{{#each line_items}}
  - {{this.title}}: {{this.price}}
{{/each}}
```

- `{{#if field}} … {{/if}}` for conditional sections; `{{#each arrayField}} … {{this.sub}} … {{/each}}` for repeaters (e.g. an `inlineData` array field).
- HTML is allowed in `body`; use it for layout/styling. Handlebars HTML-escapes `{{x}}` by default — use triple-stache `{{{x}}}` only for values you intentionally want as raw HTML.
- Keep merge fields to slugs that exist on the bound data source; expanded sub-objects need the data to actually be expanded at send time (the send path expands User/Enum/Relation).

## Data model

Table `tenant_email_template`. Fields you control: `name` (required), `subject` (required), `body` (HTML, optional), `tenant_data_source_id` (optional binding). Server-managed: `ownership` (always `CUSTOM`, not a flag, not returned), `__body_html` (derived HTML — **never write it**), `archived`, audit columns. **No sender/recipient fields exist on the template** — those live on the send path.

| CLI flag | API key | Required | Notes |
|---|---|---|---|
| `--name` | `name` | yes | |
| `--subject` | `subject` | yes | Handlebars-compiled |
| `--body` | `body` | no | HTML, Handlebars-compiled |
| `--dataSourceId` / `--dataSourceSlug` | `tenant_data_source_id` | no | optional binding |

## Worked example

A "deal won" email bound to the `deals` data source.

### 1. Inspect the fields you'll merge

```bash
docyrus studio list-fields --appSlug crm --dataSourceSlug deals --json
# confirm slugs: name (built-in), amount, stage (status/enum → {name}), owner (userSelect → {name})
```

### 2. Create the template

```bash
docyrus studio create-email-template --appSlug crm --dataSourceSlug deals \
  --name "Deal won notification" \
  --subject "🎉 {{name}} is closed-won" \
  --body '<h2>Congrats {{owner.name}}!</h2><p>The deal <strong>{{name}}</strong> ({{autonumber_id}}) closed at {{amount}} — stage {{stage.name}}.</p>' --json
# → TEMPLATE_ID
```

### 3. Read it back

```bash
docyrus studio get-email-template --templateId TEMPLATE_ID --json
```

## Validation checklist

- [ ] `get-email-template` shows `name`, `subject`, `body` exactly as authored; `tenant_data_source_id` set (if bound).
- [ ] Every `{{slug}}` in subject+body maps to a real field in `list-fields` for the bound data source.
- [ ] Every `{{x.name}}`/`{{x.sub}}` corresponds to an **expanded** field type (status/enum/relation/userSelect) — bare scalars don't have sub-paths.
- [ ] No stray sender/recipient keys were attempted via `--data` (they'd be silently dropped).
- [ ] `list-email-templates --dataSourceSlug deals` lists the template under its binding.

## Test playbook

No standalone render endpoint exists — test through a send path, then clean up.

1. **Wire into an automation `send-email` node** and fire it (see **docyrus-automation-design**):
   ```bash
   docyrus automation create-node --appSlug crm --automationId <auto-id> \
     --type send-email --name "Send won email" \
     --data '{"data":{"sender_type":"system","template":{"id":"TEMPLATE_ID"}},"field_mapping":{"to":"owner"}}' --json
   ```
   Create/modify a real deal so the trigger fires; inspect the sent/preview email — `{{name}}`, `{{amount}}`, `{{stage.name}}`, `{{owner.name}}` should all be filled.

2. **Or send via messaging** against a real record context: `docyrus messaging email send …` (see docyrus-cli-app messaging section), and confirm the rendered subject/body.

3. **Placeholder boundary:** add `{{does_not_exist}}` to a throwaway copy; confirm it renders empty (no error) — proves missing-key tolerance and why slug validation matters.

4. **Clean up:**
   ```bash
   docyrus studio delete-email-template --templateId TEMPLATE_ID --json
   # + remove any throwaway automation/nodes/records created for the send test
   ```

Report: which fields merged correctly, whether expanded objects (`stage.name`, `owner.name`) resolved, and confirmation the template + test scaffolding were removed.
