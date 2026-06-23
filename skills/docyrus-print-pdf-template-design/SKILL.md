---
name: docyrus-print-pdf-template-design
description: Design, validate, and test a Docyrus HTML/PDF print/export (report) template end-to-end using the `docyrus studio` html-template CLI commands. Use when the user wants a printable/exportable document rendered from a record — an invoice, quote, receipt, report, certificate, packing slip — authored as HTML + CSS with {{field}} Handlebars placeholders, with page setup (orientation, format, margins, header/footer, filename) and a data-source binding, then rendered to HTML/PDF for a record. Covers writing the body/header/footer/styles, the page options, marking a default template, and rendering it against a real record to prove it works. Triggers on "create a PDF/print template", "design an invoice/quote/report template", "printable document from a record", "export to PDF", "studio create-html-template", or any print/PDF report-template authoring + validation task. For emails use docyrus-email-template-design; for the automation that generates a document use docyrus-automation-design.
---

# Docyrus Print / PDF Template Design

Design a printable, data-bound report template with `docyrus studio create-html-template`, then **validate** its shape and **test** that it renders to HTML/PDF for a real record. An export template is a row in `tenant_html_template`: an HTML **body** (+ optional header/footer/CSS) with `{{field}}` Handlebars placeholders, **bound to a data source**, with page setup. Rendering is done via the app's render endpoints (reachable with `docyrus curl`).

## Workflow

1. **Confirm app + auth, and the source record shape.** An export template **must** bind to a data source, and its placeholders must match real fields.
   ```bash
   docyrus auth who --json
   docyrus apps list --json
   docyrus studio list-data-sources --appSlug crm --json     # → the data source to bind (required)
   docyrus studio list-fields --appSlug crm --dataSourceSlug quotes --json   # the field slugs to interpolate
   ```

2. **Design the document.** Decide the layout (HTML `body`), repeating sections (`{{#each}}` over an `inlineData`/child array), header/footer HTML, CSS in `styles`, the page setup (orientation, format, margins), and the output filename pattern (`filename_tmpl`). See [references/template-fields-and-rendering.md](references/template-fields-and-rendering.md).

