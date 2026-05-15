---
name: docyrus-cli-app
description: Use the Docyrus CLI (`docyrus`) to interact with the Docyrus platform from the terminal. Use when the user asks to authenticate, list apps, query or manage data records (`ds`), manage dev app data source schema objects (`studio`) including data sources, fields, enums, data views, forms, webforms, HTML/PDF/DOCX export templates, and email templates, manage automations and their triggers/action nodes (`automation`), list tenant email accounts and send emails (`messaging`), send API requests, switch environments, tenants, or accounts, discover tenant OpenAPI specs, or use the Bun-powered terminal UI via `docyrus tui`. Triggers on tasks involving docyrus CLI commands, terminal-based Docyrus operations, `docyrus ds list`, `docyrus studio`, `docyrus studio data-view`, `docyrus studio form`, `docyrus studio webform`, `docyrus studio html-template`, `docyrus studio email-template`, `docyrus automation`, `docyrus automation create-trigger`, `docyrus automation create-node`, `docyrus messaging accounts`, `docyrus messaging email send`, `docyrus discover`, `docyrus auth`, `docyrus env`, `docyrus tui`, or shell-based Docyrus workflows.
---

# Docyrus CLI

Guide for using the `docyrus` CLI to interact with the Docyrus platform from the terminal.

## Command Overview

| Command | Description |
|---------|-------------|
| `docyrus` | Show active environment, current auth context, and help summary |
| `docyrus env list` / `env use` | Manage named environments |
| `docyrus auth login` | Authenticate via OAuth2 device flow or manual tokens |
| `docyrus auth logout` | Logout the active account for the current environment |
| `docyrus auth who` | Show the active user and tenant |
| `docyrus auth accounts list` / `use` | Manage saved user accounts |
| `docyrus auth tenants list` / `use` | Manage saved tenants for a user |
| `docyrus apps list` | List apps from `/v1/apps` |
| `docyrus apps delete` / `restore` / `permanent-delete` | Archive, restore, or hard-delete apps via `/v1/dev/apps/:appId` |
| `docyrus ds get` | Get data source metadata |
| `docyrus ds list` | Query records with filters, sorting, pagination |
| `docyrus ds create` / `update` / `delete` | Mutate records, including bulk create/update |
| `docyrus studio ...` | CRUD for dev app data sources, fields, enums, data views, forms, webforms, HTML/PDF/DOCX templates, and email templates |
| `docyrus automation ...` | Manage automations, triggers, and action nodes for an app |
| `docyrus messaging accounts` | List tenant email accounts (non-credential info) |
| `docyrus messaging email send` | Send an email through a tenant email account |
| `docyrus discover api` | Download tenant OpenAPI spec |
| `docyrus discover namespaces` / `path` / `endpoint` / `entity` / `search` | Explore the downloaded tenant OpenAPI spec |
| `docyrus connect list-connectors` | List integration connectors with optional keyword search |
| `docyrus connect get-connector <slug>` | Get connector details including data sources and actions |
| `docyrus connect get-action <slug> <actionKey>` | Get action details with input/output JSON schemas |
| `docyrus connect list-connections <slug>` | Get tenant and user connections for a connector |
| `docyrus connect curl <slug> <endpoint>` | Send HTTP request through a connector's provider auth |
| `docyrus connect run-action <appSlug> <actionKey>` | Run a connector or app action |
| `docyrus curl` | Send arbitrary API requests |
| `docyrus tui` | Launch the OpenTUI terminal UI (requires Bun) |

**See [references/cli-manifest.md](references/cli-manifest.md) for complete command reference with flags and arguments.**

## Common Workflows

### Settings Scope

By default, `docyrus` stores settings in a project-local `.docyrus/` folder in the current working directory.

- Local default: `./.docyrus/`
- Global override: `~/.docyrus/` via `-g` or `--global`
- Tenant OpenAPI cache: `<settings-root>/tenans/<tenantId>/openapi.json`

Examples:

```bash
# Local project settings (default)
docyrus auth login --clientId "83a8df32-3738-4b5a-a0c7-87976adb1631"

# Force global settings for this run
docyrus -g auth login --clientId "83a8df32-3738-4b5a-a0c7-87976adb1631"
```

### Environments

The CLI does not use `API_BASE_URL`. It uses saved named environments:

- `live` (`prod` alias) -> `https://api.docyrus.com`
- `beta` -> `https://beta-api.docyrus.com`
- `alpha` -> `https://alpha-api.docyrus.com`
- `dev` -> `https://localhost:3366`

