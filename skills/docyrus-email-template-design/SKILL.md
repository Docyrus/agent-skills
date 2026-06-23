---
name: docyrus-email-template-design
description: Design, validate, and test a Docyrus email template end-to-end using the `docyrus studio` email-template CLI commands. Use when the user wants a reusable, data-bound email — a templated subject + HTML/text body with record placeholders ({{field}} Handlebars) that an automation send-email node (or a manual send) fills from a record. Covers writing the subject/body with the record context, optionally binding the template to a data source, and proving the template renders/sends. Triggers on "create an email template", "design a templated email", "email with record fields merged in", "notification email body", "studio create-email-template", or any email-template authoring + validation task in Docyrus. For HTML/PDF print/report templates use docyrus-print-pdf-template-design; for the automation that sends the email use docyrus-automation-design.
---

# Docyrus Email Template Design

Design a reusable email template with `docyrus studio create-email-template`, then **validate** its shape and **test** that it renders/sends with real record data. An email template is a row in `tenant_email_template`: a **subject** + **body**, optionally bound to a data source, with `{{field}}` placeholders filled at send time. Templates are **consumed** by the automation `send-email` action node (and other send paths) — they don't send themselves.

## Workflow

1. **Confirm app + auth, and the source record shape.** A template's placeholders must match real record fields.
   ```bash
   docyrus auth who --json
   docyrus apps list --json
   docyrus studio list-data-sources --appSlug crm --json     # the data source whose records will fill the email (optional binding)
   docyrus studio list-fields --appSlug crm --dataSourceSlug deals --json   # the field slugs you can interpolate
   ```

2. **Design the subject + body.** Decide the merge fields (`{{slug}}`), what comes from expanded relations/enums (`{{field.name}}`), and any conditional/looping sections. Keep the body as HTML. See [references/templating.md](references/templating.md) for the Handlebars context and helpers.

