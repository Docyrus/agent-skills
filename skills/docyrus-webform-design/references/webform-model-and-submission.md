# Webform Model & Submission Flow

Reference for the Docyrus webform data model, schema, and the public submission → record pipeline. Source of truth: `apps/cli/src/commands/studioCommands.ts` (webform commands + `webformFlags`), `apps/api/src/dev/dto/webform.ts`, `apps/api/src/dev/app/webform.service.ts`, `apps/api/src/webform/*` (public controller/service/entity), `libs/shared/src/edge/queue-webhook/index.ts`, `database/.../tenant_webform.sql`.

## Table of Contents

1. [Data model & casing](#data-model--casing)
2. [The `schema` shape](#the-schema-shape)
3. [Public submission flow](#public-submission-flow)
4. [Response handles (URLs / embed)](#response-handles-urls--embed)
5. [Automation `webform` trigger](#automation-webform-trigger)
6. [Worked example](#worked-example)
7. [Validation checklist](#validation-checklist)
8. [Gotchas](#gotchas)

## Data model & casing

Table `public.tenant_webform` (paired 1:1 with a `tenant_webhook`). CLI flags → API body keys:

| CLI flag | API/body key | Required | Notes |
|---|---|---|---|
| `--name` | `name` | **yes** | non-empty string |
| `--schema` | `schema` | **yes** | non-empty **JSON object** (`@IsObject`) |
| `--status` | `status` | **yes** | `@IsIn([1,2])` — **1 = active, 2 = inactive** |
| `--dataSourceId` / `--dataSourceSlug` | `dataSourceId` | no (create-only) | ⚠️ stays **camelCase** in `--data`; not `tenant_data_source_id` |
| `--webformOptions` | `options` | no | ⚠️ flag is `--webformOptions`, body key is `options`; opaque builder config |
| `--sandbox` | `sandbox` | no | marks a test form; defaults `false`; drives `form_sandbox_url` |
| `--css` | `css` | no | custom CSS injected into the rendered form |

Server-managed (not settable via these commands): `tenant_webhook_id`, analytics/tracker fields (`ga4_event`, `docyrus_tracker_*`, `gads_*`, `gtag_*`), `archived`, audit columns. **Webforms have no slug** — addressed by `--webformId` (UUID). `update-webform` (PATCH) accepts the same fields **except `dataSourceId`** (binding is create-only).

## The `schema` shape

`schema` is the form's field/layout definition (jsonb). It is authored by the form-builder UI; the backend only requires a non-empty JSON object. The shape the public renderer expects (inferred from `webform.service.ts`):

```jsonc
{
  "children": [
    { "type": "<FieldComponent>", "options": { "key": "<dataSourceFieldSlug>", "label": "..." } },
    // ...more field components...
    { "type": "KvButton", "options": { "action": "submit", "label": "Send" } }
  ]
}
```

- `children` is an ordered array of form components plus a **submit button** (`type: "KvButton"`, `options.action: "submit"`). The renderer injects a Cloudflare Turnstile component before the submit button.
- **Each input component's `options.key` must equal a field slug** on the bound data source — that key becomes the submission `data` key, which the data source maps to the record column by slug.
- The concrete field-component vocabulary (`FieldText`, etc.) lives in the form-builder UI, not this repo — when in doubt, build the form in the UI and read its `schema` back with `get-webform`, or keep keys aligned to `list-fields` slugs.

## Public submission flow

A webform is addressed publicly by its **paired webhook's short id + token**, never the webform UUID.

- **Render:** `GET /v1/webforms/{id}/{token}` → `form_url`.
- **Submit:** `POST /v1/webforms/{id}/{token}` with body `{ "data": { …field values keyed by slug… } }` → `form_submit_url`. Auth = the unguessable id+token (a `WebhookGuard`, no OAuth login). Returns `201 { id, created_at }`.

What happens on submit (`apps/api/src/webform/webform.service.ts` → `libs/shared/src/edge/queue-webhook`):
1. An **empty or empty-object `data`** is treated as a provider verification ping → stored nothing, returns `{ ok: true }`.
2. `turnstileToken` is stripped from the payload (server-side Turnstile verification exists but is currently disabled in code).
3. The submission is written to `tenant_webhook_data` (`key_type: "WEBFORM"`), which a DB trigger hands to the `queue-webhook` edge function **asynchronously**.
4. The edge function:
   - **Bound webform** → loads the data source and calls `createData({ record: data })` — the submitted `data` object **is** the record, mapped field-by-field by slug.
   - **Unbound webform** → inserts into the per-tenant `webform_record` table (`{ tenant_webform_id, data, … }`).
   - Then fires any connected, active automation `webform` trigger.

So a record (or `webform_record` row) appears a moment **after** the submit response, not synchronously.

## Response handles (URLs / embed)

`create-webform`/`get-webform` expose the public handles (computed on the entity):

- `form_id` / `form_token` — the webhook short id + token used in public URLs.
- `form_url` — the hosted render URL.
- `form_submit_url` — the POST target (`${API}/webforms/{id}/{token}`).
- `form_sandbox_url` — sandbox render URL (empty if the sandbox env is unset).
- `embed_code` — a ready `<script src="https://static.docyrus.app/webform/w.js" data-webform-id="{id}/{token}">` snippet to drop on a website.

## Automation `webform` trigger

Wire a workflow to run on each submission (see **docyrus-automation-design**):

```bash
docyrus automation create-trigger --appSlug crm --automationId <auto-id> \
  --type webform --webformId <webformId> --json   # maps --webformId → tenant_webform_id
```

On submission, the queue matches active triggers where `tenant_webform_id` equals the form's id and enqueues their action nodes, passing the submitted `data` (optionally through a JSONata `transformer`) as input. This fires **even for unbound webforms**.

## Worked example

A "Leads" intake form bound to a CRM `leads` data source.

### 1. Confirm the field slugs

```bash
docyrus studio list-fields --appSlug crm --dataSourceSlug leads --json
# slugs you'll collect: name (built-in title), email, company, message
```

### 2. Create the webform

```bash
docyrus studio create-webform --appSlug crm --dataSourceSlug leads \
  --name "Website lead form" --status 1 \
  --schema '{"children":[
    {"type":"FieldText","options":{"key":"name","label":"Full name"}},
    {"type":"FieldText","options":{"key":"email","label":"Email"}},
    {"type":"FieldText","options":{"key":"company","label":"Company"}},
    {"type":"FieldTextarea","options":{"key":"message","label":"Message"}},
    {"type":"KvButton","options":{"action":"submit","label":"Send"}}
  ]}' --json
# → WEBFORM_ID; capture form_submit_url + embed_code
```

### 3. Read back the public handles

```bash
docyrus studio get-webform --webformId WEBFORM_ID --json   # form_url, form_submit_url, embed_code
```

## Validation checklist

- [ ] `get-webform` shows `name`, `status` (1), `schema`, and the admin `tenant_data_source_id` (the binding); `form_url`/`form_submit_url`/`embed_code` present.
- [ ] Every schema input `options.key` maps to a real field slug in `list-fields` for the bound data source.
- [ ] The schema has exactly one submit button (`KvButton`, `action: "submit"`).
- [ ] `list-webforms --dataSourceSlug leads` lists the form.

## Gotchas

- **Required:** `name`, `schema` (non-empty object), `status` (1|2). The CLI marks them optional but the backend rejects omissions.
- **Casing:** raw `--data` must use `dataSourceId` (camelCase) and `options` (not `webform_options`). Prefer flags.
- **Binding is create-only** — `update-webform` can't rebind; recreate to move data sources.
- **Field keys must equal data-source slugs** — a mismatched key is accepted but silently won't populate the record.
- **Async record creation** — submit returns `{id}`; the record is created later via the queue/edge function. In local dev that pipeline may not run, so validate schema + slugs there and test the full round-trip in a real environment.
- **Empty `data` = verification ping** — stores nothing, returns `{ok:true}`.
- **Unbound forms** capture into `webform_record` (read via `docyrus curl "/v1/webforms/{webformId}/items"`).
- **Delete cascades** to the paired webhook; deleting the bound data source cascades to delete the webform.
- **No slug** — `get`/`update`/`delete` need the webform UUID.