Examples:

```bash
docyrus
docyrus env list --json
docyrus env use beta --json
```

Running `docyrus` without a subcommand returns the active environment, help summary, and current auth `context`.

### Authentication

Device flow login:

```bash
docyrus auth login --clientId "83a8df32-3738-4b5a-a0c7-87976adb1631" --json
```

Manual token login:

```bash
docyrus auth login \
  --accessToken "<access-token>" \
  --refreshToken "<optional-refresh-token>" \
  --clientId "<optional-client-id>" \
  --json
```

Rules:

- `--refreshToken` requires `--accessToken`
- if local login omits `--clientId`, the CLI falls back to the saved global client ID when available
- explicit or previously resolved client IDs are saved to config for reuse
- default scopes are hardcoded in the CLI and include `openid`, `email`, `profile`, `offline_access`, `ReadWrite.All`, `User.ReadWrite`, `Users.Read.All`, `Tenant.Read`, `Teams.Read.All`, `DS.ReadWrite.All`, `Docs.ReadWrite.All`, and `Architect.ReadWrite.All`

Multi-account and multi-tenant workflows:

```bash
docyrus auth accounts list --json
docyrus auth accounts use --userId "<user-id>" --json
docyrus auth tenants list --userId "<user-id>" --json
docyrus auth tenants use 1002 --json
docyrus auth tenants use "8d130f7a-4bc4-4be6-a05b-0f8f1b2d93e9" --userId "<user-id>" --json
docyrus auth who --json
```

`auth tenants use` takes a positional tenant selector. If it is numeric, the CLI treats it as `tenantNo`; otherwise it must be a UUID tenant ID.

### Successful Result Shape

Every successful command injects a top-level `context` field:

```json
{
  "data": {},
  "context": {
    "email": "user@example.com",
    "tenantName": "Acme",
    "tenantNo": 1002,
    "tenantDisplay": "Acme (1002)"
  }
}
```

If there is no active session, `context` is `null`.

### Discover API and Entities

Discover commands require an active session. Commands other than `discover api` auto-download the OpenAPI spec if it is missing locally.

```bash
docyrus discover api --json
docyrus discover namespaces --json
docyrus discover path /v1/users --json
docyrus discover endpoint /v1/users/me --json
docyrus discover endpoint [PUT]/v1/users/me/photo --json
docyrus discover entity UserEntity --json
docyrus discover search users,UserEntity --json
```

### Discover Data Sources

```bash
docyrus apps list --json
docyrus ds get crm contacts --json
```

### Query Records (`ds list`)

Basic listing:

```bash
docyrus ds list crm contacts --columns "name, email, phone" --limit 20
```

With filters:

```bash
docyrus ds list crm contacts \
  --columns "name, email" \
  --filters '{"rules":[{"field":"status","operator":"=","value":"active"}]}'
```

With relation expansion:

```bash
docyrus ds list crm contacts \
  --columns "name, ...related_account(account_name, account_phone)"
```

Date shortcut filter:

```bash
docyrus ds list crm tasks --filters '{"rules":[{"field":"created_on","operator":"this_month"}]}'
```

**See [references/list-query-examples.md](references/list-query-examples.md) for more filter, sort, pagination, and combined query examples.**

### Record Mutations

Create:

```bash
docyrus ds create crm contacts --data '{"name":"Jane Doe","email":"jane@example.com"}'
```

Update:

```bash
docyrus ds update crm contacts <recordId> --data '{"phone":"+1234567890"}'
```

Delete:

```bash
docyrus ds delete crm contacts <recordId>
```

Batch and file input:

```bash
docyrus ds create crm contacts --data '[{"name":"A"},{"name":"B"}]' --json
docyrus ds update crm contacts --data '[{"id":"1","phone":"+111"},{"id":"2","phone":"+222"}]' --json
docyrus ds create crm contacts --from-file ./contacts-create.csv --json
docyrus ds update crm contacts <recordId> --from-file ./contact-update.json --json
```

Array payloads route to bulk endpoints and are limited to 50 items per request.

### Studio Schema CRUD (`studio`)

Use `studio` for developer-facing schema operations: data sources, fields, enums, saved views, forms, webforms, and export templates. Most data-source-scoped commands accept either `--appId`/`--appSlug` and either `--dataSourceId`/`--dataSourceSlug`, and the CLI resolves whichever side you did not pass.

