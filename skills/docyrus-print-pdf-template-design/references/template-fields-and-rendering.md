# Export Template Fields & Rendering

Reference for the HTML/PDF export-template fields and how a template is rendered to a document. Source of truth: `apps/cli/src/commands/studioCommands.ts` (`htmlTemplateFlags`, html cmds), `apps/api/src/dev/dto/html-template.dto.ts`, `apps/api/src/dev/html-template.service.ts`, `apps/api/src/app/template.service.ts` (render), `apps/api/src/app/constants.ts` (`EXPORT_SOURCE_TYPE`, `EXPORT_PAGE_ORIENTATION`), `apps/api/src/helpers/utils/templateCompiler.ts` (Handlebars). Render engine = **Handlebars** → HTML; PDF via a hosted html2pdf service.

## Table of Contents

1. [Field table](#field-table)
2. [source_type / page_orientation option sets](#source_type--page_orientation-option-sets)
3. [The render context (Handlebars)](#the-render-context-handlebars)
4. [Render endpoints](#render-endpoints)
5. [Worked example](#worked-example)
6. [Validation checklist](#validation-checklist)
7. [Test playbook](#test-playbook)

## Field table

Table `tenant_html_template`. CLI flags camelCase → snake_case API keys.

| CLI flag | API key | Required | Notes |
|---|---|---|---|
| `--name` | `name` | **yes** | |
| `--dataSourceId` / `--dataSourceSlug` | `tenant_data_source_id` | **yes** | mandatory binding (DB column `NOT NULL`) |
| `--body` | `body` | no | document HTML, Handlebars-compiled |
| `--headerTmpl` | `header_tmpl` | no | header HTML, Handlebars |
| `--footerTmpl` | `footer_tmpl` | no | footer HTML, Handlebars |
| `--styles` | `styles` | no | CSS |
| `--filenameTmpl` | `filename_tmpl` | no | Handlebars; default `<DataSourceName>-<autonumber_id>` |
| `--pageFormat` | `page_format` | no | free string: `A4`, `Letter`, … → passed to the PDF service as `format` |
| `--pageOrientation` | `page_orientation` | no | the renderer checks the literal string `"landscape"`; pass `portrait`/`landscape` (verify live) |
| `--marginTop/Bottom/Left/Right` | `margin_top/bottom/left/right` | no | numbers (pixels) |
| `--isDefault` | `is_default` | no | default template for the data source (default false) |
| `--sourceType` | `source_type` | no | **option-set UUID** (single- vs multi-record), NOT the format — see below |

Server-managed / not a CLI flag: `__body_html` (derived — **don't write**), `ownership` (always `CUSTOM`; not a flag, not returned — same as email templates), `page_orientation2` (unused uuid column), `archived`, audit columns. `list` omits `body`/`header_tmpl`/`footer_tmpl`/`styles`; `get` includes them.

## source_type / page_orientation option sets

The biggest trap: the flag help implies `--sourceType` picks the file format and `--pageOrientation` is a free string. In reality the **format is not chosen here at all** (it's the render endpoint), `source_type` is a `uuid` column, and `page_orientation` is a plain string. All three points below were verified live.

- **`page_orientation` is a plain string — pass `portrait`/`landscape`.** *Verified:* creating with `--pageOrientation portrait` stores and renders the literal `"portrait"`, and the PDF renderer checks `page_orientation === "landscape"`. The studio UI may model it as an option set, but the API stores the string. Use the slug string.
- **`source_type` is a `uuid` column** = single-record vs multi-record export mode (NOT the format). *Verified:* `--sourceType pdf` → `invalid input syntax for type uuid: "pdf"`. Known `EXPORT_SOURCE_TYPE` UUIDs (from `apps/api/src/app/constants.ts`), only if you genuinely need to set it:
  - `3c4d13d4-0ae7-4ca8-9b6c-be14f4ebdc0c` — single record ("Printing/Exporting only one record")
  - `5d5fa52f-02e8-4519-b374-4b4b0e37c0ae` — multiple records ("Printing/Exporting multiple records")
  - **Leave `--sourceType` unset** unless you mean single/multi and have the UUID.
- **The output format (HTML vs PDF) is decided by the render endpoint** (`/html` vs `/pdf`), not by any template field. There is **no DOCX renderer** in this module.

## The render context (Handlebars)

`body`, `header_tmpl`, `footer_tmpl`, and `filename_tmpl` are each compiled against the **expanded record** (the object `ds get --expand` returns), with fields at the **root**:

- `{{slug}}` — a field value; `{{name}}`, `{{autonumber_id}}`, `{{id}}` — built-ins.
- `{{field.name}}` — expanded enum/relation/user field (object), e.g. `{{customer.name}}`, `{{status.name}}`.
- `{{#each items}} … {{this.title}} … {{/each}}` — repeat over an array field (`inlineData`/child rows) for line-item tables.
- `{{#if field}} … {{/if}}` — conditional sections.
- Missing keys render **empty** (Handlebars tolerates them). `{{x}}` HTML-escapes; `{{{x}}}` emits raw HTML.

## Render endpoints

No CLI render command — use `docyrus curl` (path only; auth automatic):

| Purpose | Method + path |
|---|---|
| HTML for a record (verified ✓) | `GET /v1/apps/:appSlug/data-sources/:dataSourceSlug/items/:recordId/templates/:templateId/html` |
| PDF for a record | `GET /v1/apps/:appSlug/data-sources/:dataSourceSlug/items/:recordId/templates/:templateId/pdf` |
| Custom PDF (upload HTML body) | `PUT /v1/apps/:appSlug/data-sources/:dataSourceSlug/items/:recordId/renderPdfTemplate/:templateId` (multipart, `body` file, `text/html`) |

`:templateId` is **UUID-validated** — there is **no `default` keyword** (verified: `.../templates/default/html` → `invalid input syntax for type uuid: "default"`). To render the default template, resolve its id first with `list-html-templates --isDefault true`, then render by that id.

The HTML endpoint returns the compiled HTML — *verified*: `{{name}}`, `{{number_field}}`, expanded `{{enum.name}}`, and footer `{{autonumber_id}}` all resolved against a real record. The PDF endpoint compiles the HTML and POSTs it to the hosted html2pdf service with `landscape` (from `page_orientation === "landscape"`), `format` (= `page_format`), and margins; `filename_tmpl` is compiled (or falls back to `<DataSourceName>-<autonumber_id>`). **The html2pdf service can return 500 in local dev** — rely on `/html` for correctness and use `/pdf` in real environments.

## Worked example

A quote PDF with a line-items table, bound to `quotes`.

### 1. Inspect fields

```bash
docyrus studio list-fields --appSlug crm --dataSourceSlug quotes --json
# slugs: name (built-in), customer (relation → {name}), total, line_items (inlineData array of {title, qty, price})
```

### 2. Create the template

```bash
docyrus studio create-html-template --appSlug crm --dataSourceSlug quotes \
  --name "Quote PDF" \
  --body '<h1>Quote {{name}}</h1>
<p>Customer: {{customer.name}}</p>
<table><thead><tr><th>Item</th><th>Qty</th><th>Price</th></tr></thead>
<tbody>{{#each line_items}}<tr><td>{{this.title}}</td><td>{{this.qty}}</td><td>{{this.price}}</td></tr>{{/each}}</tbody></table>
<p class="total">Total: {{total}}</p>' \
  --styles 'h1{font-size:22px} table{width:100%;border-collapse:collapse} th,td{border:1px solid #ddd;padding:6px} .total{font-weight:bold;text-align:right}' \
  --footerTmpl '<div style="font-size:10px;text-align:center">{{name}} · {{autonumber_id}}</div>' \
  --pageFormat A4 --pageOrientation portrait \
  --marginTop 40 --marginBottom 40 --marginLeft 30 --marginRight 30 \
  --filenameTmpl "Quote-{{autonumber_id}}" --isDefault true --json
# → TEMPLATE_ID
```

### 3. Read it back

```bash
docyrus studio get-html-template --templateId TEMPLATE_ID --json
```

## Validation checklist

- [ ] `get-html-template` shows body/header/footer/styles, page options, margins, `filename_tmpl`, `is_default`, and `tenant_data_source_id` as authored.
- [ ] Every `{{slug}}` maps to a real field in `list-fields`; `{{x.name}}` is an expanded field; `{{#each y}}` is an array field.
- [ ] `source_type` is either unset or a real EXPORT_SOURCE_TYPE UUID (not `"pdf"`/`"html"`).
- [ ] `page_orientation` is `portrait`/`landscape` (string) — confirm the PDF honors it at render time.
- [ ] `list-html-templates --dataSourceSlug quotes` lists it (and as default if `is_default`).

## Test playbook

Render against a real record, then clean up.

1. **Create/reuse a record** with data that exercises the template (including the line-items array):
   ```bash
   docyrus ds create crm quotes --data '{"name":"ACME Quote","total":1500,"line_items":[{"title":"Widget","qty":3,"price":500}]}' --json
   # → RECORD_ID
   ```
2. **HTML render** — confirm placeholders + the `{{#each}}` table fill:
   ```bash
   docyrus curl "/v1/apps/crm/data-sources/quotes/items/RECORD_ID/templates/TEMPLATE_ID/html"
   ```
3. **PDF render** — hit the `/pdf` endpoint (orientation/format/margins/footer applied):
   ```bash
   docyrus curl "/v1/apps/crm/data-sources/quotes/items/RECORD_ID/templates/TEMPLATE_ID/pdf" --format json
   ```
   In real environments this returns a PDF; in **local dev the html2pdf service may 500** (`POST https://html2pdf.cf.docyrus.app/`) — treat a clean `/html` render as the pass condition.
4. **Default lookup** — if `is_default`, confirm `list-html-templates --isDefault true` returns this template, then render by its id (there is no `templates/default/...` path).
5. **Orientation check** — `update-html-template … --pageOrientation landscape`, re-render PDF, confirm the page turns landscape (this verifies the string-vs-UUID behavior live).
6. **Placeholder boundary** — a `{{nonexistent}}` renders empty, not an error.
7. **Clean up:**
   ```bash
   docyrus ds delete crm quotes RECORD_ID
   docyrus studio delete-html-template --templateId TEMPLATE_ID --json
   ```

Report: which placeholders/sections rendered, whether the PDF honored orientation/format, and confirmation the template + test record were removed.