3. **Create the template** (`create-email-template`). `--name` and `--subject` are required; `--body` is the HTML; optionally bind a data source with `--dataSourceSlug`/`--dataSourceId`. See [Create](#create-an-email-template).

4. **Validate** — read the template back and confirm subject/body/binding landed. See [Validate](#validate).

5. **Test** — render/send it against a real record (via an automation `send-email` node, or a manual `messaging email send`) and confirm placeholders resolve. See [Test](#test).

A worked example (a "deal won" notification email bound to Deals) with validation and a send test is in [references/templating.md](references/templating.md#worked-example).

## Command cheat-sheet

Selectors: `--appId | --appSlug` (only used to resolve a slug), `--dataSourceId | --dataSourceSlug` (the optional binding), `--templateId` (the template itself). Write commands take camelCase flags **or** `--data`/`--from-file` (flags merge over JSON; API keys snake_case). Append `--json`.

### Create an email template

```bash
docyrus studio create-email-template --appSlug crm --dataSourceSlug deals \
  --name "Deal won notification" \
  --subject "🎉 {{name}} is closed-won" \
  --body '<h1>Nice work!</h1><p>{{name}} closed at {{amount}} ({{stage.name}}).</p>' --json
# → capture data.id as TEMPLATE_ID
```

- **`--name` and `--subject` are required.** Omitting either → HTTP 422.
- `--body` is the email body (HTML). It's optional at the API level, but a template with no body isn't useful.
- **Binding to a data source is optional** (`--dataSourceSlug`/`--dataSourceId` → `tenant_data_source_id`). Bind it when the template is for one entity's records (recommended — it documents intent and powers field validation in the UI). Leave unbound for generic emails.
- **Ownership is always `CUSTOM`** — it's set server-side, not a flag, and isn't returned. (Same as print/PDF templates.)

### Manage / inspect

```bash
docyrus studio list-email-templates --appSlug crm --dataSourceSlug deals --json   # filter by bound data source; omit to list all tenant templates
docyrus studio get-email-template    --templateId TEMPLATE_ID --json               # full body + __body_html
docyrus studio update-email-template --templateId TEMPLATE_ID --subject "New subject" --json   # PUT, but partial
docyrus studio delete-email-template --templateId TEMPLATE_ID --json               # soft delete (archived=true), 204
```

- `get` returns the full record incl. `body` and `__body_html`; `list` omits the body fields (metadata only).
- **`update` is HTTP PUT but behaves as a partial update** — only fields you pass are changed.
- **`delete` is a soft delete** (sets `archived=true`) returning 204; the CLI prints a deletion envelope.

## Critical rules

- **`name` + `subject` are the only required fields.** Everything else is optional.
- **Ownership is not settable** — every template is `CUSTOM` (server-set, not exposed as a flag or in responses). This mirrors print/PDF templates.
- **Data-source binding is optional** for email templates (unlike print/PDF templates, where it's required).
- **There is no `from`/`to`/`cc`/`reply_to`/`fromName` on the template.** Sender and recipients are set by the **send path** (the automation `send-email` node, or `messaging email send`), not the template. The template owns only `subject` + `body`.
- **Don't write `__body_html` directly.** It's a separate stored HTML column the CRUD services never write — the server/UI derives it. Author `body`; treat `__body_html` as read-only on `get`.
- **Unknown JSON keys are silently ignored** (`whitelist:false`). Putting `from`, `html`, `to`, `type`, etc. in `--data` neither errors nor does anything — they're dropped. Read the template back to confirm what stuck.
- **Placeholders are Handlebars `{{field}}`** over the **expanded record** at send time: bare fields `{{slug}}`, built-ins `{{name}}`/`{{autonumber_id}}`, expanded enum/relation/user fields as objects `{{stage.name}}`. Not JSONata, not `${}`. See [references/templating.md](references/templating.md).
- **The template doesn't send itself.** Wire it into an automation `send-email` node (`data.template:{id}`) or send via `messaging`. Validate by reading back; **test** by rendering through a send path.
- **Validate then test, every time.** Confirm placeholders match real field slugs, then send one against a real record. Delete throwaway templates you create.

## Validate

1. `docyrus studio get-email-template --templateId TEMPLATE_ID --json` — `name`, `subject`, `body` correct; `tenant_data_source_id` set if you bound one.
2. Cross-check every `{{placeholder}}` in subject+body against `docyrus studio list-fields --appSlug crm --dataSourceSlug <bound-ds> --json` — each bare `{{slug}}` is a real field; each `{{x.name}}` is an expanded enum/relation/user field. A placeholder with no matching field renders empty (Handlebars doesn't error on missing keys).
3. `docyrus studio list-email-templates --appSlug crm --dataSourceSlug <ds> --json` — the template appears under its bound data source.

Checklist detail in [references/templating.md](references/templating.md#validation-checklist).

## Test

Email templates have **no standalone render endpoint** — exercise them through a send path:

1. **Via automation (recommended):** create a `send-email` node referencing the template (`--data '{"data":{"template":{"id":"TEMPLATE_ID"},"sender_type":"system"},"field_mapping":{"to":"<recipient field>"}}'`) on an automation whose trigger you can fire (**docyrus-automation-design**), then fire it against a real record and inspect the sent/preview email: every `{{placeholder}}` should be filled.
2. **Via messaging:** `docyrus messaging email send …` against a real record context to confirm the rendered subject/body.
3. **Placeholder boundary:** include one `{{nonexistent_field}}` in a throwaway copy and confirm it renders empty (not an error) — this is why validating slugs against `list-fields` matters.
4. **Clean up:** `docyrus studio delete-email-template --templateId TEMPLATE_ID --json` and remove any throwaway automation/records.

Full send-test playbook in [references/templating.md](references/templating.md#test-playbook).

## References

- **[references/templating.md](references/templating.md)** — The Handlebars context (record fields, expanded objects, built-ins, helpers), authoring patterns for email bodies, the worked example, validation checklist, and the send-test playbook.
- **docyrus-automation-design** — the `send-email` action node that consumes the template. **docyrus-print-pdf-template-design** — HTML/PDF print/report templates (the sibling `studio *-html-template` flow). **docyrus-cli-app** → `references/cli-manifest.md` (flag tables) and the `messaging` section (`messaging email send`). **docyrus-platform** → `references/integrations-and-events.md` (email system concepts).