```bash
# Data sources
docyrus studio list-data-sources --appSlug crm --expand fields --json
docyrus studio get-data-source --appSlug crm --dataSourceSlug contacts --json
docyrus studio create-data-source --appSlug crm --title "Contacts" --name "contacts" --slug "contacts" --json
docyrus studio update-data-source --appId <appId> --dataSourceId <dataSourceId> --data '{"title":"Contacts v2"}' --json
docyrus studio delete-data-source --appId <appId> --dataSourceSlug contacts --json
docyrus studio restore-data-source --appId <appId> --dataSourceId <dataSourceId> --json
docyrus studio permanent-delete-data-source --appId <appId> --dataSourceId <dataSourceId> --json
docyrus studio bulk-create-data-sources --appId <appId> --from-file ./data-sources.json --json

# Fields
docyrus studio list-fields --appSlug crm --dataSourceSlug contacts --json
docyrus studio get-field --appSlug crm --dataSourceSlug contacts --fieldSlug email --json
docyrus studio create-field --appId <appId> --dataSourceId <dataSourceId> --name "Email" --slug "email" --type "text" --json
docyrus studio update-field --appId <appId> --dataSourceId <dataSourceId> --fieldId <fieldId> --data '{"name":"Primary Email"}' --json
docyrus studio delete-field --appId <appId> --dataSourceId <dataSourceId> --fieldSlug email --json
docyrus studio create-fields-batch --appId <appId> --dataSourceId <dataSourceId> --data '[{"name":"Status","slug":"status","type":"text"}]' --json
docyrus studio update-fields-batch --appId <appId> --dataSourceId <dataSourceId> --from-file ./fields-update.json --json
docyrus studio delete-fields-batch --appId <appId> --dataSourceId <dataSourceId> --data '["field-1","field-2"]' --json

# Enums
docyrus studio list-enums --appId <appId> --dataSourceId <dataSourceId> --fieldId <fieldId> --json
docyrus studio create-enums --appId <appId> --dataSourceId <dataSourceId> --fieldId <fieldId> --data '[{"name":"Open","sortOrder":1}]' --json
docyrus studio update-enums --appId <appId> --dataSourceId <dataSourceId> --fieldId <fieldId> --from-file ./enums-update.json --json
docyrus studio delete-enums --appId <appId> --dataSourceId <dataSourceId> --fieldId <fieldId> --data '["enum-1","enum-2"]' --json

# Data views (/v1/apps/:appSlug/data-sources/:dataSourceSlug/views)
docyrus studio list-data-views --appSlug crm --dataSourceSlug contacts --json
docyrus studio get-data-view --appSlug crm --dataSourceSlug contacts --viewId <viewId> --json
docyrus studio create-data-view --appSlug crm --dataSourceSlug contacts --name "Active customers" \
  --filters '{"rules":[{"field":"status","operator":"=","value":"active"}]}' --isDefault --json
docyrus studio update-data-view --appSlug crm --dataSourceSlug contacts --viewId <viewId> --data '{"name":"Renamed view"}' --json
docyrus studio delete-data-view --appSlug crm --dataSourceSlug contacts --viewId <viewId> --json

# Forms (/v1/apps/:appSlug/data-sources/:dataSourceSlug/forms)
docyrus studio list-forms --appSlug crm --dataSourceSlug contacts --json
docyrus studio get-form --appSlug crm --dataSourceSlug contacts --formId <formId> --json
docyrus studio create-form --appSlug crm --dataSourceSlug contacts --name "Lead intake" --title "New lead" --json
docyrus studio update-form --appSlug crm --dataSourceSlug contacts --formId <formId> --data '{"title":"Renamed"}' --json
docyrus studio delete-form --appSlug crm --dataSourceSlug contacts --formId <formId> --json

# Webforms (/v1/dev/webforms)
docyrus studio list-webforms --json
docyrus studio list-webforms --appSlug crm --dataSourceSlug contacts --json
docyrus studio get-webform --webformId <webformId> --json
docyrus studio create-webform --name "Contact form" --schema '{"components":[]}' --status 1 --json
docyrus studio create-webform --appSlug crm --dataSourceSlug contacts --from-file ./contact-webform.json --json
docyrus studio update-webform --webformId <webformId> --data '{"name":"Renamed"}' --json
docyrus studio delete-webform --webformId <webformId> --json

# HTML / PDF / DOCX export templates (/v1/dev/html-templates)
docyrus studio list-html-templates --appSlug crm --dataSourceSlug contacts --limit 10 --json
docyrus studio get-html-template --templateId <templateId> --json
docyrus studio create-html-template --appSlug crm --dataSourceSlug contacts \
  --name "Invoice" --sourceType pdf --pageFormat A4 --pageOrientation portrait \
  --body "<h1>{{ company.name }}</h1>" --isDefault --json
docyrus studio update-html-template --templateId <templateId> --data '{"name":"Invoice v2"}' --json
docyrus studio delete-html-template --templateId <templateId> --json

# Email templates (/v1/dev/email-templates)
docyrus studio list-email-templates --json
docyrus studio get-email-template --templateId <templateId> --json
docyrus studio create-email-template --name "Welcome" --subject "Hello {{ user.name }}" --body "<p>Hi</p>" --json
docyrus studio update-email-template --templateId <templateId> --data '{"subject":"Hi {{ user.name }}"}' --json
docyrus studio delete-email-template --templateId <templateId> --json
```