3. **Create the template** (`create-html-template`). `--name` and a data source (`--dataSourceSlug`/`--dataSourceId`) are **required**. Add body/header/footer/styles and page options. See [Create](#create-an-export-template).

4. **Validate** — read it back and confirm content, page options, and binding landed. See [Validate](#validate).

5. **Test** — render it to HTML and PDF for a real record via the app endpoints and confirm placeholders resolve and the page renders. See [Test](#test).

A worked example (a quote/invoice template with a line-items table, rendered to PDF) is in [references/template-fields-and-rendering.md](references/template-fields-and-rendering.md#worked-example).

## Command cheat-sheet

Selectors: `--appId | --appSlug` (resolves a slug), `--dataSourceId | --dataSourceSlug` (the **required** binding), `--templateId` (the template). Write commands take camelCase flags **or** `--data`/`--from-file` (flags merge over JSON; API keys snake_case). Append `--json`.

### Create an export template

```bash
docyrus studio create-html-template --appSlug crm --dataSourceSlug quotes \
  --name "Quote PDF" \
  --body '<h1>{{name}}</h1><p>Total: {{total}}</p>' \
  --styles 'h1{font-size:22px} table{width:100%;border-collapse:collapse}' \
  --headerTmpl '<div style="font-size:10px">{{name}}</div>' \
  --footerTmpl '<div style="font-size:10px;text-align:center">Page</div>' \
  --pageFormat A4 --pageOrientation portrait \
  --marginTop 40 --marginBottom 40 --marginLeft 30 --marginRight 30 \
  --filenameTmpl "{{name}}-{{autonumber_id}}" --isDefault true --json
# → capture data.id as TEMPLATE_ID
```

- **`--name` and a data source are required.** Unlike email templates, the binding (`tenant_data_source_id`) is **mandatory** — omitting it → HTTP 422. Resolve the data source first.
- `--body` is the document HTML; `--headerTmpl`/`--footerTmpl` are header/footer HTML; `--styles` is CSS. All four are Handlebars-compiled against the record.
- Page setup: `--pageFormat` (free string, e.g. `A4`, `Letter`), `--pageOrientation` (use `portrait`/`landscape` — the PDF renderer checks the literal string `"landscape"`), `--marginTop/Bottom/Left/Right` (numbers), `--filenameTmpl` (Handlebars; defaults to `<DataSourceName>-<autonumber_id>`).
- `--isDefault true` marks this the default template for the data source (rendered when a caller asks for template `default`).
- **`--sourceType` is NOT the file format.** `source_type` is an **option-set UUID** selecting single- vs multi-record export, not the format. The format is chosen by the **render endpoint** you call (`/html` vs `/pdf`). See [the field reference](references/template-fields-and-rendering.md#source_type--page_orientation-option-sets). Leave `--sourceType` unset unless you have a real option UUID.

### Manage / inspect

```bash
docyrus studio list-html-templates --appSlug crm --dataSourceSlug quotes --json   # filter by data source; --isDefault to filter defaults
docyrus studio get-html-template    --templateId TEMPLATE_ID --json                # full body/header/footer/styles
docyrus studio update-html-template --templateId TEMPLATE_ID --pageOrientation landscape --json   # PUT, but partial
docyrus studio delete-html-template --templateId TEMPLATE_ID --json                # soft delete (archived=true), 204
```

- `get` returns full content; `list` omits `body`/`header_tmpl`/`footer_tmpl`/`styles` (metadata only).
- **`update` is PUT but partial** — only passed fields change.
- **`delete` is soft** (`archived=true`), returns 204.

## Render endpoints (how a template becomes a document)

There is **no CLI render command** — render through the app endpoints with `docyrus curl` (path only; auth automatic). The `:templateId` segment **must be a real template UUID** (there is no `default` keyword — the param is UUID-validated):

```bash
# Render HTML for one record (verified to work locally):
docyrus curl "/v1/apps/crm/data-sources/quotes/items/<recordId>/templates/TEMPLATE_ID/html"
# Render PDF (compiles HTML, POSTs it to the hosted html2pdf service):
docyrus curl "/v1/apps/crm/data-sources/quotes/items/<recordId>/templates/TEMPLATE_ID/pdf" --format json
# To render the DEFAULT template, first resolve its id, then render by that id:
docyrus studio list-html-templates --appSlug crm --dataSourceSlug quotes --isDefault true --json
```

The HTML endpoint returns the compiled HTML (verified). The PDF endpoint compiles the HTML and POSTs it to the hosted html2pdf microservice — **PDF generation depends on that external service and can 500 in local dev**, so use the `/html` render as your reliable correctness check and reserve `/pdf` for real environments. **DOCX has no render path** — only HTML and PDF. See [references/template-fields-and-rendering.md](references/template-fields-and-rendering.md#render-endpoints).

## Critical rules

- **`name` + `tenant_data_source_id` are required.** The data-source binding is mandatory (resolve it before creating) — this is the key difference from email templates.
- **`source_type` is a UUID column; the output format is the render endpoint, not a field.** `source_type` selects single- vs multi-record export (a `uuid`), NOT "html"/"pdf"/"docx" — verified: passing `--sourceType pdf` fails with `invalid input syntax for type uuid: "pdf"`. The format is decided by which endpoint you hit (`/html` vs `/pdf`). **Leave `--sourceType` unset** unless you have a real EXPORT_SOURCE_TYPE UUID (listed in the field reference).
- **`page_orientation` is a plain string — pass `portrait`/`landscape`.** Verified: the value is stored and rendered as the literal string (the PDF renderer checks `page_orientation === "landscape"`). Despite the studio UI modelling it as an option set, the API accepts and stores the string — use `portrait`/`landscape`.
- **No DOCX renderer exists.** Only `/html` and `/pdf` endpoints render — there is no DOCX output path. Don't promise DOCX from these endpoints.
- **Ownership is always `CUSTOM`** — server-set, not a flag, not returned (same as email templates).
- **Body/header/footer/styles/filename are all Handlebars** over the **expanded record**: `{{slug}}`, `{{name}}`, `{{autonumber_id}}`, expanded `{{field.name}}`, and `{{#each arrayField}}…{{/each}}` for line items. Missing keys render empty (no error). HTML is escaped by `{{x}}` — use `{{{x}}}` for intentional raw HTML.
- **Unknown JSON keys are silently ignored** (`whitelist:false`). A mistyped key in `--data` neither errors nor takes effect; sending `source_type:"pdf"` (a non-UUID string) passes DTO validation but can fail at the DB uuid column. Read back to confirm.
- **`update` is PUT-but-partial; `delete` is soft (204).** `__body_html` is server-derived — **don't write it**.
- **Render is a separate endpoint, not a studio command.** Validate via `get`; **test** by curling the `/html` and `/pdf` render endpoints against a real record.
- **Validate then test, every time.** Confirm placeholders match real slugs, then render against a real record. Delete throwaway templates you create.

## Validate

1. `docyrus studio get-html-template --templateId TEMPLATE_ID --json` — `name`, `body`, `header_tmpl`, `footer_tmpl`, `styles`, `page_format`, `page_orientation`, margins, `filename_tmpl`, `is_default`, and `tenant_data_source_id` all as intended.
2. Cross-check every `{{placeholder}}` against `docyrus studio list-fields --appSlug crm --dataSourceSlug <bound-ds> --json` — bare `{{slug}}` = real field; `{{x.name}}` = expanded field; `{{#each y}}` = an array field.
3. `docyrus studio list-html-templates --appSlug crm --dataSourceSlug <ds> --json` — the template appears under its binding (and as default if `is_default`).

Checklist detail in [references/template-fields-and-rendering.md](references/template-fields-and-rendering.md#validation-checklist).

## Test

Render against a **real record** (create a throwaway one if needed):

1. **HTML render:**
   ```bash
   docyrus ds create crm quotes --data '{"name":"ACME Quote","total":1500}' --json   # → RECORD_ID (or reuse a real one)
   docyrus curl "/v1/apps/crm/data-sources/quotes/items/RECORD_ID/templates/TEMPLATE_ID/html"
   ```
   Confirm every `{{placeholder}}` is filled and the markup is well-formed.
2. **PDF render:** hit the `/pdf` endpoint. In real environments this returns a PDF; in **local dev the external html2pdf service may 500** — that's an environment limit, not a template error, so treat a clean `/html` render as the pass condition.
3. **Default lookup:** if `is_default`, confirm `list-html-templates --isDefault true` returns this template (there is no `templates/default/...` render path — render by the resolved id).
4. **Placeholder boundary:** include a `{{nonexistent}}` in a throwaway copy and confirm it renders empty (not an error).
5. **Clean up:** delete the throwaway record(s) and `docyrus studio delete-html-template --templateId TEMPLATE_ID --json`.

Full render/test playbook in [references/template-fields-and-rendering.md](references/template-fields-and-rendering.md#test-playbook).

## References

- **[references/template-fields-and-rendering.md](references/template-fields-and-rendering.md)** — Full field table (with the `source_type`/`page_orientation` option-set UUIDs and the format gotcha), the Handlebars context, the render endpoints (`/html`, `/pdf`, custom-PDF upload, `default`), a worked invoice/quote example, validation checklist, and the render test playbook.
- **docyrus-automation-design** — the `generate-document` action node that renders a template against a record inside a workflow. **docyrus-email-template-design** — the sibling email-template flow. **docyrus-cli-app** → `references/cli-manifest.md` (flag tables). **docyrus-api-dev** — REST client + its HTML-to-PDF helper.