### App Management (`apps`)

`apps list` uses `/v1/apps`, but mutations route through `/v1/dev/apps/:appId`.

```bash
docyrus apps list --json
docyrus apps delete --appId <appId> --json
docyrus apps restore --appId <appId> --json
docyrus apps permanent-delete --appId <appId> --json
```

### Automations (`automation`)

Manage automations, their triggers, and action nodes for an app. All commands route through `/v1/dev/apps/:appId/automations`. App is resolved with `--appId` or `--appSlug`. Automations and nodes have IDs only (no slugs).

```bash
# Automation CRUD
docyrus automation list --appSlug crm --json
docyrus automation get --appSlug crm --automationId <automationId> --json
docyrus automation create --appSlug crm \
  --name "Notify on new deal" --triggerType recordCreated \
  --sourceDataSourceId <dataSourceId> --status 1 --json
docyrus automation update --appSlug crm --automationId <automationId> \
  --data '{"name":"Renamed automation","status":1}' --json
docyrus automation delete --appSlug crm --automationId <automationId> --json

# Triggers (typed create/update, type-independent delete)
docyrus automation list-triggers --appSlug crm --automationId <automationId> --json
docyrus automation get-trigger --appSlug crm --automationId <automationId> --triggerId <triggerId> --json
docyrus automation create-trigger --appSlug crm --automationId <automationId> \
  --type record-modified --sourceDataSourceId <dataSourceId> \
  --modifiedColumns "status,stage" --modifiedColumnsCondition any --json
docyrus automation create-trigger --appSlug crm --automationId <automationId> \
  --type recurrence --recurrenceFrequency day --recurrenceInterval 1 --recurrenceRunAt "09:00" --json
docyrus automation update-trigger --appSlug crm --automationId <automationId> \
  --type webhook --triggerId <triggerId> --data '{"webhook_name":"renamed"}' --json
docyrus automation delete-trigger --appSlug crm --automationId <automationId> --triggerId <triggerId> --json

# Action nodes (typed create/update, type-independent delete)
docyrus automation list-nodes --appSlug crm --automationId <automationId> --json
docyrus automation get-node --appSlug crm --automationId <automationId> --nodeId <nodeId> --json
docyrus automation create-node --appSlug crm --automationId <automationId> \
  --type http-request --requestMethod POST --customEndpoint "https://example.com/webhook" \
  --contentType "application/json" --from-file ./http-node.json --json
docyrus automation create-node --appSlug crm --automationId <automationId> \
  --type external-action --actionTypeId <coreActionId> --from-file ./external-action.json --json
docyrus automation update-node --appSlug crm --automationId <automationId> \
  --type send-email --nodeId <nodeId> --data '{"data":{"to":"user@example.com","subject":"Hi"}}' --json
docyrus automation delete-node --appSlug crm --automationId <automationId> --nodeId <nodeId> --json
```

Trigger `--type` values (kebab-case URL form): `record-created`, `record-modified`, `record-deleted`, `recurrence`, `app-event`, `webhook`, `emailhook`, `webform`, `button-activation`, `manual-activation`.

Node `--type` values (kebab-case URL form): `external-action`, `send-email`, `send-notification`, `create-record`, `update-records`, `request-approval`, `request-input`, `http-request`, `data-source-query`, `custom-query`, `generate-document`, `ai-prompt`, `ai-agent`, `execute-script`.

Note: `automation create --triggerType` uses the camelCase form (e.g. `recordCreated`, `recordModified`) to match `CreateAutomationDto.trigger_type`. Trigger CRUD commands use kebab-case for `--type`.

Convenience flags are camelCase on the CLI but converted to `snake_case` in the request body. Complex nested objects (trigger `data`, node `data`, `field_mapping`, `dynamic_field_mapping`, `condition`, `input_template`, `input_transformer`, `custom_headers`, `pre_action_request`, `post_action_request`, `target_data_source_condition`, etc.) must be supplied via `--data` / `--from-file` — the CLI does not flatten them. `--recurrenceWeekDays` and `--modifiedColumns` accept comma-separated values and are sent as arrays.

`create-node --type external-action` requires `--actionTypeId` and the backend validates the supplied `data` against the linked `core_action.input_json_schema`.

### Messaging (`messaging`)

List tenant email accounts and send transactional emails through them. Routes through `/v1/messaging/email/*`. Requires the `Messaging.Email.Send` OAuth2 scope.

```bash
# List active tenant email accounts (no credentials returned)
docyrus messaging accounts --json

# Send through an account using individual flags
docyrus messaging email send \
  --accountId <accountUuid> \
  --to "ops@example.com,sales@example.com" \
  --subject "Daily summary" \
  --body "<p>Hello</p>" --json

# Send with cc/bcc/replyTo and send-as-user
docyrus messaging email send \
  --accountId <accountUuid> \
  --to "user@example.com" --cc "manager@example.com" --bcc "audit@example.com" \
  --replyTo "support@example.com" --sendAsUser \
  --subject "Update" --body "<p>...</p>" --json

# Full payload via JSON
docyrus messaging email send --accountId <accountUuid> \
  --data '{"to":["a@b.com"],"subject":"Hi","body":"<p>Hi</p>","attachments":[{"filePath":"records/abc/attachments/foo.pdf","fileName":"foo.pdf"}]}' --json
docyrus messaging email send --accountId <accountUuid> --from-file ./send.json --json
```

Limits: up to 50 recipients per `to`/`cc`/`bcc`/`replyTo`, subject max 998 chars, body max 1 000 000 chars, up to 10 attachments per send. Attachment `filePath` references a tenant-scoped storage path. The response payload contains `messageId`, `provider`, `accepted`, and `rejected`.

### Connectors and Actions (`connect`)

Connectors are external integration providers (e.g. Meta WhatsApp, Microsoft Graph, Salesforce). Use the `connect` subcommands to find connectors, inspect their data sources and actions, check connection status, send requests through their auth configuration, and run actions.

**Discovery workflow:**

```bash
# 1. Search for connectors by keyword
docyrus connect list-connectors --q whatsapp --json

# 2. Get connector details (data sources + actions)
docyrus connect get-connector meta-whatsapp --json

# 3. Get full action details with input/output schemas
docyrus connect get-action meta-whatsapp sendWhatsappMessage --json

# 4. Check if tenant/user has active connections
docyrus connect list-connections meta-whatsapp --json
```

**Send requests through connector auth (`curl`):**

The `connect curl` command sends HTTP requests to external providers using the connector's stored auth credentials (OAuth tokens, API keys, base URL).

```bash
# GET request with query params
docyrus connect curl meta-whatsapp \
  "433457363182570/phone_numbers" \
  -d '{"fields":"id,display_phone_number,verified_name"}' --json

# POST request (send WhatsApp message)
docyrus connect curl meta-whatsapp \
  "418088118057836/messages" \
  -X POST \
  -d '{"messaging_product":"whatsapp","to":"905551234567","type":"template","template":{"name":"sample_template","language":{"code":"en_US"}}}' \
  --contentType "application/json" --json

# With explicit auth header override
docyrus connect curl meta-whatsapp \
  "me/businesses" \
  --headers '{"Authorization":"Bearer <token>"}' \
  -d '{"fields":"id,name"}' --json

# With connection ID override
docyrus connect curl meta-whatsapp \
  "some/endpoint" \
  -c <connection-uuid> --json
```

Aliases: `-X` (method), `-d` (data), `-c` (connectionId).

**Run actions:**

The `connect run-action` command runs predefined connector or app actions via `POST /v1/apps/:appSlug/actions/:actionKey/run`.

```bash
# Run an action with parameters
docyrus connect run-action base sendWhatsappMessage \
  --params '{"to":"905551234567","templateName":"hello_world"}' --json

# Dry run — preview request without executing
docyrus connect run-action base sendWhatsappMessage \
  --params '{"to":"905551234567"}' --dryRun --json

# With connection override
docyrus connect run-action base sendWhatsappMessage \
  -p '{"to":"905551234567"}' -c <connection-uuid> --json
```

Aliases: `-p` (params), `-c` (connectionId), `-n` (dryRun).

### Arbitrary API Calls

```bash
docyrus curl /v1/users/me
docyrus curl /v1/apps -X GET --format json
docyrus curl /v1/some/endpoint -X POST -d '{"key":"value"}'
```

### Terminal UI

Launch the OpenTUI interface:

```bash
docyrus tui
```

It requires Bun installed locally. The TUI reuses the existing CLI command graph.

## Key Rules

- Settings are project-local by default in `./.docyrus/`; use `-g` or `--global` for `~/.docyrus/`
- The CLI uses named environments, not `API_BASE_URL`
- `apps list` uses `/v1/apps`
- `ds` commands use `appSlug` and `dataSourceSlug`
- `ds create` and `ds update` accept `--data` JSON or `--from-file` (`.json` or `.csv`), but not both
- Array payloads use bulk endpoints with a maximum of 50 items
- Bulk update requires `id` in every item and must not include positional `<recordId>`
- `--filters` accepts a JSON filter group such as `{"combinator":"and","rules":[...]}`
- Related-field filters use `rel_<relation_slug>/<field_slug>`
- `--columns` supports relation expansion `()`, spread `...`, aliasing `:`, and functions `@`
- `--format` supports `toon`, `json`, `yaml`, `md`, and `jsonl`
- Successful responses inject `context` with `email`, `tenantName`, `tenantNo`, and `tenantDisplay`
- Studio selectors are exclusive pairs: exactly one of `--appId|--appSlug`, `--dataSourceId|--dataSourceSlug`, and `--fieldId|--fieldSlug` as required
- Studio write commands accept `--data` or `--from-file` (JSON only), and explicit flags override overlapping JSON keys
- `studio` data-view and form commands route through `/v1/apps/:appSlug/data-sources/:dataSourceSlug/...`; the CLI bidirectionally resolves between id and slug, so pass either side
- `studio` webform commands route through `/v1/dev/webforms`; CRUD uses `--webformId`, and create/list accept either `--dataSourceId` or `--dataSourceSlug` (slug requires `--appId` or `--appSlug`)
- `studio` html-template and email-template commands route through `/v1/dev/html-templates` and `/v1/dev/email-templates`; CRUD uses `--templateId`, and the optional data-source binding accepts either `--dataSourceId` or `--dataSourceSlug` (slug requires `--appId` or `--appSlug`)
- `connect` subcommands use the `/v1/connectors` API endpoints, not the OpenAPI spec
- `connect curl` sends requests through the connector's provider auth (OAuth tokens, base URL); the `--headers` option can override the Authorization header
- `connect curl` data is sent as body for POST/PUT/PATCH and as query params for GET
- `connect run-action` runs actions via `/v1/apps/:appSlug/actions/:actionKey/run` with `--params` as the JSON body
- `automation` commands route through `/v1/dev/apps/:appId/automations`; trigger and node `create`/`update` use typed URLs (`/triggers/<type>`, `/nodes/<type>`), but `delete-trigger` and `delete-node` use the type-independent route
- `automation` CLI flags are camelCase and are converted to `snake_case` in the request body; nested objects (`data`, `field_mapping`, `condition`, etc.) must be supplied via `--data` / `--from-file`
- `automation create --triggerType` uses camelCase (`recordCreated`); trigger/node CRUD commands use kebab-case `--type` values (`record-created`, `http-request`, ...)
- `messaging` commands route through `/v1/messaging/email/*` and require the `Messaging.Email.Send` OAuth2 scope; the `accounts` endpoint never returns credentials
- `messaging email send` accepts either individual flags (`--to`, `--cc`, `--bcc`, `--replyTo`, `--subject`, `--body`, `--sendAsUser`) or a full JSON payload via `--data` / `--from-file`; recipient flags are comma-separated

## References

- **[CLI Manifest](references/cli-manifest.md)** — Complete command reference with flags, arguments, and command notes.
- **[List Query Examples](references/list-query-examples.md)** — Practical `ds list` examples covering columns, filters, sorting, pagination, and combined queries.
